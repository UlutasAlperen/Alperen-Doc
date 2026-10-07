---
title: "eks-vs-kubeadm"
weight: 20
---

# EKS vs Your Own kubeadm Cluster

If you've been through my [multi-node kubeadm](../../../kubernetes_v2/multi-node-kubeadm/) notes, you've done it the hard way: Debian VMs on Proxmox, `containerd`, `kubeadm init`/`join`, Cilium as the CNI, and every package upgrade lands on your desk. That cluster is _yours_ - you know every screw.

EKS is the same Kubernetes API with most of the screws hidden. The point of this note is exactly what to compare: what disappears, what changes, and what stays stubbornly identical.

## What AWS takes off your plate

| Topic | kubeadm (your cluster) | EKS |
|---|---|---|
| Control plane | You install and run kube-apiserver, scheduler, controller-manager | AWS runs them across 3 AZs, patches them, gives you an SLA |
| etcd | You run it (stacked or [external](../../../kubernetes_v2/ha-control-plane/)), you snapshot it, you defrag it | AWS runs it; you never touch it. No `etcdctl`, no snapshots - AWS backs it up |
| Node bootstrap | Kernel modules, swap off, sysctl, containerd, k8s apt packages, `kubeadm join` | A managed node group: AWS builds the AMI (AL2 / Bottlerocket), joins the nodes for you |
| Upgrades | [`kubeadm upgrade`](../../../kubernetes_v2/kubeadm-upgrade-etcd/), one minor at a time, control plane first, your calendar | Pick a version in the console/CLI; AWS rolls the control plane, you roll the node group AMIs |
| Endpoint | `--control-plane-endpoint=<VIP>`, your keepalived/nginx LB | A regional API endpoint (public and/or private) managed by AWS |
| Cost | The VMs. Nothing else | ~**$0.10/hour** for the control plane, _plus_ worker nodes |

> Not: kubeadm'de `--control-plane-endpoint` VIP'i ve failover'ı sen kuruyordun. EKS'te o iş bitti - endpoint regional DNS, arkasında AWS'in çok-AZ'li control plane'i var. [ha-control-plane](../../../kubernetes_v2/ha-control-plane/) notundaki "stable endpoint" bölümünün tamamı AWS'e devredilmiş durumda.

## What changes but is still yours

- **CNI:** kubeadm'de Cilium kurmuştuk ([notlar](../../../kubernetes_v2/multi-node-kubeadm/)). EKS varsayılan olarak [VPC CNI](https://docs.aws.amazon.com/eks/latest/userguide/pod-networking.html) kullanır - pod IP'leri **VPC'nin** IP havuzundan gelir. Bu, pod'ların VPC'de "gerçek" IP'lerle dolaşması demek (güzel: VPC SG'leri pod'ları da korur), ama dikkat: private subnet IP'leri hızla tükenir. Büyük kümelerde `--max-pods`, prefix delegation veya Cilium'a geçiş gerekir.
- **IAM ↔ Kubernetes RBAC:** kubeadm'de olmayan bir katman. EKS'te AWS IAM ile Kubernetes kimlikleri köprülenir: [IRSA](../eks-connect-s3/) (pod bazlı roller) ve EKS Access Entries (`aws-auth` ConfigMap'inin modern hali). Biz bunu [S3](../eks-connect-s3/) ve [RDS](../eks-use-rds/) derslerinde kullandık.
- **Node lifecycle:** `kubectl delete node` + VM'yi yakmak yerine managed node group'ları `eksctl scale` / console ile büyütürsün. Node group bir launch template gibidir - yenisini eklersin, eskisini kapatırsın.
- **Addon'lar:** kubeadm'de Cilium/`metrics-server`'ı elle kurdun; EKS'te `eksctl utils associate-iam-oidc-provider` + addon katalogu (VPC CNI, CoreDNS, kube-proxy, eks-pod-identity-agent) tek yerden.

## What is literally the same

This is the part people underestimate. Once your `kubeconfig` points at EKS:

- `kubectl apply`, `Deployment`, `Service`, `StatefulSet`, `Job` - aynı
- Taints, tolerations, affinity, topology spread - aynı (EKS'e özgü detaylar için [eks-node-affinity-taints](../eks-node-affinity-taints/))
- RBAC (`Role`, `ClusterRole`, `RoleBinding`) - aynı
- CRD'ler, operator'lar, Helm chart'ları - aynı

Kısacası: kubeadm notlarında öğrendiğin Kubernetes bilgisi EKS'te birebir geçerli. Kaybedilen şey _operasyonel yük_, kazanılan şey _AWS faturası_.

## Assignment

**Compare your own cluster against EKS - and prove the API really is the same.**

1.  Point `kubectl` at the EKS cluster (`eksctl utils write-kubeconfig --cluster patientping-eks`), then run the same three commands you run on your homelab cluster:

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl api-resources | wc -l
```

2.  On your kubeadm cluster, run `kubectl get nodes -o wide` too. Note the differences: `INTERNAL-IP`'ler EKS'te VPC IP'leri, `VERSION`'lar AWS'in yönettiği AMI'lerden geliyor, `ROLES` sütununda `control-plane` node'u **yok** (control plane görünür değil).
3.  List what you'd have to do on kubeadm to get what EKS gave you for free (HA control plane, etcd backups, upgrades) - that list is the real price of "free" Kubernetes.

> Sanity check: `kubectl version` - her iki tarafta da aynı komut, aynı çıktı formatı. API sunucusu kim olursa olsun Kubernetes Kubernetes'tir.
