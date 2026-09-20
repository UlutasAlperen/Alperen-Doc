---
title: "longhorn"
weight: 10
---
# Longhorn

Everything we've done with storage so far lived on _one_ node. In the [v1 notes](../../kubernetes/kubernetes-persistent-volumes/) a PVC was a local disk on (the only) Minikube node, and in the [multi-node notes](multi-node-kubeadm-k3s/) our workers each keep their own local directories. That means the day `kworker1` dies, every PVC hosted on it dies with it - block storage was the last single point of failure in our homelab.

[Longhorn](https://longhorn.io/docs/) fixes that: it's a distributed block storage system built as a CSI driver, purpose-made for Kubernetes. Every volume it serves is split into an **engine** plus **N replicas** (default 3) that live on _different_ nodes - scheduled exactly like pods are.

## The Architecture

Three pieces matter:

- **longhorn-manager** - a DaemonSet that owns all the bookkeeping (volumes, replicas, settings) and talks to the Kubernetes API
- **longhorn-driver** - the CSI plugin. Pods request volumes through PVCs; this is the component that creates/attaches them
- **instance-manager** pods - the heavy lifters. Each volume gets an **engine** process (one per volume) and its **replicas** run here, distributed across nodes

The data path for a single write:

```text
pod → CSI attach (iSCSI) → longhorn engine → write to all replicas → ACK
```

Reads are served by the engine from the replicas. The engine/replica split is Longhorn's signature: when the node hosting the engine dies, a new engine is started on another node and replicas are reattached. When a replica's node dies, the remaining replicas keep serving and Longhorn starts rebuilding a new replica in the background.

> Replication here is **synchronous** at the block layer: the engine considers a write done once it's persisted on the replicas. This is protection against node/disk loss - and it is _not_ the same thing as PostgreSQL's streaming replication, which we'll layer on top in the next chapters.

## Snapshot vs Backup

The distinction that saves you from a false sense of safety:

| | Snapshot | Backup |
|---|---|---|
| Nerede | Longhorn cluster'ın kendi node'larında | Dışarıda: S3/NFS (cluster dışı) |
| Ne korur | Yanlış silme, bad upgrade sonrası geri dönüş | Tüm cluster'ın kaybı |
| Nasıl | CoW anlık görüntü | Tarayıcı (sync agent) ile uzun süreli dosya |

**Türkçe özet:** snapshot'lar cluster ile birlikte yaşar - cluster yanarsa snapshot da yanar. Gerçek backup her zaman S3/NFS'e. Homelab'da en azından snapshot + offsite backup kombinasyonunu düşün.

## The Honest Requirements

- **iscsid** on every node - the CSI attachment path is iSCSI-based. Missing `iscsid` is the #1 "volumes stuck in attaching" cause
- Enough free disk under `/var/lib/longhorn` on each node (the default data location)
- A Linux kernel ≥ 4.18 (5.8+ recommended)
- NFS client only if you want RWX volumes (the NFS-server-based sharing layer)
- **A real multi-node cluster.** On Minikube's single node Longhorn installs and runs, but replicas collapse onto one node: zero redundancy. Test it there only to see the UI; this chapter assumes the 3-node cluster from the [multi-node notes](multi-node-kubeadm-k3s/)

## How to Prepare the Nodes

1. On _every_ node (all three), install the iSCSI pieces:

```bash
sudo apt-get install -y open-iscsi
sudo systemctl enable --now iscsid
sudo systemctl status iscsid --no-pager
```

> Do this _before_ installing Longhorn. If `iscsid` is missing, the CSI driver installs fine and then the first volume attach hangs forever - a `Pending` that reads like a scheduling bug but isn't. This is exactly the kind of trap the [troubleshooting](kubernetes-troubleshooting/) checklist is for.

2. Check disk headroom on each node - Longhorn stores replicas here:

```bash
df -h /var/lib/longhorn 2>/dev/null || df -h /
```

3. If `multipathd` is running, make sure it won't grab Longhorn's devices (on stock Debian VMs it usually isn't - verify rather than assume):

```bash
systemctl is-active multipathd || true
```

## How to Install Longhorn with Helm

1. Add the repo and install into a dedicated namespace:

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
kubectl create namespace longhorn-system
helm install longhorn longhorn/longhorn --namespace longhorn-system --version 1.7.3
```

2. Wait for the fleet to come up (there are many pods; that's normal):

```bash
kubectl -n longhorn-system get pods --watch
```

You want `longhorn-manager` pods on every node, `longhorn-driver-deployer` completed, and CSI pods (`csi-attacher`, `csi-provisioner`, `longhorn-csi-plugin`) running.

3. Verify the StorageClass was created and check the UI:

```bash
kubectl get sc
kubectl -n longhorn-system port-forward svc/longhorn-frontend 8085:80
```

Open `http://localhost:8085` - the UI shows nodes, disks, volumes and their replica placement. Seeing the 3 replicas of a volume spread across your 3 nodes is the "aha" moment of this chapter.

## How to Make Longhorn the Default StorageClass

Our cluster already has a default SC (Minikube's `standard`, or k3s' `local-path` from the [multi-node notes](multi-node-kubeadm-k3s/)). Two defaults = new PVCs silently going to the wrong backend.

1. Demote the old one and promote Longhorn:

```bash
kubectl patch sc standard -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
kubectl patch sc longhorn -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl get sc
```

The default is the one with `(default)` suffix.

2. Existing PVCs keep their original class - this only affects _new_ claims. If you already have stateful workloads on the old SC, plan a migration instead of assuming a repatch moves them.

## How to Test Node Failover with a Real Workload

1. Deploy a writer so we can watch the volume migrate:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: writer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: writer
  template:
    metadata:
      labels:
        app: writer
    spec:
      containers:
        - name: shell
          image: busybox:1.36
          command: ["sh", "-c", "echo $(date) hello >> /data/heartbeat.log && sleep 5"]
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: data-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 2Gi
```

2. Find where the volume lives, then take that node out of scheduling (simulating a failure, the polite way):

```bash
kubectl -n longhorn-system get volumes.longhorn.io
kubectl cordon <node-with-the-pod>
kubectl delete pod <writer-pod>
kubectl get pods -o wide --watch
```

3. Watch the sequence: new pod schedules on another node → Longhorn stops the engine on the old node, starts it on the new one, reattaches → pod starts, `/data/heartbeat.log` continues from where it left off. In the UI you'll see the volume flip nodes while its 3 replicas stay spread.

> This is the Longhorn contract: the pod is scheduled anywhere, its block volume follows it. It's not magic - it's replicas + an engine that can move. Un-cordon the node afterwards (`kubectl uncordon <node>`), or your next test surprises you.

4. Also test the sad path once: stop a node hard (VM shutdown in Proxmox) instead of cordoning, and watch Longhorn mark the replica as failed and rebuild it when the node returns.

for more [postgresql-statefulset-replication](postgresql-statefulset-replication/)
