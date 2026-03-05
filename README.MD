# Local Kubernetes Observability Lab (Kind + Helm + Prometheus + Grafana)

A hands-on observability lab that runs fully local on Kubernetes with Kind. You will install Prometheus and Grafana, deploy a StatsD-style `metrics-demo` service, generate synthetic traffic, build dashboards, create Grafana-managed alerts, run a failure simulation, and cleanly tear everything down.

## Table of Contents
- [Overview](#overview)
- [Architecture (ASCII diagram)](#architecture-ascii-diagram)
- [Prerequisites](#prerequisites)
- [Quickstart](#quickstart)
- [Install Prometheus + Grafana](#install-prometheus--grafana)
- [Access Prometheus + Grafana (port-forward)](#access-prometheus--grafana-port-forward)
- [Deploy metrics-demo](#deploy-metrics-demo)
- [Configure Prometheus to scrape metrics-demo (preferred Helm method + fallback)](#configure-prometheus-to-scrape-metrics-demo-preferred-helm-method--fallback)
- [Generate traffic](#generate-traffic)
- [Validate in Prometheus (Targets, Configuration, key queries)](#validate-in-prometheus-targets-configuration-key-queries)
- [Grafana dashboards (recommended panels + Code mode note)](#grafana-dashboards-recommended-panels--code-mode-note)
- [Alerting (create rules + evaluation group + contact point/policy note)](#alerting-create-rules--evaluation-group--contact-pointpolicy-note)
- [Chaos test (scale down/up, what to expect)](#chaos-test-scale-downup-what-to-expect)
- [Cleanup (stop traffic generator, uninstall helm, delete cluster)](#cleanup-stop-traffic-generator-uninstall-helm-delete-cluster)
- [Troubleshooting](#troubleshooting)

## Overview
This lab demonstrates:
- Prometheus pull-based scraping from Kubernetes services.
- StatsD push-to-exporter flow (`9125/UDP`) translated to Prometheus metrics (`9102/TCP`).
- Grafana dashboards and Grafana-managed alert rules.
- Alert lifecycle behavior during a real failure simulation.

Stable lab settings used in this repo:

| Item | Value |
|---|---|
| Kind cluster name | `obs-lab` |
| Monitoring namespace | `monitoring` |
| Prometheus Helm release | `prom` |
| Grafana Helm release | `graf` |
| Prometheus port-forward target | `svc/prom-prometheus-server` (`9090:80`) |
| Grafana port-forward target | `svc/graf-grafana` (`3000:80`) |
| Grafana admin secret | `graf-grafana` key `admin-password` |
| Demo deployment/service | `metrics-demo` in `default` |
| Demo scrape target | `metrics-demo.default.svc.cluster.local:9102` |
| Demo metrics endpoint | `http://metrics-demo.default.svc.cluster.local:9102/metrics` |
| Demo StatsD input | `9125/UDP` |

## Architecture (ASCII diagram)
```text
              +-------------------------+
              |      Kind Cluster       |
              |       (obs-lab)         |
              +-----------+-------------+
                          |
          +---------------+----------------+
          |                                |
+---------v---------+            +---------v---------+
|  monitoring ns    |            |    default ns     |
|                   |            |                   |
|  Prometheus (prom)|<-----------| metrics-demo svc  |
|  Grafana (graf)   |  scrape    | 9102/TCP /metrics|
|                   |            | 9125/UDP StatsD   |
+---------+---------+            +---------+---------+
          |                                ^
          | query                           |
          v                                 |
      Grafana UI                      synthetic traffic
      dashboards + alerts             (busybox nc -u)
```

## Prerequisites
| Tool | Check command |
|---|---|
| Docker | `docker version` |
| kubectl | `kubectl version --client` |
| kind | `kind version` |
| helm | `helm version` |

```bash
docker version
kubectl version --client
kind version
helm version
```

## Quickstart
```bash
export CLUSTER_NAME=obs-lab
export MON_NS=monitoring
export PROM_RELEASE=prom
export GRAF_RELEASE=graf

kind create cluster --name "$CLUSTER_NAME"
kubectl create namespace "$MON_NS"

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install "$PROM_RELEASE" prometheus-community/prometheus -n "$MON_NS"
helm install "$GRAF_RELEASE" grafana/grafana -n "$MON_NS"

kubectl apply -f yaml_store/metrics-demo.yaml
```

## Install Prometheus + Grafana
```bash
export MON_NS=monitoring
export PROM_RELEASE=prom
export GRAF_RELEASE=graf

helm install "$PROM_RELEASE" prometheus-community/prometheus -n "$MON_NS"
helm install "$GRAF_RELEASE" grafana/grafana -n "$MON_NS"
kubectl get pods -n "$MON_NS" -w
```

Helpful expected state:
```text
prom-prometheus-server-...            2/2 Running
graf-grafana-...                      1/1 Running
prom-kube-state-metrics-...           1/1 Running
prom-prometheus-node-exporter-...     1/1 Running
```

## Access Prometheus + Grafana (port-forward)
Use two terminals for port-forwards:

```bash
kubectl port-forward -n monitoring svc/prom-prometheus-server 9090:80
```

```bash
kubectl port-forward -n monitoring svc/graf-grafana 3000:80
```

Get Grafana admin password:
```bash
kubectl get secret -n monitoring graf-grafana -o jsonpath="{.data.admin-password}" | base64 --decode; echo
```

Open:
- Prometheus: http://localhost:9090
- Grafana: http://localhost:3000 (user: `admin`)

## Deploy metrics-demo
Apply the demo exporter/service manifest:
```bash
kubectl apply -f yaml_store/metrics-demo.yaml
kubectl get deploy,svc metrics-demo
```

The service exposes:
- `9125/UDP` for StatsD input.
- `9102/TCP` for Prometheus scraping (`/metrics`).

## Configure Prometheus to scrape metrics-demo (preferred Helm method + fallback)

### Preferred: Helm values with `extraScrapeConfigs`
Create values file:
```bash
cat > prom-extra-scrape.yaml <<'YAML'
server:
  extraScrapeConfigs: |
    - job_name: "metrics-demo"
      static_configs:
        - targets: ["metrics-demo.default.svc.cluster.local:9102"]
YAML
```

Upgrade Prometheus:
```bash
helm upgrade prom prometheus-community/prometheus -n monitoring -f prom-extra-scrape.yaml
kubectl rollout status -n monitoring deploy/prom-prometheus-server
```

### Fallback (warning): edit ConfigMap directly
Use only for quick recovery/testing. Helm upgrades can overwrite manual edits.

```bash
kubectl edit configmap -n monitoring prom-prometheus-server
```

Add under `scrape_configs`:
```yaml
- job_name: "metrics-demo"
  static_configs:
    - targets: ["metrics-demo.default.svc.cluster.local:9102"]
```

Then restart server to force reload if needed:
```bash
kubectl rollout restart deploy/prom-prometheus-server -n monitoring
kubectl rollout status deploy/prom-prometheus-server -n monitoring
```

## Generate traffic
Run a synthetic StatsD sender:

```bash
kubectl run -it --rm statsd-sender --image=busybox --restart=Never -- sh -c '
while true; do
  echo "demo.counter:1|c" | nc -u -w1 metrics-demo 9125
  sleep 1
done
'
```

This emits one counter increment per second (`demo_counter` on Prometheus side).

## Validate in Prometheus (Targets, Configuration, key queries)

### UI checks
- `Status -> Targets`: verify `metrics-demo` is `UP`.
- `Status -> Configuration`: verify active scrape job includes `metrics-demo`.

### Kubernetes checks (what config is active)
```bash
kubectl get configmap -n monitoring prom-prometheus-server -o yaml | sed -n '1,220p'
helm get values prom -n monitoring
kubectl get pod -n monitoring -l app=prometheus,component=server -o name
kubectl exec -n monitoring -it deploy/prom-prometheus-server -c prometheus-server -- sh -c 'sed -n "1,260p" /etc/config/prometheus.yml'
```

### Key PromQL queries
- Event count:
```text
demo_counter
```
- Events/sec (rate):
```text
rate(demo_counter{job="metrics-demo"}[5m])
```
- Alternate EPS:
```text
increase(demo_counter{job="metrics-demo"}[10m]) / 600
```
- Target health by job:
```text
sum by (job) (up)
```

Note: `demo_counter` is counter-like even though it does not end with `_total`.

## Grafana dashboards (recommended panels + Code mode note)
1. Add Prometheus data source in Grafana.
   - URL: `http://prom-prometheus-server.monitoring.svc.cluster.local`
2. Build a dashboard with these panels:
   - `demo_counter` (event count)
   - `rate(demo_counter{job="metrics-demo"}[5m])` (events/sec)
   - `sum by (job) (up)` (target health)
   - `increase(kube_pod_container_status_restarts_total[10m])` (pod restarts)
   - `1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)` (node memory usage)

Important note on Explore:
- Grafana Explore **Builder** mode may misparse raw PromQL if typed into the Metric field.
- Switch to **Code** mode for raw PromQL entry.

## Alerting (create rules + evaluation group + contact point/policy note)
Create Grafana-managed alert rules with an evaluation group interval (example `1m`):

1. Traffic dropped
   - Query:
   ```text
   sum(rate(demo_counter{job="metrics-demo"}[5m])) < 0.1
   ```
   - Pending period: `2m`

2. Pod restarting
   - Query:
   ```text
   increase(kube_pod_container_status_restarts_total[5m]) > 0
   ```
   - Pending period: `1m`

3. Node memory pressure
   - Query:
   ```text
   1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) > 0.9
   ```
   - Pending period: `5m`

Alert lifecycle in Grafana:
- `No data`: query has no series/value.
- `Pending`: condition met but pending timer not finished.
- `Firing`: condition still true after pending time.

Notifications:
- You must configure a **Contact Point** and **Notification Policy** to receive notifications.
- Without those, state changes are visible in Grafana UI only.

## Chaos test (scale down/up, what to expect)
Simulate outage by scaling `metrics-demo` to zero:

```bash
kubectl scale deploy/metrics-demo --replicas=0
kubectl get pods -l app=metrics-demo -w
```

Wait at least the alert pending period (for traffic drop, `>=2m`), then verify alert transitions to `Firing` in Grafana Alerting UI.

Recover service:
```bash
kubectl scale deploy/metrics-demo --replicas=1
kubectl rollout status deploy/metrics-demo
```

Expected:
- Traffic alert returns from `Firing` toward normal once metrics resume.
- You do not need to delete pods manually; scaling to `0` removes them automatically.

## Cleanup (stop traffic generator, uninstall helm, delete cluster)
Stop traffic generator:
- If interactive sender is running, press `Ctrl+C`.
- If needed, cleanup pod explicitly:
```bash
kubectl delete pod statsd-sender --ignore-not-found=true
```

Uninstall monitoring stack and delete cluster:
```bash
helm uninstall prom -n monitoring
helm uninstall graf -n monitoring
kubectl delete namespace monitoring
kind delete cluster --name obs-lab
```

## Troubleshooting

### Prometheus error: `field global already set`
Cause:
- Duplicating top-level `prometheus.yml` blocks (for example by patching `serverFiles.prometheus.yml` incorrectly) can create invalid config with repeated `global`.

Fix:
- Prefer Helm `server.extraScrapeConfigs` for adding new jobs only.
- Avoid replacing full `serverFiles.prometheus.yml` unless you fully control the entire file structure.

### Typo in values key (`extraScrapConfigs` vs `extraScrapeConfigs`)
- Correct key is `extraScrapeConfigs`.
- Wrong spelling is silently ineffective in many cases.

### How to tell what config is actually active
Check all three:
1. Prometheus UI:
   - `Status -> Configuration`
   - `Status -> Targets`
2. Kubernetes objects:
   ```bash
   kubectl get configmap -n monitoring prom-prometheus-server -o yaml
   helm get values prom -n monitoring
   ```
3. Mounted config in the live pod:
   ```bash
   kubectl exec -n monitoring -it deploy/prom-prometheus-server -c prometheus-server -- cat /etc/config/prometheus.yml
   ```

### Port-forward dropped during Helm upgrade/rollout
Symptom:
- `kubectl port-forward` exits when pods restart or service endpoints rotate.

Fix:
- Re-run port-forward commands after rollout completes:
```bash
kubectl port-forward -n monitoring svc/prom-prometheus-server 9090:80
kubectl port-forward -n monitoring svc/graf-grafana 3000:80
```

### `metrics-demo` target not `UP`
Checklist:
- Service exists and exposes `9102/TCP`.
- Prometheus job name is `metrics-demo`.
- Target exactly matches `metrics-demo.default.svc.cluster.local:9102`.
- Sender is running, and query `demo_counter` returns series.
