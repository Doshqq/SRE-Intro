# Lab 3 Submission

## Task 1 — Configure Monitoring & Build Dashboard (6 pts)

### 3.1: Docker Compose PS Output

```
NAME               IMAGE                     COMMAND                  SERVICE      CREATED          STATUS                    PORTS
app-events-1       app-events                "uvicorn main:app --…"   events       15 seconds ago   Up 7 seconds              0.0.0.0:8081->8081/tcp
app-gateway-1      app-gateway               "uvicorn main:app --…"   gateway      14 seconds ago   Up 7 seconds              0.0.0.0:3080->8080/tcp
app-grafana-1      grafana/grafana:13.0.1    "/run.sh"                grafana      15 seconds ago   Up 13 seconds             0.0.0.0:3000->3000/tcp
app-payments-1     app-payments              "uvicorn main:app --…"   payments     15 seconds ago   Up 13 seconds             0.0.0.0:8082->8082/tcp
app-postgres-1     postgres:17-alpine        "docker-entrypoint.s…"   postgres     7 days ago       Up 13 seconds (healthy)   0.0.0.0:5432->5432/tcp
app-prometheus-1   prom/prometheus:v3.11.2   "/bin/prometheus --c…"   prometheus   15 seconds ago   Up 13 seconds             0.0.0.0:9090->9090/tcp
app-redis-1        redis:7-alpine            "docker-entrypoint.s…"   redis        7 days ago       Up 13 seconds (healthy)   0.0.0.0:6379->6379/tcp
```

### 3.2: Prometheus Targets Output

```
events       up       http://events:8081/metrics
gateway      up       http://gateway:8080/metrics
payments     up       http://payments:8082/metrics
```

### 3.3: Custom Metrics List

```
events_db_pool_size
events_orders_created
events_orders_total
events_request_duration_seconds_bucket
events_request_duration_seconds_count
events_request_duration_seconds_created
events_request_duration_seconds_sum
events_requests_created
events_requests_total
events_reservations_active
gateway_request_duration_seconds_bucket
gateway_request_duration_seconds_count
gateway_request_duration_seconds_created
gateway_request_duration_seconds_sum
gateway_requests_created
gateway_requests_total
payments_charges_created
payments_charges_total
payments_request_duration_seconds_bucket
payments_request_duration_seconds_count
payments_request_duration_seconds_created
payments_request_duration_seconds_sum
payments_requests_created
payments_requests_total
```

### 3.4: PromQL Query Output (Request Rate)

```
Request rate: 0.23 req/s
```

### 3.5: PromQL Queries for Dashboard Panels

