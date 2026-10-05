---
title: "multi-node-kubeadm"
weight: 12
---
# Multi-Node: From Minikube to a Real Cluster

Minikube is a single-node cluster - we've repeated that warning since the very first [v1 notes](../../kubernetes/kubernetes-nodes-basic/). For homelab work the standard path to _real_ multi-node Kubernetes is `kubeadm` - vanilla k8s, everything in your hands, the same tool the managed distros are built on. This note walks through a kubeadm install on VMs, from node prep to a joined, working cluster, with [Cilium](https://cilium.io/) as the CNI - eBPF datapath, working [NetworkPolicy](../network-policy/) enforcement and no kube-proxy.

## The Plan

On my Proxmox box I spin up 4 Debian 12 VMs, `2 vCPU / 4GB` each:

```text
kmaster   192.168.1.200   control plane
kworker1  192.168.1.201   worker
kworker2  192.168.1.202   worker
kworker3  192.168.1.203   worker
```

Add them to `/etc/hosts` on all nodes and on your laptop, make sure SSH with key auth works, and keep the k8s minor version pinned identically everywhere - mixed minor versions within one cluster are supported only one step ahead and are a support nightmare.

## Prep on All Nodes

```bash
# Disable swap - kubelet refuses to work with it
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Kernel modules + sysctl for the pod network
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```

> `net.bridge.bridge-nf-call-iptables = 1` looks arcane but matters: without it, traffic through the bridge is not processed by iptables - which breaks Service NAT rules and makes [NetworkPolicy](../network-policy/) enforcement silently unreliable. Cilium's eBPF datapath needs `ip_forward` regardless, and even with kube-proxy gone the same prep keeps non-BPF paths (and every other doc in this series) honest. Debian 12's kernel (6.x) is comfortably above Cilium's minimum - no kernel surgery needed.

## Containerd

