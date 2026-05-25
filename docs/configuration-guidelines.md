# Configuration Guidelines

## 1. Clowder vs Local: Two Distinct Configuration Paths

The `config.Get()` function in `config/config.go` branches on `clowder.IsClowderEnabled()`. This is determined by the presence of the `ACG_CONFIG` environment variable (set automatically by the Clowder operator in OpenShift).

**When Clowder IS enabled** (deployed environment):
- Kafka brokers, CloudWatch credentials (`AwsRegion`, `AwsAccessKeyId`, `AwsSecretAccessKey`), `LogGroup`, `MetricsPort`, and Kafka topic names are all read from `clowder.LoadedConfig`. Do NOT set these via env vars; they will be ignored.

**When Clowder is NOT enabled** (local development):
- Kafka requires `QUEUE_HOST` and `QUEUE_PORT` env vars. If `QUEUE_PORT` is missing, no broker is configured and the worker will start but fail at runtime when attempting to connect.
- CloudWatch credentials come from `CW_AWS_ACCESS_KEY_ID` and `CW_AWS_SECRET_ACCESS_KEY`. If both are empty, the CloudWatch logging hook is silently skipped.
- `CLOUD_WATCH_LOG_GROUP` sets the log group name.
- `MetricsPort` defaults to `9394`.

Rule: Never add env-var overrides for fields that Clowder supplies. If you need a new config value that Clowder provides, read it from `clowder.LoadedConfig` inside the `IsClowderEnabled()` branch and fall back to `os.Getenv` in the else branch.

## 2. Sources API Connection Is Required

The worker calls the Sources API on every create and destroy request. These four env vars control the connection and are used in both Clowder and non-Clowder modes:

| Variable | Default (clowdapp.yaml) | Purpose |
|---|---|---|
| `SOURCES_SCHEME` | `http` | URL scheme for Sources API |
| `SOURCES_HOST` | `sources-api-svc` | Hostname of Sources API |
| `SOURCES_PORT` | `8000` | Port of Sources API |
| `SOURCES_PSK` | (from `internal-psk` secret) | Pre-shared key for internal auth |

Rule: When `SOURCES_PSK` is set, the worker authenticates using `x-rh-sources-psk` header (PSK mode). When empty, it falls back to forwarding the `x-rh-identity` header from the Kafka message. In deployed environments, PSK is injected from the `internal-psk` Kubernetes secret (see `deploy/clowdapp.yaml` lines 37-42). Never hardcode or commit PSK values.

## 3. Feature Flags Are Read at Startup and Use String Comparison

Two feature flags gate all message processing. They are read once via `os.Getenv` at package-level variable initialization in `main.go` (not through `config.Get()`):

| Variable | Default | Effect when `"true"` |
|---|---|---|
| `DISABLE_RESOURCE_CREATION` | `"false"` | Silently skips all `create_application` Kafka messages |
| `DISABLE_RESOURCE_DELETION` | `"false"` | Silently skips all `destroy_application` Kafka messages |

Rule: These flags compare against the literal string `"true"` (case-sensitive). Any other value, including `"True"`, `"TRUE"`, `"1"`, or `"yes"`, is treated as enabled/false. Always set them to exactly `"true"` or `"false"`.

Rule: Because these are read via `os.Getenv` at init time (before `main()` runs), changing them requires a pod restart. They cannot be hot-reloaded.

## 4. AWS_WAIT_TIME Controls IAM Race Condition Sleep

`AWS_WAIT_TIME` is read at call time (not startup) from `os.Getenv` in `superkey/forged_application.go:waitTime()`. It defines the number of seconds to sleep after creating AWS resources and before posting back to the Sources API, mitigating IAM eventual consistency.

- Default: `7` seconds (hardcoded constant `DEFAULT_SLEEP_TIME`).
- Declared in `clowdapp.yaml` with a default of `"15"`.
- Must be a valid integer string; non-numeric values log an error and fall back to `7`.

Rule: If changing this value, understand it directly affects how long each superkey creation blocks. Too low causes IAM race conditions; too high wastes time per request.

## 5. Logging Configuration

| Variable | Default | Notes |
|---|---|---|
| `LOG_LEVEL` | `WARN` (clowdapp) | Accepts `DEBUG`, `WARN`, `ERROR`; anything else maps to `INFO` |
| `LOG_HANDLER` | `built-in` | Passed through config but not branched on in current code |
| `CONTAINER_LOG_LEVEL` | `INFO` (clowdapp) | Declared in clowdapp.yaml but NOT read by any Go code; infrastructure-only |

The logger in `logger/logger.go` also supports ZincSearch integration via `logrus_zinc.FromEnv()`. This library reads its own env vars (prefixed `ZINC_`). If those vars are absent, the hook is silently skipped.

## 6. Kafka Topic Naming

The worker subscribes to `platform.sources.superkey-requests`. In Clowder, the actual topic name may differ (Clowder remaps topic names). The `KafkaTopic()` method on the config struct resolves the requested name to the Clowder-assigned name, falling back to the original if no mapping exists.

Rule: Always reference topics by their requested name (`platform.sources.superkey-requests`) in code. Never hardcode Clowder-remapped names.

## 7. Adding a New Environment Variable

Follow this pattern exactly:

1. If Clowder provides the value: read from `clowder.LoadedConfig` inside the `if clowder.IsClowderEnabled()` block in `config/config.go`.
2. In the `else` block, read from `os.Getenv("YOUR_VAR")` via `options.SetDefault`.
3. Add the field to the `SuperKeyWorkerConfig` struct.
4. Populate it in the return statement of `config.Get()`.
5. Declare the variable in `deploy/clowdapp.yaml` under both `env:` (with value or secretKeyRef) and `parameters:` (with default).
6. If the variable is a secret, use `valueFrom.secretKeyRef` in clowdapp.yaml, never a plain `value`.

Rule: Do NOT read env vars via `os.Getenv` scattered across the codebase. The only sanctioned exceptions are the two feature flags in `main.go` and `AWS_WAIT_TIME` in `superkey/forged_application.go`, which predate the config pattern.

## 8. Health Probes Depend on /tmp/healthy

Kubernetes liveness and readiness probes check for the existence of `/tmp/healthy` (created by `createHealthFile()` in `main.go`). This file is created unconditionally at startup and never deleted. The probes are `exec`-based (not HTTP), so they do not depend on any port configuration.

## Verification

```bash
# Confirm all env vars declared in clowdapp.yaml are documented above
grep -E '^\s+- name:' deploy/clowdapp.yaml | awk '{print $3}' | sort

# Confirm feature flags use exact string comparison
grep -n 'DisableCreation\|DisableDeletion' main.go

# Confirm which config fields come from Clowder vs env vars
grep -A1 'IsClowderEnabled' config/config.go

# List all os.Getenv calls outside config/config.go (should be minimal)
grep -rn 'os\.Getenv' --include="*.go" | grep -v config/config.go

# Verify SOURCES_PSK secret reference in deployment
grep -A4 'SOURCES_PSK' deploy/clowdapp.yaml

# Check AWS_WAIT_TIME default and parsing
grep -A10 'func waitTime' superkey/forged_application.go
```
