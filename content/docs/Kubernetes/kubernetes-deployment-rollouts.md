---
title: "kubernetes-deployment-rollouts"
weight: 5
---
# Rolling Updates

In the [deployments](../kubernetes-deployments-minikube/) chapter we saw that a Deployment keeps the right number of pods alive no matter what. And in [probes](../kubernetes-probes/) we made those pods _smart_ enough to fail health checks before they take traffic. Now the obvious question: what happens when we want to change the app itself - a new image, a new env var, a bug fix?

Remember the hydra? A [rolling update](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy) is the same trick applied to software versions: we grow new heads _while_ chopping off the old ones. Users see one consistent service the whole time, and if the new heads turn out to be poisonous, we grow the old ones back.

The default strategy is exactly that - `RollingUpdate`:

```yaml
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

- `maxSurge`: how many pods _above_ the desired count may exist during the update (`1` = one extra pod at a time)
- `maxUnavailable`: how many pods _below_ the desired count may be missing (`0` = never drop below 3 ready pods)

**Küçük model:** `maxUnavailable` "kaç pod eksik kalabilir", `maxSurge` "kaç pod fazla açılabilir". İkisini birlikte `0`/`1` yaparsan güncelleme boyunca kapasiten hiç düşmez - pahalı ama güvenli. `maxSurge: 0` + `maxUnavailable: 1` ise ucuz ama her adımda bir pod eksiksin.

The other option is `Recreate`: kill _everything_, then start the new version. Zero overlap, guaranteed downtime. Frankly I've only used it for apps that simply can't run two versions at once against the same database.

> The rollout is driven by the probes from the previous chapter. A new pod counts as "done" only when its readiness probe passes - which is why a bad image doesn't take down the service, it just stops the rollout in its tracks. That stall is a _feature_.

# Updating an Image

There are two ways to hand a Deployment a new image.

**The imperative way:**

```bash
kubectl set image deployment/synergychat-api api=bootdotdev/synergychat-api:v2
```

**Or the declarative way,** change the image in `api-deployment.yaml` and `kubectl apply -f` it again. Same result - but only this one lives in your git repo. You know which one I prefer.

Whatever you pick, watch it happen:

```bash
kubectl rollout status deployment/synergychat-api
```

You'll see one pod replaced at a time. Meanwhile, a second ReplicaSet quietly appears behind the scenes:

```bash
kubectl get replicaset
```

Each change creates a new ReplicaSet (and a new _revision_) and scales the old one down to zero. The old ReplicaSets stick around so we can go back.

# Rolling Back

Here's the rub: sooner or later you will ship a broken image. Not "maybe" - you will. Deployments keep revision history precisely for that day.

See what we've shipped:

```bash
kubectl rollout history deployment/synergychat-api
```

Roll back to the previous revision:

```bash
kubectl rollout undo deployment/synergychat-api
```

Or jump to a specific old revision:

```bash
kubectl rollout undo deployment/synergychat-api --to-revision=2
```

`kubectl rollout pause` / `kubectl rollout resume` lets you stage several changes and only start moving pods when you say so - useful when a config change and an image change need to go out together.

> How much history is kept is controlled by `revisionHistoryLimit` (default `10`). If you set it to `0` to save a few ReplicaSets, you also throw away `rollout undo`. There is no free lunch.

# Assignment

Let's break our own API on purpose and then save it.

1. Update the `synergychat-api` deployment to a garbage image tag:

```bash
kubectl set image deployment/synergychat-api api=bootdotdev/synergychat-api:does-not-exist
```

2. Watch the rollout:

```bash
kubectl rollout status deployment/synergychat-api
```

The status will hang - the new pod can't pull the image (`ImagePullBackOff`), and because it never becomes ready, `maxUnavailable: 0` protects the rest of the service. Old pods keep serving traffic. Go ahead and check:

```bash
kubectl get pods
```

3. Open a second terminal and keep the users happy (proof the service never went down):

```bash
kubectl port-forward svc/web-service 8080:80
```

4. Now undo the damage:

```bash
kubectl rollout undo deployment/synergychat-api
kubectl rollout history deployment/synergychat-api
```

5. Verify the API is healthy again:

```bash
kubectl get pods
kubectl exec <api-pod> -- curl -s localhost:8080/healthz
```

If the pods are `Running` and `/healthz` answers `200`, congrats - you just performed the single most common production recovery task in Kubernetes.

for more [yaml-configurations](../kubernetes-yaml-configurations/)
