---
title: "kubeadm-upgrade-etcd"
weight: 13
---
# Cluster Upgrades

The [multi-node](../multi-node-kubeadm/) notes ended with a one-liner: "upgrade the control plane first, one minor version at a time". That's the policy - this is the practice.

[Upgrading](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/) a cluster is the most common maintenance task you will ever run, and the most common way to break one. The rules that keep it boring:

- **One minor version at a time.** `1.30` → `1.31`, never `1.30` → `1.32`. The version skew policy allows the apiserver to lead kubelets by one minor; anything more is unsupported and the upgrade will refuse
- **Control plane first, workers after.** The new apiserver must understand old kubelets, not the reverse
- **One node at a time.** Drain → upgrade → uncordon → next
- **Read the output.** `kubeadm upgrade plan` exists precisely because upgrades are not a single command

> The apt packages are version-pinned with `apt-mark hold` from the install notes. Every upgrade starts with `apt-mark unhold` and ends with `apt-mark hold` again - forgetting the re-hold is how a future `apt upgrade` silently moves your cluster a minor version.

# etcd Backup and Restore

Before any upgrade - before any _anything_ risky - take a backup. The multi-node notes covered `snapshot save`; this is the full cycle, because a backup you cannot restore is a hope, not a plan.

```bash
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  snapshot save /var/backups/etcd-$(date +%F).db
```

Verify the snapshot is usable _now_, not on the day you need it:

```bash
etcdctl snapshot status /var/backups/etcd-2025-01-01.db -w table
```

**Dikkat:** `snapshot restore` mevcut bir cluster'ın üstüne yazılmaz. Restore, **yeni bir data dir** ile yeni bir etcd ayağa demektir; eski data dir'i silmeden restore etmeye kalkarsan ne yaptığını bilen bir insan olmak yerine felaket tatbikatı yapmış olursun. Prosedür aşağıda adım adım var.

# Certificate Management

kubeadm clusters issue their own certificates with a one-year expiry. Nobody remembers this until the cluster stops serving traffic at 9am on a random Tuesday.

```bash
kubeadm certs check-expiration
kubeadm certs renew all
```

Renewing writes new certs to `/etc/kubernetes/pki` - the static pods must then be restarted to pick them up. The fastest way is to touch the manifest files and let the kubelet re-create the pods:

```bash
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/ && mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
```

> Worker kubelet certs also expire. `kubeadm certs renew kubelet-client-current` on the node, or configure kubelet certificate rotation and never think about it again.

# How to Upgrade the Control Plane

1. Take a backup and make sure you could restore it:

```bash
etcdctl ... snapshot save /var/backups/etcd-$(date +%F).db
etcdctl snapshot status /var/backups/etcd-$(date +%F).db -w table
```

2. See what you're allowed to move to:

```bash
apt-mark unhold kubeadm
apt-get update && apt-get install -y kubeadm=1.31.*-1.1
kubeadm upgrade plan
```

`upgrade plan` prints the current and target versions and warns about anything unsupported. Read it.

3. Apply the control-plane upgrade:

```bash
kubeadm upgrade apply v1.31.0
```

On additional control-plane nodes (see [ha-control-plane](../ha-control-plane/)), use `kubeadm upgrade node` instead - only the first node runs `apply`.

4. Upgrade kubelet and kubectl, then restart kubelet:

```bash
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.31.*-1.1 kubectl=1.31.*-1.1
apt-mark hold kubelet kubectl kubeadm
systemctl daemon-reload && systemctl restart kubelet
```

5. Verify before touching workers:

```bash
kubectl get nodes
kubectl get pods -n kube-system
kubectl version
```

`STATUS` = `Ready`, all `kube-system` pods running, no version skew surprises.

# How to Upgrade Worker Nodes

1. Cordon and drain - the [pod-priority-disruption](../pod-priority-disruption/) notes explain why this can stall:

```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

2. On the node, same dance as the control plane, minus `upgrade apply`:

```bash
apt-mark unhold kubeadm
apt-get update && apt-get install -y kubeadm=1.31.*-1.1
kubeadm upgrade node
apt-mark unhold kubelet
apt-get install -y kubelet=1.31.*-1.1
apt-mark hold kubelet kubeadm
systemctl daemon-reload && systemctl restart kubelet
```

3. Back on the control plane, uncordon and move to the next node:

```bash
kubectl uncordon <node-name>
kubectl get nodes
```

4. Repeat one node at a time. Never drain two at once unless your [PDB](../pod-priority-disruption/) and capacity say you can.

# How to Restore etcd from a Snapshot

The day the backup earns its keep:

1. Stop the static pod from fighting you - move the apiserver and etcd manifests aside:

```bash
mv /etc/kubernetes/manifests/kube-apiserver.yaml /etc/kubernetes/manifests/kube-etcd.yaml /root/
```

2. Move the old data dir out of the way (do not delete it until the restore is verified):

```bash
mv /var/lib/etcd /var/lib/etcd.broken
```

3. Restore the snapshot into a fresh data dir:

```bash
ETCDCTL_API=3 etcdctl snapshot restore /var/backups/etcd-2025-01-01.db \
  --data-dir=/var/lib/etcd \
  --name=<node-name> \
  --initial-cluster=<node-name>=https://<node-ip>:2380 \
  --initial-advertise-peer-urls=https://<node-ip>:2380
```

The `--name` and cluster URLs must match the member's identity in the snapshot - restoring with the wrong name is how you end up with an etcd that answers but has no leader.

4. Put the manifests back and watch the cluster return:

```bash
mv /root/kube-apiserver.yaml /root/kube-etcd.yaml /etc/kubernetes/manifests/
crictl ps
kubectl get nodes
```

5. Verify the data is the data you expected - a specific namespace, a specific deployment. Then and only then, remove `/var/lib/etcd.broken`.

> On a multi-member etcd cluster ([HA notes](../ha-control-plane/)), restoring one member from snapshot is a _membership_ operation: the restored member rejoins the cluster and syncs, or you rebuild the member entirely. `etcdctl member list` before you touch anything.

# How to Renew Certificates

1. See what's near expiry:

```bash
kubeadm certs check-expiration
```

2. Renew everything at once:

```bash
kubeadm certs renew all
```

3. Restart the static pods so they load the new certs - either touch the manifests as shown above, or reboot the node. Then confirm:

```bash
kubectl get --raw=/healthz
kubeadm certs check-expiration
```

4. Distribute the updated `admin.conf` to anyone holding an old kubeconfig, and update workers' kubelet certs if you use the client cert flow. A renewed CA cert (only on `kubeadm certs renew ca`, which you should almost never do) invalidates every cert in the cluster - treat that as a disaster recovery scenario, not maintenance.

**Türkçe özet - sıralama:** backup → `upgrade plan` → `upgrade apply` (CP) → node'ları tek tek drain+upgrade+uncordon → `certs check-expiration` takvime not et. Bu sırayı bozmadığın sürece upgrade bir bakım işlemidir, bir kumar değil.

for more [ha-control-plane](../ha-control-plane/)
