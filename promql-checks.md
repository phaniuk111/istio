# PromQL checks — run before instrumenting outbound calls

Run in Cloud Monitoring → Metrics Explorer (PromQL mode), or against the GMP query API
(see the bottom of this file). Replace `<NAMESPACE>` with the target namespace.

Each check gates a decision. **Run check 1 first** — if it returns data, checks 3–5 become
optional rather than blocking, because the mesh already answers the outbound question.

| # | Check | Decides |
|---|---|---|
| 1 | Istio metrics in GMP | Whether outbound needs any app change at all |
| 2 | Direct-export path | Whether a second, scrape-independent path exists |
| 3 | `http_client_requests` | Whether Spring-managed HTTP clients are in use |
| 4 | `http_server_requests` buckets | Whether latency percentiles are available |
| 5 | URI cardinality | Whether it is safe to turn buckets on |

---

## 1. Are Istio metrics already in GMP?

```promql
count({__name__=~"istio_.*"})
```

- **Returns a number** — mesh telemetry is already flowing through the merged `:15020` endpoint
  that PodMonitoring scrapes. Outbound calls are visible **today**, with no app change and no
  platform change. This also closes the gateway-rejected-requests blind spot in the same stroke.
- **Empty** — Istio Prometheus telemetry is not enabled. Needs a platform-team change via the
  Telemetry API, or fall back to app-level instrumentation (check 3).

If non-empty, follow up to see what is actually there:

```promql
count by (__name__) ({__name__=~"istio_.*"})

# outbound calls, as reported by the calling sidecar
sum by (destination_service_name, response_code) (
  rate(istio_requests_total{reporter="source", namespace="<NAMESPACE>"}[5m])
)
```

`reporter="source"` is the outbound view (this service calling out); `reporter="destination"`
is the inbound view. External HTTPS the app originates itself (BigQuery, Airflow over TLS) shows
as `PassthroughCluster` with TCP-only metrics — connection and byte counts, no response codes.

---

## 2. Direct-export path?

```promql
istio_io:service_server_request_count{monitored_resource="istio_canonical_service"}
```

- **Returns data** — ASM is exporting canonical-service metrics straight to Cloud Monitoring, a
  path independent of the PodMonitoring scrape. Useful as a cross-check, and it survives scrape
  misconfiguration.
- **Empty** — only the scraped path exists; check 1 is the one that matters.

---

## 3. Is outbound HTTP already instrumented?

```promql
http_client_requests_seconds_count{namespace="<NAMESPACE>"}
```

- **Returns data** — clients are built from the auto-configured builders
  (`RestTemplateBuilder` / `RestClient.Builder` / `WebClient.Builder`), so Spring Boot instruments
  them already. Dashboard panels can be added immediately, no code change.
- **Empty** — clients are hand-constructed (`new RestTemplate()`), which produces no metrics.
  Fix is a one-line swap to the injected builder per service. Note that JDK `HttpClient`, raw
  Apache HttpClient and bare OkHttp have no builder to swap to and stay invisible either way.

If non-empty, check tag quality **before** building panels:

```promql
count(count by (uri) (http_client_requests_seconds_count{namespace="<NAMESPACE>"}))
count by (clientName) (http_client_requests_seconds_count{namespace="<NAMESPACE>"})
```

A high or steadily growing `uri` count means URLs are built by string concatenation instead of
templates, so IDs are landing in the tag. Spring caps this at
`management.metrics.web.client.max-uri-tags` (default 100) and then stops recording new URI tags —
no error, just a dashboard that quietly goes wrong.

---

## 4. Latency percentiles available?

```promql
http_server_requests_seconds_bucket{namespace="<NAMESPACE>"}
```

- **Empty** (this was the previous result) — histogram buckets are off. Only avg and Micrometer's
  decaying ~2-minute max exist. Enable per service:

```yaml
management:
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
```

Then `histogram_quantile(0.95, sum by (le) (rate(http_server_requests_seconds_bucket[5m])))`
becomes available. Pilot on one service — bucket series multiply per URI, so run check 5 first.

---

## 5. URI cardinality (inbound)

```promql
count(count by (uri) (http_server_requests_seconds_count{namespace="<NAMESPACE>"}))
```

- **Low tens** — safe to enable histograms.
- **Hundreds or more** — path variables are landing in the `uri` tag as literals. Fix the URI
  templates first, or enabling buckets multiplies that number by roughly 15 (one series per `le`).

See which URIs dominate:

```promql
topk(20, count by (uri) (http_server_requests_seconds_count{namespace="<NAMESPACE>"}))
```

---

## Running these from a terminal instead of the UI

```bash
PROJECT=<PROJECT_ID>
QUERY='count({__name__=~"istio_.*"})'

curl -s -G \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  --data-urlencode "query=${QUERY}" \
  "https://monitoring.googleapis.com/v1/projects/${PROJECT}/location/global/prometheus/api/v1/query" \
  | jq '.data.result'
```

Swap `/query` for `/query_range` with `start`, `end` and `step` parameters for a time series
rather than an instant value.
