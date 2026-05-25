# AGENTS.md

This file is the onboarding document for AI coding agents working on `sources-superkey-worker`.

## Project summary

A Go Kafka consumer that processes superkey lifecycle events (`create_application`, `destroy_application`) to provision and deprovision AWS resources (IAM roles, policies, S3 buckets, Cost and Usage Reports) on behalf of Red Hat customers, then posts results back to the Sources API. Currently only the `amazon` provider is implemented.

## Quick reference

| Item | Value |
|---|---|
| Language | Go 1.24 (toolchain go1.24.3) |
| Module path | `github.com/redhatinsights/sources-superkey-worker` |
| Build | `make build` or `make container` |
| Lint | `make lint` (runs `go vet` + `golangci-lint run -E gofmt,gci,bodyclose,forcetypeassert,misspell`) |
| Tests | **None exist.** There are zero `*_test.go` files in the repository. |
| CI | GitHub Actions (`gofmt` via Jerome1337/gofmt-action, `go vet`, `golangci-lint --enable gci,bodyclose,forcetypeassert,misspell`); Konflux/Tekton for container images. CI uses `go-version: "1.23"`. |
| Entry point | `main.go` |
| Deployment | Clowder ClowdApp on OpenShift (`deploy/clowdapp.yaml`) |
| License | Apache 2.0 (`licenses/LICENSE`) |

## Cross-cutting conventions

### Package dependency hierarchy (hard rule)

```
main.go
  ├─ provider/    (orchestration — bridges amazon/, superkey/, sources/)
  │    ├─ amazon/     (AWS SDK wrappers only — never imports superkey/, provider/, sources/)
  │    ├─ superkey/   (domain types — may import sources/, never imports provider/ or amazon/)
  │    └─ sources/    (Sources API HTTP client — never imports superkey/, provider/, or amazon/)
  ├─ config/      (standalone leaf — no internal imports)
  └─ logger/      (standalone leaf — no internal imports)
```

`main.go` must never import `amazon/` or `sources/` directly. Only `provider/` bridges them.

### Struct definitions go in `types.go`

Every package places struct definitions and interfaces in `types.go`. Methods on those structs go in separate, purpose-named files. Never put business logic in `types.go`.

### Logger alias and context-based logging

Import the logger as `l` (`l "github.com/redhatinsights/sources-superkey-worker/logger"`). Always use `l.LogWithContext(ctx)` when a `context.Context` is available. Use bare `l.Log` only in startup/shutdown code where no request context exists. Never use `logrus` directly outside the `logger/` package.

**Known exception:** `sources/helpers.go` imports the logger without an alias (`"github.com/redhatinsights/sources-superkey-worker/logger"`) and uses `logger.Log` directly. Do not "fix" this without checking all callers.

### Error wrapping

Use `fmt.Errorf("...: %w", err)` with resource-identifying context. The `amazon/` package returns raw SDK errors without wrapping; wrapping happens in `provider/`. Never use `log.Fatal` during message processing — reserve it for startup failures only.

### Naming conventions

- AWS resources: `redhat-<apptype>-<resource>-<guid>` (the naming scheme is load-bearing; the Sources API stores these names for teardown).
- Provider implementations: `provider/<name>_provider.go`.
- Per-AWS-service files in `amazon/`: named after the service (`iam.go`, `s3.go`, `costandusagereports.go`).

### Known code quirks

- `substiteInPayload` (typo) in `provider/amazon_provider.go` — preserve this spelling; do not rename without a repo-wide search.
- Import path casing matters: `RedHatInsights` (capital R/H/I) for `sources-api-go`; `redhatinsights` (lowercase) for `app-common-go` and `platform-go-middlewares`.
- The project uses both AWS SDK v1 (only in `logger/logger.go` for CloudWatch) and AWS SDK v2 (everywhere else). All new AWS code must use v2.
- `GOTOOLCHAIN` ARG in `Dockerfile` must match the `toolchain` directive in `go.mod`. CI uses `go-version: "1.23"` in `.github/workflows/actions.yml`, which is intentionally pinned below the module's `go 1.24` directive — do not change CI without understanding that the toolchain download behavior differs from the local build.

