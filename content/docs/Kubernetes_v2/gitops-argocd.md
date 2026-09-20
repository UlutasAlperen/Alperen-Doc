---
title: "gitops-argocd"
weight: 6
---
# GitOps

My site deploys with GitHub Actions + rsync over SSH (see [github_workflow](../../version_control_systems/github_workflow/)). That's a **push** model: CI runs, then _pushes_ files to the server. GitOps flips it: the cluster _pulls_ its own desired state from git and reconciles itself continuously.

The four principles (from [OpenGitOps](https://opengitops.dev/)):

1. **Declarative** - the entire desired state is expressed declaratively (manifests, charts, kustomize overlays)
2. **Versioned and immutable** - state lives in git; every change is a commit, every deploy is a merge
3. **Pulled automatically** - an agent inside the cluster pulls; nothing outside pushes into the cluster
4. **Continuously reconciled** - actual state is compared with desired state _all the time_, and drift is corrected

That fourth principle is what push-based CD structurally cannot give you. With push CD, the 2am `kubectl edit` sticks forever - nobody re-runs the pipeline for a change they didn't make. With pull CD, drift is detected within seconds and reverted.

# ArgoCD Architecture

[ArgoCD](https://argo-cd.readthedocs.io/) is the most popular GitOps controller. It runs _inside_ the cluster and keeps it matching git:

- **API server + UI + CLI**: dashboard, gRPC API, and the `argocd` binary for imperative operations
- **Repo server**: clones and renders your repo - plain YAML, Helm, or kustomize (it detects `kustomization.yaml` automatically)
- **Application controller**: the reconciler. Continuously diffs rendered git state vs live cluster state, computes health, and (when auto-sync is on) corrects it

The unit of work is the `Application` CRD:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: synergychat
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-user>/k8s-manifests
    targetRevision: main
    path: base
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

- `source`: repo, revision, path. For private repos you register credentials first (`argocd repo add` or via UI)
- `syncPolicy.automated`: sync on git changes without human click
- `prune: true`: resource deleted in git gets deleted in cluster too - without this, deletions silently do nothing
- `selfHeal: true`: manual `kubectl edit/scale/delete` gets reverted to git's state

## Sync Waves and Hooks

Deployments have _ordering_ requirements: the namespace first, then configmaps, then deployments, then an ingress. ArgoCD supports this with annotations:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"
```

Lower waves sync first and must be Healthy before the next wave proceeds. For one-off work (migrations, seed data), hooks run Jobs at sync phases:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync
```

`helm install`'ün [hooks](helm/) kavramına aynen benziyor - sadece burada tetikleyici git sync'i.

## App-of-Apps and ApplicationSet

Once you have more than a handful of Applications, you get the chicken-and-egg problem: who manages the Applications themselves? Two standard answers:

- **App-of-apps**: one root `Application` whose source directory contains only other `Application` manifests. ArgoCD bootstraps itself
- **ApplicationSet**: generates Applications from templates (one app per git directory, per cluster, per PR - [PR environments](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/) fall out almost for free)

## Push mı Pull mı?

| | GitHub Actions (push) | ArgoCD (pull) |
|---|---|---|
| Tetikleme | CI pipeline'ı deploy eder | Cluster kendi durumunu çeker |
| Kimlik | CI'ın cluster'a SSH/API erişimi gerekir | Sadece içeriden çıkan bağlantı; dışarıya açık erişim yok |
| Drift | 2am `kubectl edit` kalıcı olur | `selfHeal` geri alır |
| Secrets | GitHub secrets'ta | Repo/cluster'da (External Secrets ile dışarı taşınır) |
| State | Actions logları | `Application` CRD + canlı UI |

Benim sonucum: build/test CI'da kalır; **deploy'u** ArgoCD'ye devretmek drift'i ve "kim ne deploy etti" sorusunu ortadan kaldırıyor. CI image'ı build edip registry'ye push'lar, manifest'teki tag'i güncelleyen PR'ı açar, gerisini ArgoCD halleder.

## How to Install ArgoCD and Log In via CLI

1. Install with Helm (see [helm](helm/)):

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo/argo-cd --namespace argocd --create-namespace
kubectl -n argocd get pods --watch
```

2. Expose the UI locally and fetch the admin password:

```bash
kubectl port-forward svc/argocd-server 8080:443 -n argocd
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```

> That initial secret contains a _generated_ admin password - after your first login, change it or disable local admin entirely and use SSO/RBAC. Leaving the initial secret around is a documented footgun.

3. Install the CLI and log in (self-signed cert in dev, hence `--insecure`):

```bash
argocd login localhost:8080 --username admin --password <pw> --insecure
argocd account list
```

4. Change the admin password immediately:

```bash
argocd account update-password
```

## How to Onboard an Application with Auto-Sync

1. Push your SynergyChat manifests (the `base/` folder from [kustomize](kustomize/)) to a GitHub repo.

2. Register the repo (needed once, and mandatory for private repos):

```bash
argocd repo add https://github.com/<your-user>/k8s-manifests.git
```

3. Create the Application (the YAML from the architecture section above), then watch it through the CLI instead of the UI:

```bash
kubectl apply -f synergychat-app.yaml
argocd app list
argocd app get synergychat
argocd app sync synergychat   # only needed if automated isn't on yet
```

`argocd app get` shows the interesting state: `Synced`/`OutOfSync`, and per-resource `Health` (Progressing/Healthy/Degraded) - which is k8s health checks read for you.

4. The UI (`kubectl port-forward svc/argocd-server 8080:443 -n argocd`) renders the resource tree - worth seeing once, since it visualizes exactly which manifest owns which live resource.

## How to Test Drift Correction

1. Scale by hand, like the 2am cowboy you're not supposed to be:

```bash
kubectl scale deployment/synergychat-api --replicas=5
argocd app get synergychat
```

Status flips to `OutOfSync`; with `selfHeal` on, the controller reconciles it back within seconds - verify the replica count returns to git's value.

2. Change git instead: open a PR that bumps `replicas` in the base, merge it, and watch `argocd app get` flip to `Synced` after the refresh loop picks it up. _Deploy = git push._

3. Test `prune`: delete a manifest from the repo, commit, and confirm the resource disappears from the cluster. That's the property that makes git the single source of truth - _including deletions_.

> Watch out: `prune: true` means git deletions are destructive on the cluster, too. Deleting the wrong directory in a PR takes down real workloads - that's why prod repos get branch protection and why `argocd app sync --prune` in manual mode deserves a second look before Enter.

for more [vpa-cluster-autoscaler](vpa-cluster-autoscaler/)
