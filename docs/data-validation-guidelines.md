# Data Validation Guidelines

Rules for validating data flowing through the superkey worker: Kafka messages, identity headers, AWS credentials, and API payloads.

## 1. Kafka Message Header Validation

The entry point is `processSuperkeyRequest` in `main.go`. Three headers are extracted before any body parsing:

- `event_type` determines the processing branch (`create_application` or `destroy_application`). An unrecognized value logs an error and drops the message silently. When adding a new event type, add a `case` in the `switch` block; do not use a fallthrough or default handler that processes unknown events.
- `x-rh-identity` and `x-rh-sources-org-id` are checked with an OR gate: the message is skipped only when **both** are empty. Either header alone is sufficient. Do not change this to require both; the Sources API client in `sources/api_client.go` uses whichever is available to build request headers.
- Headers are extracted via `msg.GetHeader()` which returns an empty string for missing headers, not an error. Never treat the return value as proof the header existed in the Kafka record.

## 2. Kafka Message Body Deserialization (`msg.ParseTo`)

- `msg.ParseTo(&req)` deserializes the Kafka message value (JSON) into the target struct. It is the **only** deserialization boundary for incoming requests.
- On parse failure, the message is logged with the raw `msg.Value` and dropped. No retry, no dead-letter queue. If you need retry semantics, it must be added explicitly.
- After `ParseTo` succeeds, `IdentityHeader` and `OrgIdHeader` on `CreateRequest` are assigned from the Kafka headers -- they are **not** part of the JSON body. Any new header-derived field must be assigned the same way (after parse, before use).
- `DestroyRequest` does **not** carry identity headers at all. Its `TenantID` comes from the JSON body.

## 3. Identity Header Decoding (`sources/types.go`)

- `parseXRhIdentity` performs Base64 (standard encoding) decode then JSON unmarshal into `XRhIdentity`.
- The struct only extracts `identity.account_number`. All other fields in the x-rh-identity blob are silently discarded. If you need `org_id` or `type` from the identity header, add fields to `XRhIdentity` with matching JSON tags under the `identity` nesting.
- `getAccountNumber` returns an empty string (not an error) when the `account_number` field is absent from a valid JSON payload. Callers must handle empty account numbers.
- The `encodeIdentity` helper in `sources/helpers.go` uses `identity.XRHID` from the platform-go-middlewares library, which has a different struct layout than the local `XRhIdentity`. These two structs are not interchangeable.

## 4. Authentication Credential Validation

The only post-parse field validation in the codebase is the Username/Password guard:

- **`provider/forge.go`**: After `GetInternalAuthentication` returns, `auth.Username` and `auth.Password` are checked for empty strings inside `getProvider`. If either is empty, the entire forge is aborted with an error. This is the sole gatekeeper before AWS credentials are used.
- **`sources/api_client.go`**: A weaker check exists inside `GetInternalAuthentication` itself, but it only triggers when `json.Unmarshal` also returns an error (the `&&` condition). A response that unmarshals cleanly but has empty Username/Password will pass this check and rely on the `forge.go` guard.
- When adding a new provider, replicate the empty-credential check in `getProvider` before constructing the provider client. The `amazon.NewClient` constructor does **not** validate that key/secret are non-empty.

## 5. CostReport JSON Deserialization

- `CostReport` in `amazon/types.go` is deserialized from the `step.Payload` string (after substitution) via `json.Unmarshal` in `provider/amazon_provider.go`.
- JSON tags use `snake_case` (e.g., `report_name`, `s3_bucket`, `s3_region`, `time_unit`). The payload in `SuperKeySteps[].Payload` must match these exact tag names.
- After unmarshal, `ReportName` is mutated by appending `-{GUID}`. No other fields are validated. If `S3Bucket` or `S3Region` are empty, the AWS SDK call will fail at runtime, not at the validation layer.
- The type fields (`Compression`, `Format`, `TimeUnit`, `S3Region`) are AWS SDK enum types (`costtypes.*`). Invalid enum string values will be accepted by `json.Unmarshal` without error but will cause AWS API failures.

