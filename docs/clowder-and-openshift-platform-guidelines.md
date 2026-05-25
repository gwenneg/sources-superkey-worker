# Clowder & OpenShift Platform Guidelines

Rules for working with the Clowder deployment model and OpenShift platform in sources-superkey-worker.

## 1. Clowder Config Branching Pattern

The `config/config.go` file uses a strict two-branch pattern controlled by `clowder.IsClowderEnabled()` (which checks the `ACG_CONFIG` environment variable). Every new config field that has a Clowder-managed equivalent MUST follow this pattern:

- **Inside the `if clowder.IsClowderEnabled()` block**: read from `clowder.LoadedConfig` (the parsed `ACG_CONFIG` JSON).
- **Inside the `else` block**: read from environment variables or hardcoded defaults for local development.
- **After both branches** (shared section): set values for config fields that are the same regardless of Clowder mode (e.g., `LOG_LEVEL`, `SOURCES_HOST`, `SOURCES_PSK`).

Never read `clowder.LoadedConfig` outside the Clowder-enabled branch. Never read `QUEUE_HOST`/`QUEUE_PORT` inside the Clowder-enabled branch.

## 2. Adding a New Environment Variable

Follow this decision tree:

| Source of truth | Where to add | Example |
|---|---|---|
| Clowder injects it (Kafka, CloudWatch, metrics port) | Clowder branch only, read from `clowder.LoadedConfig` | `cfg.Logging.Cloudwatch.Region` |
| App-specific, same in both modes | Shared section after both branches, use `os.Getenv()` | `SOURCES_HOST`, `LOG_LEVEL` |
| App-specific, different defaults per mode | Both branches with different defaults | `MetricsPort` (Clowder: `cfg.MetricsPort`, local: `9394`) |

After adding the field:
1. Add it to the `SuperKeyWorkerConfig` struct.
2. Set its default with `options.SetDefault(...)` in the correct branch.
3. Read it in the return statement with the matching `options.Get*` call.
4. If it needs to be in the ClowdApp template, add it to `deploy/clowdapp.yaml` under both `env:` and `parameters:`.

## 3. Kafka Topic Declaration

- The single Kafka topic `platform.sources.superkey-requests` is declared in two places that must stay in sync:
  - `deploy/clowdapp.yaml` under `kafkaTopics:` (the Clowder CRD declaration).
  - `main.go` as the constant `superkeyRequestedTopic`.
- In Clowder mode, topic names are remapped via `clowder.KafkaTopics`. The code uses `conf.KafkaTopic(requestedName)` to resolve the actual topic name. Always use this method; never hardcode the Clowder-remapped topic name.
- Current topic spec: **3 partitions, 3 replicas**. Do not change these without coordinating with the platform team.

## 4. Kafka Broker Configuration

- In Clowder mode, brokers come from `cfg.Kafka.Brokers` as `[]clowder.BrokerConfig`.
- In local mode, brokers are built manually from `QUEUE_HOST` and `QUEUE_PORT` env vars.
- The `BrokerConfig` type from `app-common-go` (`github.com/redhatinsights/app-common-go/pkg/api/v1`) is used in both paths. The `Port` field is a `*int` (pointer), not an `int`.
- The Kafka reader is obtained through `sources-api-go/kafka` (not `segmentio/kafka-go` directly in production code). The `util/produce_messages.go` utility uses `segmentio/kafka-go` directly but is a standalone dev tool, not production code.

## 5. Health Probes

Both liveness and readiness probes use a file-based check: `stat /tmp/healthy`.

- The file is created at startup in `main.go` via `createHealthFile()` before the Kafka consumer starts.
- There is no dynamic health check against Kafka brokers. The comment in `main.go` explains this is intentional: the wrapped Kafka clients do not return errors suitable for probe-based restarts.
- If you add an HTTP-based probe, you must also update `deploy/clowdapp.yaml` to change from `exec` to `httpGet` probes.

