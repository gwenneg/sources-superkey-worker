# Dependency Management Guidelines

## Critical: GOTOOLCHAIN and Dockerfile Sync

The Dockerfile uses `ARG GOTOOLCHAIN=go1.24.3` to override the Go version provided by the UBI9 base image (`registry.access.redhat.com/ubi9/ubi-minimal:latest`), because the UBI image ships an older Go that cannot satisfy `go.mod`'s minimum version requirement.

- When updating the `toolchain` directive in `go.mod`, also update the `GOTOOLCHAIN` ARG in `Dockerfile` to the same version.
- When updating the `go` directive in `go.mod` (currently `go 1.24`), verify the UBI9 base image still works with the new GOTOOLCHAIN override.
- The CI workflow (`.github/workflows/actions.yml`) pins `go-version: "1.23"` for linting/formatting. This version lags behind `go.mod` and must be updated when the minimum Go version changes, or CI will silently use a stale toolchain.

## AWS SDK: Dual-Version Constraint

This repository uses **both** AWS SDK v1 and AWS SDK v2 simultaneously:

- **AWS SDK v2** (`github.com/aws/aws-sdk-go-v2`) is the primary SDK used in the `amazon/` package for all AWS service calls: IAM, S3, and Cost and Usage Reports.
- **AWS SDK v1** (`github.com/aws/aws-sdk-go`) is used **only** in `logger/logger.go` for the CloudWatch logging hook via `platform-go-middlewares/logging/cloudwatch`. A TODO comment in that file notes this should be migrated to v2 but is blocked by the upstream `platform-go-middlewares` library.

Rules:
- All new AWS service integrations must use AWS SDK v2 exclusively.
- Do not add new imports of `github.com/aws/aws-sdk-go` (v1). The only acceptable use is the existing CloudWatch logger path.
- AWS SDK v2 service packages (iam, s3, costandusagereportservice) must be updated together as a group -- they share internal modules (`internal/configsources`, `internal/endpoints/v2`) that must stay version-aligned.
- Do not remove the v1 SDK dependency until `platform-go-middlewares` provides a v2-compatible CloudWatch hook.

## Red Hat Platform Dependencies

Three Red Hat-internal dependencies require special attention:

| Dependency | Import Path | Pinning Strategy |
|---|---|---|
| sources-api-go | `github.com/RedHatInsights/sources-api-go` | Pseudo-version (commit hash), no tagged releases |
| app-common-go | `github.com/redhatinsights/app-common-go` | Tagged semver (`v1.6.8`) |
| platform-go-middlewares | `github.com/redhatinsights/platform-go-middlewares` | Tagged semver (`v1.0.0`) |

Rules:
- `sources-api-go` is consumed for its `model` and `kafka` packages (used in `main.go`, `sources/api_client.go`, `superkey/`). It uses pseudo-versions because the upstream repo does not cut releases. When updating, use `go get github.com/RedHatInsights/sources-api-go@<commit-sha>` to pin to a specific commit.
- Note the **mixed casing** in import paths: `RedHatInsights` (capital R, H, I) for sources-api-go vs `redhatinsights` (all lowercase) for app-common-go and platform-go-middlewares. Preserve this casing exactly.
- `app-common-go` provides Clowder configuration (`clowder.BrokerConfig`, `clowder.IsClowderEnabled()`). Upgrades may change Clowder config struct shapes and break `config/config.go`.
- `platform-go-middlewares` provides identity structs used in `sources/helpers.go`. This dependency is at v1.0.0 and is unlikely to change often.

## Dependabot Configuration

Dependabot is configured in `.github/dependabot.yml` with:
- **gomod** ecosystem: weekly schedule, 50 open PR limit
- **github-actions** ecosystem: weekly schedule, default PR limit

The high PR limit (50) means Dependabot will open many PRs simultaneously. Rules:
- AWS SDK v2 Dependabot PRs often arrive in groups (core + service packages). Merge them together or in quick succession to avoid transient version mismatches.
- Dependabot cannot update `sources-api-go` because it uses pseudo-versions with no tagged releases. This dependency must be updated manually.
- Review Dependabot PRs for `app-common-go` carefully -- Clowder API changes can break runtime config loading without compile errors.

## Transitive Dependencies to Watch

Several transitive dependencies come in through `sources-api-go` and are **not directly used** by this repo but appear in `go.sum`:
- `gorm.io/gorm`, `gorm.io/datatypes`, `gorm.io/driver/mysql` -- ORM dependencies from sources-api-go's model package
- `github.com/labstack/gommon` -- from sources-api-go's Echo dependency
- `github.com/go-sql-driver/mysql` -- database driver not used by this worker

Do not add direct imports of these transitive dependencies. If you need database or web framework functionality, discuss architecture changes first.

## Kafka Client

This repo uses `github.com/segmentio/kafka-go` (v0.4.48) **indirectly** -- the `util/produce_messages.go` utility tool imports it directly, but the main application consumes Kafka through `sources-api-go`'s `kafka` package wrapper. Rules:
- The main application's Kafka usage goes through `github.com/RedHatInsights/sources-api-go/kafka`. Do not import `segmentio/kafka-go` directly in production code.
- `util/produce_messages.go` is a standalone development tool (not part of the main binary) that imports `segmentio/kafka-go` directly. This is acceptable.

## Logging Dependencies

The logging stack has a specific dependency chain:
- `github.com/sirupsen/logrus` -- primary logger
- `github.com/lindgrenj6/logrus_zinc` -- ZincSearch hook (pinned to a 2022 commit pseudo-version, likely unmaintained)
- `github.com/redhatinsights/platform-go-middlewares` -- CloudWatch hook (requires AWS SDK v1)

Do not replace logrus with another logger without also updating the ZincSearch and CloudWatch hooks.

## Adding New Dependencies

Before adding a new dependency:
1. Check if `sources-api-go` already provides the functionality (it brings in many transitive deps).
2. Prefer standard library where possible -- this is a small, focused worker service.
3. Run `go mod tidy` after any dependency change to clean up `go.sum`.
4. Verify the Dockerfile still builds: `docker build . -t sources-superkey-worker -f Dockerfile`

## Verification

```bash
# Confirm go.mod and go.sum are in sync
go mod tidy && git diff --exit-code go.mod go.sum

# Verify GOTOOLCHAIN in Dockerfile matches go.mod toolchain directive
grep "^toolchain" go.mod
grep "GOTOOLCHAIN" Dockerfile

# Verify CI go-version is compatible with go.mod minimum
grep "go-version" .github/workflows/actions.yml
grep "^go " go.mod

# Check for unauthorized AWS SDK v1 usage (should only be logger/logger.go)
grep -rn '"github.com/aws/aws-sdk-go/' --include="*.go" .

# Check for direct segmentio/kafka-go imports outside util/
grep -rn '"github.com/segmentio/kafka-go"' --include="*.go" . | grep -v "util/"

# Verify no replace directives crept into go.mod
grep "^replace" go.mod

# Build the container to validate dependency resolution
make container
```
