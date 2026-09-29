---
title: "control-plane-troubleshooting"
weight: 7
---
# Control Plane Components

In the [namespaces](../../kubernetes/kubernetes-namespaces/) notes we listed what lives in `kube-system` and said "you don't want to mess with it". Good advice - but understanding it is non-negotiable, because when the control plane misbehaves _every_ debugging tool you own stops working. `kubectl get pods` that returns `the connection to the server was refused` is not a pod problem.

The four components, and what they own:

| Bileşen | Ne yapar | Nasıl koşar |
|---|---|---|
| kube-apiserver | Tek API yüzeyi; her `kubectl` buraya gider | static pod |
| etcd | Tüm cluster state'inin tek kaynağı(key-value storage)| static pod |
| kube-scheduler | Pod'ları node'lara (birden fazla kube-scheduler olusturulabilir) yerleştirir | static pod |
| kube-controller-manager | Reconcile döngüleri (deployment, rs, node...) | static pod |

On a [kubeadm](../multi-node-kubeadm/) cluster these run as **static pods** - not Deployments, not systemd units. The kubelet reads YAML files from `/etc/kubernetes/manifests/` and keeps whatever is in there alive, forever. Delete a manifest and the component dies; restore the file and it comes back. That directory is the single most important path on a control-plane node.

```bash
ls /etc/kubernetes/manifests/
crictl ps
```

> On Minikube you can see the same objects with `kubectl get pods -n kube-system` because the kubelet still reports static pods through the API. On a bare kubeadm node with a broken API server, `crictl ps` is your only window - which is exactly why it earns a spot in the [cluster-extensions](../cluster-extensions/) CRI notes.

# The kubectl Failure Tree

Before blaming components, classify the failure. The message tells you where to look:

| Hata | Kim | İlk bakılacak yer |
|---|---|---|
| `connection refused` | apiserver ayakta değil | `crictl ps`, `/etc/kubernetes/manifests/kube-apiserver.yaml`, kubelet log |
| `Unauthorized` / `Forbidden` | RBAC | `kubectl auth can-i --list`, [rbac](../../kubernetes/kubernetes-rbac/) notları |
| `timeout` | network / firewall / LB | node → apiserver port 6443, `ss -tlnp` |
| `i/o timeout` to etcd | etcd sorunu | `etcdctl endpoint health`, disk IO |
| pods stuck `Pending` | scheduler | scheduler log, node resources |
| changes revert themselves | controller-manager | CM log, leader election |

**Ozetlersek:** kubectl -> apiserver -> etcd. Zincirin her halkası koparsa kullanıcıya aynı "bozuk cluster" görünür, ama sebebi tamamen farklıdır. Sırayla test et: apiserver'a `curl`, etcd'ye `etcdctl`, scheduler'a log.

# etcd

etcd is the database behind everything. If it's unhappy, the whole cluster is unhappy - slow writes, lost leader elections, API errors that look random.

Health, from the control-plane node (cert paths are kubeadm defaults):

```bash
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  endpoint health
```

You want `is healthy: true`. `endpoint status` shows who the leader is and how much data is in there. When the DB grows unboundedly (big ConfigMaps, chatty CRDs), `defrag` reclaims space:

```bash
etcdctl ... defrag
etcdctl ... endpoint status -w table
```

> **Dikkat:** defrag etcd'yi bir süreliğine yavaşlatır - uzun süredir defrag edilmemiş, dolu bir etcd'yi prime time'da defrag etmek "hızlandırma" değil, kendine DoS yapmaktır. Bunun yedeğini almadan da restore senaryosu deneme - backup/restore doc'u [kubeadm-upgrade-etcd](../kubeadm-upgrade-etcd/) notlarında.

# kubelet and Nodes

A node that shows `NotReady` is reporting "the kubelet on me is not healthy" - the kubelet is the node's local babysitter. The decision tree is short:

1. Is the kubelet process alive? `systemctl status kubelet`
2. If it's crash-looping: `journalctl -u kubelet -e --no-pager | tail -n 40`
3. Look for the classics: cgroup driver mismatch (see [multi-node](../multi-node-kubeadm/) notes), expired certs, wrong `--kubeconfig`, containerd down
4. Is the node out of resources? `kubectl describe node <node-name>` - look at `Conditions` and `Allocatable` vs `Allocated`

Common kubelet log signatures:

```
failed to load Kubelet config ... cgroup driver
x509: certificate has expired
container runtime is down
```

