# Code Organization Guidelines

## Package Dependency Hierarchy

Packages follow a strict layered dependency order. Never introduce imports that violate this direction:

```
main.go
  └─ provider/    (orchestration)
       ├─ amazon/     (cloud SDK wrappers)
       ├─ superkey/   (domain types + request logic)
       │    └─ sources/   (Sources API HTTP client)
       └─ sources/
  └─ superkey/
  └─ config/      (standalone, no internal imports)
  └─ logger/      (standalone, no internal imports)
```

**Hard rules:**
- `config/` and `logger/` are leaf packages -- they must never import other internal packages (except `config` imports into `logger` for initialization).
- `amazon/` must never import `superkey/`, `provider/`, or `sources/`. It only wraps AWS SDK calls.
- `sources/` must never import `superkey/`, `provider/`, or `amazon/`. It only wraps Sources API HTTP calls.
- `superkey/` may import `sources/` and `logger/` but must never import `provider/` or `amazon/`.
- `provider/` is the only package that bridges `amazon/`, `superkey/`, and `sources/` together.
- `main.go` imports `provider/`, `superkey/`, `config/`, and `logger/` -- never `amazon/` or `sources/` directly.

## The types.go Convention

Every package that defines structs must place struct definitions and interface declarations in a file named `types.go`. Methods on those structs go in separate, purpose-named files.

| Package    | `types.go` contains                                              | Methods live in                                        |
|------------|------------------------------------------------------------------|--------------------------------------------------------|
| `superkey` | `CreateRequest`, `DestroyRequest`, `Step`, `App`, `ForgedApplication`, `Provider` interface | `create_request.go`, `forged_application.go`          |
| `amazon`   | `Client` struct, `CostReport` struct, `NewClient` constructor, `CostS3Policy` variable | `iam.go`, `s3.go`, `costandusagereports.go`, `credentials.go` |
| `logger`   | `CustomLoggerFormatter`, `Marshaler` interface                   | `logger.go`, `logger_context.go`                      |
| `sources`  | `XRhIdentity` struct, identity-parsing helpers                   | `api_client.go`, `helpers.go`                         |

**Exception:** `sources/api_client.go` declares `SourcesClient` inline rather than in `types.go` because it is tightly coupled to its methods. When adding a new client struct to `sources/`, prefer moving it to `types.go`.

**Rules:**
- When adding a new struct to an existing package, put the struct definition in that package's `types.go`.
- The `types.go` file may also contain constructors (`NewClient`), variables (`CostS3Policy`), and small helper functions that are tightly coupled to the type definitions.
- Never put business logic (forge steps, API calls, teardown sequences) in `types.go`.

## The Provider Interface

`superkey/types.go` defines the `Provider` interface:

```go
type Provider interface {
    ForgeApplication(ctx context.Context, createRequest *CreateRequest) (*ForgedApplication, error)
    TearDown(ctx context.Context, forgedApplication *ForgedApplication) []error
}
```

When adding a new cloud provider:
1. Create a new package under the repository root (e.g., `azure/`) containing `types.go` for client structs and per-service files for SDK methods.
2. Create `provider/<name>_provider.go` with a struct that implements `superkey.Provider`.
3. Add a `case` to the `switch request.Provider` block in `provider/forge.go` `getProvider()`.
4. Do not modify `superkey/types.go` unless the interface itself needs a new method.

## File Naming Within Packages

- **Per-AWS-service files in `amazon/`:** Each AWS service gets its own file named after the service: `iam.go`, `s3.go`, `costandusagereports.go`. The file contains all `Client` methods that call that service.
- **`credentials.go`:** Holds AWS credential/config construction (`NewAmazonConfig`), separate from service operations.
- **Provider implementations:** Named `<provider>_provider.go` (e.g., `amazon_provider.go`). Contains both `ForgeApplication` and `TearDown` for that provider, plus provider-specific helpers like `substiteInPayload` and `generateGUID`.
- **`forge.go`:** The provider dispatcher. Contains `Forge()`, `TearDown()`, and `getProvider()` -- the switch that routes to the correct provider implementation.
- **`create_request.go` vs `forged_application.go`:** Methods are split by which struct they are defined on: methods on `*CreateRequest` go in `create_request.go`, methods on `*ForgedApplication` go in `forged_application.go`.
- **`api_client.go`:** All `SourcesClient` methods (HTTP calls to Sources API).
- **`helpers.go`:** Standalone utility functions that support the package but are not methods on the main struct.

## main.go Role and Structure

`main.go` is the application entry point and Kafka message router. It handles:
- Kafka consumer setup and signal handling
- Message dispatching by `event_type` header (`create_application` / `destroy_application`)
- Context creation with logger fields for each request
- Orchestration calls to `provider.Forge()` and `provider.TearDown()`
- Metrics and health file initialization

**Rules:**
- Keep `main.go` as a thin orchestrator. Business logic belongs in `provider/` and `superkey/`.
- New Kafka event types get a new `case` in the `switch eventType` block inside `processSuperkeyRequest`.
- All per-request log context fields (`tenant_id`, `source_id`, `application_id`, `application_type`) must be set via `logger.With*` functions before calling into provider code.

## Logger Import Alias

The `logger` package is imported with the alias `l` throughout most of the codebase:

```go
l "github.com/redhatinsights/sources-superkey-worker/logger"
```

Use `l.Log` for the global logger and `l.LogWithContext(ctx)` for context-enriched logging. Never use `logrus` directly except inside the `logger/` package itself.

**Exception:** `sources/helpers.go` imports `logger` without an alias and uses `logger.Log.WithFields(...)` directly. This is a known inconsistency — if refactoring that file, switch to the aliased import.

## util/ Package

`util/` contains `produce_messages.go` which is a standalone CLI tool (its own `package main`) for manually producing Kafka test messages. It is **not** a library package -- do not import from `util/` in application code.

## Verification

```bash
# Confirm no circular imports compile successfully
go build ./...

# List all internal import relationships to check dependency direction
grep -rn '"github.com/redhatinsights/sources-superkey-worker/' --include='*.go' | grep -oP '"[^"]+"' | sort -u

# Verify types.go files exist for packages that define structs
for pkg in amazon superkey logger sources; do
  test -f "${pkg}/types.go" && echo "OK: ${pkg}/types.go" || echo "MISSING: ${pkg}/types.go"
done

# Verify no amazon/ or sources/ imports leak into main.go
grep -c '"github.com/redhatinsights/sources-superkey-worker/amazon"' main.go && echo "VIOLATION: main.go imports amazon/" || echo "OK: main.go does not import amazon/"
grep -c '"github.com/redhatinsights/sources-superkey-worker/sources"' main.go && echo "VIOLATION: main.go imports sources/" || echo "OK: main.go does not import sources/"

# Verify amazon/ does not import superkey/ or provider/
grep -rn 'sources-superkey-worker/superkey\|sources-superkey-worker/provider' amazon/ && echo "VIOLATION" || echo "OK: amazon/ has no upward imports"
```
