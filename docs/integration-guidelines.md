# Integration Guidelines

Rules for working with the three external systems this worker integrates: Kafka, Sources API, and AWS.

## Kafka Consumer

1. **Single topic, two event types.** The worker consumes only `platform.sources.superkey-requests`. Clowder may remap the topic name at runtime; always resolve through `conf.KafkaTopic(superkeyRequestedTopic)`, never hard-code the resolved name.

2. **Consumer group is fixed.** The group ID is `sources-superkey-worker` (set in `config/config.go`). Do not make it configurable per-instance; all replicas must share the same group so Kafka partitions are distributed, not duplicated.

3. **Event routing is header-based.** The `event_type` Kafka header determines the handler: `create_application` or `destroy_application`. Any other value is logged and skipped. Do not add body-based routing; keep the switch in `processSuperkeyRequest` as the single dispatch point.

4. **Identity headers are mandatory.** Every message must carry either `x-rh-identity` or `x-rh-sources-org-id` in its Kafka headers. Messages missing both are rejected before parsing. When adding new message handling, enforce this same gate before any processing.

5. **Feature flags disable processing, not consumption.** `DISABLE_RESOURCE_CREATION` and `DISABLE_RESOURCE_DELETION` env vars skip the handler logic but still consume and acknowledge the message. Never stop consuming; unconsumed messages block the partition for all replicas.

## Sources API Authentication

6. **PSK mode vs. identity-header mode.** The `headers()` method in `sources/api_client.go` implements a two-branch auth strategy:
   - When `SOURCES_PSK` is set (production): sends `x-rh-sources-psk` plus optional `x-rh-sources-account-number`, `x-rh-identity`, and `x-rh-org-id`.
   - When `SOURCES_PSK` is empty (local dev): sends only `x-rh-identity`, synthesized from account number and org ID via `encodeIdentity()`.
   
   Never mix these modes. All Sources API calls in a single request share one `SourcesClient` instance whose headers are set once from the Kafka message context.

7. **Identity synthesis.** When no identity header arrives on the Kafka message and PSK is not set, `encodeIdentity()` in `sources/helpers.go` builds a base64-encoded `XRHID` struct from `AccountNumber` and `OrgID`. This is the fallback path. Do not add additional identity construction paths; keep this as the single factory.

## Sources API Endpoints

8. **Two API versions in use.** Public endpoints use `/api/sources/v3.1/` (applications, authentications, sources, application_authentications, check_availability). The internal endpoint uses `/internal/v2.0/authentications/{id}?expose_encrypted_attribute[]=password` to retrieve secrets. Never call the internal endpoint for anything other than reading decrypted credentials.

9. **Internal auth fetch retries.** `GetInternalAuthentication` retries up to 5 times with a 3-second sleep between attempts. This exists because the authentication record may not be immediately available after creation. Do not reduce the retry count without understanding the Sources API's eventual consistency window.

10. **SourcesClient is per-request, not global.** Each Kafka message creates its own `SourcesClient` with tenant-specific identity fields. The `SourcesClient` struct is instantiated in `provider/forge.go` (`getProvider`) and in `superkey/forged_application.go` (`CreateInSourcesAPI`). Never use a shared or cached client across requests.

## AWS Credential Lifecycle

11. **Credentials are ephemeral and per-request.** AWS access key and secret key are fetched at runtime from Sources internal API for each superkey request (the `SuperKey` field is the authentication ID). These are the customer's AWS credentials, not the worker's own. The worker has no stored AWS credentials for customer accounts.

12. **AWS clients are initialized on-demand.** `amazon.NewClient` creates only the SDK service clients (IAM, S3, CostReporting) needed for the specific steps in the request. The `getRequiredApis` function maps step names to API clients. When adding a new AWS service, add a new case in both `getRequiredApis` and `NewClient`.

13. **Region is hard-coded to `us-east-1`.** Set in `amazon/credentials.go`. IAM is global so this works for IAM operations. S3 bucket creation without a `LocationConstraint` defaults to `us-east-1`. CUR is also `us-east-1` only. Do not parameterize the region without verifying all three services support it.

## Superkey Request Lifecycle

14. **Create flow: Forge, then post to Sources, then check availability.**
    - `getProvider` fetches customer AWS creds from Sources internal API
    - `ForgeApplication` executes ordered steps (s3 -> cost_report -> policy -> role -> bind_role) creating AWS resources
    - `CreateInSourcesAPI` sleeps (AWS_WAIT_TIME, default 7s in code; 15s as configured in ClowdApp), patches the application with `_superkey` metadata, creates an Authentication record with the role ARN as username, then triggers `check_availability`
    - On any forge failure: `TearDown` reverses completed steps, then `MarkSourceUnavailable` patches both application and source as unavailable

15. **Destroy flow: Reconstruct, then tear down.** `ReconstructForgedApplication` rebuilds a `ForgedApplication` from the `DestroyRequest`'s `StepsCompleted` map. The `Client` field is nil and gets lazily initialized in `TearDown` by re-fetching credentials from Sources. The teardown order is: unbind_role, then policy/role/cost_report in parallel, then s3 last.

16. **Step ordering matters.** The `SuperKeySteps` array defines execution order. Steps reference outputs of prior steps via `Substitutions` (e.g., `"s3"` substitution reads the bucket name from `StepsCompleted["s3"]["output"]`). Never reorder steps without verifying substitution dependencies.

17. **GUID uniqueness.** Each forged application gets an 8-byte random hex GUID (`generateGUID`). This GUID is appended to all AWS resource names (buckets, policies, roles, cost reports) to prevent collisions across tenants. The GUID is stored in the `_superkey.guid` field of the application's `extra` payload for teardown reconstruction.

18. **AWS_WAIT_TIME prevents IAM race conditions.** After creating AWS resources and before posting back to Sources, the worker sleeps for `AWS_WAIT_TIME` seconds (default 7 in code; configured as 15 in ClowdApp). This prevents Sources from attempting to use the IAM role before AWS has fully propagated it. Do not remove this sleep.

## Error Handling

19. **Partial creation requires teardown.** If any forge step fails, the `ForgedApplication` is returned with `StepsCompleted` reflecting partial progress. The caller in `createResources` calls `TearDown` on the partial result, then `MarkSourceUnavailable`. Always return `ForgedApplication` from `ForgeApplication` even on error so teardown can clean up.

20. **Teardown errors are collected, not fatal.** `TearDown` returns `[]error`, not a single error. Each step teardown is attempted independently. Log all errors but do not stop tearing down remaining resources on the first failure.

## Verification

```bash
# Build and lint
make build
make lint

# Verify Kafka topic constant matches ClowdApp
grep 'superkeyRequestedTopic' main.go
grep 'topicName' deploy/clowdapp.yaml

# Verify Sources API version paths are consistent
grep -rn '/api/sources/v' sources/
grep -rn '/internal/v' sources/

# Verify auth header logic branches
grep -A 20 'func (sc \*SourcesClient) headers' sources/api_client.go

# Verify teardown order (unbind before delete)
grep -A 30 'func.*TearDown' provider/amazon_provider.go

# Verify AWS_WAIT_TIME is used before Sources API calls
grep -B 5 -A 5 'waitTime' superkey/forged_application.go
```