The first is a config mismatch, the second is cert renewal (covered in [kubeadm-upgrade-etcd](../kubeadm-upgrade-etcd/)), the third is your CRI.

# kube-proxy and Networking

Pod-to-pod and service traffic is programmed by [kube-proxy](../network-policy/) (iptables or IPVS) plus your CNI. When services don't route:

```bash
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=50
kubectl get endpoints <service>
iptables-save | grep <service-ip>
```

Empty endpoints mean the selector matches no ready pods - the [troubleshooting](../kubernetes-troubleshooting/) notes covered that one. Endpoints present but no traffic means the datapath: kube-proxy rules stale, or the CNI broken.

# How to Debug a Dead API Server

1. From the control-plane node, confirm who's dead:

```bash
curl -k https://127.0.0.1:6443/healthz
crictl ps -a | grep kube-apiserver
```

`curl` refusing = apiserver down. `curl` returning `401`/`200` = apiserver fine, your kubeconfig is the problem.

2. If it's down, look at the manifest and the kubelet:

```bash
ls -l /etc/kubernetes/manifests/kube-apiserver.yaml
journalctl -u kubelet -e --no-pager | tail -n 30
```

A missing or malformed manifest is a static-pod problem; a crash-looping apiserver shows up as a pod that keeps restarting in `crictl ps`.

3. Read why it crashes - the apiserver log is inside the container:

```bash
crictl logs $(crictl ps -a -q --name kube-apiserver | head -1) | tail -n 40
```

The usual suspects, in order: etcd unreachable (check etcd next), expired apiserver certs, a bad flag in the manifest after someone edited it, disk full on `/var`.

4. Verify etcd is answering before you blame anything else:

```bash
etcdctl ... endpoint health
```

5. Fix the cause (not the symptom), then confirm:

```bash
curl -k https://127.0.0.1:6443/healthz
kubectl get --raw=/healthz
```

# How to Diagnose a NotReady Node

1. Get the condition and the age - `NotReady` for 10 seconds is not `NotReady` for 10 hours:

```bash
kubectl describe node <node-name> | sed -n '/Conditions/,$p'
```

`MemoryPressure`, `DiskPressure`, `PIDPressure`, `NetworkUnavailable`, `Ready=False` each point to a different subsystem.

2. On the node itself, check the babysitter:

```bash
systemctl status kubelet
journalctl -u kubelet -e --no-pager | tail -n 40
```

3. Check the CRI and the disk:

```bash
crictl info >/dev/null && echo runtime-ok
df -h
```

A full disk is the quietest cluster-killer there is - the kubelet stops posting status, etcd stops accepting writes, and every symptom is downstream.

4. Fix, then wait (or force) the kubelet to re-register. Pods already running on the node usually survive a kubelet restart; they don't survive the node being rebooted if their PVCs are stuck elsewhere.

# How to Debug the Scheduler and the Controller Manager

These two are quieter than the apiserver but just as load-bearing. Both use leader election - only one instance is active at a time.

1. Check the leader (with apiserver up):

```bash
kubectl get lease -n kube-system
```

2. Check the logs for why work isn't happening:

```bash
kubectl logs -n kube-system kube-scheduler --tail=40
kubectl logs -n kube-system kube-controller-manager --tail=40
```

For the scheduler, pods stuck `Pending` with no events mean it's not running (or can't find a feasible node - `describe pod` shows the feasibility errors). For the controller-manager, Deployments that don't scale and nodes that never get `NotReady` are the tells.

3. On a static-pod cluster, `kubectl logs` only works if the apiserver is up. Otherwise go to `crictl logs` on the node, same as the apiserver.

Türkçe homelab checklist (kafam karıştığında sırayla uyguladığım liste):

1. `kubectl` erişim hatası mı, yoksa iş bitmiyor mu? İkisi farklı arızalar
2. `curl -k https://127.0.0.1:6443/healthz` ile apiserver
3. `crictl ps -a` ile static pod'lar ayakta mı
4. `etcdctl endpoint health` ile etcd
5. `systemctl status kubelet` + `journalctl -u kubelet` ile node tarafı
6. `kubectl get lease -n kube-system` ile scheduler/CM liderliği
7. `df -h` ve `kubectl describe node` ile kaynak baskısı
8. Sonra - ve ancak ondan sonra - uygulama logları

for more [gitops-argocd](../gitops-argocd/)
