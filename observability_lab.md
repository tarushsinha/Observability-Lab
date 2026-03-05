## Requirements
* Docker
* kubectl
* kind
* helm
* Prometheus
* Grafana


1) Verify

```bash
docker version
kubectl version --client
kind version
helm version
```

2) Create kind cluster

```bash
kind create cluster --name obs-lab
kubectl cluster-info
kubectl get nodes
```

Expected:
NAME                    STATUS   ROLES           AGE   VERSION
obs-lab-control-plane   Ready    control-plane   41s   v1.35.0

3) Install Prometheus & Grafana

```bash
kubectl create namespace monitoring
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

Expected:
namespace/monitoring created

"prometheus-community" has been added to your repositories

"grafana" has been added to your repositories

Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "grafana" chart repository
...Successfully got an update from the "prometheus-community" chart repository
Update Complete. ⎈Happy Helming!⎈

```bash
helm install prom prometheus-community/prometheus -n monitoring
helm install graf grafana/grafana -n monitoring

kubectl get pods -n monitoring -w
```
Expected:

NAME                                          READY   STATUS    RESTARTS   AGE
graf-grafana-555c9c7bc6-x4w96                 1/1     Running   0          28s
prom-alertmanager-0                           1/1     Running   0          60s
prom-kube-state-metrics-7f9c8bf475-4vx4k      1/1     Running   0          60s
prom-prometheus-node-exporter-ql944           1/1     Running   0          60s
prom-prometheus-pushgateway-5f9f7959f-cdpcw   1/1     Running   0          60s
prom-prometheus-server-6d7b5b6b94-w52mh       2/2     Running   0          60s

4) Open Prometheus & Grafana locally

```bash
kubectl port-forward -n monitoring svc/prom-prometheus-server 9090:80

kubectl get secret --namespace monitoring graf-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

kubectl port-forward -n monitoring svc/graf-grafana 3000:80
```

Expected:

Open exposed K8s Services at:
- Prometheus: http://localhost:9090
- Grafana: http://localhost:3000

Login admin/{echo'd password}

5) Connect Grafana -> Prometheus (data source)

In the Grafana UI:
* Connections -> Data Sources -> Add data source -> Prometheus
* URL:
    http://prom-prometheus-server.monitoring.svc.cluster.local

Save & test

6) Deploy a demo metrics source + scrap it

    1) Deploy statsd-exporter (exposes /metrics and accepts StatsD)
        *** WHY **
        StatsD -> push (over UDP)
        Prometheus -> pull (HTTP)
        OpenTelemetry -> push (OLTP (gRPC/HTTP))

        statsd exporter is a translation bridge:
        - receives statsD udp metrics
        - converts into prometheus metrics
        - exposes them on /metrics

     Create a metrics-demo.yaml (see created file in repo), and apply it:

     ```bash
     kubectl apply -f metrics-demo.yaml
     ```

    2) Tell Prometheus to scrape it
     Create prom-extra-scrape.yaml (see created file in repo), and apply it:

     ```bash
     helm upgrade prom prometheus-community/prometheus -n monitoring -f prom-extra-scrap.yaml
     ```

     if for some reason when verifying in prometheus UI (status -> targets -> metrics-demo should be UP)
      -> create prom-values.yaml (see example)

      ```bash
       helm upgrade prom prometheus-community/prometheus -n monitoring -f prom-values.yaml
       kubectl rollout status -n monitoring deploy/prom-prometheus-server

       kubectl exec -n monitoring -it PODNAME -c prometheus-server -- sh -c 'grep -n "metrics-demo" /etc/config/prometheus.yml'
      ```

      --- some issues with the prom-[].yaml caused the job to not be added to the configmap - manually added the job to config map to proceed for the time being ---

      Expected from last command when verifying: job_name: "metrics-demo"

7) Inject Metrics
```bash
kubectl run -it --rm statsd-sender --image=busybox --restart=Never -- sh -c '
while true; do
 echo "demo.counter:1|c" | nc -u -w1 metrics-demo 9125
 sleep 1
done
'
```

8) Test PromQL, Build Dashboards, and Alerts

Query on Prometheus w/ PromQL for verification before import to Grafana
demo_counter
increase(demo_counter{job="metrics-demo"}[10m]) / 600

See images folder for Grafana dashboard samples

## Dashboards created:
* Event Count: demo_counter
* Event Rate: increase(demo_counter{job="metrics-demo"}[10m]) / 600

* Cluster CPU Usage: sum(rate(container_cpu_usage_seconds_total{container!=""}[5m]))
* Pod Count by Namespace: count(kube_pod_info) by (namespace)
* Restarting Pods: increase(kube_pod_container_status_restarts_total[10m])
* Node Memory Usage: 1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
* K8s Target Health: sum by (job) (up)

## Alerts created
* metrics-demo traffic dropped
* Container Restarted
* Alert #3: Node memory pressure (>90%)

## Graceful Shutdown
```bash
helm uninstall prom -n monitoring
helm uninstall graf -n monitoring

kubectl delete namespace monitoring
kind delete cluster --name obs-lab
```