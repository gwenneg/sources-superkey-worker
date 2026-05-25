# API Contracts Guidelines

Rules governing HTTP calls from this worker to the Sources API. Every `SourcesClient` method in `sources/api_client.go` must follow these conventions.

---

## 1. API Versioning — Two Distinct Namespaces

| Namespace | Base path | Purpose |
|---|---|---|
| **Public** | `/api/sources/v3.1/` | CRUD for sources, applications, authentications, application_authentications, and availability checks |
| **Internal** | `/internal/v2.0/` | Fetching authentications with decrypted secrets (`expose_encrypted_attribute[]=password`) |

**Rules:**

- Never call `/internal/v2.0/` for any operation other than reading credentials that require decrypted passwords.
- Never send write requests (POST/PATCH) to the internal namespace.
- All public-facing resource mutations (create authentication, patch application, patch source) go through `/api/sources/v3.1/`.
- URL construction uses `conf.SourcesScheme`, `conf.SourcesHost`, `conf.SourcesPort` from config — never hardcode scheme, host, or port.

## 2. Authentication Header Strategy

The `headers()` method on `SourcesClient` implements a dual-mode authentication scheme. The mode is selected by whether `SOURCES_PSK` is set.

**PSK mode** (when `conf.SourcesPSK != ""`):
- `x-rh-sources-psk` — the pre-shared key (always sent)
- `x-rh-sources-account-number` — sent only when `AccountNumber` is non-empty
- `x-rh-identity` — forwarded only when `IdentityHeader` is non-empty
- `x-rh-org-id` — sent only when `OrgId` is non-empty

**Identity mode** (when `conf.SourcesPSK == ""`):
- `x-rh-identity` — a base64-encoded XRHID JSON structure; synthesized from `AccountNumber` + `OrgId` via `encodeIdentity()` if no raw header was provided

**Rules:**

- Every request must include `Content-Type: application/json`.
- Never mix PSK and identity-only modes in the same request; the `headers()` method handles this — do not bypass it.
- When constructing a `SourcesClient`, populate `IdentityHeader`, `OrgId`, and `AccountNumber` from the Kafka message headers (`x-rh-identity` and `x-rh-sources-org-id`). At least one of `IdentityHeader` or `OrgId` must be non-empty or the message is dropped.

## 3. Expected Status Codes per Endpoint

| Method | Endpoint pattern | HTTP verb | Success code | Failure check |
|---|---|---|---|---|
| `CheckAvailability` | `.../sources/{id}/check_availability` | POST | **202** (exact) | `!= 202` |
| `CreateAuthentication` | `.../authentications` | POST | **2xx** | `> 299` |
| `createApplicationAuthentication` | `.../application_authentications` | POST | **2xx** | `> 299` |
| `PatchApplication` | `.../applications/{id}` | PATCH | **2xx** | `> 299` |
| `PatchSource` | `.../sources/{id}` | PATCH | **2xx** | `> 299` |
| `GetInternalAuthentication` | `/internal/v2.0/authentications/{id}` | GET | **200** (exact) | `!= 200` |

**Rules:**

- `CheckAvailability` is the only endpoint that expects 202. All other success checks use `> 299` as the failure threshold.
- On non-2xx responses, read the body with `io.ReadAll` and include it in the error message. Do not discard response bodies on failure. Exception: `CheckAvailability` does not read the response body on non-202 — it returns only the status code in the error.
- `GetInternalAuthentication` is the only endpoint that checks for an exact 200; all other mutating endpoints accept any 2xx.

## 4. Retry Policy

Only `GetInternalAuthentication` retries. All other `SourcesClient` methods fail immediately on error.

- Retry count: **5 attempts** with a fixed **3-second sleep** between attempts.
- Retry triggers: non-200 HTTP status codes only. Transport errors (`err != nil`) break the loop immediately without retrying — the loop condition `if err != nil || res.StatusCode == 200` causes an immediate break in both the success and transport-error cases.
- On exhaustion: return an error wrapping the original error with `"unable to fetch internal authentication ... after 5 retries"`.
- No other endpoint has retry logic — do not add retries to mutating endpoints without considering idempotency.

## 5. HTTP Client Usage

