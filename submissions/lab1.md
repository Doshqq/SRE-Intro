# Lab 1 Submission

## Task 1 — Deploy & Break QuickTicket (6 pts)

### 1.1: Docker Compose PS Output

```
NAME             IMAGE                COMMAND                  SERVICE    CREATED          STATUS                    PORTS
app-events-1     app-events           "uvicorn main:app --…"   events     37 seconds ago   Up 31 seconds             0.0.0.0:8081->8081/tcp
app-gateway-1    app-gateway          "uvicorn main:app --…"   gateway    37 seconds ago   Up 31 seconds             0.0.0.0:3080->8080/tcp
app-payments-1   app-payments         "uvicorn main:app --…"   payments   37 seconds ago   Up 37 seconds             0.0.0.0:8082->8082/tcp
app-postgres-1   postgres:17-alpine   "docker-entrypoint.s…"   postgres   37 seconds ago   Up 37 seconds (healthy)   0.0.0.0:5432->5432/tcp
app-redis-1      redis:7-alpine       "docker-entrypoint.s…"   redis      37 seconds ago   Up 37 seconds (healthy)   0.0.0.0:6379->6379/tcp
```

### 1.2: Critical Path Output

**List Events:**
```json
[
    {
        "id": 1,
        "name": "Go Conference 2026",
        "venue": "Main Hall A",
        "date": "2026-09-15T09:00:00+00:00",
        "total_tickets": 100,
        "price_cents": 5000,
        "available": 100
    },
    {
        "id": 4,
        "name": "Python Workshop",
        "venue": "Lab 301",
        "date": "2026-09-22T14:00:00+00:00",
        "total_tickets": 25,
        "price_cents": 2000,
        "available": 25
    },
    {
        "id": 2,
        "name": "SRE Meetup",
        "venue": "Room 204",
        "date": "2026-10-01T18:00:00+00:00",
        "total_tickets": 30,
        "price_cents": 0,
        "available": 30
    },
    {
        "id": 5,
        "name": "Kubernetes Deep Dive",
        "venue": "Auditorium B",
        "date": "2026-10-10T10:00:00+00:00",
        "total_tickets": 80,
        "price_cents": 8000,
        "available": 80
    },
    {
        "id": 3,
        "name": "Cloud Native Summit",
        "venue": "Expo Center",
        "date": "2026-11-20T10:00:00+00:00",
        "total_tickets": 500,
        "price_cents": 15000,
        "available": 500
    }
]
```

**Reserve Ticket:**
```json
{
    "reservation_id": "5f459103-7a85-4e3f-b128-8910bf067df8",
    "event_id": 1,
    "quantity": 1,
    "total_cents": 5000,
    "expires_in_seconds": 300
}
```

**Pay for Reservation:**
```json
{
    "order_id": "5f459103-7a85-4e3f-b128-8910bf067df8",
    "event_id": 1,
    "quantity": 1,
    "total_cents": 5000,
    "status": "confirmed"
}
```

### 1.3: Health Check (All Services Healthy)

```json
{
    "status": "healthy",
    "checks": {
        "events": "ok",
        "payments": "ok",
        "circuit_payments": "CLOSED"
    }
}
```

### 1.4: Dependency Map

```
gateway → events → postgres
gateway → events → redis
gateway → payments
```

### 1.5: Failure Table

| Component Killed | Events List | Reserve | Pay | Health Check | User Impact |
|-----------------|-------------|---------|-----|--------------|-------------|
| payments        | Works       | Works   | Fails (502: Payment service unavailable) | Degraded (payments: down) | Users can browse events and reserve tickets, but cannot complete payment |
| events          | Fails (502: Events service unavailable) | Fails (502: Events service unavailable) | N/A | Degraded (events: down) | Complete outage - users cannot browse events, reserve, or pay |
| redis           | Works       | Fails (504: Events service timeout) | N/A | Degraded (events: down) | Users can browse events but cannot reserve tickets (reservation system unavailable) |
| postgres        | Fails (502: Events service unavailable) | Fails (500: Internal Server Error) | N/A | Degraded (events: degraded) | Complete outage - users cannot browse events, reserve, or pay |

### 1.6: Load Generator Output (Error Rate Spike When Payments Killed)

```
QuickTicket Load Generator
Target: http://localhost:3080 | RPS: 5 | Duration: 30s
---
[10s] requests=40 success=39 fail=1 error_rate=2.5%
[10s] requests=41 success=40 fail=1 error_rate=2.4%
[10s] requests=42 success=41 fail=1 error_rate=2.3%
[10s] requests=43 success=42 fail=1 error_rate=2.3%
[20s] requests=80 success=76 fail=4 error_rate=5.0%
[20s] requests=81 success=77 fail=4 error_rate=4.9%
[20s] requests=82 success=78 fail=4 error_rate=4.8%
[20s] requests=83 success=79 fail=4 error_rate=4.8%
---
Done. total=119 success=113 fail=6 error_rate=5.0%
```

The error rate spiked from 0% to 2.5% immediately after killing payments, then increased to 5% as more payment-related requests failed. This demonstrates the blast radius of the payments service failure.

---

## Task 2 — Graceful Degradation (3 pts)

### Gateway Code Diff

