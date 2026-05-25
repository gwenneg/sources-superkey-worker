# Red Hat Identity and Tenancy Guidelines

Domain guideline for sources-superkey-worker identity propagation, authentication headers, and tenant context management.

## Critical Rules

### 1. Always populate all three SourcesClient identity fields

Every `sources.SourcesClient` instantiation MUST set `IdentityHeader`, `OrgId`, AND `AccountNumber`. The `headers()` method on `SourcesClient` branches on whether `SOURCES_PSK` is configured. When PSK mode is active, `AccountNumber` maps to `x-rh-sources-account-number` and `OrgId` maps to `x-rh-org-id` as separate headers. When PSK is absent, `AccountNumber` and `OrgId` are encoded together into a synthesized `x-rh-identity` base64 blob. Omitting any field silently drops that tenant identifier from outbound requests.

Reference sites where the client is constructed:
- `provider/forge.go`
- `superkey/create_request.go`
- `superkey/forged_application.go`

### 2. TenantID is the EBS account number, not the org-id

The JSON field `tenant_id` in `CreateRequest` and `DestroyRequest` is mapped to `SourcesClient.AccountNumber`. Do not confuse it with `OrgIdHeader`, which comes from the Kafka header `x-rh-sources-org-id`. When constructing a `SourcesClient`, always pass `request.TenantID` as `AccountNumber` and `request.OrgIdHeader` as `OrgId`.

### 3. Kafka message identity extraction requires both header checks

In `processSuperkeyRequest` (main.go), the worker reads two Kafka headers:
- `x-rh-identity` (base64-encoded XRHID JSON)
- `x-rh-sources-org-id` (plain org-id string)

The message is rejected only when BOTH are empty. Either one alone is sufficient to proceed. New Kafka consumers must replicate this dual-check pattern.

### 4. destroy_application path does not propagate identity headers

The `destroy_application` case in `processSuperkeyRequest` does NOT set `IdentityHeader` or `OrgIdHeader` on the `DestroyRequest` struct. The `ReconstructForgedApplication` function copies only `TenantID` from the destroy payload. Any code path triggered by `destroy_application` that needs to call the Sources API must rely solely on `TenantID` (which becomes `AccountNumber`) and cannot forward the original `x-rh-identity`.

### 5. PSK vs XRHID auth is a global toggle, not per-request

The `SourcesPSK` config value is read from the `SOURCES_PSK` environment variable at startup and stored in the singleton config. The `headers()` method on `SourcesClient` checks `conf.SourcesPSK == ""` to decide the auth strategy. You cannot mix PSK and non-PSK auth for different API calls within the same process.

## Header Mapping Reference

| Kafka Header | SourcesClient Field | Outbound Header (no PSK) | Outbound Header (PSK mode) |
|---|---|---|---|
| `x-rh-identity` | `IdentityHeader` | `x-rh-identity` (passthrough) | `x-rh-identity` (if present) |
| `x-rh-sources-org-id` | `OrgId` | Encoded into `x-rh-identity` | `x-rh-org-id` |
| (from JSON `tenant_id`) | `AccountNumber` | Encoded into `x-rh-identity` | `x-rh-sources-account-number` |
| N/A | N/A | N/A | `x-rh-sources-psk` (from env) |

## PSK Mode Header Assembly Rules

When `SOURCES_PSK` is set, `headers()` always emits `x-rh-sources-psk`. Additionally:
- `x-rh-sources-account-number` is set only if `AccountNumber != ""`
- `x-rh-identity` is forwarded only if `IdentityHeader != ""`
- `x-rh-org-id` is set only if `OrgId != ""`

When `SOURCES_PSK` is empty, `headers()` synthesizes a single `x-rh-identity` header:
- If `IdentityHeader` is already set, it is used as-is (passthrough from Kafka)
- If `IdentityHeader` is empty, `encodeIdentity(AccountNumber, OrgId)` builds a new base64 XRHID

## XRHID Encoding

Two separate XRHID struct definitions exist in this repo:

1. **`sources/types.go`** -- Minimal `XRhIdentity` struct used only for *decoding*. Contains only `identity.account_number`. The `parseXRhIdentity`/`getAccountNumber` functions use this. Note: this struct does NOT include `org_id`.

2. **`sources/helpers.go`** -- Uses `identity.XRHID` and `identity.Identity` from `platform-go-middlewares` for *encoding*. Sets both `AccountNumber` and `OrgID` into the full platform struct.

When decoding an inbound `x-rh-identity`, the local minimal struct silently drops `org_id`. If you need to extract org-id from the identity header, use the platform middleware struct instead.

## Internal Authentication Path

`GetInternalAuthentication` calls `/internal/v2.0/authentications/{id}?expose_encrypted_attribute[]=password`. This endpoint:
- Uses the same `headers()` method, so it respects the PSK/XRHID toggle
- Has a 5-retry loop with 3-second sleep between attempts
- Returns username + password for the SuperKey credential
- Is the only call that hits the internal (non-public) Sources API path

## Context Propagation

Tenant identity flows through `context.Context` for logging only, not for auth:
- `logger.WithTenantId(ctx, req.TenantID)` stores tenant_id in context
- `logger.LogWithContext(ctx)` extracts `tenant_id`, `source_id`, `application_id`, `application_type` into logrus fields
- Auth identity (`IdentityHeader`, `OrgIdHeader`) is stored on the request struct, NOT on the context
- The `destroy_application` path only sets `tenant_id` in context (no source_id or application_id)

## Common Mistakes

- **Passing OrgIdHeader as AccountNumber**: These are different identifiers from different Kafka headers. `TenantID` (from JSON body) maps to `AccountNumber`. `OrgIdHeader` (from Kafka header `x-rh-sources-org-id`) maps to `OrgId`.
- **Assuming x-rh-identity always exists**: The worker proceeds with only `x-rh-sources-org-id` present. In PSK mode, `x-rh-identity` may be empty.
- **Using the local XRhIdentity struct to extract org-id**: It only has `account_number`. Use `identity.XRHID` from platform-go-middlewares.
- **Adding identity to context for auth**: Context carries identity for logging only. Auth credentials must be on `SourcesClient` fields or request structs.

## Verification

```bash
# Confirm all SourcesClient instantiations set all three fields
grep -rn 'sources.SourcesClient{' $(find . -name '*.go') | grep -v vendor

# Check that destroy_application does not set IdentityHeader/OrgIdHeader
grep -A5 'destroy_application' main.go

# Verify PSK toggle references are consistent
grep -rn 'SourcesPSK' --include='*.go'

# Verify Kafka header extraction covers both identity headers
grep -n 'GetHeader' main.go

# Check that encodeIdentity uses the platform middleware struct
grep -A5 'encodeIdentity' sources/helpers.go

# Confirm the deploy secret source for PSK
grep -A3 'SOURCES_PSK' deploy/clowdapp.yaml
```
