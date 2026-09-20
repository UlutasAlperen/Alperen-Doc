---
title: "kustomize"
weight: 2
---
# Kustomize

Helm solves packaging with templates and values. [Kustomize](https://kustomize.io/) refuses templates entirely: keep your plain YAML as the **base**, and build **overlays** that patch in environment differences. Since v1.27 `kubectl` ships with it built in - no extra binary:

```bash
kubectl apply -k ./my-app
```

## The Mental Model

```text
k8s-manifests/
├── base/
│   ├── api-deployment.yaml
│   ├── api-configmap.yaml
│   ├── web-deployment.yaml
│   └── kustomization.yaml      # lists resources
├── overlays/
│   ├── dev/
│   │   └── kustomization.yaml  # patches: replicas: 1
│   └── prod/
│       ├── kustomization.yaml
│       └── patch-replicas.yaml
```

- **base** = the truth of the application (works as-is on any cluster)
- **overlay** = a set of patches on top of the base (environment-specific deltas)

The rule that makes it work: overlays only _touch_ fields they declare. Everything else passes through unchanged.

## Patches: Strategic Merge vs JSON6902

Two patch flavors, and picking the right one matters:

**Strategic merge** (the default when you give a plain YAML patch) - merges by field name, YAML-style:

```yaml
# patch-replicas.yaml - strategic merge
apiVersion: apps/v1
kind: Deployment
metadata:
  name: synergychat-api
spec:
  replicas: 3
```

> Strategic merge has one hard limitation: it can't remove list items cleanly, because lists have no "key". If you need to replace an element of a list (say, one container's env entry), strategic merge will _merge_ it weirdly or duplicate it.

**JSON6902** (explicit patch type) - RFC 6902 operations, surgical and precise:

```yaml
# kustomization.yaml
patches:
  - target:
      kind: Deployment
      name: synergychat-api
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 3
      - op: remove
        path: /spec/template/spec/containers/0/resources
```

Use strategic merge for "set these fields", JSON6902 when you need `remove`, `replace` on list elements, or to target by something other than name/kind.

## The Transforms That Scale

- `namePrefix`/`nameSuffix`: prefix every resource (great for a "team" prefix without editing files)
- `namespace`: stamp every resource into a namespace (pairs with the [namespaces](../../kubernetes/kubernetes-namespaces/) notes)
- `labels` (current) vs `commonLabels` (deprecated): adds labels to _every_ resource including selectors

> Deprecation trap: older tutorials use `commonLabels`. On current kustomize it warns; the replacement is the `labels:` field with `includeSelectors: true|false`. `includeSelectors: false` is safer when you don't want the label change to trigger deployment updates.

- `configMapGenerator` / `secretGenerator`: generate resources with content hashes (more below)
- `--load-restrictor`: kustomize refuses to read files outside the kustomization directory by default. That security restriction exists because remote/`../` bases can pull arbitrary content; `--load-restrictor LoadRestrictionsNone` is the "I know what I'm doing" escape hatch

## Validate Before You Apply

```bash
kubectl kustomize overlays/prod          # render to stdout
kubectl apply -k overlays/prod --dry-run=client
```

And for schema-level validation, pipe into [kubeconform](https://github.com/yannh/kubeconform):

```bash
kubectl kustomize overlays/prod | kubeconform -strict -summary
```

## configMapGenerator: The Hash Trick

```yaml
configMapGenerator:
  - name: api-config
    files:
      - config.env
```

The generated ConfigMap is named `api-config-hg8b7fcfd2`. Change `config.env`, re-apply, and the hash changes. If your Deployment references the ConfigMap (via `envFrom`), kustomize rewrites the reference - so the pod template changes and **pods roll automatically**.

> This closes a silent failure mode from the v1 notes: with a hand-written ConfigMap, `kubectl apply` of a changed ConfigMap does _nothing_ to already-running pods. The container keeps its old env until it restarts for some other reason. With generated ConfigMaps, a config change is a rollout - visible, verifiable, and no stale-config mystery.

## Helm mi Kustomize mı?

| | Helm | Kustomize |
|---|---|---|
| Yaklaşım | Go template'leri + values | Saf YAML + patches |
| Ne için | 3. parti uygulamaları paketten kurmak | _Kendi_ manifest'lerini ortam bazlı yönetmek |
| Release/rollback | Helm'in kendi state'i (revision) | Git'te versiyonlarsan `git revert` |
| Öğrenme eğrisi | Template dili + chart yapısı | Neredeyse yok |

Benim pratik kuralım: başkasının yazdığı karmaşık uygulamayı (Prometheus, ArgoCD, cert-manager) kuracaksam **Helm**; kendi deployment'larımı ortamlara göre yönetiyorsam **Kustomize**. İkisi bir arada da yaygın - ve [ArgoCD](gitops-argocd/) ikisini de source olarak destekler.

## How to Structure a Base and Overlays

1. Move your SynergyChat manifests from v1 into the structure shown above, and write `base/kustomization.yaml` listing every resource.

2. The base must be independently applyable - test it first:

```bash
kubectl apply -k base
kubectl get pods
```

> If the base alone doesn't work, the overlays never will. Fix base errors here, where nothing environmental can mask them.

3. Create `overlays/dev/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patches:
  - target:
      kind: Deployment
      name: synergychat-api
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 1
```

4. Preview the difference, then apply:

```bash
kubectl kustomize overlays/dev | grep -B2 -A2 "replicas"
kubectl apply -k overlays/dev
```

## How to Promote a Change from dev to prod

The whole point of overlays: promotion is _another overlay_, not a copy-paste of files.

1. Create `overlays/prod/kustomization.yaml` with a higher replica patch plus an image pin:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patches:
  - path: patch-replicas.yaml
images:
  - name: bootdotdev/synergychat-api
    newTag: v1.4.2
```

2. Render and compare dev vs prod before applying - the diff _is_ your promotion review:

```bash
diff <(kubectl kustomize overlays/dev) <(kubectl kustomize overlays/prod)
```

3. Apply and verify what actually landed:

```bash
kubectl apply -k overlays/prod
kubectl get deploy synergychat-api -o jsonpath='{.spec.replicas} {.spec.template.spec.containers[0].image}{"\n"}'
```

## How to Auto-Roll Pods on Config Change

1. Delete the hand-written `api-configmap.yaml` from the base's `resources:` list, put its content in `base/config.env`, and add to `base/kustomization.yaml`:

```yaml
configMapGenerator:
  - name: api-config
    files:
      - config.env
```

2. Update the deployment's `envFrom` to reference the _unhashed_ name (kustomize rewrites it at build time):

```yaml
envFrom:
  - configMapRef:
      name: api-config
```

3. Verify the hash rewriting:

```bash
kubectl kustomize base | grep -A2 "name: api-config"
```

4. Change a value in `config.env`, re-apply, and watch the rollout happen by itself:

```bash
kubectl apply -k overlays/dev
kubectl get pods --watch
```

> If the ConfigMap is referenced via `envFrom` the roll happens because the pod template changes. If some pod reads the ConfigMap via a mounted volume at runtime, the _file_ updates on disk but pods still won't restart automatically unless the hash trick applies - that's exactly why `envFrom` + generator is the sanest pattern here.

for more [network-policy](network-policy/)
