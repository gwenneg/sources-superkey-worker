# Logging and Observability Guidelines

## Architecture Overview

This worker uses `logrus` as its logging library with a custom JSON formatter (`CustomLoggerFormatter`). Log output goes to stdout and optionally to CloudWatch and ZincSearch via logrus hooks. Prometheus metrics are exposed via a separate HTTP server. There is no distributed tracing.

---

## Rules

### 1. Always use `LogWithContext(ctx)` when a `context.Context` is available

Every function that receives a `context.Context` must log with `l.LogWithContext(ctx)`, never `l.Log` directly. `LogWithContext` automatically extracts and attaches `tenant_id`, `source_id`, `application_id`, and `application_type` from the context. Using `l.Log` directly drops these fields and makes log correlation impossible.

```go
// CORRECT
l.LogWithContext(ctx).Info("Processing request")

// WRONG - loses tenant/source/application context
l.Log.Info("Processing request")
```

The only legitimate uses of `l.Log` directly are in startup/shutdown code where no request context exists (e.g., `main()` before message processing begins).

### 2. Populate the context with all four identity fields at the message boundary

When a Kafka message is received, build the context using all available `With*` helpers before passing it downstream. The canonical pattern from `processSuperkeyRequest` in `main.go`:

```go
ctx := l.WithTenantId(context.Background(), req.TenantID)
ctx = l.WithSourceId(ctx, req.SourceID)
ctx = l.WithApplicationId(ctx, req.ApplicationID)
ctx = l.WithApplicationType(ctx, req.ApplicationType)
```

For `destroy_application` events, `source_id`, `application_id`, and `application_type` are not available in the `DestroyRequest` struct -- only `tenant_id` is set. Do not invent values; set only what the request provides.

### 3. Attach ad-hoc fields with `WithField`/`WithFields`, do not embed them in the message string

Structured fields must be added via logrus field methods so they appear as distinct JSON keys, not buried in the `message` string.

```go
// CORRECT
l.LogWithContext(ctx).WithField("request_url", reqURL).Debug("Requesting availability check")

// WRONG - URL not queryable as a structured field
l.LogWithContext(ctx).Debugf("Requesting availability check at %s", reqURL)
```

Commonly used ad-hoc fields in this codebase: `org_id`, `request_url`, `body`, `kafka_message`, `authentication_id`.

### 4. Use correct log levels consistently

| Level | When to use | Example from codebase |
|-------|-------------|----------------------|
| `Fatal` | Unrecoverable startup failure (exits process) | CloudWatch hook init failure, Kafka reader failure, health file creation failure |
| `Error` | Operation failed but worker continues | Parse failure on Kafka message, teardown failure |
| `Warn` | Recoverable degradation, retries | `"Unable to fetch internal authentication. Retrying..."` |
| `Info` | Significant business events | Resource created/destroyed, request processing start/finish |
| `Debug` | Detailed operational data, request payloads | Request URLs, bodies, intermediate step details |

Never use `Fatal` for per-message errors -- it kills the entire worker process. Reserve `Fatal` exclusively for initialization failures.

### 5. Do not log secrets or credentials

The `body` field logged in `sources/api_client.go` Debug calls contains request payloads sent to Sources API. These payloads must never include raw credentials. The `GetInternalAuthentication` response contains `Username` and `Password` fields -- do not log this response. Currently this is handled correctly, but any change to add debug logging of authentication responses must exclude password fields.

### 6. Implement the `Marshaler` interface for custom log representations

The custom formatter checks for the `logger.Marshaler` interface on all field values. Any struct attached as a logrus field that should have a custom log representation must implement:

```go
type Marshaler interface {
    MarshalLog() map[string]interface{}
}
```

Error types are automatically converted via `.Error()`. All other types are logged as-is.

### 7. Log format is fixed JSON -- do not change field names

