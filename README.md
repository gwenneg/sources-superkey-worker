# Sources SuperKey Worker

A Go Kafka consumer that processes superkey lifecycle events to provision and deprovision AWS resources (IAM roles, policies, S3 buckets, Cost and Usage Reports) on behalf of Red Hat customers. It runs on OpenShift as a Clowder-managed ClowdApp, consuming from the `platform.sources.superkey-requests` Kafka topic and posting results back to the Sources API. Currently only the AWS provider is implemented.

## Table of contents

- [Package layout](#package-layout)
- [Getting started](#getting-started)
- [Environment variables](#environment-variables)
- [Makefile targets](#makefile-targets)
- [Deployment](#deployment)
- [Documentation](#documentation)
- [License](#license)

## Package layout

| Path | Description |
|---|---|
| `main.go` | Entry point: Kafka consumer setup, signal handling, event routing (`create_application` / `destroy_application`) |
| `amazon/` | AWS SDK v2 wrappers for IAM (`iam.go`), S3 (`s3.go`), Cost and Usage Reports (`costandusagereports.go`), and credential handling (`credentials.go`) |
| `config/` | Runtime configuration struct populated from environment variables and Clowder |
| `deploy/` | Clowder ClowdApp OpenShift template (`clowdapp.yaml`) |
| `docs/` | 15 guideline files covering architecture, conventions, and operational details |
| `licenses/` | Apache 2.0 license file |
| `logger/` | Logrus-based logger with context-based structured logging and CloudWatch/ZincSearch hooks |
| `provider/` | Provider interface and orchestration layer: `forge.go` (provider instantiation), `amazon_provider.go` (AWS superkey provisioning logic) |
| `scripts/` | SonarQube scanner configuration |
| `sources/` | HTTP client wrapper for the Sources API (`api_client.go`, `helpers.go`) |
| `superkey/` | Domain types and request structs: `CreateRequest`, `DestroyRequest`, `ForgedApplication` |
| `util/` | Utility functions for producing Kafka messages |

Struct definitions and interfaces live in `types.go` within each package. Methods on those structs are in separate, purpose-named files.

## Getting started

### Prerequisites

- Go 1.24+ (toolchain go1.24.3)
- A running Kafka broker
- Access to a Sources API instance
- Docker (for container builds)

### Build and run locally

```sh
# Build the binary
make build

# Run locally (requires Kafka and Sources API)
make run

# Or run without building a binary first
make inlinerun
```

### Build and run in a container

```sh
# Build the container image
make container

# Run in the container (connects to localhost:9092 for Kafka)
make runcontainer
```

### Lint

```sh
make lint
```

This runs `go vet` followed by `golangci-lint` with the linters: `gofmt`, `gci`, `bodyclose`, `forcetypeassert`, `misspell`. If `make lint` fails on import order, run `make gci` to auto-fix.

### Tests

There are currently no tests in this repository (zero `*_test.go` files).

## Environment variables

When running under Clowder, most configuration is injected automatically. For local development, set these:

| Variable | Description | Default |
|---|---|---|
| `QUEUE_HOST` | Kafka broker hostname | _(none)_ |
| `QUEUE_PORT` | Kafka broker port | _(none)_ |
| `SOURCES_HOST` | Sources API hostname | `sources-api-svc` |
| `SOURCES_PORT` | Sources API port | `8000` |
| `SOURCES_SCHEME` | Sources API URL scheme | `http` |
| `SOURCES_PSK` | Pre-shared key for Sources API authentication | _(from K8s secret)_ |
| `LOG_LEVEL` | Log level | `WARN` |
| `LOG_HANDLER` | Log handler (`built-in` or external) | `built-in` |
| `CONTAINER_LOG_LEVEL` | Container-level log verbosity | `INFO` |
| `AWS_WAIT_TIME` | Seconds to sleep between creating AWS resources and posting to Sources API | `15` |
| `DISABLE_RESOURCE_CREATION` | Set to `true` to skip `create_application` processing | `false` |
| `DISABLE_RESOURCE_DELETION` | Set to `true` to skip `destroy_application` processing | `false` |
| `CW_AWS_ACCESS_KEY_ID` | AWS access key for CloudWatch logging (local only) | _(none)_ |
| `CW_AWS_SECRET_ACCESS_KEY` | AWS secret key for CloudWatch logging (local only) | _(none)_ |
| `CLOUD_WATCH_LOG_GROUP` | CloudWatch log group name (local only) | _(none)_ |

For the full configuration reference, see [`docs/configuration-guidelines.md`](docs/configuration-guidelines.md).

## Makefile targets

| Target | Description |
|---|---|
| `make` / `make build` | Compile the Go binary |
| `make tidy` | Run `go mod tidy` |
| `make clean` | Remove the compiled binary |
| `make container` | Build a Docker container image |
| `make run` | Build and run the binary |
| `make inlinerun` | Run via `go run .` without building a binary first |
| `make fancyrun` | Build and run, filtering for JSON log lines and piping through `jq` |
| `make runcontainer` | Build and run inside a Docker container |
| `make debug` | Start a Delve debugger session |
| `make remotedebug` | Start a headless Delve session for remote debugging |
| `make lint` | Run `go vet` and `golangci-lint` |
| `make gci` | Auto-fix import ordering via `golangci-lint` |

## Deployment

The service is deployed as a Clowder ClowdApp on OpenShift. The container image uses a two-stage build: UBI9 minimal as both the builder and runtime base, with Go installed via `microdnf` in the builder stage, running as non-root user `1001`. The ClowdApp template is at [`deploy/clowdapp.yaml`](deploy/clowdapp.yaml).

CI runs via GitHub Actions (`.github/workflows/actions.yml`) for linting, and Konflux/Tekton for container image builds.

For full deployment details, see [`docs/deployment-guidelines.md`](docs/deployment-guidelines.md).

## Documentation

Detailed architecture and operational guidelines are maintained in [`AGENTS.md`](AGENTS.md) and the `docs/` directory:

| Guideline | Topic |
|---|---|
| [`api-contracts-guidelines.md`](docs/api-contracts-guidelines.md) | Sources API HTTP contracts, authentication, retry policy |
| [`async-and-messaging-guidelines.md`](docs/async-and-messaging-guidelines.md) | Kafka consumer setup, topic/group config, message dispatch |
| [`aws-resource-provisioning-guidelines.md`](docs/aws-resource-provisioning-guidelines.md) | AWS resource creation/teardown order, step tracking |
| [`clowder-and-openshift-platform-guidelines.md`](docs/clowder-and-openshift-platform-guidelines.md) | Clowder config, ClowdApp conventions, health probes |
| [`code-organization-guidelines.md`](docs/code-organization-guidelines.md) | Package hierarchy, naming conventions, adding providers |
| [`configuration-guidelines.md`](docs/configuration-guidelines.md) | All environment variables, feature flags |
| [`data-validation-guidelines.md`](docs/data-validation-guidelines.md) | Kafka message validation, identity header decoding |
| [`dependency-management-guidelines.md`](docs/dependency-management-guidelines.md) | AWS SDK v1/v2 constraints, Dependabot, toolchain sync |
| [`deployment-guidelines.md`](docs/deployment-guidelines.md) | Dockerfile, CI/CD pipelines, local dev setup |
| [`error-handling-guidelines.md`](docs/error-handling-guidelines.md) | Error strategies for Kafka, teardown, and Sources API |
| [`integration-guidelines.md`](docs/integration-guidelines.md) | End-to-end Kafka/Sources API/AWS integration flows |
| [`logging-and-observability-guidelines.md`](docs/logging-and-observability-guidelines.md) | Structured logging, CloudWatch hooks, Prometheus metrics |
| [`performance-guidelines.md`](docs/performance-guidelines.md) | Synchronous processing model, resource limits |
| [`red-hat-identity-and-tenancy-guidelines.md`](docs/red-hat-identity-and-tenancy-guidelines.md) | `x-rh-identity` header propagation, PSK auth |
| [`security-guidelines.md`](docs/security-guidelines.md) | Credential lifecycle, non-root container, GUID generation |

## License

This project is available as open source under the terms of the [Apache License 2.0](http://www.apache.org/licenses/LICENSE-2.0).