### What NOT to do

- Do not add test files without first establishing a test framework — there are currently zero tests.
- Do not introduce concurrency in the Kafka consumer without a semaphore/limiter and careful offset-commit strategy.
- Do not add new `os.Getenv()` reads scattered across the codebase — use `config/config.go` unless following the existing exception pattern for feature flags.
- Do not log AWS credentials, passwords, or the `amazon.Client` struct.
- Do not add dependencies on transitive packages pulled in by `sources-api-go` (gorm, Echo, etc.).

## Docs index

| Guideline file | What it covers |
|---|---|
| [`docs/api-contracts-guidelines.md`](docs/api-contracts-guidelines.md) | HTTP calls to Sources API: versioned endpoints, authentication headers, status codes, retry policy, request sequencing |
| [`docs/async-and-messaging-guidelines.md`](docs/async-and-messaging-guidelines.md) | Kafka consumer setup, topic/group config, message dispatch by `event_type` header, consumer lifecycle and shutdown |
| [`docs/aws-resource-provisioning-guidelines.md`](docs/aws-resource-provisioning-guidelines.md) | AWS resource creation/teardown order, `StepsCompleted` tracking, substitution system, S3 bucket deletion, IAM race-condition sleep |
| [`docs/clowder-and-openshift-platform-guidelines.md`](docs/clowder-and-openshift-platform-guidelines.md) | Clowder config branching pattern, ClowdApp template conventions, health probes, resource limits, non-root container requirement |
| [`docs/code-organization-guidelines.md`](docs/code-organization-guidelines.md) | Package hierarchy, `types.go` convention, `Provider` interface, file naming, `main.go` role, adding new providers |
| [`docs/configuration-guidelines.md`](docs/configuration-guidelines.md) | All environment variables, Clowder vs local config paths, feature flags (`DISABLE_RESOURCE_CREATION/DELETION`), `AWS_WAIT_TIME`, adding new env vars |
| [`docs/data-validation-guidelines.md`](docs/data-validation-guidelines.md) | Kafka header/body validation, identity header decoding, credential checks, `StepsCompleted` nil-guard requirements, unvalidated fields |
| [`docs/dependency-management-guidelines.md`](docs/dependency-management-guidelines.md) | AWS SDK v1/v2 dual constraint, Red Hat platform deps (`sources-api-go` pseudo-version pinning), Dependabot config, `GOTOOLCHAIN` sync |
| [`docs/deployment-guidelines.md`](docs/deployment-guidelines.md) | Dockerfile two-stage build, UBI9 base image, Konflux/Tekton pipelines, GitHub Actions CI, `build_deploy.sh`, local dev setup |
| [`docs/error-handling-guidelines.md`](docs/error-handling-guidelines.md) | Log-and-skip for Kafka errors, `TearDown` collects all errors without short-circuiting, `ForgeApplication` returns partial state, `MarkSourceUnavailable` escalation |
| [`docs/integration-guidelines.md`](docs/integration-guidelines.md) | End-to-end Kafka/Sources API/AWS integration: credential lifecycle, create/destroy flows, step ordering, GUID uniqueness |
| [`docs/logging-and-observability-guidelines.md`](docs/logging-and-observability-guidelines.md) | `LogWithContext` usage, log levels, JSON log schema, CloudWatch/ZincSearch hooks, Prometheus metrics (default only), no distributed tracing |
| [`docs/performance-guidelines.md`](docs/performance-guidelines.md) | Synchronous single-goroutine processing, `AWS_WAIT_TIME` blocking, no HTTP timeouts, unpaginated S3 deletion, tight resource limits |
| [`docs/red-hat-identity-and-tenancy-guidelines.md`](docs/red-hat-identity-and-tenancy-guidelines.md) | `x-rh-identity`/`x-rh-sources-org-id` header propagation, PSK vs XRHID auth, `SourcesClient` identity fields, `TenantID` vs `OrgId` distinction |
| [`docs/security-guidelines.md`](docs/security-guidelines.md) | Customer AWS credential lifecycle (ephemeral, per-request), PSK from K8s secret, internal auth endpoint, non-root container, `crypto/rand` for GUIDs |
