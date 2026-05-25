# Deployment Guidelines

## Container Image Build

- **Base image is UBI9 minimal** (`registry.access.redhat.com/ubi9/ubi-minimal:latest`). Never switch to a non-UBI base; Red Hat certification requires it.
- **The `GOTOOLCHAIN` build ARG exists as a workaround** because the UBI9 image ships an older Go version than `go.mod` requires (currently Go 1.24). When updating `go.mod`'s Go version, also update the `ARG GOTOOLCHAIN` default in the Dockerfile to match. If they drift apart the build will fail.
- **The Dockerfile is a two-stage build.** Stage 1 (`build`) compiles the binary; stage 2 copies only the binary and the license file. Do not add runtime dependencies to stage 1 or dev tools to stage 2.
- **The binary name is `sources-superkey-worker`** (derived from the Go module name). The ENTRYPOINT expects it at `/sources-superkey-worker`. If you rename the module, update the `COPY --from=build` path and the ENTRYPOINT.
- **`licenses/LICENSE` must exist.** The Dockerfile copies it to `/licenses/LICENSE` in the final image for Red Hat license compliance scanning. Deleting or moving it breaks the image build.
- **The container runs as non-root `USER 1001`.** The pod spec in `deploy/clowdapp.yaml` enforces `runAsNonRoot: true`. Never add `USER root` to the final stage. The only writable path the app needs at runtime is `/tmp/healthy` (for the health probe file), which is writable by any user in the UBI minimal image.

## ClowdApp Deployment (deploy/clowdapp.yaml)

- **Resource defaults are deliberately small** (CPU: 20m request / 50m limit, Memory: 50Mi request / 100Mi limit). This service is a lightweight Kafka consumer. Do not raise limits without load-testing justification.
- **Replica count is templated via `MIN_REPLICAS`** (default `1`). The parameter uses the `${{}}` integer interpolation syntax (double braces), not the string `${}` syntax. Using single braces would inject the value as a string and break the ClowdApp schema.
- **Liveness and readiness probes both exec `stat /tmp/healthy`.** The application creates this file once at startup in `main.go:createHealthFile()`. There is no ongoing health check against Kafka brokers -- the file is created unconditionally. If the file is somehow removed, the pod will be killed by the liveness probe (period: 60s, initial delay: 10s).
- **Readiness probe initial delay is 3 seconds; liveness is 10 seconds.** The readiness probe fires first so traffic is accepted quickly. Do not set readiness delay higher than liveness delay or the pod may be killed before it is ever marked ready.
- **Kafka topic `platform.sources.superkey-requests`** is declared with 3 partitions and 3 replicas. Changing the partition count requires coordination with consumers -- the consumer group ID is hardcoded to `sources-superkey-worker` in `config/config.go`.
- **The `SOURCES_PSK` env var** is sourced from the `internal-psk` secret, key `psk`, marked `optional: true`. If the secret is missing, the worker starts but Sources API calls that require PSK authentication will fail silently.
- **Feature flags `DISABLE_RESOURCE_CREATION` and `DISABLE_RESOURCE_DELETION`** (default `"false"`) let you disable create or destroy processing independently. These are string booleans read via `os.Getenv()` in `main.go`. Set to `"true"` to disable.

## CI Pipelines

### Konflux / Tekton (.tekton/)

- **Pull request pipeline** triggers on `event == "pull_request" && target_branch == "master"`. It tags the image with `on-pr-{{revision}}` and sets `image-expires-after: 5d`. PR images are ephemeral and auto-cleaned.
- **Push pipeline** triggers on `event == "push" && target_branch == "master"`. It tags the image with the full commit SHA (no expiry). This is the production image path.
- Both pipelines reference `RedHatInsights/konflux-pipelines` at `main` branch, path `pipelines/docker-build.yaml`. Pipeline updates come from that upstream repo, not from this one.
- Both use a `1Gi` PVC workspace and the `build-pipeline-sources-superkey-worker` service account in the `hcc-integrations-tenant` namespace.
- PR images go to `quay.io/redhat-user-workloads/hcc-integrations-tenant/sources/sources-superkey-worker`. This is a different registry path than the manual `build_deploy.sh` target (`quay.io/cloudservices/sources-superkey-worker`).

### GitHub Actions (.github/workflows/)

- `actions.yml` runs `gofmt`, `go vet`, and `golangci-lint` on push to `main` and on PRs. The Go version is pinned to `1.23` in the workflow -- this may lag behind `go.mod`. Keep them in sync to avoid false lint failures.
- `security-workflow-template.yml` delegates to `RedHatInsights/platform-security-gh-workflow` for Grype vulnerability scanning and Syft SBOM generation. It triggers on pushes and PRs to `main`, `master`, and `security-compliance` branches.

### Manual Build (build_deploy.sh)

- Requires four environment variables: `QUAY_USER`, `QUAY_TOKEN`, `RH_REGISTRY_USER`, `RH_REGISTRY_TOKEN`. The script exits immediately if any are unset.
- Tags: `git rev-parse --short=7 HEAD`, plus `latest` and `qa` for backward compatibility. All three tags are pushed.
- Target registry: `quay.io/cloudservices/sources-superkey-worker`.
- Creates a local `.docker/` config directory for credential isolation. This directory is not gitignored by name but is covered by the binary entry in `.dockerignore`.

### PR Check (pr_check.sh)

- Runs `make container` (a plain `docker build`) and creates a dummy JUnit XML in `artifacts/junit-dummy.xml` to satisfy CI test-result requirements. The bonfire/ephemeral-environment integration is commented out -- smoke tests are not currently wired up.

## Local Development

- `make container` builds the Docker image locally as `sources-superkey-worker` (no registry prefix).
- `make runcontainer` runs it with `--net host` and `KAFKA_BROKERS=localhost:9092`. You need a local Kafka broker running on port 9092.
- Without Clowder (`ACG_CONFIG` not set / `clowder.IsClowderEnabled()` returns false), the app reads Kafka broker config from `QUEUE_HOST` and `QUEUE_PORT` env vars, and Sources API config from `SOURCES_HOST`, `SOURCES_SCHEME`, `SOURCES_PORT`. Metrics default to port 9394.

## Verification

```bash
# Build the container image
make container

# Verify the image runs as non-root (should show UID 1001)
docker run --rm --entrypoint id sources-superkey-worker

# Verify the license file is present in the image
docker run --rm --entrypoint sh sources-superkey-worker -c "test -f /licenses/LICENSE && echo OK"

# Validate the ClowdApp YAML syntax
python3 -c "import yaml; yaml.safe_load(open('deploy/clowdapp.yaml'))"

# Check that GOTOOLCHAIN ARG matches go.mod toolchain directive
grep 'ARG GOTOOLCHAIN' Dockerfile
grep '^toolchain' go.mod

# Run the same lint checks as GitHub Actions CI
go vet ./...
golangci-lint run -E gofmt,gci,bodyclose,forcetypeassert,misspell
```
