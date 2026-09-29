---
title: "kubernetes-dns-coredns"
weight: 14
---
# Cluster DNS

In the [namespaces](../kubernetes-namespaces/) chapter we casually dropped the magic formula `<service>.<namespace>.svc.cluster.local` and told you to wire `CRAWLER_BASE_URL` with it. It worked, which means _something_ in the cluster is doing name resolution for us. What is it?

Kubernetes runs a DNS server inside the cluster and automatically creates a record for every Service. That's why pods can find each other by name instead of by IP - and why pod IPs (which are ephemeral and worthless to hardcode) don't scare anyone.

The record formats:

```
web-service.crawler.svc.cluster.local   # service, fully qualified
web-service.crawler                     # service + namespace
web-service                             # same namespace only
```

Pods get a record too, but only with a headless service (see [statefulsets](../kubernetes-statefulsets/)) - normally you reach pods through Services anyway.

**Özetlersek:** cluster DNS'i bir telefon rehberi gibi düşün. Servis = kayıtlı numara, isim = kayıt adı. Pod'un kendi numarası da var ancak rehberde "özel" kategoride bulunuyorsa "headless servis" istemedikçe oraya bakmazsın.

# CoreDNS

The server is [CoreDNS](https://coredns.io/), running as a Deployment in `kube-system`:

```bash
kubectl get deploy,pods -n kube-system -l k8s-app=kube-dns
```

Your cluster may label it `k8s-app=kube-dns` - that's the historical name; the implementation is CoreDNS. Its configuration lives in a ConfigMap called `coredns`:

```bash
kubectl get configmap coredns -n kube-system -o yaml
```

The `Corefile` inside is a list of plugin blocks. The interesting one is `kubernetes`:

```
.:53 {
    errors
    health
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
    }
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}
```

- `kubernetes cluster.local` - the zone the cluster answers for
- `forward . /etc/resolv.conf` - anything that isn't a cluster name goes upstream to the node's resolver
- `cache 30` - 30s TTL, which is why "I fixed the service name but it still resolves" is usually impatience, not a bug

Editing this ConfigMap is rare but legal - CoreDNS hot-reloads it. Common tweaks: adding a `stubdomain` for internal corporate DNS, or raising the cache TTL.

# dnsPolicy

A pod's `dnsPolicy` decides where _its_ lookups go:

- `ClusterFirst` (default): cluster names are answered by CoreDNS, everything else forwarded upstream
- `Default`: inherit the node's resolver, skip CoreDNS entirely
- `None`: bring your own `dnsConfig` - explicit nameservers, `search` domains, `options`

The one option that trips everyone: `ndots`. By default a name with fewer dots than `ndots: 5` gets every search domain appended first. That's why `db` inside a pod sometimes resolves to something surprising - the search list is doing work behind your back. Setting `ndots: 1` for chatty apps is a classic optimization.

# DNS Troubleshooting

When a pod "can't reach the database", it's DNS far more often than anyone admits. The debug flow is always the same:

```bash
kubectl exec <pod-name> -- nslookup web-service.crawler.svc.cluster.local
kubectl exec <pod-name> -- nslookup web-service
kubectl exec <pod-name> -- cat /etc/resolv.conf
```

If the fully qualified name works but the short one doesn't, you have a `search` domain problem. If _nothing_ resolves, go one level down:

```bash
kubectl get svc -n kube-system kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
kubectl exec <pod-name> -- nslookup kubernetes.default.svc.cluster.local
```

A pod that can't resolve `kubernetes.default` has no working DNS at all - look at the CoreDNS pods before blaming the app.

> **Dikkat:** `nslookup` her imajda bulunmaz (hele ki distroless olanlarda). Olan bir debug imajı ile ayrı bir pod açmak (v2 [troubleshooting](../../kubernetes_v2/kubernetes-troubleshooting/) notlarda `kubectl debug` ile ephemeral container anlattim) çoğu zaman en hızlı yol. Netshoot imajı bu iş için yapılmış ideal bir imaj.

# Assignment

Let's prove the phone book works - and then break it.

1. From inside a pod, resolve the full name and the short name of a service in another namespace:

```bash
kubectl exec <web-pod> -- nslookup api-service.default.svc.cluster.local
kubectl exec <web-pod> -- nslookup api-service
```

If the web pod lives in `default` and the service is in `crawler`, the short name should fail and the long one should succeed. That asymmetry is the search domain at work.

2. Read the resolver config the pod actually uses:

```bash
kubectl exec <web-pod> -- cat /etc/resolv.conf
```

You'll see `search` lines listing `<namespace>.svc.cluster.local` and `svc.cluster.local`, plus `nameserver` pointing at the cluster DNS service IP (`kubectl get svc -n kube-system kube-dns` should match).

3. Break resolution on purpose: point an env var at a service that doesn't exist (`api-services`), restart the deployment, and watch the crash in slow motion:

```bash
kubectl logs <new-pod> --previous | tail
```

The app dies with something that looks like a connection error but is really "no such host". Fix the name and roll it back with the [rollouts](../kubernetes-deployment-rollouts/) skills from earlier.

4. Scale CoreDNS down to zero and back - the fastest way to feel how load-bearing DNS is:

```bash
kubectl scale deploy coredns -n kube-system --replicas=0
kubectl exec <web-pod> -- nslookup web-service
kubectl scale deploy coredns -n kube-system --replicas=2
```

Everything should break and then heal. Restore it before you walk away.

for more [scaling-vertical](../kubernetes-scaling-vertical/)
