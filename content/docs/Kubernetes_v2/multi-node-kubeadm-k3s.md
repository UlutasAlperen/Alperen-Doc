---
title: "multi-node-kubeadm-k3s"
weight: 9
---
# Multi-Node: From Minikube to a Real Cluster

Minikube is a single-node cluster - we've repeated that warning since the very first [v1 notes](../../kubernetes/kubernetes-nodes-basic/). For homelab work there are two realistic paths to _real_ multi-node Kubernetes: `kubeadm` (vanilla k8s, everything in your hands) and `k3s` (single binary, batteries included). This note walks through kubeadm on VMs, with k3s as the lightweight alternative.

## The Plan

On my Proxmox box I spin up 3 Debian 12 VMs, `2 vCPU / 4GB` each:

```text
kmaster   192.168.1.50   control plane
kworker1  192.168.1.51   worker
kworker2  192.168.1.52   worker
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

> `net.bridge.bridge-nf-call-iptables = 1` looks arcane but matters: without it, traffic through the bridge is not processed by iptables - which breaks Service NAT rules and makes [NetworkPolicy](network-policy/) enforcement silently unreliable.

## Containerd

Kubernetes ships its own [containerd](https://github.com/containerd/containerd) expectations; the important detail is the cgroup driver:

```bash
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
```

> `SystemdCgroup = true` is not optional anymore: kubelet defaults to the systemd cgroup driver, and a mismatch with the container runtime is the classic cause of pods stuck in `ContainerCreating` or kubelet crash-looping at join time. The v1 [troubleshooting](kubernetes-troubleshooting/) checklist applies to nodes too.

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

- `--pod-network-cidr` must match what your CNI expects (flannel's default is exactly `10.244.0.0/16`; calico defaults elsewhere - check first, this is painful to change later)
- The output prints a `kubeadm join ...` command - **save it**
- For anything beyond a lab, add `--control-plane-endpoint=<stable-DNS-or-VIP>` - this is what makes a later control-plane HA setup possible without rebuilding everything

Set up kubeconfig, then install a CNI. Flannel is the no-frills homelab choice:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

(Ben policy enforcement'i de denemek istiyorsam [calico](https://docs.tigera.io/calico/latest/about/about-kubernetes) kuruyorum - [NetworkPolicy](network-policy/) notundaki CNI konusunun ta kendisi.)

## Workers Join

On each worker, run the saved join command. If you lost it:

```bash
kubeadm token create --print-join-command   # on the control plane
```

Verification - both nodes join, then everything flips to `Ready` (CNI first, then nodes):

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

Proxmox tarafında ayrıca VM-level snapshot almak bedava sigortadır - ama etcd snapshot'ı deployment'ları da kurtarırken, VM snapshot'ı tüm control-plane'i olduğu geri alır; ikisinin yerini tutmaz.

Upgrades: control plane first, one minor at a time, `apt-mark unhold` before, hold again after, drain per node in between. Never skip minors on kubeadm clusters.

## k3s Alternatifi

If the VMs are small (2GB) and the goal is "lightweight but real cluster", [k3s](https://docs.k3s.io/) installs in five minutes:

```bash
# Master
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644
sudo cat /var/lib/rancher/k3s/server/node-token

# Worker (agent)
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.1.50:6443 K3S_TOKEN=<token> sh -
```

Differences worth knowing before choosing:

- Single binary with kubelet, kube-proxy, embedded containerd, flannel and CoreDNS bundled - far fewer moving parts to break
- State store defaults to **sqlite** (single server); for HA it supports embedded etcd (`--cluster-init`) or external datastore - not a toy, but read the HA docs before assuming parity with kubeadm's etcd
- Ships [traefik](https://traefik.io/) and the local-path storage provisioner by default; `--disable traefik` if you run your own ingress - my Proxmox VMs are small, and this alone saves RAM
- No `kubeadm`; upgrades are a re-run of the install script with a version flag

Benim tercihim: kaynak bolken kubeadm (standart k8s, her şeyi sen yönetiyorsun); 2GB'lık VPS'lerde ve düşük kaynaklı Proxmox VM'lerinde k3s.

## How to Bootstrap a 3-Node Cluster with kubeadm

1. Prep all three VMs (swap, modules, sysctl, containerd, packages) - identical commands everywhere, and verify each step's output rather than assuming.

2. On `kmaster`, initialize and capture the join command:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

Copy kubeconfig, verify `kubectl get nodes` shows one `NotReady` control plane (no CNI yet - that's expected).

3. Apply flannel and watch the control plane flip to `Ready`:

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
kubectl get nodes --watch
```

> If nodes stay `NotReady` after flannel, check `kubectl get pods -n kube-system` - flannel pods should be Running on every node. NotReady + flannel Running is usually a CIDR mismatch from step 2; there's no clean fix for a wrong `--pod-network-cidr`, plan the CIDR before `init`.

## How to Join Workers and Verify the Cluster

1. On each worker, paste the join command. A successful join ends with `This node has joined the cluster`.

2. From the control plane, confirm the fleet and that the worker role reads as `<none>`:

```bash
kubectl get nodes -o wide
```

Check `INTERNAL-IP`, `OS-IMAGE`, `VERSION` columns - and confirm all nodes run the _same_ k8s minor version (the `hold` pins should guarantee it).

3. Deploy the web app with 3 replicas and watch placement:

```bash
kubectl get pods -o wide
```

Pods landing on _different_ worker nodes is the v1 [minikube limits](../../kubernetes/kubernetes-minikube/) table happening for real.

4. Sanity-check the control-plane taint keeps pods off the master:

```bash
kubectl describe node kmaster | grep -iA3 taints
```

## How to Remove and Re-add a Node Safely

1. Evacuate first - drain moves the workloads away before you touch anything:

```bash
kubectl drain kworker1 --ignore-daemonsets
kubectl get pods -o wide --watch
```

Watch the web pods reschedule onto remaining nodes (this is also the moment you notice if you lack a [spread constraint](taints-affinity-quotas/) - everything may pile onto one node).

2. From the cluster's view, remove the node object:

```bash
kubectl delete node kworker1
```

3. On the node itself, clean k8s state:

```bash
sudo kubeadm reset -f
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F
sudo rm -rf /etc/cni /var/lib/cni/ /var/lib/kubelet/*
```

> The iptables flush and kubelet cleanup are what make a _re_-join work cleanly; skipping them produces join failures that look like token problems but are actually stale state.

4. Re-join with a fresh token from the control plane and verify it returns to `Ready`:

```bash
kubeadm token create --print-join-command   # on kmaster
# paste on kworker1
kubectl get nodes --watch
```

for more [longhorn](longhorn/)
