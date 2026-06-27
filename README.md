# Kubernetes Monitoring Cluster

A self-contained observability stack for Kubernetes. It collects **metrics** with Prometheus, **logs** with Loki + Promtail, and presents both in **Grafana** through pre-provisioned datasources and a ready-made dashboard.

A sample application stack (`test-app` + Postgres + Redis) is included so you can see real metrics and logs flowing end-to-end.

---

## Table of contents

1. [Architecture](#architecture)
2. [How each component works](#how-each-component-works)
3. [Data flow, end to end](#data-flow-end-to-end)
4. [Namespaces, ports & storage](#namespaces-ports--storage)
5. [Prerequisites](#prerequisites)
6. [Setup — step by step](#setup--step-by-step)
7. [Accessing the UIs](#accessing-the-uis)
8. [Using the dashboard](#using-the-dashboard)
9. [File layout](#file-layout)
10. [Troubleshooting](#troubleshooting)
11. [Tear down](#tear-down)
12. [Production considerations](#production-considerations)

---

## Architecture

The cluster is split into **two namespaces** with a clear separation of concerns:

- **`appspace`** — the workload being observed (`test-app`, Postgres, Redis).
- **`monitoring`** — the observability platform (Prometheus, Loki, Promtail, Grafana).

The monitoring namespace *pulls* metrics and *receives* logs from the app namespace. The app namespace knows nothing about monitoring — it just exposes a `/metrics` endpoint and writes to stdout/stderr like any normal container. This is the key design idea: **observability is bolted on from the outside, not baked into the app.**

```
┌──────────────────────────────────────────────────────────────────────────┐
│                            Kubernetes Cluster                              │
│                                                                            │
│   ┌───────────────────────────┐         ┌──────────────────────────────┐  │
│   │     namespace: appspace   │         │    namespace: monitoring     │  │
│   │                           │         │                              │  │
│   │   ┌─────────────────┐     │         │   ┌──────────────────────┐   │  │
│   │   │   test-app      │     │  scrape │   │     Prometheus       │   │  │
│   │   │  :8080 /metrics ◀─────┼─────────┼───│  (pull, every 15s)   │   │  │
│   │   └────────┬────────┘     │         │   └──────────┬───────────┘   │  │
│   │            │              │         │              │ PromQL        │  │
│   │   ┌────────▼───┐ ┌──────┐ │         │              ▼               │  │
│   │   │  postgres  │ │ redis│ │         │   ┌──────────────────────┐   │  │
│   │   │   :5432    │ │:6379 │ │         │   │       Grafana        │   │  │
│   │   └────────────┘ └──────┘ │         │   │  :3000  (dashboards) │   │  │
│   │         │ stdout/stderr   │         │   └──────────▲───────────┘   │  │
│   └─────────┼─────────────────┘         │              │ LogQL         │  │
│             │ container logs            │   ┌──────────┴───────────┐   │  │
│             │ on node filesystem        │   │         Loki         │   │  │
│             │ /var/log/pods/...         │   │  :3100 (log store)   │   │  │
│             ▼                           │   └──────────▲───────────┘   │  │
│   ┌──────────────────────┐  push logs   │              │ HTTP push     │  │
│   │  Promtail (DaemonSet) │─────────────┼──────────────┘               │  │
│   │  one pod per node     │              └──────────────────────────────┘  │
│   └──────────────────────┘                                                 │
│                                                                            │
└──────────────────────────────────────────────────────────────────────────┘
        Browser ──▶ NodePort 30090 (Prometheus)   NodePort 30300 (Grafana)
```

**Two independent pipelines feed Grafana:**

- **Metrics pipeline (pull):** Prometheus discovers and scrapes targets over HTTP.
- **Logs pipeline (push):** Promtail tails node log files and pushes them to Loki.

Grafana queries both and renders them side by side.

---

## How each component works

### Prometheus — metrics (pull model)

Prometheus is the metrics database. It **scrapes** HTTP endpoints every 15s and stores the results as time series on a PersistentVolume.

It is configured with three scrape jobs (`k8s/monitoring/prometheus.yaml`):

| Job               | What it scrapes                                                          |
|-------------------|--------------------------------------------------------------------------|
| `prometheus`      | Its own internal metrics (`localhost:9090`)                              |
| `test-app`        | The sample app at `test-app.appspace.svc.cluster.local:8080/metrics`     |
| `kubernetes-pods` | **Any** pod cluster-wide that opts in via annotations                    |

The `kubernetes-pods` job uses the Kubernetes service-discovery API (`role: pod`) plus relabeling rules. A pod is scraped automatically when it carries these annotations — exactly what `test-app` declares:

```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"
  prometheus.io/path: "/metrics"
```

To reach the Kubernetes API for discovery, Prometheus runs under a dedicated **ServiceAccount** bound to a **ClusterRole** (read access to nodes, pods, services, endpoints, ingresses). It also enables `--web.enable-lifecycle` so config can be reloaded without a restart.

### Loki — log storage (push model)

Loki is the log database. Unlike Prometheus, it does **not** pull — agents push logs to it at `:3100/loki/api/v1/push`. It indexes only log **labels** (namespace, pod, container…) and stores the raw log lines compressed on disk, which keeps it lightweight.

Key config (`k8s/monitoring/loki.yaml`):

- Single-binary mode, `auth_enabled: false`, replication factor 1 (fine for one node).
- `filesystem` storage with the `boltdb-shipper` index — everything lives on the Loki PVC.
- **7-day retention** (`retention_period: 168h`); older logs are rejected/compacted.

### Promtail — log collector (DaemonSet)

Promtail is the agent that gets logs *into* Loki. It runs as a **DaemonSet**, so Kubernetes schedules exactly one Promtail pod on every node.

How it works (`k8s/monitoring/promtail.yaml`):

1. Mounts the node's log directories read-only via `hostPath`: `/var/log/pods`, `/var/log/containers`, `/var/lib/docker/containers`.
2. Uses Kubernetes service discovery to find pods scheduled **on its own node** (filtered by `${HOSTNAME}` / `spec.nodeName`).
3. Relabels each log stream with `namespace`, `pod`, `container`, and `node_name` so they're queryable in Loki.
4. Parses the container runtime format (`cri: {}`) and pushes the lines to Loki.
5. Tracks read offsets in `positions.yaml` so it resumes cleanly after a restart instead of re-sending everything.

Like Prometheus, it has its own ServiceAccount + ClusterRole for pod discovery.

### Grafana — visualization

Grafana is the UI layer. It is **provisioned automatically** at startup (no manual clicking required):

- **Datasources** (`grafana-datasources` ConfigMap): Prometheus (default, `uid: prometheus`) and Loki (`uid: loki`), wired to their in-cluster service URLs.
- **Dashboard provider** (`grafana-dashboards-provider`): tells Grafana to load any dashboard JSON found in `/var/lib/grafana/dashboards`.
- **Dashboard** (`grafana-dashboard-pod-logs`): the "Kubernetes Pod Logs" dashboard, mounted as a ConfigMap.
- **Admin credentials**: read from the `grafana-secret` Secret (default `admin` / `admin`).

Dashboards and credentials persist on the Grafana PVC.

---

## Data flow, end to end

**Metrics**

```
test-app exposes /metrics  ──▶  Prometheus scrapes every 15s  ──▶  stored as time series
                                                                        │
                                              Grafana queries via PromQL ┘  ──▶  panels
```

**Logs**

```
container writes stdout  ──▶  kubelet writes /var/log/pods/...  ──▶  Promtail tails & labels
                                                                            │
                                                       pushes to Loki :3100 ┘  ──▶  stored
                                                                            │
                                              Grafana queries via LogQL ─────┘  ──▶  log panels
```

---

## Namespaces, ports & storage

### Services & ports

| Component  | Namespace    | Service type | In-cluster address                                 | External port |
|------------|--------------|--------------|----------------------------------------------------|---------------|
| Prometheus | `monitoring` | NodePort     | `prometheus.monitoring.svc.cluster.local:9090`     | **30090**     |
| Grafana    | `monitoring` | NodePort     | `grafana.monitoring.svc.cluster.local:3000`        | **30300**     |
| Loki       | `monitoring` | ClusterIP    | `loki.monitoring.svc.cluster.local:3100`           | internal only |
| Promtail   | `monitoring` | DaemonSet    | (no service — it pushes outbound)                  | —             |
| test-app   | `appspace`   | NodePort     | `test-app.appspace.svc.cluster.local:8080`         | **30080**     |
| Postgres   | `appspace`   | ClusterIP    | `postgres.appspace.svc.cluster.local:5432`         | internal only |
| Redis      | `appspace`   | ClusterIP    | `redis.appspace.svc.cluster.local:6379`            | internal only |

### Persistent storage (PVCs)

| PVC              | Namespace    | Size | Used by    |
|------------------|--------------|------|------------|
| `prometheus-pvc` | `monitoring` | 2Gi  | Prometheus |
| `loki-pvc`       | `monitoring` | 2Gi  | Loki       |
| `grafana-pvc`    | `monitoring` | 1Gi  | Grafana    |
| `postgres-pvc`   | `appspace`   | 1Gi  | Postgres   |
| `redis-pvc`      | `appspace`   | 1Gi  | Redis      |

### Resource quotas

Both namespaces have a `ResourceQuota` to cap usage:

- **`monitoring`**: 10 pods, 2 CPU / 2Gi requested, 4 CPU / 4Gi limits, 3 PVCs.
- **`appspace`**: 6 pods, 1 CPU / 1.5Gi requested, 2 CPU / 2Gi limits, 2 PVCs.

> Note: the monitoring stack is deployed **separately** from the app stack. `k8s/kustomization.yaml` intentionally manages only the `appspace` workload; the `monitoring/` folder is applied on its own.

---

## Prerequisites

1. **kubectl** — [install guide](https://kubernetes.io/docs/tasks/tools/)
2. **A running Kubernetes cluster** — e.g. [minikube](https://minikube.sigs.k8s.io/docs/start/), [kind](https://kind.sigs.k8s.io/), or a cloud cluster.
3. **A default StorageClass** that can provision PVCs (minikube and kind include one out of the box).

Verify connectivity:

```bash
kubectl cluster-info
kubectl get nodes
kubectl get storageclass   # confirm a (default) StorageClass exists
```

---

## Setup — step by step

Clone the repo first:

```bash
git clone git@github.com:NerojuPavan/KubernetesMonitoring.git
cd KubernetesMonitoring
```

### Step 1 — Namespace & quota

```bash
kubectl apply -f k8s/monitoring/namespace.yaml
```

### Step 2 — Persistent volumes

```bash
kubectl apply -f k8s/monitoring/pvc.yaml
kubectl get pvc -n monitoring          # wait until all show STATUS=Bound
```

### Step 3 — Prometheus (metrics)

```bash
kubectl apply -f k8s/monitoring/prometheus.yaml
```

### Step 4 — Loki (log store)

```bash
kubectl apply -f k8s/monitoring/loki.yaml
```

### Step 5 — Promtail (log collector)

```bash
kubectl apply -f k8s/monitoring/promtail.yaml
```

### Step 6 — Grafana + dashboards

Apply the dashboard ConfigMaps **before** Grafana so they're mounted at startup:

```bash
kubectl apply -f k8s/monitoring/grafana-dashboards.yaml
kubectl apply -f k8s/monitoring/grafana.yaml
```

### Step 7 — Verify the stack

```bash
kubectl get pods -n monitoring
```

Everything should reach `Running` (one `promtail-*` pod per node):

```
NAME             READY   STATUS    RESTARTS   AGE
grafana-...      1/1     Running   0          1m
loki-...         1/1     Running   0          1m
prometheus-...   1/1     Running   0          1m
promtail-...     1/1     Running   0          1m
```

### Step 8 — Deploy the sample app (optional but recommended)

Without a workload there are no app metrics or logs to look at. Deploy `test-app`, Postgres, and Redis into `appspace`:

```bash
kubectl apply -k k8s/
kubectl get pods -n appspace
```

> The `test-app` image (`pavankumar5/test_app`) is pulled per `k8s/kustomization.yaml`. On a local cluster you may need to load/build the image into the node first (e.g. `minikube image load ...`).

### One-shot deploy (after cloning)

```bash
kubectl apply -f k8s/monitoring/namespace.yaml
kubectl apply -f k8s/monitoring/pvc.yaml
kubectl apply -f k8s/monitoring/prometheus.yaml
kubectl apply -f k8s/monitoring/loki.yaml
kubectl apply -f k8s/monitoring/promtail.yaml
kubectl apply -f k8s/monitoring/grafana-dashboards.yaml
kubectl apply -f k8s/monitoring/grafana.yaml
```

---

## Accessing the UIs

### Grafana (port 30300)

**minikube:**

```bash
minikube service grafana -n monitoring --url
```

**kind / generic:** find a node IP and browse to `http://<NODE_IP>:30300`:

```bash
kubectl get nodes -o wide
```

**Or port-forward (works everywhere):**

```bash
kubectl port-forward -n monitoring svc/grafana 3000:3000
# open http://localhost:3000
```

| Field    | Value   |
|----------|---------|
| Username | `admin` |
| Password | `admin` |

> Defined in `k8s/monitoring/grafana.yaml` (`grafana-secret`). Fine as-is for local use.

### Prometheus (port 30090)

```bash
kubectl port-forward -n monitoring svc/prometheus 9090:9090
# open http://localhost:9090
```

Useful checks:
- **Status → Targets** — `test-app` and `kubernetes-pods` should be **UP**.
- **Graph** — try `up`, or `up{job="test-app"}`.

---

## Using the dashboard

In Grafana:

1. **Dashboards → Logs → Kubernetes Pod Logs**.
2. Pick a **Namespace** (defaults to `appspace`).
3. Pick a **Pod** (or leave **All**).
4. Panels are grouped by tier — `test-app`, `postgres`, `redis` — plus an all-logs panel.

Datasources are already wired up, so you can also explore freely:
- **Explore → Prometheus** for metrics (PromQL).
- **Explore → Loki** for logs (LogQL), e.g. `{namespace="appspace", pod=~"test-app.*"}`.

---

## File layout

```
k8s/
├── kustomization.yaml          # Application stack ONLY (appspace); excludes monitoring/
├── namespace.yaml              # appspace namespace + quota
├── pvc.yaml                    # postgres + redis PVCs
├── postgres.yaml               # Postgres deployment + service + secret
├── redis.yaml                  # Redis deployment + service
├── app/
│   ├── app.yaml                # test-app deployment + service (Prometheus annotations)
│   ├── config.yaml             # test-app ConfigMap (env)
│   └── secret.yaml             # test-app Secret (JWT, DB creds)
└── monitoring/                 # Observability stack — deploy SEPARATELY
    ├── namespace.yaml          # monitoring namespace + quota
    ├── pvc.yaml                # prometheus, loki, grafana PVCs
    ├── prometheus.yaml         # config + RBAC + deployment + NodePort service
    ├── loki.yaml               # config + deployment + service
    ├── promtail.yaml           # RBAC + config + DaemonSet
    ├── grafana.yaml            # secret + datasources + deployment + NodePort service
    └── grafana-dashboards.yaml # dashboard provider + Pod Logs dashboard
```

---

## Troubleshooting

**PVCs stuck in `Pending`** — no default StorageClass:

```bash
kubectl get storageclass
kubectl describe pvc -n monitoring
minikube addons enable storage-provisioner   # minikube only
```

**Prometheus target `test-app` is DOWN** — app not deployed or `/metrics` unreachable:

```bash
kubectl get pods -n appspace
kubectl port-forward -n appspace svc/test-app 8080:8080
curl http://localhost:8080/metrics
```

**Grafana shows "No data" for logs** — check the log pipeline:

```bash
kubectl get pods -n monitoring -l app=loki         # Loki running?
kubectl get daemonset -n monitoring                # one Promtail per node?
kubectl logs -n monitoring -l app=promtail --tail=50
```

Also confirm `appspace` actually has running pods producing logs.

**Resource quota exceeded** — namespace caps hit:

```bash
kubectl describe resourcequota -n monitoring
kubectl describe resourcequota -n appspace
```

**Datasource errors in Grafana** — verify in-cluster DNS resolves:

```bash
kubectl exec -n monitoring deploy/grafana -- wget -qO- http://prometheus.monitoring.svc.cluster.local:9090/-/healthy
kubectl exec -n monitoring deploy/grafana -- wget -qO- http://loki.monitoring.svc.cluster.local:3100/ready
```

---

## Tear down

```bash
# Monitoring stack
kubectl delete -f k8s/monitoring/ --ignore-not-found

# Application stack
kubectl delete -k k8s/ --ignore-not-found

# Persistent data (irreversible)
kubectl delete pvc -n monitoring --all
kubectl delete pvc -n appspace --all
```

---

## Scope & notes

This is a **local development / learning setup** — it runs on minikube/kind and was tested locally. It is intentionally simple:

- Plain `admin`/`admin` Grafana login and inline secrets — convenient for a local cluster, nothing sensitive here.
- `filesystem` storage and single replicas — perfect for one node, no external dependencies.
- NodePort access instead of Ingress/TLS — easy to reach from the host.

If you ever decide to take this beyond a local sandbox, the natural next steps would be: real credentials via a secrets manager, object storage (S3/GCS) for Loki, Ingress + TLS, Alertmanager for alerting, pinned image tags with automated rollout (Argo CD / Helm), and higher replica counts. None of that is needed for local use.
```
