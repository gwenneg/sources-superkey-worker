# Async and Messaging Guidelines

This worker is a single-topic Kafka consumer that processes superkey lifecycle events. All Kafka interaction flows through the shared `sources-api-go/kafka` library — this project does not implement its own consumer loop, reader, or message types.

## Topic and Consumer Configuration

- **Single topic only.** The worker consumes exactly `platform.sources.superkey-requests`. The requested topic name is defined as a constant `superkeyRequestedTopic` in `main.go`; the actual runtime topic name may differ because Clowder remaps it via `conf.KafkaTopic()`.
- **Never hardcode the runtime topic name.** Always resolve through `conf.KafkaTopic("platform.sources.superkey-requests")` where `conf = config.Get()`. Clowder populates `KafkaTopics` with `requestedName -> actualName` mappings; the `KafkaTopic()` method falls back to the requested name when running outside Clowder.
- **Consumer group ID is `sources-superkey-worker`.** Set as a Viper default in `config/config.go`. Changing this value resets consumer offsets in the cluster — coordinate with the platform team before modifying.
- **Topic is declared in `deploy/clowdapp.yaml`** under `spec.kafkaTopics` with 3 partitions and 3 replicas. Any partition or replica changes must go through this file; they are not configurable at runtime.

## Message Dispatch Pattern

- **Dispatch is header-based, not body-based.** The `event_type` Kafka header determines which code path runs. The message body is parsed only after the event type is matched.
- **Exactly two event types are supported:** `create_application` and `destroy_application`. Any other value logs an error and the message is skipped (not retried).
- **Identity is required.** Every message must carry either an `x-rh-identity` or `x-rh-sources-org-id` header. Messages missing both are logged at ERROR and skipped. Do not add a fallback — unauthenticated messages must be rejected.
- **Use `msg.GetHeader(key)` for header extraction.** This is provided by the `sources-api-go/kafka` package. Do not access raw Kafka headers directly.

## Message Parsing

- **Use `msg.ParseTo(&struct{})` for deserialization.** This method (from `sources-api-go/kafka`) unmarshals the message value into the target struct. Parse errors are logged and the message is skipped — there is no retry or dead-letter mechanism.
- **Request types are in `superkey/types.go`.** `CreateRequest` is used for `create_application`; `DestroyRequest` is used for `destroy_application`. Both are JSON-tagged structs. Add new fields there, not in `main.go`.
- **Identity and org-id headers are copied onto the request struct after parsing.** `req.IdentityHeader` and `req.OrgIdHeader` are set from headers, not from the message body. Do not move these into the JSON payload.

## Consumer Lifecycle and Shutdown

- **The consumer runs in a goroutine.** `kafka.Consume(reader, handlerFunc)` blocks inside the goroutine. The main goroutine waits on a signal channel.
- **Graceful shutdown listens for `os.Interrupt` and `syscall.SIGTERM`.** On signal receipt, the reader is closed via `kafka.CloseReader(reader, label)` and the process exits with code 0. Do not add `SIGKILL` handling — it cannot be caught.
- **No context cancellation propagation.** The current design does not pass a cancellable context from the signal handler to in-flight message processing. An in-flight AWS resource creation will NOT be cancelled on SIGTERM — `os.Exit(0)` terminates the process immediately, orphaning any partially-created resources. Keep this in mind when modifying shutdown behavior.
- **Health probe is file-based.** A `/tmp/healthy` file is created at startup for Kubernetes liveness/readiness probes. There is no health endpoint — the probes use `stat /tmp/healthy`.

## Error Handling Conventions

- **Parse failures skip the message.** If `msg.ParseTo()` fails, the error is logged (including the raw message value) and the handler returns. The consumer auto-commits the offset.
- **AWS resource creation errors trigger teardown.** If `provider.Forge()` fails partway, `provider.TearDown()` is called on the partially-created `ForgedApplication` to clean up, then the source and application are marked `unavailable` via the Sources API.
- **Teardown errors are collected, not short-circuited.** `TearDown()` returns `[]error`. All steps are attempted even if earlier ones fail. Log each error individually.
- **No dead-letter queue.** Failed messages are not re-enqueued. Errors are logged. If you need retry semantics, implement them at the application level (see `GetInternalAuthentication` in `sources/api_client.go` for the existing retry pattern: 5 attempts with 3-second sleep).

## Feature Flags

- **`DISABLE_RESOURCE_CREATION` and `DISABLE_RESOURCE_DELETION`** are string-compared against `"true"` (not parsed as bool). They are read from `os.Getenv()` at package-level variable initialization in `main.go`. When enabled, the corresponding event type is logged and skipped.
- **`AWS_WAIT_TIME`** controls the sleep (in seconds) between AWS resource creation and posting back to Sources API. Defaults to 7 seconds if unset or unparseable. Read from `os.Getenv()` at call time in `superkey/forged_application.go` (not at startup). Configured in `clowdapp.yaml` with a default of `"15"`.

## Utility Producer (`util/produce_messages.go`)

- **This is a standalone CLI tool, not part of the worker.** It uses `segmentio/kafka-go` directly (not `sources-api-go/kafka`) and hardcodes `localhost:9092`. It exists for local development/testing only.
- **Do not import this package from the main application.** It is in `package main` under `util/` and has its own `main()` function.
- **The producer does not set `event_type` or identity headers.** When using it for testing, you must add appropriate headers manually or the worker will reject the message.

## Logging in Message Handlers

- **Build a context with tenant/source/application IDs before processing.** Use `l.WithTenantId()`, `l.WithSourceId()`, `l.WithApplicationId()`, and `l.WithApplicationType()` to attach structured fields to the context. Then use `l.LogWithContext(ctx)` for all subsequent log calls.
- **Do not log raw message values at INFO or WARN.** The raw `msg.Value` appears at DEBUG level for routine tracing and at ERROR level inside parse-failure log lines. Never add raw message logging at INFO or WARN — they may contain sensitive data.

## Clowder Integration

- **Broker configuration comes from Clowder when `ACG_CONFIG` is set.** `clowder.LoadedConfig.Kafka.Brokers` provides the broker list. Outside Clowder, set `QUEUE_HOST` and `QUEUE_PORT` environment variables.
- **Topic name remapping is automatic.** Clowder may rename `platform.sources.superkey-requests` to a namespace-prefixed variant. The `KafkaTopic()` method handles this transparently.

## Verification

```bash
# Confirm the topic constant matches clowdapp declaration
grep -n 'superkeyRequestedTopic' main.go
grep 'topicName' deploy/clowdapp.yaml

# Confirm consumer group ID default
grep 'KafkaGroupID' config/config.go

# Confirm all event_type cases are handled
grep -n 'case "' main.go | grep -v '//'

# Confirm signal handling is present
grep -n 'signal.Notify\|SIGTERM' main.go

# Confirm identity header validation
grep -n 'x-rh-identity\|x-rh-sources-org-id' main.go

# Confirm feature flag checks
grep -n 'DISABLE_RESOURCE' main.go

# Lint and vet
make lint

# Build to verify compilation
make build
```
