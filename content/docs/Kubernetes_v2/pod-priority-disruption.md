---
title: "pod-priority-disruption"
weight: 11
---
# DaemonSet

Deployments spread replicas wherever there's room. Sometimes "wherever there's room" is the wrong answer - a log shipper, a CSI node agent, or a CNI plugin needs exactly one pod per node, no more, no less. That's a [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/):

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-agent
spec:
  selector:
    matchLabels:
      app: node-agent
  template:
    metadata:
      labels:
        app: node-agent
    spec:
      tolerations:
        - operator: Exists
      containers:
        - name: agent
          image: busybox:1.36
          command: ["sh", "-c", "sleep infinity"]
```

Look at that toleration - `operator: Exists` tolerates _everything_, which is exactly what you want when the DaemonSet must land on control-plane nodes too. Most system DaemonSets (kube-proxy, Longhorn managers, Calico's node agents) ship with this, because a plugin that avoids the control plane is a plugin that half-works.

A DaemonSet uses the same scheduling machinery as everything else from the [taints and affinity](../taints-affinity-quotas/) notes - you can pin it with node affinity and evict it with taints. What's different is the controller: it watches _nodes_ instead of replicas.

**Küçük model:** Deployment = "3 tane çalışsın, nerede olduğu fark etmez"; DaemonSet = "her node'da tam 1 tane çalışsın". Birincisi iş yükü, ikincisi altyapı.

# PriorityClass

When the cluster runs out of room, somebody doesn't get scheduled. [PriorityClass](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/) decides who loses:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "SynergyChat api pods"
```

A pod claims it with `priorityClassName: high-priority`. Higher `value` wins. Higher-priority pods can also _preempt_ - evict lower-priority pods to free up room - which is exactly as dramatic as it sounds.

Kubernetes ships two reserved classes: `system-cluster-critical` and `system-node-critical`, both enormous values. That's why, when the node is packed, your app dies before CoreDNS does - by design.

> Priority is not the same as requests. Requests decide what a pod _asks_ for; priority decides who gets asked first. A tiny high-priority pod still fits in tight space; a huge low-priority pod is the first to go.

# PodDisruptionBudget

Now the uncomfortable question: when a node is drained for maintenance ([multi-node](../multi-node-kubeadm/) notes do this routinely), how many of your pods may be down at once? The Deployment's `maxUnavailable` governs rolling updates, but _voluntary_ evictions (drain, upgrade, scale-down) are governed by a [PodDisruptionBudget](https://kubernetes.io/docs/concepts/workloads/pod-disruption-budget/):

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: synergychat-api-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: synergychat-api
```

Or the inverse form: `maxUnavailable: 1`. The budget applies to the pods matching the selector, and the eviction API refuses to take down more than it allows.

The sharp edge: a PDB that says `minAvailable: 2` on a 2-replica Deployment makes `kubectl drain` hang forever. The drain is respecting your budget - it simply cannot evict anything without violating it. That's the constraint working, not a bug, and it's why "drain never finishes" is almost always a PDB.

> The [VPA](../vpa-cluster-autoscaler/) notes mentioned PDB in passing because VPA's updater evicts pods the same way `drain` does - both go through the eviction API, both respect the budget. One budget per app, not per tool.

**Türkçe özet - üçlü ilişki:**
- **DaemonSet** = her node'da 1 pod (altyapı ajanları)
- **PriorityClass** = yer yokken kim yaşar, kim ölür
- **PDB** = bakım/drain sırasında en az kaç pod ayakta kalır

# How to Run One Agent Per Node with a DaemonSet

1. Apply the DaemonSet above and watch it populate every node:

```bash
kubectl apply -f node-agent.yaml
kubectl get ds node-agent
kubectl get pods -l app=node-agent -o wide
```

One pod per node, including the control plane if the toleration is there.

2. Add a node and confirm the DaemonSet notices without being told:

```bash
# on a multi-node cluster, add or uncordon a node
kubectl get pods -l app=node-agent -o wide --watch
```

3. Remove the toleration and re-apply - control-plane pods vanish while worker pods stay. That asymmetry is the toleration, not magic.

4. Clean up:

```bash
kubectl delete ds node-agent
```

# How to Protect a Service During Node Drain

1. Scale the api deployment to 3 replicas and create the PDB:

```bash
kubectl scale deploy synergychat-api --replicas=3
kubectl apply -f synergychat-api-pdb.yaml
kubectl get pdb
```

`ALLOWED DISRUPTIONS: 1` means one pod may be evicted at a time.

2. Drain one node:

```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

Watch it evict one pod and then stall on the second. `kubectl get pdb` shows `ALLOWED DISRUPTIONS: 0` - the drain is waiting for the first pod to become ready elsewhere.

3. Unblock it the honest way - add capacity first, then drain. On a single-node Minikube you can also observe the API's behavior directly:

```bash
kubectl get pdb synergychat-api-pdb -o yaml | sed -n '/status/,$p'
```

4. Tighten the budget to `minAvailable: 3` and drain again. The drain now refuses to move anything. Delete the PDB and the drain proceeds - proof the budget, not the drain command, is in charge.

> **Dikkat:** `minAvailable` replica sayısına eşit veya büyükse drain **asla** bitemez. Node bakımına çıkarken bütçeyi (ya da replica sayısını) bunu bilerek ayarla - aksi hâlde "node drain takıldı" diye 2 saat debug edersin.

for more [multi-node-kubeadm](../multi-node-kubeadm/)
