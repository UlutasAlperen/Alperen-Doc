---
title: "taints-affinity-quotas"
weight: 8
---
# Taints and Tolerations

On a single-node Minikube cluster scheduling is trivial - there's nowhere else to put a pod. The moment you add nodes ([multi-node notes](multi-node-kubeadm-k3s/) are coming), scheduling decisions become real. [Taints](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) mark a node as "keep away", and _tolerations_ let specific pods opt in anyway:

```bash
kubectl taint nodes <node-name> workload=ai:NoSchedule
```

The three effects behave differently:

- `NoSchedule`: new pods without a matching toleration can't be scheduled here (running pods are left alone)
- `PreferNoSchedule`: soft version - scheduler _tries_ to avoid, but won't reject
- `NoExecute`: also _evicts_ pods already running there that lack the toleration - the aggressive one

The matching toleration in a pod spec:

```yaml
tolerations:
  - key: "workload"
    operator: "Equal"
    value: "ai"
    effect: "NoSchedule"
```

`operator: "Exists"` (no value) matches any value for that key. And some taints are _well-known_ - k8s itself taints nodes when things go wrong (`node.kubernetes.io/not-ready`, `.../unreachable`, `memory-pressure`); these auto-expire when the condition clears, which is why pods can't schedule onto an unhealthy node even though you never taint it.

**Küçük model:** taint node'a "yasak" tabelası asar, toleration o tabelanın geçiş iznidir. Yaygın kullanım: GPU node'ları (sadece AI job'lar girsin), control-plane node'ları (kubeadm bunu zaten yapar), spot instance'lar (`PreferNoSchedule` + toleration ile "tercihen kaçın, ama gerektiğinde kullan").

# Affinity

Taints push pods _away_ from nodes. [Affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) pulls them _toward_ - or away from - each other. Three flavors:

**nodeSelector** - the simple label match:

```yaml
spec:
  nodeSelector:
    disktype: ssd
```

**nodeAffinity** - nodeSelector with boolean expressions and soft variants:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: disktype
              operator: In
              values: ["ssd", "nvme"]
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
            - key: zone
              operator: In
              values: ["eu-west"]
```

- `required...` = hard rule; no node matches → pod stays `Pending` (this is a [troubleshooting](kubernetes-troubleshooting/) classic)
- `preferred...` = soft preference with a weight; used when "best effort" is acceptable
- `IgnoredDuringExecution` = if a running pod's node stops matching, the pod keeps running (only _new_ pods are constrained) - there's a `requiredDuringSchedulingRequiredDuringExecution` beta feature for "please move me" semantics

**podAntiAffinity** - spread or co-locate pods _relative to each other_ (the `topologyKey` decides what counts as "different domain" - hostname = per node, zone = per availability zone).

# Topology Spread Constraints

podAntiAffinity is binary per node. [Topology spread](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/) expresses the modern, more nuanced goal: _skew_:

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: synergychat-web
```

`maxSkew: 1` means: across all nodes matching the `topologyKey` domain, the difference between the node with the most and least matching pods may be at most 1. With 3 replicas over 3 nodes you get exactly one pod per node; with 4 replicas you get 2/1/1 - never 3/1/0. `whenUnsatisfiable: DoNotSchedule` makes it a hard constraint; `ScheduleAnyway` is the soft mode.

Why prefer this over podAntiAffinity? Anti-affinity refuses to co-locate _any_ two replicas - at scale (50 replicas, 10 nodes) that's unsatisfiable and pods pend forever. Spread constraints quantify fairness instead of forbidding co-location.

# ResourceQuota and LimitRange

Taints and affinity decide _where_ pods go. [Quotas](https://kubernetes.io/docs/concepts/policy/resource-quota/) decide _how much_ a namespace may consume in total:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: backend-quota
  namespace: backend
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
```

A [LimitRange](https://kubernetes.io/docs/concepts/policy/resource-limits/) complements it with per-container defaults, so devs don't have to remember resources on every pod:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
spec:
  limits:
    - default:
        cpu: "500m"
        memory: 512Mi
      defaultRequest:
        cpu: "100m"
        memory: 128Mi
      type: Container
```

> The quota-LimitRange interaction has a sharp edge: a pod with _no_ resources declared can't be admitted to a namespace with CPU/memory quota, because it can't be accounted. With a LimitRange present, undeclared resources get defaults and admission succeeds - that's why the pair ships together, not alone.

Also worth knowing: quotas can be scoped - `scopeSelector` with `Terminating`/`NotTerminating` or `BestEffort`/`NotBestEffort` lets you, say, cap only bursty jobs while leaving long-running services alone.

## How to Reserve a Node for Special Workloads

1. Taint the node and verify the taint took:

```bash
kubectl taint nodes <node-name> team=backend:NoSchedule
kubectl describe node <node-name> | grep -iA3 taints
```

2. Delete a `synergychat-web` pod and watch its replacement go `Pending`:

```bash
kubectl delete pod <web-pod>
kubectl describe pod <new-pod>
```

Find the event: `node(s) had untolerated taint {team: backend}` - the scheduler is telling you the only node is off-limits.

3. Give the web deployment the toleration (pod template):

```yaml
tolerations:
  - key: "team"
    operator: "Equal"
    value: "backend"
    effect: "NoSchedule"
```

Apply and watch it schedule. Then remove the taint to restore the shared setup:

```bash
kubectl taint nodes <node-name> team=backend:NoSchedule-
```

> Note the trailing `-` on the taint command - that's kubectl's "remove" suffix, same as labels. Forgetting it creates a duplicate taint error that reads like a mystery.

## How to Spread Replicas Across Nodes

1. Add to the web deployment's pod spec:

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: synergychat-web
```

2. Scale to more replicas than you have nodes, and watch placement:

```bash
kubectl get pods -o wide --watch
```

3. Delete one pod and watch the replacement avoid clustering on an already-heavy node. With `whenUnsatisfiable: DoNotSchedule` on a tight cluster, some pods may stay `Pending` until capacity allows skew 1 - that's the constraint working, not a bug; switch to `ScheduleAnyway` if you'd rather degrade gracefully.

## How to Cap a Namespace with ResourceQuota

1. Create a sandbox namespace with a strict pod budget:

```bash
kubectl create ns sandbox
kubectl -n sandbox apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: sandbox-quota
spec:
  hard:
    pods: "3"
    requests.memory: 1Gi
EOF
```

2. Deploy a 5-replica nginx there:

```bash
kubectl -n sandbox create deployment nginx --image=nginx:1.27 --replicas=5
kubectl -n sandbox get deployment nginx
```

`3/5` ready, and the rest never come up.

3. Find the verdict - quota exhaustion shows up in deployment conditions and events, not pod statuses (there are no pods to describe!):

```bash
kubectl -n sandbox describe deployment nginx | tail -n 5
kubectl -n sandbox get resourcequota sandbox-quota -o yaml
```

The quota's `status.used` vs `hard` tells the story.

4. Now add the LimitRange (defaults above) to the same namespace, delete the deployment, redeploy with _no_ resource declarations at all - the pods inherit defaults and count against the quota automatically. That's the mechanism that keeps "forgot to set requests" developers from bypassing your accounting.

for more [multi-node-kubeadm-k3s](multi-node-kubeadm-k3s/)
