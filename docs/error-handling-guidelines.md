# Error Handling Guidelines

## 1. Kafka Message Errors: Always Log and Skip

This worker has no dead-letter queue. When a Kafka message cannot be processed (missing headers, parse failure, unknown event type), log the error with `org_id` and `return` to advance the consumer offset. Never panic, never block the consumer loop.

```go
// CORRECT: log with org_id field, then return to skip the message
l.Log.WithFields(logrus.Fields{"org_id": orgIdHeader}).Errorf(`Error parsing request "%s": %s`, string(msg.Value), err)
return

// WRONG: calling log.Fatal (kills the entire worker over one bad message)
```

A skipped message is gone forever. Include the raw `msg.Value` in the log line so operators can manually replay it.

## 2. TearDown Collects All Errors Without Short-Circuiting

`Provider.TearDown` returns `[]error`, not a single error. Every cleanup step (unbind role, destroy policy, destroy role, destroy cost report, destroy S3 bucket) runs regardless of whether earlier steps failed. Append each failure to the slice; never `return` early from TearDown.

```go
// CORRECT pattern (from provider/amazon_provider.go):
errors := make([]error, 0)
if f.StepsCompleted["policy"] != nil {
    err := a.Client.DestroyPolicy(policyArn)
    if err != nil {
        errors = append(errors, fmt.Errorf(`failed to destroy policy "%s": %w`, policyArn, err))
    }
}
// ...continue to next step regardless...
return errors
```

The caller in `main.go` iterates and logs each error individually. Do not collapse the slice into a single joined error.

## 3. ForgeApplication Must Return Partial State on Error

`ForgeApplication` returns `(f, err)` where `f` is the partially-built `ForgedApplication`. The caller needs `f` even on failure to know which AWS resources to tear down via `StepsCompleted`. Never return `(nil, err)` after any step has been marked completed.

```go
// CORRECT: return f so TearDown knows what to clean up
return f, fmt.Errorf(`failed to create role "%s": %w`, name, err)

// WRONG: return nil, err — orphans any resources created in earlier steps
```

The only case where `nil` is acceptable is when no steps have executed yet (e.g., GUID generation failure).

## 4. Always Wrap Errors with %w and Add Context

Every `fmt.Errorf` in the codebase uses `%w` to preserve the original error for `errors.Is`/`errors.As` unwrapping. The context string must identify the specific resource that failed (name, ARN, ID).

Existing pattern — layer-specific prefixes:
- **amazon layer**: raw AWS SDK errors, returned unwrapped (no `fmt.Errorf`)
- **provider layer**: `"failed to create/destroy <resource> "<name>": %w"`
- **superkey layer**: `"error while <action> in Sources: %w"`
- **forge.go (provider init)**: `"unable to get provider: %w"`, `"error while fetching internal authentication \"<id>\" from Sources: %w"`

Do not add wrapping in the `amazon/` package itself; it returns raw SDK errors. Wrapping happens one layer up in `provider/amazon_provider.go`.

## 5. MarkSourceUnavailable Is the Escalation Path

When `Forge` fails, `createResources` in `main.go` follows a strict sequence:
1. Log the forge error
2. Call `TearDown` (clean up any partial AWS resources)
3. Call `req.MarkSourceUnavailable(ctx, err, newApp)` to PATCH the application and source as `"unavailable"` via the Sources API

`MarkSourceUnavailable` formats the original error into `availability_status_error` with the prefix `"Resource Creation error: failed to create resources in Amazon. Error: "`. It also persists the `_superkey` extra payload so operators can see how far resource creation progressed.

If `MarkSourceUnavailable` itself fails, log the error but do not retry. The source remains in whatever state it was in.

## 6. GetInternalAuthentication Retry: Fixed Interval, No Backoff

`SourcesClient.GetInternalAuthentication` retries up to 5 times with a fixed 3-second sleep between attempts. It only retries on non-200 status codes (not on transport errors — transport errors break immediately via the `err != nil` check in the loop condition).

Known subtlety: if the HTTP call returns a non-200 status **and** `err` is nil after all 5 retries, the final `fmt.Errorf` wraps a nil error with `%w`. This produces a message like `unable to fetch internal authentication "X" after 5 retries: <nil>`. Do not change the retry count or timing without coordinating with the Sources API team.

## 7. Sources API Client Error Patterns

All `SourcesClient` methods follow the same two-tier error pattern:
1. **Transport error**: `"unable to send request: %w"` — the HTTP call itself failed
2. **Status code error**: `expecting a 200 status code, got "%d" with body "%s"` — the API responded with an error; the response body is included for debugging

`CheckAvailability` is the exception: it expects 202 (not 200) and does not include the response body in the error message.

HTTP status errors are **not** wrapped with `%w` (they use `%d` and `%s`). This means `errors.Is` and `errors.As` will not work on status-code errors — they are terminal log-and-report errors, not errors you programmatically branch on.

## 8. Logging Conventions

- **Before context is built** (missing headers, parse failures): use `l.Log.WithFields(logrus.Fields{"org_id": orgIdHeader})` or `l.Log.WithFields(logrus.Fields{"kafka_message": ...})`
- **After context is built**: always use `l.LogWithContext(ctx)` which automatically includes `tenant_id`, `source_id`, `application_id`, and `application_type`
- **Error level**: use `.Errorf()` for failures that affect the current request but do not crash the worker
- **Fatal level**: reserved exclusively for startup failures (`Kafka reader creation`, `health file creation`). Never use `Fatal` during message processing
- **Warn level**: used only for retriable situations (`GetInternalAuthentication` retry loop)
- **Info level**: marks processing boundaries (`Processing "create_application"`, `Finished processing`, resource creation confirmations)

Use `%s` (not `%v`) when formatting errors in log messages — the `Error()` string is what you want, not the Go representation.

## 9. The amazon/ Package Never Logs

The `amazon/` package (IAM, S3, Cost Reporting) does not import the logger. It returns errors to the caller and never logs directly. The sole exception is `NewClient` in `types.go`, which logs an error for unsupported API names but continues without failing. Keep this boundary: AWS SDK interaction code stays silent; the `provider/` layer decides what to log.

## Verification

```bash
# Confirm all fmt.Errorf calls use %w for wrapping (except status-code errors which use %d/%s)
grep -rn 'fmt.Errorf' --include='*.go' | grep -v '%w' | grep -v '%d'

# Confirm amazon/ package does not import the logger (except types.go for NewClient)
grep -rn 'logger' amazon/ --include='*.go'

# Confirm no log.Fatal calls outside of main() startup
grep -rn '\.Fatal' --include='*.go' | grep -v 'main.go' | grep -v 'logger.go' | grep -v '_test.go' | grep -v 'config.go'

# Confirm ForgeApplication always returns f (not nil) after any MarkCompleted call
grep -A1 'MarkCompleted' provider/amazon_provider.go

# Confirm LogWithContext is used (not bare l.Log) inside createResources/destroyResources
grep -n 'l\.Log\.' main.go
```
