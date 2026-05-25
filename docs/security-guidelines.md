# Security Guidelines — sources-superkey-worker

## 1. AWS Credential Lifecycle (Critical)

Customer AWS credentials (access key + secret) are fetched from Sources API per Kafka message and used to create an ephemeral `amazon.Client`. These credentials MUST remain request-scoped.

- Never store AWS credentials in package-level variables, caches, or connection pools. The `amazon.Client` struct is created in `provider/forge.go:getProvider` and must not outlive the `processSuperkeyRequest` call in `main.go`.
- The `amazon/credentials.go` file uses `credentials.StaticCredentialsProvider` with customer-supplied keys. Do not replace this with shared/ambient credentials (e.g., EC2 instance roles) — those are the platform's own credentials, not the customer's.
- The `amazon.Client` struct (`amazon/types.go`) holds `AccessKey` and `SecretKey` as plain strings. Never add serialization tags (json/yaml) to these fields. Never log the `Client` struct.
- When adding new AWS service clients to `amazon/types.go:NewClient`, always derive them from the per-request `*aws.Config` returned by `NewAmazonConfig`, not from a cached config.

## 2. Sources API Authentication Headers

All HTTP requests to Sources API go through `sources/api_client.go:headers()`. This method implements a dual-auth strategy:

- **PSK mode** (production): When `SOURCES_PSK` is set, the `x-rh-sources-psk` header is sent. The PSK value comes from the `internal-psk` Kubernetes secret (`deploy/clowdapp.yaml`). When adding new Sources API calls, always use `sc.headers()` — never construct headers manually.
- **x-rh-identity mode** (fallback): When PSK is empty, a base64-encoded JSON identity (XRHID) is sent. The encoding logic is in `sources/helpers.go:encodeIdentity`. This uses the `platform-go-middlewares/identity` package.
- In PSK mode, `x-rh-sources-account-number` and `x-rh-org-id` headers are sent alongside the PSK for tenant scoping. Both must be present when available. Do not remove the conditional org-id/account-number headers from `headers()`.

## 3. Internal Authentication Endpoint

`sources/api_client.go:GetInternalAuthentication` calls `/internal/v2.0/authentications/{id}?expose_encrypted_attribute[]=password` to retrieve decrypted credentials.

- This endpoint returns plaintext passwords. Responses must never be logged at any level. The current code correctly avoids logging the response body — preserve this.
- The `model.AuthenticationInternalResponse` contains `Username` and `Password` fields. These flow into `amazon.NewClient` as `key` and `sec`. Ensure no intermediate logging captures these values.

## 4. Kafka Message Identity Validation

In `main.go:processSuperkeyRequest`, a message is skipped only when **neither** `x-rh-identity` nor `x-rh-sources-org-id` is present (`if identityHeader == "" && orgIdHeader == ""`). A message carrying either header alone is processed normally.

- Never remove the identity/org-id header check. If both headers are missing, the message must be skipped.
- The `x-rh-identity` header from Kafka is forwarded as-is to Sources API calls. Do not decode, modify, or re-encode it unless explicitly required.
- When adding new Kafka message types (beyond `create_application`/`destroy_application`), always enforce the same header presence check.

## 5. Container Security Constraints

- **Dockerfile**: The final stage runs as `USER 1001` (non-root). Never add a `USER root` directive or remove the existing `USER 1001` line.
- **ClowdApp**: `deploy/clowdapp.yaml` sets `runAsNonRoot: true` under `securityContext`. Do not remove this. If adding init containers or sidecars, they must also set `runAsNonRoot: true`.
- The health probe writes to `/tmp/healthy` (`main.go:createHealthFile`). This path is writable by non-root users. Do not change it to a path requiring elevated permissions.

## 6. No TLS Configuration Exists

The codebase has zero TLS references. Kafka connections and Sources API HTTP calls rely entirely on the platform's network-level encryption (Clowder-managed).

- `SOURCES_SCHEME` defaults to `http` in the ClowdApp parameters. In production, the service mesh provides TLS termination. Do not hardcode `https` — the scheme is environment-dependent.
- All Sources API calls use `http.DefaultClient` with no custom transport. If you need to add TLS verification (e.g., for external endpoints), create a dedicated `http.Client` with a configured `tls.Config` rather than modifying `http.DefaultClient`.

## 7. Secret Management

- `SOURCES_PSK` is injected from a Kubernetes secret (`internal-psk`, key `psk`) via `deploy/clowdapp.yaml`. It is never set as a plain-text parameter default. Keep it that way.
- CloudWatch logging credentials (`CW_AWS_ACCESS_KEY_ID`, `CW_AWS_SECRET_ACCESS_KEY`) in `config/config.go` are used only for log shipping, not for customer-facing AWS operations. These are separate from customer credentials.

## 8. Logging Sensitive Data

- `sources/api_client.go` logs request URLs and bodies at Debug level for `CreateAuthentication`, `PatchApplication`, and `createApplicationAuthentication`. The `AuthenticationCreateRequest` body may contain a `Username` field. Ensure `Password`/`SecretAccessKey` fields are never added to these request payloads.
- The `amazon.Client` struct contains `AccessKey` and `SecretKey`. Never pass this struct to `logrus.WithField` or `fmt.Sprintf` in log statements.

## 9. Supply Chain Security

- **Anchore Grype/Syft**: `.github/workflows/security-workflow-template.yml` runs the `RedHatInsights/platform-security-gh-workflow` on pushes and PRs to `main`/`master`. This performs container vulnerability scanning and SBOM generation. Do not remove or skip this workflow.
- **Dependabot**: `.github/dependabot.yml` monitors `gomod` and `github-actions` weekly. Do not reduce the update frequency or lower the PR limit.

## 10. GUID Generation

`provider/amazon_provider.go:generateGUID` uses `crypto/rand` for generating resource name suffixes. Do not replace this with `math/rand` — AWS resource names that collide could cause cross-tenant resource conflicts.

## Verification

```bash
# Confirm container runs as non-root
grep -n "USER 1001" Dockerfile
grep -n "runAsNonRoot: true" deploy/clowdapp.yaml

# Confirm PSK comes from a Kubernetes secret, not a plaintext default
grep -A4 "SOURCES_PSK" deploy/clowdapp.yaml | grep "secretKeyRef"

# Confirm no TLS skip or insecure settings exist
grep -rn "InsecureSkipVerify\|TLSClientConfig" --include="*.go" .

# Confirm AWS credentials are never logged
grep -rn "SecretKey\|SecretAccessKey\|Password" --include="*.go" . | grep -i "log\|print\|debug\|info\|warn\|error" | grep -v "go.sum"

# Confirm crypto/rand is used for GUID generation (not math/rand)
grep -n "crypto/rand" provider/amazon_provider.go

# Confirm identity header validation in message processing
grep -n 'identityHeader == "" && orgIdHeader == ""' main.go

# Confirm no AWS credentials in environment variable defaults
grep -n "SOURCES_PSK\|AWS_SECRET\|AWS_ACCESS" deploy/clowdapp.yaml | grep -v "secretKeyRef\|CW_"
```
