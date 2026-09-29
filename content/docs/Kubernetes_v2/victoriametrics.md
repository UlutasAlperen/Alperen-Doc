---
title: "victoriametrics"
weight: 19
---
# VictoriaMetrics

The [prometheus-stack](../prometheus-stack/) chapter gave you a working monitoring stack. This one asks a different question: if you were starting over, would you still pick Prometheus as the storage engine?

[VictoriaMetrics](https://docs.victoriametrics.com/) is a metrics database that speaks the same scrape protocol and mostly the same query language, but is built differently underneath - columnar storage, aggressive compression, no local TSDB churn. The usual reasons people move: less memory per series, longer retention on the same disk, and cardinality explosions that don't take the database down with them.

It is **not** a drop-in replacement for everything Prometheus does. Alert evaluation, scraping and the operator model all still exist - but they come from VictoriaMetrics' own operator and CRDs, which is the part that trips people up.

**Özetlersek:** Prometheus ekosistemi (scrape, PromQL, alert) aynen kalır; değişen şey verinin nerede durduğu. "Daha az kaynak, daha uzun retansiyon" için geçilir - ama CRD isimleri değiştiği için her objeyi yeniden yazman gerekir.

## Why Not Just Prometheus

An honest comparison, because neither wins outright:

| | Prometheus | VictoriaMetrics |
|---|---|---|
| Storage | local TSDB, per-scrape head churn | columnar, better compression |
| Memory per series | higher | noticeably lower |
| Long retention | painful on disk | cheap |
| High cardinality | degrades, then hurts | handles it much better |
| Query language | PromQL | MetricsQL (PromQL superset) |
| Ecosystem / docs | enormous | smaller but good |
| Maturity of the operator | very mature | catching up fast |
| Downsampling / tiering | needs Thanos/Mimir | built in |

Where Prometheus still wins: community answers, Stack Overflow, the sheer number of dashboards and blog posts. Where VictoriaMetrics wins: the same hardware stores several times more history, and your `topk` queries stop timing out.

## The Architecture

The pieces map onto the Prometheus ones almost one-for-one:

| Prometheus concept | VictoriaMetrics equivalent | Notes |
|---|---|---|
| prometheus-operator | **victoria-metrics-operator** | reconciles VM CRDs |
| Prometheus | **VMSingle** (one node) / **VMCluster** (distributed) | the TSDB |
| - | **VMAgent** | scraping + remote write (replaces Prometheus' scrape role) |
| - | **VMAlert** | rule/alert evaluation |
| PrometheusRule | **VMRule** | same rule syntax, different kind |
| ServiceMonitor / PodMonitor | **VMServiceScrape** / **VMPodScrape** | same spec shape, different kind |
| Alertmanager | **VMAlertmanager** | usually the same Alertmanager |

Two architectural notes that matter:

- **VMAgent does the scraping**, not the database. In clustered mode `vminsert` receives from VMAgent, `vmstorage` keeps the data, `vmselect` answers queries. That split is what lets it scale write and read independently.
- **VMAlert evaluates rules**, so alerting keeps working even if the query path is busy. Don't assume "Prometheus evaluates" - check where your rules actually landed.

## The Honest Requirements

- **Same as the Prometheus stack**: memory headroom, a StorageClass for retention, namespace discipline
- **The VM operator CRDs.** These are `victoriametrics.com` not `monitoring.coreos.com`, so nothing you wrote for Prometheus is reusable as-is
- **A decision on single vs cluster before you install.** Changing later means a data migration, not a values change
- **Grafana still comes along.** The chart bundles it; you don't need to add a separate Grafana install

## How to Install victoria-metrics-k8s-stack with Helm

1. Three repos - the VM chart pulls Grafana and kube-state-metrics from elsewhere:

```bash
helm repo add vm https://victoriametrics.github.io/helm-charts/
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm show values vm/victoria-metrics-k8s-stack > vm-defaults.yaml
```

2. Write your values. Note that the scrape config block is named `serviceMonitors` in the values even though it generates `VMServiceScrape` objects - a naming leftover that confuses everyone once:

```yaml
# vm-values.yaml
victoria-metrics-operator:
  enabled: true

# what gets deployed as the TSDB
vmsingle:
  enabled: true
  spec:
    retentionPeriod: "30d"
    storage:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 20Gi

# what scrapes
vmagent:
  enabled: true

# what evaluates rules
vmalert:
  enabled: true
  spec:
    evaluationInterval: 30s

grafana:
  adminPassword: "changeme"
  defaultDashboardsEnabled: true

kube-state-metrics:
  enabled: true
prometheus-node-exporter:
  enabled: true
```

3. Dry-run first (the chart's own docs recommend it - the values surface is large):

```bash
helm install vmks vm/victoria-metrics-k8s-stack \
  -n monitoring --create-namespace \
  -f vm-values.yaml --debug --dry-run
```

4. Install and watch the CRs land:

```bash
helm install vmks vm/victoria-metrics-k8s-stack \
  -n monitoring --create-namespace -f vm-values.yaml
kubectl -n monitoring get pods --watch
kubectl -n monitoring get vmsingle,vmagent,vmalert
```

5. Confirm the operator is reconciling rather than just running:

```bash
kubectl -n monitoring logs -l app.kubernetes.io/name=victoria-metrics-operator --tail=20
kubectl -n monitoring get crd | grep victoriametrics
```

## How to Translate Prometheus Objects to VM Ones

This is the whole migration, condensed. Same intent, different `kind` and different `apiVersion`:

| You have | You write | Field differences |
|---|---|---|
| `ServiceMonitor` (`monitoring.coreos.com/v1`) | `VMServiceScrape` (`operator.victoriametrics.com/v1beta1`) | `spec.endpoints[].port` → `spec.endpoints[].port`; most fields carry over |
| `PodMonitor` | `VMPodScrape` | same |
| `PrometheusRule` | `VMRule` | rule `expr` syntax is MetricsQL, which accepts PromQL |
| `Prometheus` CR | `VMSingle` or `VMCluster` | `retention` → `retentionPeriod` |
| - | `VMAgent` | scraping config lives here |

A concrete one - the `synchat-api` monitor from the Prometheus chapter becomes:

```yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMServiceScrape
metadata:
  name: synchat-api
  namespace: monitoring
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

Note what **disappeared**: the `release: kps` label. VM's operator doesn't gate discovery on a Helm release label the way Prometheus does. Fewer silent failures.

> **Dikkat:** Prometheus'tan VM'e geçerken "CRD'ları değiştir, spec'i koru" beklentisi tuzağı. `VMRule` içindeki `expr` MetricsQL'dir - PrometheusQL'in üst kümesi olduğu için çoğu sorgu birebir çalışır ama `rate()`/`histogram_quantilaround` davranışları ve bazı fonksiyon isimleri farklıdır. Kuralları tek tek test et, toplu taşıma.

## How to Choose Single vs Cluster

| | `VMSingle` | `VMCluster` |
|---|---|---|
| Topology | one pod, one disk | `vminsert` / `vmstorage` / `vmselect` |
| Scale | up (bigger box) | out (more nodes) |
| HA | restart, lose in-flight writes | replicated storage |
| Complexity | low | three Deployments + services |
| Right for | homelab, one team, one cluster | many teams, long retention, real QPS |

Start on `VMSingle`. It's the [longhorn](../longhorn/) lesson applied to metrics: don't buy the distributed system before you have the distributed problem. When `vmstorage` needs its own nodes, you'll know.

```yaml
vmcluster:
  enabled: false   # flip to true and disable vmsingle
  spec:
    retentionPeriod: "12"
    vmstorage:
      replicaCount: 2
      storage:
        resources:
          requests:
            storage: 100Gi
    vminsert:
      replicaCount: 2
    vmselect:
      replicaCount: 2
```

## How to Verify It Actually Works

1. Open the VMUI - VictoriaMetrics' own query interface, no Grafana required:

```bash
kubectl -n monitoring port-forward svc/vmks-vmsingle 8428:8428
```

`http://localhost:8428/vmui`. Run `up` and then something from your own app. If `up` returns nothing, the scrape path is broken; if your app's metric is missing but `up` works, it's the `VMServiceScrape` selector.

2. Check the targets the same way you would in Prometheus - VMAgent has its own `/targets`:

```bash
kubectl -n monitoring port-forward svc/vmks-vmagent 8429:8429
```

3. Compare against the Prometheus stack if you have both running. Same query, same window, `vm_memory_*` on each. The difference in resident memory is the argument for the migration, measured rather than blogged about.

4. Break an app and confirm `VMAlert` fires - the alerting path is separate from the query path here, so it's worth proving explicitly.

## When to Pick Which

| Situation | Pick |
|---|---|
| Learning monitoring for the first time | **Prometheus** - the docs and answers are everywhere |
| Homelab / one cluster / modest retention | either; `VMSingle` is lighter |
| Long retention, high cardinality, many teams | **VictoriaMetrics** |
| Already invested in `PrometheusRule`/`ServiceMonitor` + dashboards | **Prometheus**, unless you'll actually do the translation |
| Need Thanos-style global view across clusters | VictoriaMetrics cluster mode, or Thanos on Prometheus |
| Team knows PromQL but nothing else | VictoriaMetrics (MetricsQL is a superset) |

The honest summary: Prometheus is the safer default and the better ecosystem; VictoriaMetrics is the better engine. Most organisations pick Prometheus and regret the disk bill two years later.

**Özetlersek:** engine değişir, ekosistem kalır. Prometheus'u öğren, VictoriaMetrics'i kaynak/retansiyon sıkıştığında seç. İkisini birden çalıştırıp ölçmek, blog yazısı okumaktan daha ikna edicidir.

for more [longhorn](../longhorn/)
