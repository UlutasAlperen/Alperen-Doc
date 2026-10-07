---
title: "eks-ha-cluster"
weight: 70
---

# High Availability on EKS

In the [HA notes](../../../kubernetes_v2/ha-control-plane/) for the kubeadm cluster, HA was a project: stacked etcd members, an external etcd option, a keepalived VIP in front of the apiserver, and a failover test where you pull the power on a VM. On EKS, most of that project is **already done for you** - but "done for you" is not the same as "you have no homework".

## The control plane: AWS's job

When you create `patientping-eks`, AWS runs the control plane across **three Availability Zones**, with an etcd cluster that tolerates member loss and a regional API endpoint your `kubeconfig` already points at. There is no VIP to keep alive, no `--control-plane-endpoint` to design, no etcd quorum to babysit.

What you give up in exchange: **no access**. No `etcdctl snapshot save`, no manual defrag, no `kubectl exec` into the apiserver. Backups and restores of the control plane are AWS's responsibility (they exist - you just can't take them). For most teams that trade is a bargain; if you need etcd-level forensics, you're on kubeadm for a reason.

> Not: [ha-control-plane](../../../kubernetes_v2/ha-control-plane/) notundaki "stable endpoint" ve "external etcd" bölümleri EKS'te anlamsız - endpoint regional DNS, etcd AWS'in kontrolünde. Senin HA sorumluluğun sadece **worker katmanı**.

## The worker layer: your job

Control plane HA without worker HA is a fancy car with one wheel. Three things make the worker layer survive an AZ loss:

1.  **Spread the node group across AZs.** A managed node group created with multiple subnets (like the two private subnets in our [cluster config](../eks-cluster/)) places nodes in each. For real AZ coverage, give it all the private subnets in the VPC.
2.  **Topology spread constraints** - the same tool from your [scheduling notes](../../../kubernetes_v2/taints-affinity-quotas/):

```yaml
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels: { app: patientping-web }
```

3.  **Pod Disruption Budgets** - from the [pod priority and disruption](../../../kubernetes_v2/pod-priority-disruption/) notes. A PDB is what keeps a node drain (or a node group upgrade) from killing every replica at once:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: patientping-web-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels: { app: patientping-web }
```

## Node replacement

On kubeadm, replacing a dead node means a new VM and `kubeadm join`. On EKS, delete the instance (or let the node group's instance refresh do it) and the group launches a replacement that joins automatically - labels and taints included (see [eks-node-affinity-taints](../eks-node-affinity-taints/)). For unplanned node loss, [Cluster Autoscaler](../../../kubernetes_v2/vpa-cluster-autoscaler/) (or Karpenter) keeps a buffer of capacity so the replacement is fast.

## Assignment

**Make the PatientPing deployment survive losing an entire node - and prove it.**

**Cost check:** Spreading nodes across AZs doesn't cost extra; you're already paying for the nodes. The drain test below is free, just disruptive.

1.  Patch the deployment with the topology spread constraint above and make sure the PDB exists:

```bash
kubectl apply -f patientping-web.yaml
kubectl apply -f patientping-web-pdb.yaml
kubectl get pdb
```

2.  Confirm both replicas are on **different nodes** (and therefore, in our multi-AZ group, different AZs):

```bash
kubectl get pods -o wide
kubectl get nodes --show-labels | grep zone
```

3.  Simulate losing a node - the EKS version of the "pull the power" failover test from your HA notes:

```bash
NODE=$(kubectl get pods -l app=patientping-web -o wide | awk 'NR==2{print $7}')
kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data
```

4.  Watch the PDB and the pods do their job:

```bash
kubectl get pods -w
```

The evicted replica should be recreated on the surviving node/zone. `kubectl get pdb` shows `ALLOWED DISRUPTIONS: 1` before the drain - that's the budget letting exactly one replica go down at a time.

5.  Bring the node back (uncordon it) and check the node group in the console - or simply delete the drained node's instance and watch a fresh one join.

```bash
kubectl uncordon $NODE
```

> Sanity check: tek node'a `kill -9` atmak yerine `kubectl drain` kullanın - drain, PDB'ye saygılı ve kontrollü bir tahliyedir. Gerçek AZ kaybını test etmek isterseniz node'un instance'ını console'dan terminate edin; node group otomatik yenisini açar.
