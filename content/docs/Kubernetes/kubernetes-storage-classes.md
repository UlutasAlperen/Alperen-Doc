---
title: "kubernetes-storage-classes"
weight: 21
---
# StorageClasses

In the [persistent volumes](../kubernetes-persistent-volumes/) chapter we created a PVC and a PV just... appeared. Nobody wrote a `PersistentVolume` object. So who decided its size, its disk type, and what happens when we delete it?

That somebody is a [StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/). Think of it as a _template_ for dynamically provisioned volumes: a name a PVC can ask for, plus the recipe the cluster uses to go create a real disk.

```bash
kubectl get storageclass
```

Minikube has one called `standard` (usually marked `default`). Longhorn, if you installed the [longhorn](../../kubernetes_v2/longhorn/) chapter, adds another. The `(default)` annotation is what makes `storageClassName` optional in a PVC - leave it out and you get that one.

# Anatomy of a StorageClass

Here's a full one, with every field that matters:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-retain
provisioner: rancher.io/local-path
parameters:
  type: ssd
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

- `provisioner`: the plugin that actually creates the volume (a cloud disk, a CSI driver like [Longhorn](../../kubernetes_v2/longhorn/), a local path)
- `parameters`: opaque key/values passed to the provisioner - what they mean depends entirely on the driver
- `reclaimPolicy`: what happens to the underlying volume when the PVC is deleted
- `volumeBindingMode`: when the volume gets created
- `allowVolumeExpansion`: whether you can grow a PVC later (you usually can't shrink)

**Küçük model:** PVC = "bana şu özellikte bir disk lazım" siparişi; StorageClass = o siparişi karşılayan katalog kalemi; provisioner = siparişi gerçekten getiren kargo şirketi.

`volumeBindingMode` deserves a second look. The default `Immediate` provisions the volume as soon as the PVC exists - which can pin a volume to a zone where no pod is going to run. `WaitForFirstConsumer` waits until a pod actually claims it, then provisions near the pod. For multi-zone clusters that's not a nicety, it's the difference between working and `Pending` forever.

# Access Modes

A PVC also states _how_ the volume may be mounted. Three modes, and they are about _nodes_, not pods:

- `ReadWriteOnce` (RWO): mounted read-write by a **single node** at a time
- `ReadOnlyMany` (ROX): mounted read-only by many nodes
- `ReadWriteMany` (RWX): mounted read-write by many nodes at once

> **Dikkat:** `ReadWriteOnce` pod başına değil, **node** başınadır. Aynı node üzerinde birden çok pod aynı RWO volume'ü aynı anda read-write mount edebilir. Cluster'ın tamamında tek bir pod çalıştırabilirim sanmak en yaygın yanlış anlaşılmadır. (Eski notlardaki "multiple pods at the same time" ifadesi de bu yüzden yanıltıcıydı - düzeltilmiş hali budur.)

Whether a driver actually _supports_ a mode is up to the provisioner. Most cloud block disks only do RWO; NFS-style and Longhorn RWX do more. Ask the StorageClass, don't assume.

# Reclaim Policies

When you delete a PVC, the PV behind it doesn't just vanish - `reclaimPolicy` decides its fate:

- `Delete`: the volume and its data go away with the claim (the default for dynamic provisioning, and the reason people lose databases)
- `Retain`: the volume survives in `Released` state, manually recoverable, with the data intact

`Retain` is the safety net for anything that matters. A released PV keeps the data but refuses new claims until you clean it manually (delete the old `claimRef` and rebind it, or wipe it and let it return to the pool).

Static provisioning ties in here too: sometimes you have a disk - a real one, plugged into a machine - and you want to hand it to the cluster yourself. That's a `PersistentVolume` you write by hand:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-manual-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data
```

No `provisioner`, no `storageClassName` magic - just a volume waiting for a claim. Bind it with a PVC that matches the size and access modes.

# Assignment

Let's watch `Retain` save our data from us.

1. Create a file called `fast-retain-sc.yaml` with the StorageClass above (`fast-retain`, `reclaimPolicy: Retain`). Apply it and check the class list:

```bash
kubectl apply -f fast-retain-sc.yaml
kubectl get sc
```

2. Create a PVC that asks for it (`api-pvc.yaml`):

- `kind`: `PersistentVolumeClaim`
- `spec/storageClassName`: `fast-retain`
- `spec/accessModes`: `["ReadWriteOnce"]`
- `spec/resources/requests/storage`: `1Gi`

3. Attach it to the `synergychat-api` deployment at `/persist`, post a few chat messages in the app, and verify the data is there.

4. Now delete the PVC and look at the PV:

```bash
kubectl delete pvc <pvc-name>
kubectl get pv
```

You want `Released`, not `Gone`. The data is still on the disk even though nothing is using it - that's `Retain` keeping its promise.

5. Repeat with `reclaimPolicy: Delete` on a throwaway class and watch the PV disappear entirely. Compare the two outcomes, then re-create the Retain setup for the rest of the course.

6. Bonus: try to grow the PVC after the fact:

```bash
kubectl patch pvc <pvc-name> -p '{"spec":{"resources":{"requests":{"storage":"2Gi"}}}}'
```

With `allowVolumeExpansion: true` on the class this works; without it the patch is accepted but the capacity never changes. Shrinking is never allowed.

for more [statefulsets](../kubernetes-statefulsets/)
