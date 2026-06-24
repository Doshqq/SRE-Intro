# Lab 4 — Kubernetes: Deploy QuickTicket to a Cluster

## Task 1 — Write Manifests & Deploy to k3d

### 1. Output of `kubectl get nodes`

```
NAME                       STATUS   ROLES           AGE   VERSION
k3d-quickticket-server-0   Ready    control-plane   11m   v1.35.5+k3s1
```

### 2. Output of `kubectl get pods,svc` showing all running

```
NAME                           READY   STATUS    RESTARTS   AGE
pod/events-6c4df7d6-hj2rp      1/1     Running   0          5m37s
pod/gateway-6fc44f68c5-fggjz   1/1     Running   0          65s
pod/payments-58fb468db-k47c6   1/1     Running   0          5m37s
pod/postgres-7c7ffc4b-lnhks    1/1     Running   0          5m36s
pod/redis-c46d5dffc-crh8d      1/1     Running   0          5m36s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/events       ClusterIP   10.43.218.40    <none>        8081/TCP   5m37s
service/gateway      ClusterIP   10.43.10.136    <none>        8080/TCP   5m37s
service/kubernetes   ClusterIP   10.43.0.1       <none>        443/TCP    11m
service/payments     ClusterIP   10.43.121.52    <none>        8082/TCP   5m37s
service/postgres     ClusterIP   10.43.230.254   <none>        5432/TCP   5m36s
service/redis        ClusterIP   10.43.117.134   <none>        6379/TCP   5m36s
```

### 3. Output of `curl localhost:3080/events` via port-forward (proving the full stack works)

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

### 4. Output of pod deletion and auto-recovery

**Before deletion:**
```
NAME                       READY   STATUS    RESTARTS   AGE
events-6c4df7d6-hj2rp      1/1     Running   0          5m5s
gateway-6fc44f68c5-wgctb   1/1     Running   0          5m5s
payments-58fb468db-k47c6   1/1     Running   0          5m5s
postgres-7c7ffc4b-lnhks    1/1     Running   0          5m4s
redis-c46d5dffc-crh8d      1/1     Running   0          5m4s
```

**After deletion:**
```
pod "gateway-6fc44f68c5-wgctb" deleted
```

**After auto-recovery:**
```
NAME                       READY   STATUS    RESTARTS   AGE
events-6c4df7d6-hj2rp      1/1     Running   0          5m37s
gateway-6fc44f68c5-fggjz   1/1     Running   0          33s
payments-58fb468db-k47c6   1/1     Running   0          5m37s
postgres-7c7ffc4b-lnhks    1/1     Running   0          5m36s
redis-c46d5dffc-crh8d      1/1     Running   0          5m36s
```

### 5. Answer: "How long did K8s take to recreate the deleted pod? How does this compare to docker-compose restart?"

Kubernetes took approximately 33 seconds to recreate the deleted gateway pod. This is significantly faster than docker-compose restart, which typically requires manual intervention (`docker compose start` or `docker compose restart`) and doesn't have automatic self-healing capabilities. With Kubernetes, the Deployment controller automatically detects when a pod is deleted and creates a replacement to maintain the desired replica count, without any manual intervention required. This is a key advantage of Kubernetes over docker-compose - it provides built-in self-healing and maintains the desired state automatically.

## Task 2 — Probes & Resource Limits

### 1. Output of `kubectl describe pod` showing probes configured

```
Liveness:   http-get http://:8080/health delay=10s timeout=1s period=10s #success=1 #failure=3
Readiness:  http-get http://:8080/health delay=0s timeout=1s period=5s #success=1 #failure=2
```

### 2. Output during Redis deletion showing readiness probe failure

```
Warning  Unhealthy  2m26s (x3 over 2m28s)  kubelet            Readiness probe failed: Get "http://10.42.0.16:8081/health": dial tcp 10.42.0.16:8081: connect: connection refused
Warning  Unhealthy  100s                   kubelet            Readiness probe failed: Get "http://10.42.0.16:8081/health": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
```

The events pod showed readiness probe failures when Redis was deleted, as expected. The pod was marked as not ready (0/1 Ready) during the Redis outage, which would have removed it from the Service endpoints, preventing traffic from being routed to it.

### 3. Output of `kubectl describe node` showing allocated resources

```
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                350m (4%)   600m (7%)
  memory             332Mi (8%)  938Mi (23%)
  ephemeral-storage  0 (0%)      0 (0%)
  hugepages-1Gi      0 (0%)      0 (0%)
  hugepages-2Mi      0 (0%)      0 (0%)
  hugepages-32Mi     0 (0%)      0 (0%)
  hugepages-64Ki     0 (0%)      0 (0%)
```

### 4. Answer: "What's the difference between liveness and readiness probe failure? Which one should you use for checking database connectivity, and why?"

**Difference between liveness and readiness probe failure:**
- **Liveness probe failure**: When a liveness probe fails, Kubernetes kills and restarts the pod. This is designed to detect when a pod is in a broken state that can be fixed by restarting it.
- **Readiness probe failure**: When a readiness probe fails, Kubernetes marks the pod as not ready (removes it from Service endpoints) but does not restart it. Traffic is no longer routed to the pod, but the pod continues running.

**For database connectivity:**
You should use **readiness probes**, not liveness probes, for checking database connectivity. If the database is down, restarting the pod won't fix the problem - the database will still be down. Using a readiness probe will stop traffic to the pod (preventing errors for users) while allowing the pod to continue running and automatically recover when the database comes back online. Using a liveness probe would cause the pod to be killed and restarted repeatedly in a crash loop, which is wasteful and doesn't solve the underlying issue.

## Bonus Task — Helm Chart

### 1. Chart.yaml

```yaml
apiVersion: v2
name: quickticket
description: QuickTicket SRE learning project
version: 0.1.0
```

### 2. values.yaml

```yaml
gateway:
  replicas: 1
  image: quickticket-gateway:v1
  events:
    replicas: 1
    image: quickticket-events:v1
  db:
    host: postgres
    port: 5432
    name: quickticket
    user: quickticket
    password: quickticket
payments:
  replicas: 1
  image: quickticket-payments:v1
  failureRate: "0.0"
  latencyMs: "0"
```

Templates check in `k8s/chart/templates/` directory.

### 3. Output of `helm list` showing the installed release

```
NAME       	NAMESPACE	REVISION	UPDATED                             	STATUS  	CHART            	APP VERSION
quickticket	default  	1       	2026-06-19 15:00:06.135233 +0300 MSK	deployed	quickticket-0.1.0
```

### 4. Output of `kubectl get pods` after Helm install

```
NAME                       READY   STATUS    RESTARTS   AGE
events-675d86c77-8ww5l     1/1     Running   0          44s
gateway-7cd55d8774-tflqt   1/1     Running   0          44s
payments-d7dc94485-b4zxb   1/1     Running   0          44s
postgres-7c7ffc4b-psnpw    1/1     Running   0          44s
redis-c46d5dffc-dts89      1/1     Running   0          44s
```

All 5 pods are running successfully after Helm installation. The Helm chart was created with templates for all services (gateway, events, payments, postgres, redis) with configurable values for replicas, images, and environment variables.