Kubernetes ships its own [containerd](https://github.com/containerd/containerd) expectations; the important detail is the cgroup driver:

```bash
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
```

> `SystemdCgroup = true` is not optional anymore: kubelet defaults to the systemd cgroup driver, and a mismatch with the container runtime is the classic cause of pods stuck in `ContainerCreating` or kubelet crash-looping at join time. The v1 [troubleshooting](../kubernetes-troubleshooting/) checklist applies to nodes too.

## K8s Packages (apt, the careful way)

Same discipline as my [Docker hardening doc](../../docker/docker-kurulum-hardening-debian/): verify with GPG, pin versions, avoid surprise upgrades.

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod a+r /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/v1.31:/deb/ /" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

> `apt-mark hold` is the part everyone skips and then regrets: unattended-upgrades bumping kubelet on one node only is a textbook way to end up with version-skew symptoms that look like random network failures.

## Control Plane

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

- `--pod-network-cidr` is where kubeadm carves out per-node pod CIDRs from - Cilium reads those from `node.spec.podCIDR` (its default `ipam.mode=kubernetes`), so pick this **before** `init`, it's painful to change later
- The output prints a `kubeadm join ...` command - **save it**
- For anything beyond a lab, add `--control-plane-endpoint=<stable-DNS-or-VIP>` - this is what makes a later control-plane HA setup possible without rebuilding everything

Set up kubeconfig:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## CNI: Cilium

Flannel is the no-frills homelab CNI - and it implements _zero_ NetworkPolicies, which is the one thing this series kept promising. [Cilium](https://cilium.io/docs/) is the modern answer: an eBPF-based CNI that enforces [NetworkPolicy](../network-policy/), replaces kube-proxy entirely (Service load-balancing in eBPF), and throws in Hubble for network observability. The [cluster-extensions](../cluster-extensions/) notes called CNI "the socket" - Cilium is the plugin we now plug into it.

Installed with [Helm](../helm/), the same way we'll treat every cluster add-on later:

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update

helm install cilium cilium/cilium --version 1.16.5 \
  --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=192.168.1.200 \
  --set k8sServicePort=6443
```

Three flags worth understanding, because each one is a different way to install a broken cluster:

- `kubeProxyReplacement=true` - Cilium programs Service VIPs in eBPF; kube-proxy becomes redundant
- `k8sServiceHost` / `k8sServicePort` - with kube-proxy gone, Cilium is the thing that has to find the API server; point it at `kmaster`'s IP (or your `controlPlaneEndpoint` if you set one). Forgetting these leaves every Service unroutable with no error anywhere
- `--version` - pin it, same philosophy as `apt-mark hold`. `helm install cilium cilium/cilium` with no version grabs the latest chart, which is how clusters get surprise minor upgrades

Then remove the now-dead kube-proxy DaemonSet - leaving it running is confusing at best and fighting for iptables at worst:

```bash
kubectl -n kube-system scale daemonset kube-proxy --replicas=0
kubectl -n kube-system get daemonset kube-proxy   # 0 desired, confirm nothing breaks
kubectl -n kube-system delete daemonset kube-proxy
```

Verify before joining anything:

```bash
cilium status --wait          # or: kubectl -n kube-system get pods -l k8s-app=cilium
kubectl get nodes             # kmaster flips to Ready
```

> Cilium pods are a DaemonSet (`--ignore-daemonsets` in every drain below), and the agent needs `CAP_BPF`-level access to the kernel - on a plain Debian VM with the prep above that just works. If `cilium status` shows `Controller: ... Failing`, check `k8sServiceHost` first, `dmesg | grep -i bpf` second. For a deeper look: `cilium sysdump` bundles everything worth reading.

> Hubble (Cilium's network observability layer) is a flag away - `--set hubble.enabled=true --set hubble.relay.enabled=true --set hubble.ui.enabled=true` - and turns "which pod is talking to which" from a guessing game into `hubble observe`. Off by default here to keep the install lean; it's a per-node sidecar-free design so it costs almost nothing.

(Ben policy enforcement'i artık her yerde deneyebilirim - Cilium varsayılan olarak standardı uygular. eBPF tabanlı ekstra `CiliumNetworkPolicy`'ler için [NetworkPolicy](../network-policy/) notuna bak; o nottaki CNI gerçekliği tablosunda flannel'in "policies exist, are ignored" satırının neden bizi Cilium'a getirdiğini göreceksin.)

## Workers Join

On each worker, run the saved join command. If you lost it:

```bash
kubeadm token create --print-join-command   # on the control plane
```

Verification - three nodes join, then everything flips to `Ready` (Cilium first, then nodes):

```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system
```

Note: control-plane nodes carry the taint `node-role.kubernetes.io/control-plane:NoSchedule` by default - workloads land on workers only. The v1 notes' [control plane vs workers](../../kubernetes/kubernetes-nodes-basic/) distinction is now something you can literally `kubectl describe`.

## Node Lifecycle

- `kubectl cordon <node>` - mark unschedulable, running pods keep running
- `kubectl drain <node> --ignore-daemonsets` - evict everything reschedulable (daemonsets excluded, they're per-node by design)
- `kubectl uncordon <node>` - schedulable again
- On the node itself: `sudo kubeadm reset` cleans up most of the k8s state (PVC data and some config survive - read the warning it prints), then `kubectl delete node <name>` on the control plane removes the object

## Backups and Upgrades

Control-plane state lives in etcd. Snapshot it periodically:

```bash
sudo etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /backup/etcd-snapshot-$(date +%F).db
```

Proxmox tarafında ayrıca VM-level snapshot almak olabilecek en kolay sigortadır - tabi etcd snapshot'ı deployment'ları da kurtarırken, VM snapshot'ı tüm control-plane'i hepten geri alır; ikisinin yerini tutmaz.

Upgrades: control plane first, one minor at a time, `apt-mark unhold` before, hold again after, drain per node in between. Never skip minors on kubeadm clusters. The full command-by-command walkthrough - including etcd restore and cert renewal - is in [kubeadm-upgrade-etcd](../kubeadm-upgrade-etcd/); HA topologies that make an upgrade survivable are in [ha-control-plane](../ha-control-plane/).

**Özetle:** kubeadm upgrade'i ve Cilium upgrade'i ayrı islerdir. `apt-get upgrade` cluster'ı hareket ettirir; `helm upgrade cilium ...` datapath'i. İkisini aynı anda yapma - biri bozulursa hangisinde problem olduğunu bilemezsin.

## How to Bootstrap a 4-Node Cluster with kubeadm

1. Prep all four VMs (swap, modules, sysctl, containerd, packages) - identical commands everywhere, and verify each step's output rather than assuming.

2. On `kmaster`, initialize and capture the join command:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

Copy kubeconfig, verify `kubectl get nodes` shows one `NotReady` control plane (no CNI yet - that's expected).

3. Install Cilium and watch the control plane flip to `Ready`:

```bash
helm repo add cilium https://helm.cilium.io/ && helm repo update
helm install cilium cilium/cilium --version 1.16.5 \
  --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=192.168.1.200 \
  --set k8sServicePort=6443

cilium status --wait
kubectl get nodes --watch
```

> If the node stays `NotReady` after the install, check `kubectl get pods -n kube-system` - `cilium` pods should be `Running` on every node and `cilium-operator` at least once. CrashLooping cilium pods with no obvious error is almost always `k8sServiceHost` pointing at the wrong address; `NotReady` + cilium Running is usually a CIDR mismatch from step 2 - there's no clean fix for a wrong `--pod-network-cidr`, plan the CIDR before `init`.

## How to Join Workers and Verify the Cluster

1. On each worker, paste the join command. A successful join ends with `This node has joined the cluster`. Three times - `kworker1`, `kworker2`, `kworker3`.

2. From the control plane, confirm the fleet and that the worker role reads as `<none>`:

```bash
kubectl get nodes -o wide
```

Check `INTERNAL-IP`, `OS-IMAGE`, `VERSION` columns - and confirm all nodes run the _same_ k8s minor version (the `hold` pins should guarantee it).

3. Confirm the datapath claim: kube-proxy is gone and Services still work:

```bash
kubectl -n kube-system get daemonset kube-proxy   # NotFound = success
kubectl get svc kubernetes -o wide
```

If `kubernetes` ClusterIP answers on `:443` from a worker, eBPF is load-balancing - the flag did what it promised.

4. Deploy the web app with 3 replicas and watch placement:

```bash
kubectl get pods -o wide
```

Pods landing on _different_ worker nodes is the v1 [minikube limits](../../kubernetes/kubernetes-minikube/) table happening for real.

5. Sanity-check the control-plane taint keeps pods off the master:

```bash
kubectl describe node kmaster | grep -iA3 taints
```

## How to Remove and Re-add a Node Safely

1. Evacuate first - drain moves the workloads away before you touch anything:

```bash
kubectl drain kworker1 --ignore-daemonsets
kubectl get pods -o wide --watch
```

Watch the web pods reschedule onto remaining nodes (this is also the moment you notice if you lack a [spread constraint](../taints-affinity-quotas/) - everything may pile onto one node). Cilium agents stay - they're a DaemonSet and per-node by design.

2. From the cluster's view, remove the node object:

```bash
kubectl delete node kworker1
```

3. On the node itself, clean k8s + Cilium state:

```bash
sudo kubeadm reset -f
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F
sudo rm -rf /etc/cni /var/lib/cni/ /var/lib/kubelet/* /var/run/cilium /var/lib/cilium
```

> The iptables flush, kubelet cleanup and Cilium state removal are what make a _re_-join work cleanly; skipping them produces join failures that look like token problems but are actually stale state. Stale Cilium state is especially nasty - the node rejoins, the agent starts, and the datapath silently keeps programming old identities.

4. Re-join with a fresh token from the control plane and verify it returns to `Ready`:

```bash
kubeadm token create --print-join-command   # on kmaster
# paste on kworker1
kubectl get nodes --watch
kubectl -n kube-system get pods -l k8s-app=cilium -o wide   # a cilium pod for the new node
```

for more [kubeadm-upgrade-etcd](../kubeadm-upgrade-etcd/) and [ha-control-plane](../ha-control-plane/)
