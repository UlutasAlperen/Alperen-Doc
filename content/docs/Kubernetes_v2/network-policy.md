---
title: "network-policy"
weight: 3
---
# NetworkPolicy

When we built custom bridge networks in [Docker](../../docker/docker-network/), the goal was isolation: containers on `network A` can't see containers on `network B`. [NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/) is the cluster-level equivalent - with a twist that trips everyone up at first.

**The default is "allow everything"**. No policy = every pod can talk to every pod, everywhere. But the moment a pod is selected by _any_ NetworkPolicy (Ingress or Egress, even an empty one), that flips: all traffic to/from that pod is denied unless another policy explicitly allows it. Policies are **additive**: the union of all rules selecting a pod defines its allowed traffic. There is no "deny" rule - you deny by omission.

- `policyTypes: [Ingress]` - only controls who may send traffic _to_ the selected pods
- `policyTypes: [Ingress, Egress]` - also controls what the selected pods may reach
- `podSelector: {}` - selects all pods in the namespace
- An empty `ingress:` list = allow nothing (that's your default-deny)

## Selector Semantics: AND vs OR

The `from` field mixes two selector types, and their combination rules are subtle:

```yaml
ingress:
  - from:
      - podSelector:
          matchLabels:
            app: web
      - namespaceSelector:
          matchLabels:
            team: backend
```

Two separate items in the list = **OR**: pods labeled `app: web` from _any_ namespace, _plus_ any pod from the `team: backend` namespace. That's almost never what you meant.

```yaml
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            team: backend
        podSelector:
          matchLabels:
            app: web
```

Two selectors in the _same_ item = **AND**: pods labeled `app: web` that live in namespaces labeled `team: backend`. Same YAML elements, one less space of indentation - completely different rule.

> Egress'te ise selector'ların beraberinde DNS'i düşünmek zorunlu. Egress'i kısıtladığın pod, CoreDNS'e ulaşamazsa her domain çözümlemesi ölür. Bu yüzden her egress policy'sine `kube-dns`'e `53/UDP+TCP` izni eklemek standart pratiktir.

## ipBlock, Ports and Named Ports

`from` can also be a CIDR:

```yaml
from:
  - ipBlock:
      cidr: 192.168.1.0/24
      except:
        - 192.168.1.20/32
```

> `ipBlock` matches pod IPs, which are virtual and rescheduled freely. It's appropriate for pods that run on the node host itself, external VMs or (in some setups) node IPs - not for "that other deployment". Use label selectors between pods; save CIDRs for genuinely external sources.

Ports can be numeric or - better - **named**, matching the service/container port names:

```yaml
ports:
  - port: http
    protocol: TCP
```

Renaming a numeric port later breaks policies silently; named ports keep the policy meaningful. But check your CNI supports named ports (calico does; some older ones don't).

## The CNI Reality Check

NetworkPolicy is just an API. **Enforcement happens in the CNI plugin**, and that's where most homelab confusion lives:

- flannel: implements _nothing_ - policies exist, are ignored
- calico: full enforcement, its own iptables programs
- kube-router, cilium: enforce as well (cilium adds richer eBPF-based policies beyond the standard API)

Minikube's default CNI doesn't enforce policies. To test anything here, start the cluster policy-aware:

```bash
minikube start --cni=calico
# or: minikube addons enable network-policy
```

## How to Build a Default-Deny Zone

1. Make sure the cluster's CNI actually enforces (see above).

2. Apply the namespace-wide deny:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

```bash
kubectl apply -f default-deny-ingress.yaml
```

3. Prove it works - a pod outside your allowed sources should time out, not get an error page:

```bash
kubectl run nettest --rm -it --image=cilium/netshoot:latest -- curl --max-time 3 http://api-service:8080
```

> Timeout (not connection refused) is the signature of a _dropped_ packet at policy level. Connection refused would mean the pod answered - i.e. your policy isn't being enforced at all.

4. Check what k8s thinks the policy means:

```bash
kubectl describe networkpolicy default-deny-ingress
```

## How to Allow Only web → api Traffic

1. Write the allow rule with the AND-semantics you actually mean:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-api
spec:
  podSelector:
    matchLabels:
      app: synergychat-api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: synergychat-web
      ports:
        - port: 8080
          protocol: TCP
```

2. Apply, then test the allowed path _from a properly labeled pod_ - `kubectl run busybox` creates an unlabeled pod, so label it at creation:

```bash
kubectl run nettest --rm -it --image=cilium/netshoot:latest \
  --labels="app=synergychat-web" -- curl --max-time 3 http://api-service:8080
```

3. Test the denied path: the same curl _without_ `--labels` must now fail. That asymmetry (labeled works, unlabeled times out) proves the policy is enforcing, not just present.

4. Roll out the same pattern to the crawler namespace: default-deny there, then allow `api → crawler` on the crawler's port only.

## How to Verify Policies with netshoot

[cilium/netshoot](https://github.com/cilium/netshoot) is a Swiss-army network toolbox container. A few checks worth scripting:

1. Confirm DNS still resolves (egress policy sanity):

```bash
kubectl run dnstest --rm -it --image=cilium/netshoot:latest -- nslookup api-service
```

2. Enumerate what a pod is actually allowed - kubectl has no "effective policy" view, so check by source:

```bash
kubectl get networkpolicy -o wide
kubectl get netpol allow-web-to-api -o yaml | grep -A8 "ingress:"
```

3. Delete the allow policy and watch the allowed path break, then restore it. Ending on a positive control is what turns "I think it works" into "it demonstrably works".

for more [security-context](security-context/)