**Latency panel (Golden Signal #1):**
- p50: `histogram_quantile(0.50, sum(rate(gateway_request_duration_seconds_bucket[1m])) by (le))`
- p95: `histogram_quantile(0.95, sum(rate(gateway_request_duration_seconds_bucket[1m])) by (le))`
- p99: `histogram_quantile(0.99, sum(rate(gateway_request_duration_seconds_bucket[1m])) by (le))`
- Unit: seconds

**Saturation panel (Golden Signal #4):**
- Query: `events_db_pool_size`
- Visualization: Gauge
- Min: 0, Max: 10
- Thresholds: green default, yellow at 7, red at 9

### 3.6: Dashboard Observations

**Normal traffic:**
- Request rate: ~0.2-0.5 req/s
- Error rate: 0%
- All services healthy
- Latency: low (p99 < 100ms)
- DB pool size: stable at low values

**Payments failure (stopped payments service):**
- Error rate spiked from 0% to ~12-15% immediately after killing payments
- Error rate remained elevated (~12-15%) for the duration of the failure
- Request rate continued but payment-related requests failed
- Service health showed payments as down
- After restarting payments, error rate gradually returned to 0%

### 3.7: Which Golden Signal Showed Failure First?

**Error Rate** showed the failure first. The error rate spiked from 0% to ~12-15% within 10-15 seconds after killing the payments service. This was the first and most obvious indicator of the failure, as payment-related requests immediately started failing with 502/503 errors.

---

## Task 2 — Define SLOs & Recording Rules (4 pts)

### 3.8: SLI/SLO Definitions with Error Budget Math

**SLI 1 — Availability:** % of gateway requests returning non-5xx
- SLO target: 99.5% over a 7-day window
- Error budget: 100% - 99.5% = 0.5%
- With ~1000 requests/day = 7000 requests/week
- Error budget allows: 7000 × 0.005 = 35 failures per week

**SLI 2 — Latency:** % of gateway requests completing under 500ms
- SLO target: 95%
- Error budget: 100% - 95% = 5%
- With ~1000 requests/day = 7000 requests/week
- Error budget allows: 7000 × 0.05 = 350 slow requests per week

### 3.9: Rules Loaded Output

```
gateway:sli_availability:ratio_rate5m         = ok
gateway:sli_latency_500ms:ratio_rate5m        = ok
gateway:error_budget_burn_rate:ratio_rate5m   = ok
```

### 3.10: SLO Panel Query

**SLO Gauge panel:**
- Query: `gateway:sli_availability:ratio_rate5m * 100`
- Visualization: Gauge
- Min: 99, Max: 100
- Threshold: 99.5 (SLO target)

**Observation during failure:**
- During normal operation: SLO gauge shows 100% availability
- When payments service was killed: SLO gauge dropped to ~85-88% (reflecting the 12-15% error rate)
- After restarting payments: SLO gauge gradually returned to 100% as error rate normalized

---

## Recording Rules Configuration

The recording rules are defined in `monitoring/prometheus/rules.yml`:

```yaml
groups:
  - name: slo_rules
    interval: 30s
    rules:
      - record: gateway:sli_availability:ratio_rate5m
        expr: |
          sum(rate(gateway_requests_total{status!~"5.."}[5m]))
          /
          sum(rate(gateway_requests_total[5m]))

      - record: gateway:sli_latency_500ms:ratio_rate5m
        expr: |
          sum(rate(gateway_request_duration_seconds_bucket{le="0.5"}[5m]))
          /
          sum(rate(gateway_request_duration_seconds_count[5m]))

      - record: gateway:error_budget_burn_rate:ratio_rate5m
        expr: |
          (1 - gateway:sli_availability:ratio_rate5m)
          /
          (1 - 0.995)
```

---

## Prometheus Configuration

The Prometheus configuration is defined in `monitoring/prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "rules.yml"

scrape_configs:
  - job_name: 'gateway'
    static_configs:
      - targets: ['gateway:8080']

  - job_name: 'events'
    static_configs:
      - targets: ['events:8081']

  - job_name: 'payments'
    static_configs:
      - targets: ['payments:8082']
```

## Bonus Task — Correlation Report

### 1. Timeline
* **[22:46:24]** — Load generator started (`./loadgen/run.sh 5 120 &`). Initial traffic is healthy with an `error_rate=0%`.
* **[22:46:59]** — Failure injection. Executed the command to restart the `payments` service with `PAYMENT_FAILURE_RATE=0.5` and `PAYMENT_LATENCY_MS=1000`.
* **[22:47:01]** — First error recorded in `gateway` logs, triggered by a downstream service failure.
* **[22:47:02]** — First signs of artificial latency injection appear in the `payments` service logs.
* **[22:47:30]** — Grafana dashboard reflects a sharp spike in both **HTTP Error Rate** and **Request Latency** metrics.
* **[22:48:23]** — The load generator script completes its 120-second run. System traffic drops back to zero (idle state), though the `payments` container remains in its degraded configuration.

---

### 2. Log Excerpts

#### Gateway Logs (Moment of first error):

```text
gateway-1  | 2026-06-18T22:47:01.283924549Z {"time":"2026-06-18 22:47:01,283","level":"INFO","service":"gateway","msg":"HTTP Request: POST http://payments:8082/charge "HTTP/1.1 500 Internal Server Error""}
gateway-1  | 2026-06-18T22:47:01.289001008Z INFO:     192.168.65.1:61106 - "POST /reserve/7874d23c-9692-443a-adf8-45ac28c93e28/pay HTTP/1.1" 500 Internal Server Error
```

#### Payments Logs (Moment of failure & latency injection):

```text
payments-1  | 2026-06-18T22:47:02.656568008Z {"time":"2026-06-18 22:47:02,656","level":"INFO","service":"payments","msg":"Injecting 1000ms latency for 10d85975-0d26-419c-87c7-290b7be7705b"}
payments-1  | 2026-06-18T22:47:03.658587467Z {"time":"2026-06-18 22:47:03,658","level":"INFO","service":"payments","msg":"Payment success: PAY-3E5F67AF for 10d85975-0d26-419c-87c7-290b7be7705b"}
```

### 3. Root Cause Explanation

#### Connecting Metrics to Logs
The root cause of the system performance degradation is the manual injection of environment variables into the `payments` service, which immediately disrupted the upstream `gateway` service during the full checkout flow `(/reserve/:id/pay)`.

**Error Rate Spike**: At `22:47:01`, the `gateway` log captures a `500 Internal Server Error` response directly from `http://payments:8082/charge`. Because the `PAYMENT_FAILURE_RATE` was set to `0.5`, the payment service began failing exactly half of the processing attempts, forcing the API gateway to pass these 5xx errors down to the load generator clients.

**Latency Spike**: At `22:47:02`, the `payments` log explicitly confirms that it started adding an artificial delay (`Injecting 1000ms latency...`). Even when a transaction eventually succeeded at `22:47:03`, it took exactly **1 second** longer than usual. This massive hop in execution time degraded the gateway's total response time, causing a matching upward surge on the Grafana latency charts.

#### Observability Delay Explanation

There is a noticeable ~29-second gap between the first log error (`22:47:01`) and the visible spike on the Grafana dashboard (`22:47:30`). This delay is a standard characteristic of metric collection systems rather than an application delay. It is caused by:

**Prometheus Scrape Interval**: Prometheus polls (scrapes) metrics endpoints from containers at fixed intervals (every 5 seconds).

**Prometheus Time-Series Aggregation**: Functions like `rate()` or `sum()` over time used in Grafana panels require a window of data points to calculate and render a trend line, resulting in a slight delay before the visual spike peaks on the dashboard.
