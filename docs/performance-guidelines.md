# Performance Guidelines — sources-superkey-worker

## P1: AWS_WAIT_TIME sleep blocks the single consumer goroutine

The `CreateInSourcesAPI` path calls `time.Sleep(waitTime())` to work around an AWS IAM eventual-consistency race condition. Because Kafka messages are consumed **synchronously in a single goroutine** (no worker pool, no concurrency limit), this sleep blocks *all* message processing for the duration.

- **Default is 15 seconds** when `AWS_WAIT_TIME` is set in `deploy/clowdapp.yaml`, but the code-level constant `DEFAULT_SLEEP_TIME` in `superkey/forged_application.go` is **7 seconds**. A deploy without the env var falls back to 7s silently — verify which value is active in each environment.
- Never raise `AWS_WAIT_TIME` above 20 seconds without measuring queue lag. Each additional second directly adds to per-message latency and causes backpressure across the 3-partition topic.
- If you must change the default, update **both** the `DEFAULT_SLEEP_TIME` constant in `superkey/forged_application.go` *and* the parameter default in `deploy/clowdapp.yaml` to keep them consistent.

## P2: Synchronous message processing — no concurrency, no back-pressure control

`main.go` runs a single `kafka.Consume` loop inside one goroutine. Each message is fully processed (AWS API calls, Sources API calls, sleep) before the next message is read. There is no worker pool, no buffered channel fan-out, and no concurrency limit.

- Do **not** add a naive `go processSuperkeyRequest(msg)` without also adding a concurrency limiter (e.g., semaphore channel). Unbounded goroutine spawning would create uncontrolled AWS API call rates and risk IAM throttling.
- If introducing concurrency, ensure the Kafka consumer's commit strategy still guarantees at-least-once delivery. The current synchronous model provides this implicitly.
- The topic has **3 partitions** (`deploy/clowdapp.yaml`). Scaling beyond 3 replicas (`MIN_REPLICAS`) provides no additional parallelism because Kafka assigns at most one consumer per partition within a consumer group.

## P3: GetInternalAuthentication retry loop has fixed 3-second sleep, no exponential backoff

`sources/api_client.go` `GetInternalAuthentication` retries up to 5 times with a hardcoded `time.Sleep(3 * time.Second)` between attempts. Worst-case: **15 seconds of blocking** (plus the AWS_WAIT_TIME sleep that follows).

- Do not increase the retry count without adding exponential backoff. Five retries at 3s each already risks 15s of stalled processing.
- The retry triggers on any non-200 status code, including 4xx client errors that will never succeed on retry. If modifying this code, distinguish retryable (5xx, network) from non-retryable (4xx) responses.

## P4: All HTTP calls use http.DefaultClient — no timeouts, no connection pooling control

Every Sources API call in `sources/api_client.go` uses `http.DefaultClient`, which has **no request timeout**. A hung Sources API response will block the consumer goroutine indefinitely.

- If adding an `http.Client` with a `Timeout`, set it to a value shorter than the Kafka consumer session timeout to avoid rebalance storms.

## P5: AWS SDK calls use context.Background() — no cancellation or timeout propagation

Every AWS API call in `amazon/iam.go`, `amazon/s3.go`, and `amazon/costandusagereports.go` creates a fresh `context.Background()` instead of propagating the request context from the Kafka message handler.

- If adding timeouts or cancellation (e.g., on SIGTERM), you must thread the parent `ctx` from `processSuperkeyRequest` through `provider.Forge` / `provider.TearDown` into each AWS SDK call. Currently, a SIGTERM during a multi-step forge will not cancel in-flight AWS operations.
- The `ForgeApplication` method in `provider/amazon_provider.go` already receives `ctx` but does not pass it to AWS SDK methods. Fix the SDK calls before relying on context cancellation.

## P6: S3 bucket deletion is not paginated

`amazon/s3.go` `DestroyS3Bucket` calls `ListObjects` once and deletes each object sequentially. `ListObjects` returns a maximum of 1000 keys per call.

- If a cost-reporting bucket accumulates more than 1000 objects, `DestroyS3Bucket` will leave the bucket non-empty and the `DeleteBucket` call will fail. Use `ListObjectsV2` with pagination, or the `s3.NewListObjectsV2Paginator`.
- Object deletion is sequential (one API call per object). For buckets with many objects, use `DeleteObjects` (batch delete, up to 1000 keys per call).

## P7: Resource limits are very tight

`deploy/clowdapp.yaml` defaults:

| Resource | Request | Limit |
|----------|---------|-------|
| CPU      | 20m     | 50m   |
| Memory   | 50Mi    | 100Mi |

- The 50m CPU limit can cause throttling during TLS handshakes with AWS APIs, especially when multiple AWS service clients are initialized in `NewClient`.
- 100Mi memory is adequate for the current single-message-at-a-time model. If you add concurrency or buffering, re-measure with `go tool pprof` under load.
- The `MIN_REPLICAS` default is **1**. There is no pod disruption budget; a rolling deploy will drop in-flight messages.

## P8: Graceful shutdown does not wait for in-flight work

`main.go` receives SIGTERM, closes the Kafka reader, and calls `os.Exit(0)` immediately. If a `create_application` message is mid-processing (e.g., after creating an IAM role but before binding the policy), the partial resources are orphaned in AWS.

- Do not add a `select` with a timeout on shutdown without also ensuring the `TearDown` path runs for partially forged applications.

## Verification

```bash
# Confirm AWS_WAIT_TIME is set in the deployment manifest
grep -n "AWS_WAIT_TIME" deploy/clowdapp.yaml

# Confirm DEFAULT_SLEEP_TIME constant matches expectations
grep -n "DEFAULT_SLEEP_TIME" superkey/forged_application.go

# Find all time.Sleep calls (each one blocks the consumer)
grep -rn "time.Sleep" --include="*.go" .

# Find all uses of http.DefaultClient (no timeout enforcement)
grep -rn "http.DefaultClient" --include="*.go" .

# Find all context.Background() in AWS SDK calls (no cancellation)
grep -rn "context.Background()" amazon/ --include="*.go"

# Verify retry parameters in GetInternalAuthentication
grep -A5 "for retry" sources/api_client.go

# Check resource limits and replica count
grep -A2 'limits:\|requests:\|MIN_REPLICAS' deploy/clowdapp.yaml

# Confirm S3 ListObjects is not paginated
grep -B2 -A5 "ListObjects" amazon/s3.go
```
