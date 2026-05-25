# AWS Resource Provisioning Guidelines

Rules for modifying the AWS resource lifecycle in `provider/amazon_provider.go`, the `amazon/` package, and the `superkey/` state-tracking types.

## 1. Teardown Order Must Mirror the Dependency Graph, Not the Creation Order

Teardown in `TearDown()` follows a **fixed dependency-driven order**, not a simple reversal of creation steps:

1. **Unbind role-policy** (`bind_role`) -- must happen first so IAM role and policy can be deleted independently.
2. **Delete policy, role, cost_report** -- these are independent after unbinding and can fail independently.
3. **Delete S3 bucket** -- always last because other resources (e.g., cost reports) may reference it.

When adding a new resource step, place its teardown at the correct dependency tier. Do not assume "reverse creation order" is correct; analyze what depends on what.

## 2. StepsCompleted Is the Single Source of Truth for Partial Rollback

`ForgedApplication.StepsCompleted` (`map[string]map[string]string`) records every successfully created resource. The teardown path checks each key with a nil guard (`if f.StepsCompleted["key"] != nil`).

**Rules:**
- Call `f.MarkCompleted(stepName, data)` immediately after the AWS API call succeeds, before any further work in that step.
- The `"output"` key in the data map must contain the identifier needed to destroy the resource (bucket name, policy ARN, role name, report name).
- For IAM roles, also store `"arn"` because the role ARN is needed for the authentication payload while the role *name* is used for destruction.
- `bind_role` stores an empty map (`map[string]string{}`); its presence alone signals that unbinding is needed.
- Never modify StepsCompleted entries after they are set. They are persisted to the Sources API as `_superkey.steps` and are replayed during `destroy_application`.

## 3. Resource Naming Convention Is Load-Bearing

All AWS resource names follow `redhat-<apptype>-<resource>-<guid>`:
- `getShortName()` produces `redhat-<base of ApplicationType URL>`.
- The GUID is a 16-hex-char random string from `generateGUID()`.
- Concrete patterns: `redhat-<apptype>-bucket-<guid>`, `redhat-<apptype>-policy-<guid>`, `redhat-<apptype>-role-<guid>`.
- Cost report names append `-<guid>` to whatever `ReportName` is in the step payload.

Do not change the naming scheme without coordinating with the Sources API, which stores these names for later teardown via `DestroyRequest.StepsCompleted`.

## 4. ForgeApplication Returns the Partial ForgedApplication on Error

`ForgeApplication` returns `(f, err)` where `f` is non-nil even when `err` is non-nil. The caller (`createResources` in `main.go`) uses the partial `f` to:
1. Tear down any resources that were already created.
2. Store the partial `StepsCompleted` in the Sources API so a future `destroy_application` message can clean up.

Any new step must follow the same pattern: return `f` alongside the error so the caller can roll back. Never return `(nil, err)` after any step has been marked completed.

## 5. Step Ordering Is Determined by the Kafka Message, Not the Code

The creation order is driven by `request.SuperKeySteps`, an ordered slice from the Kafka message. The switch-case in `ForgeApplication` handles each step name but does not enforce ordering.

**Implicit ordering dependencies the message must respect:**
- `role` and `policy` must precede `bind_role` (which reads their outputs from StepsCompleted).
- `s3` must precede `cost_report` when the cost report references the bucket.
- `s3` must precede its own bucket-policy attachment (handled inline when the step payload is `"\"create_cost_policy\""`).

If you add a step that reads from StepsCompleted, document which prior steps it depends on.

## 6. Substitution System Has Three Hard-Coded Tokens

`substiteInPayload()` (note: the typo is in the codebase) recognizes exactly three substitution types:
- `"get_account"` -- replaces the token with `f.Request.Extra["account"]`.
- `"s3"` -- replaces the token with the S3 bucket name from `f.StepsCompleted["s3"]["output"]`.
- `"generate_external_id"` -- replaces the token with `f.Request.Extra["external_id"]` (no-op if absent).

Adding a new substitution requires adding a new case here. The substitution map keys are the literal placeholder strings in the payload; the values are the substitution-type identifiers above.

## 7. S3 Bucket Deletion Drains Objects First

`DestroyS3Bucket` in `amazon/s3.go` lists and deletes all objects before deleting the bucket. This uses `ListObjects` (not `ListObjectsV2`) and does not handle pagination -- it processes only the first 1000 objects. If adding support for buckets that may contain more objects, switch to paginated listing.

## 8. IAM Race Condition Sleep

After all AWS resources are created but before posting them to the Sources API, there is an intentional sleep (default 7 seconds, configurable via `AWS_WAIT_TIME` env var) in `CreateInSourcesAPI()`. This exists because IAM propagation is eventually consistent. Do not remove this sleep or move the Sources API call earlier in the flow.

## 9. Feature Flags Are Kill Switches, Not Partial Controls

- `DISABLE_RESOURCE_CREATION=true` -- skips all `create_application` messages entirely (logs and returns).
- `DISABLE_RESOURCE_DELETION=true` -- skips all `destroy_application` messages entirely.

These are read once at startup from the environment. They are binary: there is no per-step or per-resource-type granularity. They are checked in `main.go` before any provider code runs.

## 10. The Client Initializes Only the AWS API Clients Needed

`amazon.NewClient()` accepts a variadic `apis` list derived from the step names. `getRequiredApis()` maps step names to SDK clients (`s3` -> S3, `role`/`policy`/`bind_role` -> IAM, `cost_report` -> Cost and Usage Report Service). Only the needed clients are instantiated.

When adding a new step that uses a new AWS service, add a mapping in `getRequiredApis()` and a new field + initialization case in `NewClient()`.

## 11. Region Is Hard-Coded

`amazon/credentials.go` hard-codes `us-east-1`. All AWS resources are created in this region. Cost and Usage Reports require `us-east-1` regardless. If multi-region support is ever added, the cost reporting service must remain in `us-east-1`.

## 12. Teardown Collects Errors Instead of Failing Fast

`TearDown()` returns `[]error`, not a single error. It attempts every teardown step regardless of prior failures (each guarded by its own StepsCompleted nil check). Callers log all errors individually. Preserve this behavior: never short-circuit teardown on the first error.

## Verification

```bash
# Confirm the five known step names are handled in ForgeApplication
grep -c 'case "s3"\|case "cost_report"\|case "policy"\|case "role"\|case "bind_role"' provider/amazon_provider.go
# Expected: 6 (5 in ForgeApplication switch + 1 in substiteInPayload)

# Confirm teardown checks all step keys with nil guards
grep -c 'StepsCompleted\[.*\] != nil' provider/amazon_provider.go
# Expected: 5

# Confirm MarkCompleted is called for each creation step
grep -c 'MarkCompleted' provider/amazon_provider.go
# Expected: 5

# Confirm teardown returns []error (not single error)
grep 'func.*TearDown.*\[\]error' provider/amazon_provider.go

# Confirm the naming helper produces the expected prefix
grep 'redhat-' provider/amazon_provider.go

# Confirm no test files exist (known gap)
find . -name "*_test.go" -type f | wc -l
# Expected: 0
```
