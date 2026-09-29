---
title: "kubernetes-cert-manager"
weight: 13
---
# TLS Without Hand-Rolling Secrets

In the [ingress](../kubernetes-ingress/) chapter we said that real clusters "get this issued automatically by cert-manager instead of hand-rolling it". That was a promise we haven't paid off yet. Let's pay it off.

Here is what a TLS certificate actually requires, every single time:

1. a private key
2. a certificate signing request built from that key
3. someone - a Certificate Authority - to sign it
4. a renewal before it expires (Let's Encrypt certs last 90 days)
5. the result copied into a `Secret` in the right namespace, in the right format

Do that by hand for two hostnames and you have already lost an afternoon. [cert-manager](https://cert-manager.io/) is a controller that does all five as a Kubernetes reconciliation loop: you declare _what_ you want, it keeps it true forever - including the renewal.

**Özetlersek:** elle sertifika = tek seferlik iş ama süresi dolunca yine sen sorumlusun. cert-manager = "bu isimlerde, şu imzalayıcıdan, şu süreyle sertifika istiyorum" deyip arkana yaslanmak.

# The Three Objects

Everything cert-manager does circles three objects:

- **`Issuer` / `ClusterIssuer`** — _who_ signs. An `Issuer` is valid in one namespace, a `ClusterIssuer` across the whole cluster. The spec is identical; only the reach differs.
- **`Certificate`** — _what_ we want: which names (`dnsNames`), from which signer (`issuerRef`), written to which `Secret` (`secretName`).
- **`Secret`** — the result. `tls.crt`, `tls.key`, usually `ca.crt`. This is the only thing the Gateway and the Ingress actually consume.

The division of labour is clean: the Issuer is policy, the Certificate is the request, the Secret is the outcome. The Gateway steps in at the end and serves that Secret to your users as TLS.

# Installing cert-manager

Throughout this course we install everything with `kubectl apply` - we'll get to [Helm](../../kubernetes_v2/helm/) later. Same approach here:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.21.0/cert-manager.yaml
```

The install is three parts: the CRDs (`Certificate`, `Issuer`, `CertificateRequest`...), the `cert-manager` namespace, and the controllers themselves. Give it a few seconds and check:

```bash
kubectl get pods -n cert-manager
kubectl get crds | grep cert-manager
```

You want three pods: `cert-manager`, `cert-manager-cainjector`, `cert-manager-webhook`. Don't create any `Certificate` until all three are `Running` - if the webhook isn't up, requests get rejected.

> **Dikkat:** Gateway API CRD'ları cert-manager'dan **önce** kurulmuş olmalı, ya da cert-manager'ı restart etmelisin. Bazı bileşenler bu kontrolü sadece startup'ta yapar. Önceki bölümde Envoy Gateway'i kurduğumuz için CRD'lar zaten orada - ama sıralamayı bil. Çözümü tek satır: `kubectl rollout restart deployment cert-manager -n cert-manager`

# Path A: Explicit Certificate

Let's start with the mechanism spelled out - this path needs no extra flags.

## The SelfSigned issuer

`synchat.internal` is not a real domain; no public CA will sign it. So we use an issuer that signs with itself:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned
spec:
  selfSigned: {}
```

```bash
kubectl get clusterissuer selfsigned
```

The `READY` column should say `True`. A `SelfSigned` issuer has no external dependency at all - the certificate is signed with its own private key and there is no separate CA involved.

## The Certificate

Now we say what we want:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: synchat-tls
spec:
  secretName: synchat-tls
  dnsNames:
    - synchat.internal
    - synchatapi.internal
  subject:
    organizations:
      - SynergyChat
  issuerRef:
    name: selfsigned
    kind: ClusterIssuer
    group: cert-manager.io
```

That `subject` line is not decoration. On a self-signed certificate Subject DN equals Issuer DN, so leaving the subject empty leaves the Issuer DN empty too - and the X.509 spec technically calls that invalid. cert-manager emits a `BadConfig` event when it sees it. One line to avoid the whole problem.

```bash
kubectl get certificate synchat-tls
kubectl describe certificate synchat-tls
```

Once `READY` is `True` the `Secret` exists too:

```bash
kubectl get secret synchat-tls
kubectl get secret synchat-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text
```

Both domains should show up under `Subject:` and `X509v3 Subject Alternative Name`.

## Wiring it into the Gateway

Now let's turn the `app-gateway` from the [gateway](../kubernetes-gateway-minikube/) chapter into real HTTPS. It's a `tls` block on the listeners:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: app-gateway
spec:
  gatewayClassName: app-gatewayclass
  listeners:
    - name: http
      protocol: HTTP
      port: 80
    - name: web-https
      protocol: HTTPS
      port: 443
      hostname: "synchat.internal"
      tls:
        mode: Terminate
        certificateRefs:
          - name: synchat-tls
    - name: api-https
      protocol: HTTPS
      port: 443
      hostname: "synchatapi.internal"
      tls:
        mode: Terminate
        certificateRefs:
          - name: synchat-tls
```

Two listeners, the **same** `certificateRefs[].name`. That's deliberate: cert-manager merges the names and produces a single `Certificate`, de-duplicating `dnsNames`. Two hostnames, one certificate, one Secret. You _can_ certify them separately, but why?

> `tls.mode: Terminate` ends TLS at the gateway - the certificate is opened inside Envoy. `Passthrough` does not terminate and forwards the crypto to the backend, which this model does not support.

`certificateRefs` is only a name; where the Secret lives is assumed to be the Gateway's **own namespace** (`default` for `app-gateway`). That's why the Certificate went there too.

It's good practice to say which listener an HTTPRoute attaches to with `sectionName`:

```yaml
spec:
  parentRefs:
    - name: app-gateway
      sectionName: web-https
```

# Path B: The Annotation

In Path A we wrote the `Certificate` by hand. The real productivity gain is deriving it from the Gateway itself:

```yaml
metadata:
  name: app-gateway
  annotations:
    cert-manager.io/cluster-issuer: selfsigned
```

With that annotation cert-manager starts watching the Gateway and generates the `Certificate` objects from its HTTPS listeners. You can stop writing `Certificate` YAML entirely.

There is a cost: you have to **turn on** Gateway support. Installed with raw manifests (which is what we did), that's a flag:

```bash
kubectl patch deployment cert-manager -n cert-manager --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--enable-gateway-api"}]'
kubectl rollout restart deployment cert-manager -n cert-manager
```

> On Helm the equivalent is `--set config.gatewayAPI.enabled=true`. Raw manifests have no values.yaml, so the flag is the right tool.

## Which listeners get a Certificate?

cert-manager evaluates each listener individually and **silently skips** the ones that don't qualify. If a Certificate refuses to appear, this table is almost always why:

| Field | Requirement |
|---|---|
| `hostname` | must not be empty |
| `tls.mode` | must be `Terminate`; `Passthrough` is unsupported |
| `tls.certificateRefs[].name` | required |
| `tls.certificateRefs[].kind` | if set, must be `Secret` |
| `tls.certificateRefs[].group` | if set, must be `""` |
| `tls.certificateRefs[].namespace` | if set, must match the Gateway's |

> **Dikkat:** `dnsNames` HTTPRoute'un `hostnames` alanından **değil**, listener'ın `hostname` alanından gelir. HTTPRoute'ta yazdığın isimler yönlendirme içindir; TLS isimleri listener'da yaşar. İkisini karıştırmak, "Certificate oluştu ama yanlış isimlere" probleminin bir numaralı sebebi.

**Özetlersek:** Path A = her şeyi sen yaz, her yerde çalışır. Path B = annotation yaz, cert-manager üretir, ama `--enable-gateway-api` ve listener kısıtları gerekir. İkisi aynı sonucu verir; fark YAML'ı kimin yazdığı.

# Trust: A Real CA Instead

Everything so far worked - but open `https://synchat.internal` in a browser and you get "Your connection is not private". The reason matters: every self-signed certificate **is its own root**. To trust it you have to add that specific certificate to your trust store. Two hostnames, two separate trust problems.

The real world doesn't work like that. Let's Encrypt and corporate PKIs have a **root CA**; every leaf is signed by it and that one root is trusted once. You can build the same shape locally - and it is exactly what cert-manager recommends `SelfSigned` for:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: synchat-root-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: synchat-root-ca
  secretName: synchat-root-ca-secret
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned
    kind: ClusterIssuer
    group: cert-manager.io
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: synchat-ca
spec:
  ca:
    secretName: synchat-root-ca-secret
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: synchat-tls
spec:
  secretName: synchat-tls
  dnsNames:
    - synchat.internal
    - synchatapi.internal
  issuerRef:
    name: synchat-ca
    kind: ClusterIssuer
    group: cert-manager.io
```

Reading the chain matters, because every line has a reason:

1. `selfsigned` only performs the **first step** - its whole job is bootstrapping the root
2. `synchat-root-ca` is a certificate but flagged `isCA: true`; it self-signs and is written to a `Secret` in the `cert-manager` namespace
3. a second `ClusterIssuer` called `synchat-ca` uses that `Secret` as its CA
4. `synchat-tls` now asks `synchat-ca`, not `selfsigned` - so it is **signed by the root**

The `namespace: cert-manager` line is not arbitrary: a `ClusterIssuer` is not namespaced, so `ca.secretName` is assumed to live in the `cert-manager` namespace. That's where the root goes.

See the difference for yourself:

```bash
kubectl get secret synchat-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -issuer -subject
```

Now `issuer=` and `subject=` must be **different**. Different means the chain is real.

Add the root to your trust store once (macOS Keychain, or `/usr/local/share/ca-certificates/` on Linux) and both hostnames open without warnings. `ca.crt` is already in the Secret:

```bash
kubectl get secret synchat-root-ca-secret -n cert-manager -o jsonpath='{.data.ca\.crt}' | base64 -d > synchat-root-ca.pem
```

| | `SelfSigned` alone | `isCA` + `CA` chain |
|---|---|---|
| Objects | 2 | 4 |
| Trust anchor | each certificate separately | one root, once |
| Two hostnames | trust twice | trust the root once |
| Issuer chain | none | real |
| Browser | warns | clean |

# Ingress vs Gateway

cert-manager works with both APIs, but where TLS is **declared** changes. This is where Ingress users get stuck:

| | Ingress | Gateway |
|---|---|---|
| Annotation | on the `Ingress` | on the `Gateway` |
| TLS lives in | `spec.tls[].secretName` | listener `tls.certificateRefs[].name` |
| Names come from | `spec.tls[].hosts` | listener `hostname` |
| Who owns it | app team (their own Ingress) | platform team (shared Gateway) |
| Sub-resources | none | `sectionName` to pick a listener |

The annotations themselves are identical - `cert-manager.io/issuer`, `cert-manager.io/cluster-issuer`, `cert-manager.io/duration`, `cert-manager.io/renew-before`. Only the object they sit on changes.

> `ingress-nginx` has been officially EOL since early 2026. For anything new, Gateway API is the destination. Know Ingress, but don't build on it.

**Özetlersek:** Ingress'te TLS'i *sen* Ingress'in içinde tarif edersin, app ekibi kendi başına yönetir. Gateway'de TLS *Gateway*'in içinde yaşar ve genelde platform ekibinin kontrolündedir - ki bu yüzden `ListenerSet` diye bir kaynak geliştiriliyor.

# Troubleshooting

TLS that doesn't work doesn't come with an error message, it comes with silence. In order:

**`Certificate` is `Ready: False`** — the detail is always there:

```bash
kubectl describe certificate synchat-tls
kubectl get certificaterequest
kubectl describe certificaterequest <name>
```

The `message` under Conditions is the real reason (`referenced signer resource does not exist`, `secret "..." not found`...).

**The `Secret` never appears** — is the `Certificate` `Ready`? Does `certificateRefs[].namespace` match the Gateway's? A cross-namespace reference is skipped silently.

**No Certificate is generated (Path B)** — three checks in order: is `--enable-gateway-api` on (`kubectl get deployment cert-manager -n cert-manager -o jsonpath='{.spec.template.spec.containers[0].args}'`), is the annotation spelled right, does the listener satisfy the constraints table. An empty `hostname` skips the listener.

**HTTPS works but the name is wrong** — look at `dnsNames`. If it's wrong, fix `hostname` on the **listener**, not on the HTTPRoute.

**The browser still warns** — that's not a cert-manager problem, that's a trust problem. If you're on `SelfSigned`, switch to the CA chain or add the root to your trust store. Use `openssl x509 -noout -issuer -subject` to tell whether a chain exists at all.

**The old certificate is still being served** — the Secret updated but Envoy hasn't reloaded it. Usually just waiting works; controllers pick up Secret changes.

# Assignment

Let's make the `app-gateway` from the gateway chapter into real HTTPS - without warnings.

1. Build the CA chain: a `selfsigned` ClusterIssuer, a `synchat-root-ca` with `isCA: true` (in the `cert-manager` namespace), and a `synchat-ca` ClusterIssuer. One file is fine.

2. Write the `synchat-tls` Certificate. `secretName: synchat-tls`, `dnsNames` with `synchat.internal` **and** `synchatapi.internal`, `issuerRef` → `synchat-ca`.

3. Add the `cert-manager.io/cluster-issuer: synchat-ca` annotation to `app-gateway`. If you take Path B, confirm `--enable-gateway-api` is on and then delete the hand-written Certificate - cert-manager will generate it. Either way, rewrite the listeners as `web-https` and `api-https`, both with `tls.mode: Terminate` and the same `certificateRefs[].name`.

4. Don't forget `sectionName` on the HTTPRoutes: `web-https` for web, `api-https` for api.

5. Verify:

```bash
kubectl get certificate
kubectl get secret synchat-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -issuer -subject -ext subjectAltName
curl -kv https://synchat.internal 2>&1 | grep -E "subject:|issuer:|SSL certificate"
```

What you want: `certificate` `READY` `True`, issuer and subject **different**, `subjectAltName` containing both names, and both visible in the `curl` output.

6. The last step is the actual payoff. Add the root CA to your trust store and open `https://synchat.internal` in a browser:

```bash
kubectl get secret synchat-root-ca-secret -n cert-manager -o jsonpath='{.data.tls\.crt}' | base64 -d > synchat-root-ca.pem
```

When it opens with no warning, you've seen TLS actually **work**. Getting a certificate and plugging it into a Gateway was the easy half; making it trusted and self-renewing was the real job.

> **Dikkat:** `Certificate`'ı Gateway'in namespace'i dışında yazarsan sonuç sessizce bozulur. `Certificate` namespace'i, `certificateRefs`'un bakacağı namespace ile **aynı** olmalı - `app-gateway` için bu `default`.

The next chapter takes this to a real cluster: ACME, Let's Encrypt, wildcards, renewal windows, trust distribution and monitoring.

for more [cert-manager-production](../kubernetes-cert-manager-production/)
