---
title: "cluster-extensions"
weight: 3
---
# Extension Interfaces

Everything we've installed so far - the [CNI plugin](../multi-node-kubeadm/) that gives pods their network, the [Longhorn](../longhorn/) CSI driver that gives them disks, the [containerd](../multi-node-kubeadm/) runtime that runs them - shares one thing in common: none of them are "Kubernetes". They're plugins, and Kubernetes just provides the sockets they plug into.

Those sockets are the extension interfaces, and knowing them turns "the cluster is broken" into "the _network plugin_ is broken":

| Arayüz | Ne işe yarar | Kim implement eder |
|---|---|---|
| CNI | Pod'lara IP ve network connectivity | Calico, Flannel, Cilium, Weave |
| CSI | Volume provisioning, attach, mount | Longhorn, cloud disk drivers, Ceph |
| CRI | Container'ları gerçekten çalıştırmak | containerd, CRI-O, (dockershim gitti) |

The contract matters more than the vendor. CNI says "here's a pod, give it an IP and a veth"; every plugin answers that the same _way of thinking_, differently in the details. Same for CSI (`CreateVolume`/`Attach`/`Mount` are verbs every driver must know) and CRI (`RunPodSandbox`/`CreateContainer`/`StopContainer`).

### Özetlersek
`kubelet`, container runtime ile **CRI (Container Runtime Interface)** üzerinden konuşur.  
`crictl` ise bu CRI arayüzüyle konuşmak için kullanılan CLI aracıdır.

Bu yüzden Kubernetes node'unda container'ları incelerken:

```bash
crictl pods
crictl ps
```

kullanmak, `docker ps` kullanmaktan daha doğru bir yaklaşımdır.

Çünkü:

- `crictl pods` → kubelet tarafından yönetilen **Pod sandbox'larını**
- `crictl ps` → CRI runtime tarafından yönetilen **container'ları**
- `docker ps` → yalnızca **Docker daemon** tarafından yönetilen container'ları gösterir.

Modern Kubernetes kurulumlarında runtime çoğunlukla `containerd` veya `CRI-O` olduğu için, `docker ps` çıktısı Kubernetes container'larını göstermeyebilir.

Kısaca:

```text
kubelet ──CRI──> containerd / CRI-O
                  ▲
                  │
                crictl
```

Yani `crictl`, kubelet'in container runtime ile konuştuğu dünyaya bakmak için kullanılan araçtır.

A quick taste of the CRI side when a pod is stuck in `ContainerCreating` and you want to see what the runtime thinks:

```bash
crictl ps -a
crictl pods
crictl inspect <container-id> | head -n 20
```

### Özetlersek

- **CNI** → Kubernetes'in **ağa** bağlandığı standart arayüzdür.
- **CSI** → Kubernetes'in **depolamaya** bağlandığı standart arayüzdür.
- **CRI** → Kubernetes'in **container runtime'a** bağlandığı standart arayüzdür.

Basit bir benzetmeyle:

```text
CNI = ağ kablosunun takıldığı yer
CSI = diskin takıldığı yer
CRI = container motorunun takıldığı yer
```

Kubernetes bu işleri doğrudan kendisi yapmaz. Bunun yerine **standart arayüzleri tanımlar**; gerçek işi CNI eklentileri, CSI sürücüleri ve container runtime'lar gerçekleştirir. CRDs

So far every object we've touched - `Deployment`, `Service`, `NetworkPolicy` - is built into Kubernetes. [CustomResourceDefinitions](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) are how _you_ add new object types to the API:

```bash
kubectl get crd
```

On a cluster with [Longhorn](../longhorn/) and [ArgoCD](../gitops-argocd/) installed, this list is long. Each CRD is a schema: a `kind`, an `apiGroup`, a spec. Once applied, the API server accepts objects of that kind and stores them in etcd next to `Pod` and `Service` - `kubectl get <your-kind>` just works.

```bash
kubectl get crd | grep -E 'argoproj|longhorn|postgresql'
kubectl describe crd applications.argoproj.io | head -n 30
```

We've been using CRDs all along without calling them that: the ArgoCD `Application` object, the CloudNativePG `Cluster` object from the [Postgres notes](../postgresql-longhorn-read-replicas/), even VPA's `VerticalPodAutoscaler`. The API surface is uniform on purpose.

# Operators

A CRD only adds _data_. Something still has to look at that data and _do the work_ - create pods, take backups, fail over databases. That something is a controller loop, and a well-packaged controller + CRD pair is called an [operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/).

The pattern is always the same, and it's the same pattern as Deployments reconciling replica counts:

1. User applies a `Cluster` CR with `instances: 3`
2. The operator observes it, compares "what's running" with "what's declared"
3. It takes action (creates a StatefulSet, sets up replication)
4. It updates the CR's `status`, and goes back to sleep until something changes

You declare your hopes and dreams, and it's the operator's job to make them come true.

**Türkçe özet - üçlü ilişki:**
- **CRD** = yeni obje türünün şeması (API'ye "şu türden şey kabul et" der)
- **Custom Resource (CR)** = o şemayla yazılmış somut bir örnek (`Cluster`, `Application`)
- **Operator** = CR'leri okuyup gerçekten işi yapan controller pod'u

> Operators don't have any special powers - they're regular pods with a ServiceAccount and some RBAC. Which means the [RBAC](../../kubernetes/kubernetes-rbac/) notes from v1 apply directly: an operator that can't `create` StatefulSets is a very expensive no-op. When an operator "doesn't work", check its service account's permissions before its logs.

# How to Inspect a CRD and its Objects

1. See what's installed and pick one to dig into:

```bash
kubectl get crd
kubectl explain <kind>.<group>
```

`kubectl explain` works on custom kinds exactly like on built-ins - it's the fastest way to learn an unfamiliar operator's spec without leaving the terminal.

2. List the custom resources and their real state:

```bash
kubectl get <kind> -A
kubectl get <kind> <name> -o yaml
```

The `status` section is where operators tell you the truth - conditions, phase, ready replicas. When `spec` says `instances: 3` and `status` says `ready: 1`, the gap is the story.

3. Look at the `finalizers` field while you're in there. Objects with finalizers refuse to be deleted until the operator says so - that's why a CR can hang in `Terminating` forever when its operator is down.

# How to Install an Operator

Most operators ship as a Helm chart, which is what our [helm](../helm/) notes were building toward:

1. Add the repo and look before you leap:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm show values prometheus-community/kube-prometheus-stack | head -n 40
```

2. Install it and watch the CRDs land first:

```bash
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
kubectl get crd | grep monitoring
```

3. Verify the operator is reconciling:

```bash
kubectl get pods -n monitoring
kubectl get prometheus -n monitoring
kubectl logs -n monitoring -l app.kubernetes.io/name=prometheus-operator --tail=20
```

4. Make a declarative change - bump `replicas` or a retention setting in the CR - and watch the operator translate it into real pods. That's the reconcile loop, visible in real time.

> **Dikkat:** Helm ile kurulan operatörlerin çoğu CRD'leri `crds/` klasöründen bir kez apply eder ve **asla** upgrade/rollback ile değiştirmez ([helm](../helm/) notlarındaki o tuhaf kural). CRD sürümünü yükseltmen gerekiyorsa bu ayrı bir, elle yapılan bir işlemdir.

for more [network-policy](../network-policy/)



