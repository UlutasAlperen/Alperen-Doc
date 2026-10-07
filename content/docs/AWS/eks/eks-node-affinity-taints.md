---
title: "eks-node-affinity-taints"
weight: 60
---

# Node Affinity and Taints on EKS

Everything in my [taints, affinity and quotas](../../../kubernetes_v2/taints-affinity-quotas/) notes works on EKS **verbatim**. `kubectl taint`, tolerations, `nodeAffinity`, `topologySpreadConstraints` - the API doesn't know or care that AWS is underneath. What _is_ different is how nodes appear, how they get their labels and taints, and what happens to all of it when the autoscaler is involved.

> Kısa hatırlatma: **taint** node'un kapıdaki tabelası ("her pod giremez"), **toleration** pod'un geçiş izni, **nodeAffinity** ise "şu node'a git" talebi. İkisini karıştırmayın - toleration sizi oraya göndermez, sadece engeli kaldırır.

## How nodes get labels and taints on EKS

On kubeadm you `kubectl taint nodes ...` after joining. On EKS, worker nodes are cattle owned by a **managed node group**, and the right place to declare labels and taints is **when the group is created** - in the `eksctl` config:

```yaml
managedNodeGroups:
  - name: patientping-workers
    instanceType: t3.small
    desiredCapacity: 2
    privateNetworking: true
  - name: gpu-workers
    instanceType: g4dn.xlarge
    desiredCapacity: 1
    privateNetworking: true
    labels:
      workload: ai
    taints:
      - key: workload
        value: ai
        effect: NoSchedule
```

Why not just `kubectl taint` after the fact? You can - it works - but the next [node group scale-up](../eks-ha-cluster/) or instance refresh creates brand new nodes from the group's template. A taint applied by hand to an old node vanishes with that node; a taint in the group config comes back on every replacement.

> Not: node group'taki `labels`/`taints` node'lara **join anında** basılır. Elle `kubectl label` ile eklenenler node ölünce gider. "Cattle, not pets" - EKS bunu node'larda da zorunlu kılıyor.

## EKS's automatic labels

You don't label zone or instance type yourself; the [node admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#nodelabelselector) (via the AWS cloud provider and the AMI) stamps them on join:

- `eks.amazonaws.com/nodegroup` - node group adı
- `topology.kubernetes.io/zone`, `topology.kubernetes.io/region`
- `node.kubernetes.io/instance-type`
- AMI ailesine göre `eks.amazonaws.com/compute-type` (`ec2` veya `fargate`)

That means the `nodeAffinity` rules from your kubeadm notes work unchanged:

```yaml
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: topology.kubernetes.io/zone
                    operator: In
                    values: ["us-east-1a"]
```

## Fargate is a different animal

With [Fargate](https://docs.aws.amazon.com/eks/latest/userguide/fargate.html) there are **no nodes at all**, so taints and node affinity have nothing to bite on. Instead, a **Fargate profile** selects pods by namespace + labels and decides "these run serverless". Same idea (isolate workloads), different mechanism.

## The autoscaler interaction

If a pod can't schedule because of a taint, the [Cluster Autoscaler](../../../kubernetes_v2/vpa-cluster-autoscaler/) may respond by scaling up _another_ node group - or doing nothing, if no group can satisfy the pod. Taints and affinity are therefore not just scheduling rules; they're **capacity rules**. A misconfigured toleration can silently double your node count (and your bill).

## Assignment

**Prove that a taint from the node group config keeps a normal pod off the tainted nodes - and that a toleration lets a special pod in.**

**Cost check:** An extra `t3.small` node group of 1 node adds a few dollars per month. Delete the `gpu-workers`-style group when you're done (`eksctl delete nodegroup`).

1.  Add the `gpu-workers`-style node group from the config above to `patientping-eks.yaml` and apply it:

```bash
eksctl create nodegroup --cluster patientping-eks -f patientping-eks.yaml
```

2.  Deploy a plain pod with _no_ toleration, and one with a toleration:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: normal-pod
spec:
  containers: [{ name: web, image: nginx }]
---
apiVersion: v1
kind: Pod
metadata:
  name: ai-pod
spec:
  tolerations:
    - key: workload
      operator: Equal
      value: ai
      effect: NoSchedule
  nodeSelector:
    workload: ai
  containers: [{ name: train, image: nginx }]
```

```bash
kubectl apply -f taint-test.yaml
```

3.  Check where they landed:

```bash
kubectl get pods -o wide
kubectl describe pod normal-pod | grep -A3 Events
```

Expected: `ai-pod` runs on the tainted node (it has both the toleration _and_ the nodeAffinity via `nodeSelector`), `normal-pod` sits `Pending` with `0/3 nodes are available: 1 node(s) had untolerated taint...` in its events.

> Hatırlatma: `Pending` + "untolerated taint" mesajı taint'in **çalıştığını** gösterir - hata değil, doğrulamadır. Pod'u temizlemek için `kubectl delete pod normal-pod`.