## 6. CreateRequest and DestroyRequest Field Coverage

`CreateRequest` JSON fields: `identity_header`, `org_id_header`, `tenant_id`, `source_id`, `application_id`, `application_type`, `super_key`, `provider`, `extra`, `superkey_steps`.

`DestroyRequest` JSON fields: `tenant_id`, `super_key`, `guid`, `provider`, `steps_completed`, `superkey_steps`.

**No field in either struct is validated after deserialization.** Specifically:
- `Provider` is validated only when it hits the `switch` in `getProvider` (only `"amazon"` is supported). An empty or unknown provider returns an error there.
- `SuperKey` (the authentication ID) is used directly in an API call to Sources without length or format checks.
- `Extra["account"]` and `Extra["external_id"]` are used in payload substitution without nil or empty checks. A missing key returns an empty string from the map, which silently produces malformed AWS policy documents.
- `StepsCompleted` map values are accessed by key without nil checks on the outer map entry in `ForgeApplication`. Specifically, the `bind_role` step reads `f.StepsCompleted["role"]["output"]` and `f.StepsCompleted["policy"]["output"]`, and the role ARN is read at `f.StepsCompleted["role"]["arn"]`, all without nil guards. A create flow where a prior step did not complete will panic. In `TearDown`, all outer map accesses do have nil guards.

## 7. Substitution Payload Validation

`substiteInPayload` in `provider/amazon_provider.go` performs string replacement on raw JSON/policy payloads:
- `get_account` substitution reads `f.Request.Extra["account"]` -- if absent, the placeholder is replaced with an empty string.
- `s3` substitution reads `f.StepsCompleted["s3"]["output"]` -- if the s3 step has not completed, this is a nil map access (panic).
- `generate_external_id` uses a safe `ok` check and skips replacement if absent.
- The function name has a typo (`substiteInPayload`). Maintain this spelling when referencing it; do not rename without a repo-wide search.

## 8. Sources API Response Validation

- HTTP response status codes are checked with `> 299` (not `!= 200`). Any 2xx is accepted as success.
- `GetInternalAuthentication` retries up to 5 times on non-200 responses with a 3-second sleep. After exhausting retries, the error message wraps the last `err` with `%w`; if all retries received non-200 HTTP responses with no transport error, `err` is nil and the message will contain `<nil>`. This is cosmetically odd but not a nil-pointer dereference.
- Response body unmarshalling errors in `CreateAuthentication` and `GetInternalAuthentication` have different handling: the former returns the error; the latter only returns an error if Username or Password are also empty.

## 9. Rules for Adding New Validated Fields

1. Add the JSON struct tag with the exact `snake_case` name matching the Kafka message contract.
2. If the field is required for processing, add an explicit empty-string check **in the calling code** (like `forge.go`), not inside the deserialization layer.
3. For fields sourced from Kafka headers (not body), assign them after `ParseTo`, following the pattern in `processSuperkeyRequest`.
4. For AWS enum types, validate the value against known constants before passing to AWS SDK calls.
5. Never assume map entries exist in `Extra` or `StepsCompleted` -- always use the `value, ok := map[key]` idiom.

## Verification

```bash
# Find all JSON struct tags to audit field naming consistency
grep -rn 'json:"' --include='*.go' .

# Find all ParseTo / Unmarshal call sites (deserialization boundaries)
grep -rn 'ParseTo\|json\.Unmarshal' --include='*.go' .

# Find all post-parse validation checks (empty string guards)
grep -rn '== ""' --include='*.go' .

# Find unguarded map access patterns (potential panics)
grep -rn 'StepsCompleted\[' --include='*.go' . | grep -v '!= nil'

# Find all base64 decode call sites
grep -rn 'base64\.' --include='*.go' .

# Confirm no test files exist (validation is untested)
find . -name '*_test.go' -type f

# Check for direct Extra map access without ok-guard
grep -rn 'Extra\[' --include='*.go' . | grep -v ', ok'
```
