---
title: "ha-control-plane"
weight: 14
---
# High Availability

The [multi-node](../multi-node-kubeadm/) notes left a check on the wall: `--control-plane-endpoint=<VIP>` "makes later HA possible". This is later.

A cluster with one control-plane node has one single point of failure: kill that node and `kubectl` stops working, the scheduler stops placing pods, and every controller stops reconciling. The _workloads_ keep running (kubelets and containers don't need the apiserver to stay alive), but the cluster stops being managed. For a homelab that's an annoyance; for anything real it's an outage.

High availability means removing that single point of failure: multiple control-plane members, an etcd that tolerates member loss, and one stable endpoint clients can always reach.

**Küçük model:** tek kontrollü node = tek ışık anahtarı; 3 kontrollü node = 3 anahtar ve oybirliği. Işık yanık kalır çünkü biri düşse bile diğer ikisi kararı verir.

# Topologies

Two ways to arrange an HA control plane, and the only difference is _where etcd lives_:

| | Stacked etcd | External etcd |
|---|---|---|
| etcd nerede | Her CP node'unun üzerinde | Ayrı makinelerde |
| Toplam makine | 3 CP yeter (3+3) | 3 CP + 3 etcd (3+3) |
| Karmaşıklık | Düşük - kubeadm kurar | Yüksek - etcd'yi sen kurarsın |
| Kayıp toleransı | 1 üye (3'te) | 1 üye (3'te) |
| Kim kullanır | Çoğu kubeadm kurulumu | Büyük/regulated ortamlar |

Both need an **odd** number of members: 3 or 5. That's quorum - the majority that has to agree before anything is written. Two members is worse than one: either one dying takes the quorum away. Three members tolerate one loss; five tolerate two.

> Adding members without quorum math is how people build clusters that survive a failure _less_ well than before. `etcdctl endpoint status -w table` shows the member count and who the leader is - check it after every change.

# The Stable Endpoint

Clients need one address that survives node failures. That's `controlPlaneEndpoint` - a DNS name or VIP in front of an API server load balancer:

```
controlPlaneEndpoint: "k8s-api.local:6443"
```

The LB (HAProxy, nginx, keepalived + VIP, or a cloud LB) forwards to all healthy apiservers. kubeadm writes this endpoint into every kubeconfig and every component's flags at init time - which is why it must be decided _before_ the first `kubeadm init`, not retrofitted after.

> The apiserver LB only load-balances the API. Peer traffic between etcd members goes over port 2380 directly - don't put that behind the same LB.

# How to Add a Second Control Plane Node

1. On an existing control-plane node, generate a join command with a certificate key:

```bash
kubeadm init phase upload-certs --upload-certs
kubeadm token create --print-join-command
```

The `--certificate-key` from the first command is single-use and short-lived. The [multi-node](../multi-node-kubeadm/) notes used the same token flow for workers; the `--control-plane` flag and the cert key are what make this a control-plane join.

2. Prepare the new node exactly like the first one - swap off, kernel modules, containerd, matching versions ([multi-node](../multi-node-kubeadm/) prep steps verbatim).

3. Join it as a control plane member:

```bash
kubeadm join k8s-api.local:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <key>
```

4. Verify the quorum grew:

```bash
kubectl get nodes
etcdctl ... member list
kubectl get pods -n kube-system -o wide
```

You want three apiserver pods on three nodes, and `member list` showing three members.

# How to Set Up External etcd

1. Set up three dedicated etcd hosts - versions and certs matching your cluster ([kubeadm-upgrade-etcd](../kubeadm-upgrade-etcd/) covers cert layout). Generate the etcd PKI once, and hand every member its own cert signed by the same CA.

2. Configure each member with the peer and client URLs, and the full initial cluster:

```
--initial-cluster=etcd1=https://10.0.0.11:2380,etcd2=https://10.0.0.12:2380,etcd3=https://10.0.0.13:2380
--initial-advertise-peer-urls=https://10.0.0.11:2380
```

Start them, then check they formed a cluster:

```bash
etcdctl --endpoints=https://10.0.0.11:2379,... endpoint status -w table
```

3. Write the kubeadm config with an `external` etcd section pointing at all three endpoints and their client certs, then `kubeadm init` against it. The control-plane nodes never run etcd themselves.

4. Backups now have to target one of the external members - same `snapshot save` as always, different `--endpoints`.

# How to Test Failover

An HA setup you haven't tested is a hope, not a design.

1. Record the baseline - who's leader, how many members:

```bash
etcdctl ... endpoint status -w table
kubectl get nodes
```

2. Take one control-plane node down cleanly (or hard-fail it if you're testing the ugly case):

```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
# or: virsh destroy <vm> / pull the power
```

3. Prove the cluster still answers during the loss:

```bash
kubectl get nodes
kubectl run failover-test --image=nginx:1.27
kubectl get pods -o wide --watch
```

If `kubectl` responds and pods schedule on the remaining nodes, you have HA. If `kubectl` hangs, you tested the LB and discovered you don't have one.

4. Bring the node back and confirm it rejoins as a full member:

```bash
etcdctl ... member list
kubectl get nodes
```

5. Repeat for the other members until you've killed each one once. It's the only way to know.

> **Dikkat:** quorum düşerse (`etcdctl member list` "unstarted" / "learner" ya da leader yok) etcd yazmayı reddeder ve cluster **read-only** gibi davranır. Eksik üyeyi geri getirmek, yeni bir üye eklemekten daha hızlıdır - `etcdctl member remove` + yeni makineyle yeniden join, çoğu durumda kurtarma yoludur. Önce [backup](../kubeadm-upgrade-etcd/) alsan iyi olur.

for more [devops-control-plane](../devops-control-plane/)
