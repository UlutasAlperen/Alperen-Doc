---
title: "cert-manager"
weight: 16
---
# cert-manager

In the v1 [ingress](../../kubernetes/kubernetes-ingress/) notes we said that real clusters "get this issued automatically by cert-manager instead of hand-rolling it". That was a promise we haven't paid off yet. Let's pay it off.

Here is what a TLS certificate actually requires, every single time:

1. a private key
2. a certificate signing request built from that key
3. someone - a Certificate Authority - to sign it
4. a renewal before it expires (Let's Encrypt certs last 90 days)
5. the result copied into a `Secret` in the right namespace, in the right format

Do that by hand for two hostnames and you have already lost an afternoon. [cert-manager](https://cert-manager.io/) is a controller that does all five as a Kubernetes reconciliation loop: you declare _what_ you want, it keeps it true forever - including the renewal.

**Özetlersek:** elle sertifikalama = tek seferlik iş ama süresi dolunca yine elle serfialama durumunda sen sorumlusun. cert-manager = "bu isimlerde, şu imzalayıcıdan, şu süreyle sertifika istiyorum" otomasyona baglayim rahatla yatmak desem daha dogru olur.

## The Three Objects

Everything cert-manager does circles three objects - and they map exactly onto the CRD / Custom Resource / operator split from the [cluster-extensions](../cluster-extensions/) notes:

- **`Issuer` / `ClusterIssuer`** : _who_ signs. An `Issuer` is valid in one namespace, a `ClusterIssuer` across the whole cluster. The spec is identical; only the reach differs.
- **`Certificate`** : _what_ we want: which names (`dnsNames`), from which signer (`issuerRef`), written to which `Secret` (`secretName`).
- **`Secret`** : the result. `tls.crt`, `tls.key`, usually `ca.crt`. This is the only thing the Gateway and the Ingress actually consume.

The division of labour is clean: the Issuer is policy, the Certificate is the request, the Secret is the outcome. The Gateway steps in at the end and serves that Secret to your users as TLS.

## The Honest Requirements

- **A cluster where you can `kubectl apply` CRDs.** cert-manager ships about a dozen of them (`Certificate`, `Issuer`, `CertificateRequest`, `Order`, `Challenge`)
- **Three pods' worth of headroom.** The controller, the cainjector and the webhook together want roughly 200Mi; the webhook is the one that blocks everything if it's down
- **Gateway API CRDs installed before cert-manager starts** - or restart cert-manager afterwards. Some components only check at startup
- **Patience for the first `Certificate`.** The webhook has to be `Ready` before anything else is accepted; applying a `Certificate` in the first ten seconds is the classic false negative

## How to Install cert-manager

Throughout the v1 notes we installed with `kubectl apply`; [helm](../helm/) is the platform-team's tool and the [devops-control-plane](../devops-control-plane/) notes use it for add-ons. Either works - the static manifest is one command:

1. Install and let it settle:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.21.0/cert-manager.yaml
```

2. Confirm the three pods and the CRDs:

```bash
kubectl get pods -n cert-manager
kubectl get crds | grep cert-manager
```

You want `cert-manager`, `cert-manager-cainjector`, `cert-manager-webhook` all `Running`. Don't create any `Certificate` until they are - if the webhook isn't up, requests get rejected and it looks like your YAML is wrong.

3. Or, the Helm way (the same three components, one release):

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true
```

> **Dikkat:** Helm ile kurarsan CRD'lar `crds/` klasöründen bir kez apply edilir ve `helm upgrade` ile **asla** güncellenmez - [helm](../helm/) notlarındaki o tuhaf kural burada da geçerli. Sürüm yükseltirken CRD'ları elle uygulamak ayrı bir adımdır.

## How to Issue a Certificate by Hand

Let's start with the mechanism spelled out - this path needs no extra flags and works the same on any Gateway controller.

1. Create a `SelfSigned` issuer. `synchat.internal` is not a real domain; no public CA will sign it:

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

2. Say what you want:

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

3. Watch it land:

```bash
kubectl get certificate synchat-tls
kubectl describe certificate synchat-tls
kubectl get secret synchat-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text
```

Both domains should show up under `Subject:` and `X509v3 Subject Alternative Name`.

4. Wire it into the `app-gateway` from the v1 [gateway](../../kubernetes/kubernetes-gateway-minikube/) notes. It's a `tls` block on the listeners:

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

Two listeners, the **same** `certificateRefs[].name`. That's deliberate: cert-manager merges the names and produces a single `Certificate`, de-duplicating `dnsNames`. Two hostnames, one certificate, one Secret.

> `tls.mode: Terminate` ends TLS at the gateway - the certificate is opened inside Envoy. `Passthrough` does not terminate and forwards the crypto to the backend, which this model does not support.

`certificateRefs` is only a name; where the Secret lives is assumed to be the Gateway's **own namespace** (`default` for `app-gateway`). That's why the Certificate went there too.

5. Pin the HTTPRoutes to their listeners with `sectionName`:

```yaml
spec:
  parentRefs:
    - name: app-gateway
      sectionName: web-https
```

## How to Derive Certificates from the Gateway

Writing `Certificate` YAML by hand gets old. The productivity gain is letting cert-manager read the Gateway and generate them:

```yaml
metadata:
  name: app-gateway
  annotations:
    cert-manager.io/cluster-issuer: selfsigned
```

With that annotation cert-manager starts watching the Gateway and creates the `Certificate` objects from its HTTPS listeners.

There is a cost: you have to **turn on** Gateway support. With the static manifest install that's a flag on the controller:

```bash
kubectl patch deployment cert-manager -n cert-manager --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--enable-gateway-api"}]'
kubectl rollout restart deployment cert-manager -n cert-manager
```

> On Helm the equivalent is `--set config.gatewayAPI.enabled=true`. The static manifest has no values.yaml, so the flag is the right tool.

### Which listeners get a Certificate?

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

**Özetlersek:** elle yaz = her yerde çalışır. annotation = cert-manager üretir ama `--enable-gateway-api` ve listener kısıtları gerekir. İkisi aynı sonucu verir; fark YAML'ı kimin yazdığı.

## How to Build a Real CA Chain

Everything so far worked - but open `https://synchat.internal` in a browser and you get "Your connection is not private". The reason matters: every self-signed certificate **is its own root**. To trust it you have to add that specific certificate to your trust store. Two hostnames, two separate trust problems.

The real world doesn't work like that. Let's Encrypt and corporate PKIs have a **root CA**; every leaf is signed by it and that one root is trusted once. You can build the same shape locally - and it is exactly what cert-manager recommends `SelfSigned` for.

1. Keep the `selfsigned` issuer, but use it only to bootstrap a root. Note `namespace: cert-manager`: a `ClusterIssuer` is not namespaced, so `ca.secretName` is assumed to live in the `cert-manager` namespace.

```yaml
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
```

2. Point a second `ClusterIssuer` at that Secret:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: synchat-ca
spec:
  ca:
    secretName: synchat-root-ca-secret
```

3. Re-issue the leaf from the new issuer - `issuerRef` changes from `selfsigned` to `synchat-ca`:

```yaml
spec:
  issuerRef:
    name: synchat-ca
    kind: ClusterIssuer
    group: cert-manager.io
```

4. Prove the chain is real:

```bash
kubectl get secret synchat-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -issuer -subject
```

Now `issuer=` and `subject=` must be **different**. Different means something signed it.

5. Add the root to your trust store once (macOS Keychain, or `/usr/local/share/ca-certificates/` on Linux) and both hostnames open without warnings:

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

> Distributing that root across a real fleet is not a `cp` command - that's what [trust-manager](./cert-manager-production/) is for, and it has its own trap around CA rotation.

## Ingress vs Gateway

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

## How to Troubleshoot a Stuck Certificate

TLS that doesn't work doesn't come with an error message, it comes with silence. In order:

**`Certificate` is `Ready: False`** — the detail is always there:

```bash
kubectl describe certificate synchat-tls
kubectl get certificaterequest
kubectl describe certificaterequest <name>
```

The `message` under Conditions is the real reason (`referenced signer resource does not exist`, `secret "..." not found`...).

**The `Secret` never appears** — is the `Certificate` `Ready`? Does `certificateRefs[].namespace` match the Gateway's? A cross-namespace reference is skipped silently.

**No Certificate is generated** — three checks in order: is `--enable-gateway-api` on (`kubectl get deployment cert-manager -n cert-manager -o jsonpath='{.spec.template.spec.containers[0].args}'`), is the annotation spelled right, does the listener satisfy the constraints table. An empty `hostname` skips the listener.

**HTTPS works but the name is wrong** — look at `dnsNames`. If it's wrong, fix `hostname` on the **listener**, not on the HTTPRoute.

**The browser still warns** — that's not a cert-manager problem, that's a trust problem. Use `openssl x509 -noout -issuer -subject` to tell whether a chain exists at all.

**The old certificate is still being served** — the Secret updated but Envoy hasn't reloaded it. Usually just waiting works.

## How to Verify TLS Actually Works

The full loop, end to end:

1. Build the CA chain (`selfsigned` → `synchat-root-ca` with `isCA: true` → `synchat-ca`), then ask `synchat-ca` for `synchat-tls` covering both hostnames.

2. Add `cert-manager.io/cluster-issuer: synchat-ca` to `app-gateway`, rewrite the listeners as `web-https` and `api-https` (both `tls.mode: Terminate`, same `certificateRefs[].name`), and pin the HTTPRoutes with `sectionName`.

3. Check the artifact, not the controller:

```bash
kubectl get certificate
kubectl get secret synchat-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -issuer -subject -ext subjectAltName
curl -kv https://synchat.internal 2>&1 | grep -E "subject:|issuer:|SSL certificate"
```

What you want: `READY` `True`, issuer and subject **different**, `subjectAltName` containing both names.

4. The payoff. Add the root to your trust store and open `https://synchat.internal` in a browser. When it opens with no warning, TLS is actually **working**.

> **Dikkat:** `Certificate`'ı Gateway'in namespace'i dışında yazarsan bozulur. `Certificate` namespace'i, `certificateRefs`'un bakacağı namespace ile **aynı** olmalı - `app-gateway` için bu `default`.

Getting a certificate and plugging it into a Gateway was the easy half. The next chapter takes this to a real cluster: ACME, Let's Encrypt, wildcards, renewal windows, trust distribution and monitoring.

for more [cert-manager-production](../cert-manager-production/)
