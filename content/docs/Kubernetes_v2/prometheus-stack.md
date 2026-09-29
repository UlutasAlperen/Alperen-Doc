---
title: "prometheus-stack"
weight: 18
---
# Monitoring with kube-prometheus-stack

We've touched monitoring twice and never actually _ran_ it. The [helm](../helm/) notes installed the plain `prometheus` chart to teach values precedence, and [cluster-extensions](../cluster-extensions/) installed `kube-prometheus-stack` to show what an operator looks like. Neither taught you to operate a monitoring stack.

That's this chapter. By the end you'll have Prometheus scraping the cluster, Grafana showing it, Alertmanager paging you when it matters, and - the part people skip - a `ServiceMonitor` for your own application so its metrics land next to the system ones.

[kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) is the prometheus-community chart that packages the whole [kube-prometheus](https://github.com/prometheus-operator/kube-prometheus) project: manifests, dashboards and alerting rules as one Helm release.

## What's Actually In the Stack

One `helm install` pulls in six moving parts, and knowing which is which saves you an hour of staring at pod names:

| Component | What it does | Why you care |
|---|---|---|
| **prometheus-operator** | reconciles `Prometheus`, `ServiceMonitor`, `PrometheusRule` CRs | this is the operator; everything else is a CR it acts on |
| **prometheus** | scrapes targets, stores time series, evaluates rules | the TSDB. It's a `Prometheus` CR, not a Deployment you edit |
| **alertmanager** | deduplicates, groups and routes alerts | silence/ inhibition live here, not in Prometheus |
| **grafana** | dashboards | ships with a large library of them pre-wired |
| **node-exporter** | per-node hardware/OS metrics | a DaemonSet; one per node is normal |
| **kube-state-metrics** | object-state metrics (pod phases, restarts, owners) | this is where `kube_pod_status_phase` comes from |

The first two are the important relationship: **prometheus-operator owns Prometheus**. You don't `kubectl scale` the Prometheus deployment; you edit `spec.replicas` on the `Prometheus` CR and the operator does the rest. Same reconcile pattern as the [CloudNativePG](../postgresql-longhorn-read-replicas/) `Cluster` and the [ArgoCD](../gitops-argocd/) `Application`.

### vs the plain `prometheus` chart

The `prometheus-community/prometheus` chart from the helm notes is a single-statefulset Prometheus with a config file. `kube-prometheus-stack` is the operator model with CRDs. The difference that matters day to day: with the plain chart you edit a `ConfigMap` and restart; with the stack you apply a `ServiceMonitor` and it just works.

## The Honest Requirements

- **Roughly 2–3Gi of memory headroom.** Prometheus defaults to no retention limits and will happily use what it's given; Grafana and the operator add a few hundred Mi on top
- **A StorageClass if you want retention to survive a restart.** The chart defaults to ephemeral storage - a pod restart wipes your history. Longhorn in the [next chapter](../longhorn/) is the durable option
- **CRD lifecycle awareness.** Helm installs the `monitoring.coreos.com` CRDs once and **never upgrades them**. That's the [helm](../helm/) `crds/` rule, and it's the #1 cause of ugly upgrades here
- **The `release:` label.** `ServiceMonitor` objects must carry a label matching `prometheus.prometheusSpec.serviceMonitorSelector` or Prometheus **silently ignores** them. Default is the Helm release name
- **Namespace discipline.** The default install only discovers monitors in its own namespace unless you widen the selectors

## How to Install kube-prometheus-stack with Helm

1. Add the repo and read the defaults before you inherit them - this chart's `values.yaml` is thousands of lines:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm show values prometheus-community/kube-prometheus-stack > default-values.yaml
```

Search `default-values.yaml` for `retention`, `storageSpec`, `resources`, `serviceMonitorSelector` - the four knobs you'll almost certainly touch.

2. Write your overrides in a file (the [helm](../helm/) notes' rule: everything reproducible goes in a values file):

```yaml
# monitoring-values.yaml
prometheus:
  prometheusSpec:
    retention: 15d
    resources:
      requests:
        cpu: 250m
        memory: 1Gi
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi
grafana:
  adminPassword: "changeme"
  defaultDashboardsEnabled: true
```

> Note the nesting: `prometheus.prometheusSpec.retention` mirrors the chart's `values.yaml` hierarchy. A flat `retention: 15d` at the top silently does nothing.

3. Install into a dedicated namespace and watch it come up:

```bash
helm install kps prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  -f monitoring-values.yaml
kubectl -n monitoring get pods --watch
```

4. Verify the operator is actually reconciling, not just running:

```bash
kubectl -n monitoring get prometheus
kubectl -n monitoring get crd | grep monitoring
kubectl -n monitoring logs -l app.kubernetes.io/name=prometheus-operator --tail=20
```

`kubectl get prometheus` should show a `Prometheus` CR with `READY` replicas. That CR is the source of truth for the deployment below it.

## How to Monitor Your Own Application

The system components get scraped out of the box. Your app has to opt in, and this is where almost everyone gets stuck.

1. Deploy something that exposes `/metrics`. Any Prometheus client library works; the shape is a normal Service pointing at a container port that serves Prometheus text format.

2. Tell Prometheus to scrape it - a `ServiceMonitor`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: synchat-api
  namespace: monitoring
  labels:
    release: kps # MUST match the Helm release name
spec:
  selector:
    matchLabels:
      app: synchat-api
  namespaceSelector:
    matchNames:
      - default
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
```

The `release: kps` label is the whole ballgame. Without it Prometheus will not look at this object and will not tell you why.

3. If you'd rather not label everything, widen the selector instead - the opt-out is `serviceMonitorSelectorNilUsesHelmValues: false` in `prometheus.prometheusSpec`. Convenient in a lab, loose in production.

4. Verify from the target's side, not the YAML's:

```bash
kubectl -n monitoring port-forward svc/kps-prometheus 9090:9090
```

Open `http://localhost:9090/targets`. Your job should be `UP`. If the ServiceMonitor isn't in the list at all, it's the label. If it's there but `DOWN`, it's your port or path.

## How to Alert

Scraping without alerting is a museum exhibit. Alerts live in `PrometheusRule` CRs - same shape as the ones the chart ships, just yours.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: synchat-alerts
  namespace: monitoring
  labels:
    release: kps
spec:
  groups:
    - name: synchat
      rules:
        - alert: SynchatApiDown
          expr: up{job="synchat-api"} == 0
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "synchat-api target is down"
        - alert: SynchatHighErrorRate
          expr: |
            sum(rate(http_requests_total{job="synchat-api",status=~"5.."}[5m]))
              / sum(rate(http_requests_total{job="synchat-api"}[5m])) > 0.05
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "synchat-api 5xx rate above 5%"
```

Two rules that matter more than anything you can invent: **target down** and **cert-manager renewal overdue** (the one from the [cert-manager-production](../cert-manager-production/) chapter). Both detect the automation failing rather than the symptom arriving.

Alertmanager is where the alert goes next. Its config is a `Secret` named `alertmanager-<release>` holding `alertmanager.yml` - routing, receivers (Slack, email, PagerDuty) and inhibitions. Test the whole path by firing a rule that can't not fire:

```bash
# a rule that fires immediately, then delete it
kubectl -n monitoring apply -f - <<'EOF'
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: fire-drill
  namespace: monitoring
  labels: {release: kps}
spec:
  groups:
    - name: fire-drill
      rules:
        - alert: FireDrill
          expr: vector(1)
          labels: {severity: warning}
          annotations: {summary: "fire drill"}
EOF
```

Watch it appear in Alertmanager at `http://localhost:9093` (port-forward the same way), then delete the rule. Seeing the full path once is worth more than reading the docs twice.

## How to Reach Grafana

```bash
kubectl -n monitoring port-forward svc/kps-grafana 3000:80
```

`http://localhost:3000`, credentials from `grafana.adminPassword` in your values. The dashboards worth opening first: `Kubernetes / Compute Resources / Namespace (Pods)`, `Kubernetes / Networking / Pod`, and the node-exporter ones.

For anything beyond your laptop, front Grafana with the [Gateway](../../kubernetes/kubernetes-gateway-minikube/) from the v1 notes - a `HTTPRoute` to `kps-grafana`, and cert-manager handing it TLS from the previous two chapters. That's the moment the add-ons stop being separate toys and become one platform.

## How to Upgrade Without Breaking CRDs

This is the part everyone learns the hard way. `helm upgrade` **does not touch CRDs** - they were applied once from `crds/` and that's it.

1. When a new chart version changes CRD schemas, apply them by hand first:

```bash
helm show crds prometheus-community/kube-prometheus-stack > crds.yaml
kubectl apply --server-side -f crds.yaml
```

2. Then upgrade the release:

```bash
helm upgrade kps prometheus-community/kube-prometheus-stack \
  -n monitoring -f monitoring-values.yaml --version <chart-version>
```

3. Check `helm history kps` and the operator logs. A failed upgrade here usually means the operator can't parse a CR that the new CRD changed - roll back the release (`helm rollback`) but know that the CRD is now the new one and won't roll back with it.

> **Dikkat:** CRD'ları silmek de Helm'e ait değil. `helm uninstall` sildiğinde `monitoring.coreos.com` CRD'ları kalır - ki kalsın, çünkü `Prometheus`/`ServiceMonitor` objelerin hâlâ oradadır. Temizlemek istersen `kubectl delete crd ...` ile elle yap.

## How to Test It End to End

The only test that proves monitoring works is breaking something and getting told.

1. Confirm the loop is live: `kubectl -n monitoring get prometheus` is `READY`, `/targets` shows your app `UP`.

2. Break your app on purpose - scale the deployment to zero:

```bash
kubectl scale deploy synchat-api --replicas=0
```

3. Within a scrape interval the target goes `DOWN`, within `for: 2m` the `SynchatApiDown` alert fires, and Alertmanager routes it. Time it. If the gap is longer than you'd tolerate in an incident, your `interval`/`for` values are wrong, not your luck.

4. Break it the other way - a config change that makes the app 5xx - and watch the error-rate alert. This is the one that catches real outages.

5. Now the silent failure: `kubectl -n monitoring scale deploy kps-prometheus-operator --replicas=0`. Nothing alerts, because nothing is reconciling. That's why `CertificateNotReady`-style "the automation stopped" alerts (from the [cert-manager-production](../cert-manager-production/) notes) matter more than "the app is down" alerts - they catch the layer everyone forgets to watch.

**Özetlersek:** kurulum bir `helm install`, asıl iş `ServiceMonitor` + `release:` label'ında. Alert'i "uygulama çöktü"ye değil "otomasyon durdu"ya kur. CRD'lar Helm'in sorumluluğunda değil - upgrade öncesi elle apply et.

for more [victoriametrics](../victoriametrics/)