```diff
diff --git a/app/gateway/main.py b/app/gateway/main.py
index c86db33..4b6cc4a 100644
--- a/app/gateway/main.py
+++ b/app/gateway/main.py
@@ -332,6 +332,16 @@ async def pay_reservation(reservation_id: str):
     except CircuitOpenError:
         log.error("circuit open, skipping payments call")
         raise HTTPException(503, "Payment service temporarily unavailable (circuit open)")
+    except httpx.ConnectError:
+        log.error("payments service connection failed")
+        return JSONResponse(
+            status_code=503,
+            content={
+                "error": "payments_unavailable",
+                "message": "Payment service is temporarily down. Your reservation is held — try again in a few minutes.",
+                "reservation_id": reservation_id
+            }
+        )
     except httpx.TimeoutException:
         raise HTTPException(504, "Payment service timeout")
     except httpx.HTTPStatusError as e:
```

### Verification Output

**Reserve (works when payments is down):**
```json
{
    "reservation_id": "557009cf-ff25-4b85-a138-1a306a51b4f6",
    "event_id": 1,
    "quantity": 1,
    "total_cents": 5000,
    "expires_in_seconds": 300
}
```

**Pay (returns clear 503 with actionable message):**
```json
{
    "error": "payments_unavailable",
    "message": "Payment service is temporarily down. Your reservation is held — try again in a few minutes.",
    "reservation_id": "557009cf-ff25-4b85-a138-1a306a51b4f6"
}
```

---

## Task 3 — GitHub Community Engagement (1 pt)

### GitHub Community

Starring repositories is important in open source because it helps bookmark interesting projects for later reference, indicates project popularity, trends and community trust, appears in your GitHub profile showing your interests, and encourages maintainers by helping projects gain visibility.

Following developers helps in team projects and professional growth because you can see what other developers are working on, discover new projects through their activity, build professional connections beyond the classroom, and stay updated on classmates' work for collaboration.

p.s. follow also on me: https://github.com/Doshqq 

---

## Bonus Task — Resource Usage Under Load (2 pts)

### B.1: Baseline (idle)

| Service      | CPU %  | MEM USAGE / LIMIT     | NET I/O          | PIDS |
|--------------|--------|-----------------------|------------------|------|
| app-gateway-1| 0.33%  | 38.25MiB / 3.828GiB   | 8.66kB / 7.88kB  | 2    |
| app-events-1 | 0.26%  | 41.09MiB / 3.828GiB   | 10.4kB / 8.53kB  | 2    |
| app-payments-1| 0.36% | 33.05MiB / 3.828GiB   | 1.06kB / 264B    | 1    |
| app-postgres-1| 3.80%  | 23.61MiB / 3.828GiB   | 270kB / 308kB    | 8    |
| app-redis-1   | 1.01%  | 9.902MiB / 3.828GiB   | 79.1kB / 32.4kB  | 6    |

### B.2: Under load (10 RPS for 30s)

| Service      | CPU %  | MEM USAGE / LIMIT     | NET I/O          | PIDS |
|--------------|--------|-----------------------|------------------|------|
| app-gateway-1| 4.41%  | 38.67MiB / 3.828GiB   | 212kB / 207kB   | 2    |
| app-events-1 | 2.40%  | 41.48MiB / 3.828GiB   | 189kB / 248kB   | 2    |
| app-payments-1| 0.38% | 33.77MiB / 3.828GiB   | 5.99kB / 3.75kB  | 2    |
| app-postgres-1| 0.60%  | 23.79MiB / 3.828GiB   | 372kB / 424kB   | 8    |
| app-redis-1   | 1.18%  | 9.902MiB / 3.828GiB   | 102kB / 42.2kB  | 6    |

### B.3: Under stress with fault injection (30% failure rate, 500ms latency)

| Service      | CPU %  | MEM USAGE / LIMIT     | NET I/O          | PIDS |
|--------------|--------|-----------------------|------------------|------|
| app-payments-1| 0.92% | 35.23MiB / 3.828GiB   | 4.89kB / 3.38kB  | 2    |
| app-gateway-1| 2.51%  | 38.75MiB / 3.828GiB   | 454kB / 447kB   | 2    |
| app-events-1 | 1.53%  | 41.55MiB / 3.828GiB   | 398kB / 523kB   | 2    |
| app-postgres-1| 2.30%  | 23.99MiB / 3.828GiB   | 489kB / 561kB   | 8    |
| app-redis-1   | 1.03%  | 9.652MiB / 3.828GiB   | 124kB / 51.8kB  | 6    |

### Analysis

**Memory Usage:**
- The events service uses the most memory consistently across all scenarios (~41-42 MiB), followed by gateway (~38-39 MiB) and payments (~33-35 MiB). This pattern doesn't change significantly under load or fault injection, indicating memory usage is primarily driven by the application code and connection pools rather than request volume.

**CPU Usage:**
- Under normal load, gateway uses the most CPU (4.41%) as it routes all requests and handles the circuit breaker logic. Events service uses moderate CPU (2.40%) for database queries and Redis operations.
- Under fault injection, gateway CPU decreases to 2.51% while payments CPU increases slightly to 0.92%. The overall CPU usage is lower because the 500ms injected latency causes requests to complete more slowly, reducing the total request throughput (161 requests vs 206 in normal load).

**Fault Injection Impact on Gateway:**
- The 500ms latency injection in payments causes gateway to hold connections longer while waiting for payments to respond. This is reflected in the increased network I/O for gateway (454kB / 447kB under chaos vs 212kB / 207kB under normal load) as connections remain open longer. The gateway's CPU usage actually decreases under fault injection because fewer requests complete successfully per second due to the latency, reducing the overall processing load despite longer-held connections.