All requests use `http.DefaultClient` (no custom timeouts, no TLS config, no connection pooling overrides). The `http.Request` struct is constructed manually (not via `http.NewRequest`).

**Rules:**

- Always call `defer resp.Body.Close()` after confirming the response is non-nil.
- In the retry loop of `GetInternalAuthentication`, `defer res.Body.Close()` is called inside the loop only on break — match this pattern to avoid leaking connections during retries.
- Request bodies use `io.NopCloser(bytes.NewBuffer(body))` — do not use `bytes.NewReader` or `strings.NewReader` which would not implement `io.ReadCloser`.

## 6. Error Escalation — MarkSourceUnavailable

When resource forging fails, the worker must call `MarkSourceUnavailable` before returning. This issues two sequential PATCH calls:

1. **PATCH application** with `availability_status: "unavailable"`, `availability_status_error` (includes the AWS error), and `extra` (partial superkey progress data).
2. **PATCH source** with `availability_status: "unavailable"`.

**Rules:**

- Always pass the partially-forged application (even if nil) to `MarkSourceUnavailable` — it handles nil by creating an empty `ForgedApplication`.
- The `extra` payload must include `_superkey.steps`, `_superkey.guid`, and `_superkey.provider` so that partial progress can be inspected or cleaned up later.
- `MarkSourceUnavailable` constructs its own `SourcesClient` if the forged application does not already have one — do not assume the client is pre-initialized.

## 7. Request/Response Model Types

All request and response types come from the `github.com/RedHatInsights/sources-api-go/model` package:

| Type | Used for |
|---|---|
| `model.AuthenticationCreateRequest` | POST body to create authentications |
| `model.AuthenticationResponse` | Unmarshal response after creating an authentication (to extract the ID) |
| `model.AuthenticationInternalResponse` | Unmarshal response from internal auth endpoint (contains `Username`, `Password`) |
| `model.ApplicationAuthenticationCreateRequest` | POST body linking an authentication to an application |

**Rules:**

- After creating an authentication, immediately extract its `ID` from the `AuthenticationResponse` to create the `ApplicationAuthentication` link. These are always paired.
- For `AuthenticationInternalResponse`, the JSON `id` field comes back as a string, which causes an unmarshal warning — this is expected and safe to ignore as long as `Username` and `Password` are populated.
- Validate that both `Username` and `Password` are non-empty after fetching internal auth; treat empty credentials as a fatal error.

## 8. Request Sequencing in CreateInSourcesAPI

The `CreateInSourcesAPI` method enforces a specific order with an intentional IAM race-condition delay:

1. **Sleep** for `AWS_WAIT_TIME` seconds (default 7) to let IAM propagate.
2. **PATCH application** with `extra` (superkey metadata, S3 bucket name).
3. **POST authentication** + **POST application_authentication** (linked together).
4. **POST check_availability** on the source.

Do not reorder these steps. The availability check must be last because it validates the resources created in steps 2-3.

## 9. Payload Conventions for PATCH Endpoints

PATCH payloads are `map[string]interface{}` — not typed structs. The keys are JSON field names sent directly to the Sources API.

Known PATCH payloads:
- Application: `extra`, `availability_status`, `availability_status_error`
- Source: `availability_status`

Do not add unknown fields to PATCH payloads without verifying they are accepted by the Sources API schema.

---

## Verification

```bash
# Confirm all API paths use the correct version prefixes
grep -n 'api/sources/v3.1\|internal/v2.0' sources/api_client.go

# Confirm every HTTP call uses http.DefaultClient
grep -n 'http.DefaultClient.Do' sources/api_client.go

# Confirm every response body is deferred-closed
grep -n 'defer.*Body.Close' sources/api_client.go

# Confirm Content-Type is always set
grep -n 'Content-Type' sources/api_client.go

# Confirm retry logic only exists for GetInternalAuthentication
grep -n 'retry' sources/api_client.go

# Confirm MarkSourceUnavailable patches both application and source
grep -n 'PatchApplication\|PatchSource' superkey/create_request.go

# Verify no hardcoded host/port in URL construction
grep -n 'conf.SourcesScheme\|conf.SourcesHost\|conf.SourcesPort' sources/api_client.go

# Build check — ensures model types from sources-api-go are compatible
go build ./...
```
