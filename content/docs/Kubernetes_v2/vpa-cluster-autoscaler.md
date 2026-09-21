---
title: "vpa-cluster-autoscaler"
weight: 7
---
# The Three Autoscalers

In the [scaling notes](../../kubernetes/kubernetes-scaling-horizontal/) we built an HPA. But HPA is one of _three_ autoscalers, and they operate at different layers - mixing them up is a classic design mistake:

| Autoscaler | Katman | Soru | Ne yapar |
|---|---|---|---|
| **HPA** | Pod sayısı | "Kaç pod?" | Metriğe göre replica sayısını değiştirir |
| **VPA** | Pod kaynakları | "Bir pod ne kadar ister?" | `requests`/`limits`'i otomatik ayarlar (restart ile) |
| **Cluster Autoscaler** | Node sayısı | "Kaç node?" | Pending pod'lar için node ekler, boş node'u çıkarır |

The layering matters because they solve _different_ problems: HPA handles variable load for a fixed-per-pod footprint; VPA fixes wrong resource estimates; CA fixes fleet capacity.

# Vertical Pod Autoscaler: Architecture

A [VPA](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) is actually three cooperating components - knowing them explains every behavior you'll observe:

- **Recommender** - watches metrics-server history, builds percentile histograms per container, and computes `Low`/`Target`/`Upper` recommendations with a built-in safety margin (target estimator adds headroom ~15% above observed median-ish usage)
- **Admission controller** - a webhook that patches `requests`/`limits` into pods _at creation time_
- **Updater** - the evictor. In `Auto` mode it deletes pods so they restart with the recommended resources, at a controlled eviction rate

The three `updateMode` values map onto those components:

- `Off`: only the recommender runs; publish recommendations, touch nothing
- `Initial`: recommendations applied to _newly created_ pods only (no eviction)
- `Auto`: evict pods to apply recommendations (the admission webhook patches their requests on restart)

A recommendation-only VPA:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: testram-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: synergychat-testram
  updatePolicy:
    updateMode: "Off"
```

## The HPA+VPA Conflict (and the Rule)

The trap: VPA `Auto` and CPU-based HPA fight over the same pods. VPA changes each pod's CPU _request_, which is the exact number HPA uses to compute utilization percentages - so the HPA's math silently corrupts. The standard resolutions:

- CPU-based **HPA** + memory-based **VPA** → compatible (different resources)
- Two autoscalers on the same resource → pick one
- [KEDA](https://keda.sh/) enters when the metric isn't CPU/memory at all (queue length, Kafka lag, HTTP RPS) - it's a scaler layer, not a VPA replacement

> Cluster Autoscaler + HPA is the pairing that _does_ compose cleanly: HPA grows pod count → pods go `Pending` → CA adds nodes → pods land. VPA in `Auto` is the one that plays badly, because its tool is eviction - and eviction is also what PodDisruptionBudgets try to guard against, so the updater respects PDBs (`minAvailable`) when evicting. Put a sane PDB in front of anything VPA `Auto` touches.

Minikube'da Cluster Autoscaler'ın gerçek bir karşılığı yok (tek node zaten); davranışı elle taklit edebilirsin:

```bash
minikube node add
minikube node list
```

Üretimde bu managed service'in işidir (GKE Autopilot node autoscaling'i varsayılan olarak yönetir; [nodes-basic](../../kubernetes/kubernetes-nodes-basic/) notlarındaki "managed offering node-level autoscaling" sözünün ta kendisi).

## How to Read a VPA Recommendation

1. Install the minimum for recommendation mode (CRDs + recommender):

```bash
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/latest/download/vpa-v1-crd-gen.yaml
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/latest/download/vpa-recommender-deployment.yaml
kubectl -n kube-system get pods -l app=vpa-recommender
```

2. Point a `Off`-mode VPA at the `synergychat-testram` deployment from v1:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: testram-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: synergychat-testram
  updatePolicy:
    updateMode: "Off"
```

3. Let it collect a few minutes of metrics, then read:

```bash
kubectl describe vpa testram-vpa
```

The `Status` section carries three bands per resource: `Lower Bound`, `Target`, `Upper Bound`.

4. Interpret them against what you hand-set in v1:

- `Target` = what the recommender would set as request (observed usage + safety margin)
- `Upper Bound` = the level at which throttling/OOM becomes likely if you go lower
- `Lower Bound` = the absolute floor for saving resources

> Compare `Target` with the static `256Mi` limit we chose by hand in the [vertical scaling](../../kubernetes/kubernetes-scaling-vertical/) notes. If the target says `9Mi` for a pod limited at `256Mi`, you just learned your limits were 25x too generous - that's wasted schedulable capacity on every node it runs on.

5. Cross-check with `kubectl top pod` - the recommendation is computed from _this_ history, so a pod that just restarted has thin data. VPA needs days of history for stable numbers; treat day-one recommendations as a first draft.

## How to Apply Recommendations in Controlled Mode

1. Switch to `Initial` - apply recommendations without evicting anything:

```yaml
updatePolicy:
  updateMode: "Initial"
```

2. Delete the pod so a fresh one is created (that's all `Initial` needs):

```bash
kubectl delete pod <testram-pod>
kubectl get pod <new-pod> -o jsonpath='{.spec.containers[0].resources}{"\n"}'
```

The `requests` should now match the VPA's recommendation instead of the manifest's static values - the webhook patched them at admission.

> Notice what this means for your YAML: the manifest's `resources` values are now only a starting point. If you later run `Auto`, live pods' requests will drift from git - and if you use [ArgoCD](../gitops-argocd/) with `selfHeal`, it will fight the VPA over pod specs. Decision to make consciously: either don't use `Auto` with git-managed manifests, or exclude `resources` from diffing (`ignoreDifferences`).

3. Optional: flip to `Auto` on a throwaway deployment and watch eviction happen (`kubectl get pods --watch` - the pod terminates and restarts with new requests). Never do this on something you care about without a PodDisruptionBudget first.

for more [taints-affinity-quotas](../taints-affinity-quotas/)
