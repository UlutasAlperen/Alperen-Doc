---
title: "helm"
weight: 1
---
# Helm

In the [v1 notes](../../kubernetes/kubernetes-yaml-configurations/) we wrote every manifest by hand: deployments, configmaps, services, gateways. That's fine for a handful of resources you own. It falls apart the moment you want Prometheus + Grafana + Loki - three interdependent applications that together ship _hundreds_ of YAML lines, with version matrices between them.

[Helm](https://helm.sh/docs/) is _the_ package manager for Kubernetes. Instead of raw manifests you install _charts_: versioned, templatized bundles that ship with sane defaults you can override. Think apt/npm, but the package installs into your cluster.

## Chart Anatomy

```text
mychart/
├── Chart.yaml          # name, version (chart version), appVersion (app version)
├── values.yaml         # default configuration
├── values.schema.json  # optional: JSON schema validating values
├── charts/             # subchart dependencies (vendored or pulled via dependencies)
├── crds/               # CRDs installed BEFORE templates, never upgraded/rolled back
├── templates/
│   ├── _helpers.tpl    # reusable template snippets (labels, names)
│   ├── deployment.yaml
│   └── service.yaml
└── .helmignore
```

Things beginners usually miss:

- **`version` vs `appVersion`**: `version` is the chart's own version (can bump many times for the same app); `appVersion` is the app's version it deploys. Pin upgrades with `helm upgrade --version <chart-version>`
- **`crds/` is special**: CRDs there are applied before everything else on install, but Helm does _not_ upgrade or roll them back with the rest of the chart. That's why the Helm maintainers warn you not to manage volatile resources in it
- **`charts/` and dependencies**: a chart can declare dependencies in `Chart.yaml` (`dependencies:` with `condition:` flags); `helm dependency build` vendors them. That's how `kube-prometheus-stack` pulls in grafana, alertmanager and prometheus as subcharts

## Values and Their Precedence

This is the part that bites everyone eventually. Helm merges values from multiple sources, and the precedence is fixed:

1. Chart's `values.yaml` (lowest)
2. Each `-f my-values.yaml` (later `-f` wins over earlier ones)
3. Each `--set key=value` flag (highest)

> `--set` convenience is a trap: values set via `--set` don't live in any file, so six months later nobody knows where the config came from. `helm get values <release>` shows the _user-supplied_ values, which is your recovery tool. My rule: everything reproducible goes in a values file, `--set` only for throwaway experiments.

Also note that `--set` has its own parsing quirks (type coercion, commas in lists, `null` semantics via `--set key=null`). For anything non-trivial, `-f` with a file is more honest.

To see the defaults you're about to inherit:

```bash
helm show values prometheus-community/prometheus
```

And since some charts ship a `values.schema.json`, bad values fail fast at install time instead of producing broken manifests.

## Hooks and Weights

Helm doesn't just apply YAML; it also _executes lifecycle actions_ via annotations:

```yaml
metadata:
  annotations:
    "helm.sh/hook": pre-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

Typical use: a `pre-install`/`pre-upgrade` Job that migrates a database schema before the new pods roll out. `hook-weight` controls ordering between hooks.

## Where Does Release State Live?

`helm history`, `rollback` and friends work because Helm stores release state in the cluster - by default as Secrets named `sh.helm.release.v1.<release>.v<revision>` in the release namespace. Delete those secrets and Helm forgets everything. This also means: `kubectl` deletions and Helm state can drift apart (see the [troubleshooting notes](../kubernetes-troubleshooting/) for the `helm uninstall` vs `kubectl delete` mess this can cause).

**Özet - üçlü ilişki:**

- **Chart** = tarifeli YAML paketi (versiyonlanmış, template'li)
- **Release** = chart'ın cluster'a kurulmuş instance'ı (aynı chart'tan N tane olabilir)
- **Values** = kurulumun parametreleri (precedence sırası: chart defaults < `-f` < `--set`)
- Her `helm upgrade` bir **revision** üretir; rollback bu revision'lar üzerinden çalışır

## How to Inspect a Chart Before Installing It

Blind `helm install` is how people end up with 4Gi PVCs they didn't ask for. Inspect first:

1. Add the repo and refresh its index:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

2. Dump the chart's default values and _read them_:

```bash
helm show values prometheus-community/prometheus | less
```

Search for `retention`, `persistentVolume`, `resources` - these are the knobs you'll almost certainly want to override.

3. Render the full manifest set locally, without touching the cluster:

```bash
helm template monitor prometheus-community/prometheus > rendered.yaml
```

> `helm template` renders from files only; it doesn't know what's already in your cluster. Use it for quick reads, not final verification.

4. Do the dry run, which also catches API version and namespace issues:

```bash
helm install monitor prometheus-community/prometheus --dry-run --debug
```

## How to Install Prometheus with Custom Values

1. Install with defaults to establish a baseline:

```bash
helm install monitor prometheus-community/prometheus
kubectl get pods --watch
```

You should see `monitor-prometheus-server` running.

2. Override settings the reproducible way. Create `monitor-values.yaml`:

```yaml
server:
  persistentVolume:
    size: 4Gi
  retention: 7d
```

> Note the nesting: `server.persistentVolume.size` - the file mirrors the chart's `values.yaml` hierarchy. A flat `size: 4Gi` at the top would silently do nothing.

3. Apply it with `upgrade`:

```bash
helm upgrade monitor prometheus-community/prometheus -f monitor-values.yaml
```

4. Prove the values took effect:

```bash
helm get values monitor
kubectl get pvc
```

## How to Roll Back a Bad Upgrade

1. Break something on purpose - scale the server to an absurd replica count:

```bash
helm upgrade monitor prometheus-community/prometheus \
  --set server.replicaCount=5 --reuse-values
```

2. Watch it hurt:

```bash
kubectl get pods -l app=prometheus
helm history monitor
```

Each `upgrade` created a revision; `deployed` shows the active one.

3. Roll back to the previous revision:

```bash
helm rollback monitor <previous-revision>
kubectl get pods --watch
```

4. Verify what's actually deployed:

```bash
helm get values monitor
```

> `helm rollback` restores the release state but does _not_ revert your values files in git. Fix the file too, or your next `helm upgrade` re-introduces the problem.

for more [kustomize](../kustomize/)