Probe timing in `clowdapp.yaml`:
- **readinessProbe**: `initialDelaySeconds: 3`
- **livenessProbe**: `initialDelaySeconds: 10`, `periodSeconds: 60`

## 6. ClowdApp Template Conventions

The deployment manifest at `deploy/clowdapp.yaml` is an OpenShift Template (not raw Kubernetes YAML):

- Parameters are declared in the `parameters:` section at the bottom and referenced as `${PARAM}` (strings) or `${{PARAM}}` (non-strings like integers).
- `ENV_NAME` is the only required parameter (no default). All others have defaults.
- Use `${{MIN_REPLICAS}}` (double braces) for integer parameters to avoid string-to-int coercion issues.
- `SOURCES_PSK` is pulled from a Kubernetes Secret (`internal-psk`), not a template parameter. It is marked `optional: true`.
- Security context requires `runAsNonRoot: true`. The Dockerfile sets `USER 1001` to match.

## 7. Resource Limits

Current defaults (keep these conservative for a Kafka consumer with minimal processing):

| Resource | Request | Limit |
|----------|---------|-------|
| CPU | 20m | 50m |
| Memory | 50Mi | 100Mi |

If adding CPU-intensive work (e.g., crypto operations, large JSON parsing), update both the default parameter values and test under load before merging.

## 8. CloudWatch Logging Injection

- In Clowder mode: `Region`, `AccessKeyId`, `SecretAccessKey`, and `LogGroup` come from `cfg.Logging.Cloudwatch.*`.
- In local mode: use `CW_AWS_ACCESS_KEY_ID`, `CW_AWS_SECRET_ACCESS_KEY`, and `CLOUD_WATCH_LOG_GROUP` env vars. Region defaults to `us-east-1`.
- The CloudWatch hook in `logger/logger.go` is only added when both `key` and `secret` are non-empty. If either is missing, CloudWatch logging is silently skipped.

## 9. Metrics Port

- In Clowder mode: read from `cfg.MetricsPort` (Clowder assigns a port).
- In local mode: defaults to `9394`.
- The metrics endpoint is served at `/metrics` on this port using `promhttp.Handler()`.

## 10. Env Vars Read Outside config.go

Two environment variables are read directly in `main.go` via `os.Getenv()` at package init time, bypassing the config struct:

- `DISABLE_RESOURCE_CREATION` - set to `"true"` to skip `create_application` requests.
- `DISABLE_RESOURCE_DELETION` - set to `"true"` to skip `destroy_application` requests.

Additionally, `AWS_WAIT_TIME` is read in `superkey/forged_application.go` via `os.Getenv()`. Go-level fallback default: 7 seconds; `deploy/clowdapp.yaml` default for deployed environments: 15 seconds. This controls the sleep before posting back to Sources API after IAM resource creation.

All three are declared in the ClowdApp template. If adding similar operational toggles, follow this pattern and ensure they appear in `clowdapp.yaml` parameters.

## 11. Non-Root Container Requirement

- The Dockerfile uses `USER 1001` (non-root).
- The ClowdApp sets `securityContext.runAsNonRoot: true`.
- The health file is written to `/tmp/healthy` because `/tmp` is writable by non-root users. Do not change the probe file path to a location that requires root access.

## Verification

```bash
# Confirm Clowder branching exists and is not broken
grep -n "IsClowderEnabled" config/config.go

# Verify topic name consistency between main.go and clowdapp.yaml
grep "platform.sources.superkey-requests" main.go deploy/clowdapp.yaml

# Ensure health file path matches between Go code and YAML probes
grep "/tmp/healthy" main.go deploy/clowdapp.yaml

# Check that runAsNonRoot and USER 1001 are both present
grep "runAsNonRoot" deploy/clowdapp.yaml
grep "USER 1001" Dockerfile

# Lint and vet the Go code
make lint
```