The `CustomLoggerFormatter` emits these top-level JSON fields: `@timestamp`, `@version`, `message`, `level`, `hostname`, `app`, `caller`, `labels`, `tags`. Downstream log aggregation (CloudWatch, ZincSearch) depends on this schema. Any field rename or removal is a breaking change that requires coordinating with the logging infrastructure team.

The `@timestamp` format is `2006-01-02T15:04:05.999Z` (millisecond precision, UTC layout string but using wall-clock time). The `caller` field contains the fully qualified function name from logrus `ReportCaller`.

### 8. CloudWatch hook activates only when AWS credentials are present

The CloudWatch batching hook (`platform-go-middlewares/logging/cloudwatch`) initializes only if both `AwsAccessKeyID` and `AwsSecretAccessKey` are non-empty in the config. In Clowder environments, these come from `cfg.Logging.Cloudwatch.*`. Locally, set `CW_AWS_ACCESS_KEY_ID` and `CW_AWS_SECRET_ACCESS_KEY` environment variables. If neither is set, logs go only to stdout (and ZincSearch if configured).

The hook batches logs with a 10-second flush interval.

### 9. ZincSearch hook activates via environment variables

The `logrus_zinc.FromEnv()` call reads its own env vars (documented in the `lindgrenj6/logrus_zinc` library). If those env vars are absent, the hook silently does not activate. No explicit error is logged for missing ZincSearch configuration.

### 10. Metrics are default Prometheus process/Go runtime metrics only

The `/metrics` endpoint (`promhttp.Handler()`) on the configured `MetricsPort` (default `9394`, overridden by Clowder's `cfg.MetricsPort`) exposes only the default Go process and runtime metrics. There are **no custom application metrics** registered (no counters, gauges, or histograms for message processing, error rates, or latency). The endpoint runs in a background goroutine; if it fails to bind, the error is logged but the worker continues.

### 11. No distributed tracing exists

There is no OpenTelemetry, Jaeger, Zipkin, or any tracing instrumentation. Cross-service correlation relies entirely on log fields (`tenant_id`, `source_id`, `application_id`) matching between this worker and the Sources API.

### 12. Log level is controlled by `LOG_LEVEL` environment variable

Accepted values: `DEBUG`, `ERROR`, `WARN` (case-sensitive). Anything else defaults to `INFO`. During test runs (`flag.Lookup("test.v") != nil`), the level is forced to `FATAL` to suppress log output.

### 13. Two inconsistencies: functions that use `l.Log` directly due to missing context

Two functions in the codebase use `l.Log` (or `logger.Log`) directly without a context parameter:

- `encodeIdentity` in `sources/helpers.go` uses `logger.Log.WithFields(...)` because it does not receive a context parameter. If this function is ever refactored to accept a context, switch it to `LogWithContext`.
- `waitTime` in `superkey/forged_application.go` uses `l.Log.Errorf(...)` when `AWS_WAIT_TIME` cannot be parsed, because `waitTime()` is a pure helper that does not receive a context. If the function is ever refactored to accept a context, switch it to `LogWithContext`.

---

## Verification

```bash
# Confirm all non-startup log calls in business logic use LogWithContext, not Log directly
grep -rn "l\.Log\." --include="*.go" | grep -v "_test.go" | grep -v "main.go" | grep -v "logger/"

# Check that no credential fields are logged
grep -rn "Password\|SecretKey\|password\|secret" --include="*.go" | grep -i "log\|debug\|info\|warn\|error" | grep -v "_test.go"

# Verify the four context fields are set at the Kafka message boundary
grep -A5 "WithTenantId\|WithSourceId\|WithApplicationId\|WithApplicationType" main.go

# Confirm no custom Prometheus metrics are registered
grep -rn "prometheus.New\|promauto\|MustRegister" --include="*.go" | grep -v vendor

# Verify metrics endpoint port configuration
grep -rn "MetricsPort" --include="*.go"

# Check log level handling
grep -A10 "cfg.LogLevel" logger/logger.go
```
