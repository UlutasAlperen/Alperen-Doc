---
title: "cert-manager-production"
weight: 17
---
# cert-manager in Production

The previous chapter taught the mechanism: `Issuer`, `Certificate`, `Secret`, and a Gateway that serves it. Everything worked because we controlled every variable - a fake domain, a self-signed root, a laptop that trusts whatever we tell it to.

Production removes that control. The domain is public and someone else owns the DNS. The certificate has to be trusted by browsers you don't administer. It expires on a schedule you don't choose. And when the automation silently stops working, the outage looks like "the website is down" rather than "a controller is in a crash loop".

So the topic changes. Not _how do I get a certificate_ but _how do I keep certificates flowing for years without anyone thinking about them_.

**Özetlersek:** local mekanizmayı öğretir, production ise hata modlarını. Önceki bölümde öğrendiğin her obje aynı kalır; değişen, onların yıllarca ayakta nasıl tutulacağı.

## The Honest Requirements

- **A domain you actually control** and a cluster the internet can reach. The "no global DNS" rule of the v1 notes stops applying here
- **A DNS provider API credential**, if you want wildcards or your cluster is behind NAT. This is the single most common failure point later
- **A monitoring stack.** Renewal that fails quietly is worse than no automation at all - see the [prometheus-stack](../prometheus-stack/) chapter for the alerting half
- **Somewhere to keep the ACME account key.** It's a Secret like any other; deleting it is not a reinstall, it's a new identity at the CA

## ACME and Let's Encrypt

[ACME](https://cert-manager.io/docs/configuration/acme/) is the protocol behind Let's Encrypt, ZeroSSL, Google Trust Services and friends. The shape is always the same:

1. you create an **account** with the CA (one key pair, stored by cert-manager in `privateKeySecretRef`)
2. you submit an **order** for some names
3. the CA issues a **challenge** proving you control those names
4. you answer it, the CA validates, and you get the certificate

The account key is your identity at the CA. Delete that Secret and cert-manager registers a new account - you keep issuing certificates, but you lose the ability to revoke anything issued by the old one.

### Staging first, always

Let's Encrypt runs two environments and they behave very differently:

| | Staging | Production |
|---|---|---|
| URL | `https://acme-staging-v02.api.letsencrypt.org/directory` | `https://acme-v02.api.letsencrypt.org/directory` |
| Trusted by browsers | no | yes |
| Rate limits | generous | strict |

The staging certificate is useless to a user and invaluable to you: it exercises the entire pipeline - DNS, routing, challenge completion, Secret writing - without burning your production quota.

**Rate limits** are the reason this matters:

| Limit | Value |
|---|---|
| Certificates per registered domain | 50 / week |
| Duplicate certificates | 5 / week |
| New orders per account | 300 / 3 hours |
| Failed validations | 5 / hour |

Hit these in production and you are locked out for a week. Hit them in staging and nothing happens.

> **Dikkat:** staging ile test etmeden production'a geçme. Rate limit'e takılmak bir haftalık outage demektir - ve bunu genelde ilk kez canlıya çıkarken, en kötü anda öğrenirsin.

### The issuer pair

Always ship both, and point workloads at staging until you've proven the pipeline:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: pki-team@example.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-staging-account-key
    solvers:
      - http01:
          gatewayHTTPRoute: {}
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: pki-team@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - http01:
          gatewayHTTPRoute: {}
```

`email` goes to a **team distribution list**, not an individual. It's how the CA tells you a certificate is about to be revoked - and the person who set this up will have left by then.

## How to Choose HTTP-01 or DNS-01

Two ways to prove you own a domain, and the choice has real consequences:

| | HTTP-01 | DNS-01 |
|---|---|---|
| Proof | a file at `/.well-known/acme-challenge/` over port 80 | a TXT record at `_acme-challenge.<name>` |
| Needs public port 80 | yes | no |
| Works behind NAT / private cluster | no | yes |
| Wildcard certificates | **no** | **yes** |
| Failure mode | firewall, wrong ingress path | DNS propagation, expired API token |
| Dependency | your ingress | your DNS provider's API |

### HTTP-01 on Gateway API

The solver spins up a temporary `acmesolver` pod and an `HTTPRoute` for the duration of the challenge:

```yaml
solvers:
  - http01:
      gatewayHTTPRoute:
        parentRefs:
          - name: app-gateway
            kind: Gateway
            namespace: envoy-gateway-system
            sectionName: http
```

That `sectionName: http` matters: the challenge is answered over plain HTTP, so the Gateway needs a live **HTTP:80** listener that the internet can reach. If your cluster is behind NAT or only exposes 443, HTTP-01 will hang in `Pending` forever.

### DNS-01 with a real provider

DNS-01 proves control by writing a TXT record. cert-manager needs an API credential for your DNS provider. Cloudflare, using an API **token** (scoped, not the global key):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token
  namespace: cert-manager
type: Opaque
stringData:
  api-token: "..."
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod-dns
spec:
  acme:
    email: pki-team@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
        selector:
          dnsZones:
            - "example.com"
```

Route53 on EKS, where credentials come from the pod's IAM role rather than a Secret:

```yaml
solvers:
  - dns01:
      route53:
        region: eu-central-1
        hostedZoneID: Z1234567890ABC
```

> **Dikkat:** `ClusterIssuer` namespace'e ait değildir, bu yüzden `secretName`'lerin `cert-manager` namespace'ine bakacağı varsayılır (`--cluster-resource-namespace` ile değiştirilebilir). `Issuer` kullanıyorsan Secret aynı namespace'te olmalı. AWS tarafında ambient credential'lar yalnızca `ClusterIssuer` için geçerlidir - bunun sebebi, `Issuer` oluşturma yetkisi olan birinin senin IAM rolünü kullanamaması.

## How to Get a Wildcard Certificate

`*.example.com` can **only** be obtained with DNS-01. HTTP-01 proves one hostname at a time and cannot validate a wildcard:

```yaml
dnsNames:
  - "example.com"
  - "*.example.com"
```

Note the limits: `*.example.com` covers `api.example.com` but **not** `api.staging.example.com`. Each level needs its own wildcard, and Let's Encrypt won't nest them.

## How to Mix Solvers on One Issuer

One issuer can carry several solvers, each gated by a `selector`. Wildcards on DNS-01, everything else on HTTP-01:

```yaml
solvers:
  - dns01:
      cloudflare:
        apiTokenSecretRef:
          name: cloudflare-api-token
          key: api-token
    selector:
      dnsZones:
        - "example.com"
  - http01:
      gatewayHTTPRoute: {}
```

Selectors are `dnsZones` (prefix match, covers subdomains), `dnsNames` (exact match, does **not** resolve wildcards) and `matchLabels` (on the `Certificate`). First match wins, so put the specific ones first.

## How to Control Renewal

A certificate you cannot renew is an outage with a calendar entry. cert-manager renews when **either** two-thirds of the lifetime has passed **or** `renewBefore` is reached, whichever comes first.

```yaml
spec:
  duration: 2160h   # 90 days
  renewBefore: 720h # renew 30 days before expiry
  privateKey:
    rotationPolicy: Always
```

`rotationPolicy: Always` issues a **new private key** on every renewal. The default (`Never`) reuses the key forever, which means a stolen key stays valid across renewals. Unless something explicitly pins the key, use `Always`.

### ARI

[ARI](https://cert-manager.io/docs/configuration/acme/) (Automatic Renewal Information) lets the CA push its own renewal window instead of you guessing. With `ExperimentalOptions` enabled, cert-manager polls the CA's `renewalInfo` endpoint and schedules inside the window the CA suggests. This is what saves you during mass revocation events: the CA says "renew now" and you respond without a human.

```yaml
config:
  featureGates:
    ExperimentalOptions: true
```

### Why the cadence is tightening

The CA/Browser Forum is shortening public certificate lifetimes on a published schedule: 200 days from March 2026, 100 days in 2027, 47 days in 2029.

That is not a footnote. At 47-day lifetimes, the default renewal point is around day 31 - so you reissue roughly **every two weeks**. Every link in the chain (solver, DNS API, Secret write, workload reload) has to work reliably at that cadence.

> **Dikkat:** `subPath` ile mount edilen Secret'lar rotate edildiğini **görmez**. Pod, eski sertifikayı süresi dolana kadar okumaya devam eder. Ya `subPath` kullanma ya da yanına bir reload sidecar koy.

**Özetlersek:** renewal'ı `duration`/`renewBefore` ile tanımla, ARI'yi aç, private key'i döndür. Sertifika yaşam döngüsü 90 günden 47 güne inerken otomasyon "iyi olur" olmaktan çıkar, tek hatanın outage olduğu tek nokta olur.

## How to Distribute Trust with trust-manager

Locally we added the root to a laptop's trust store. In production nobody is going to hand-edit trust stores across a fleet. [trust-manager](https://cert-manager.io/docs/trust/trust-manager/) is the answer: a small operator that assembles X.509 trust **bundles** and syncs them into every namespace that needs them.

It adds a `Bundle` resource: a list of `sources`, and a `target` saying where the result goes.

```yaml
apiVersion: trust.cert-manager.io/v1alpha1
kind: Bundle
metadata:
  name: synchat-trust
spec:
  sources:
    - configMap:
        name: synchat-root-source
        key: root-cert.pem
  target:
    configMap:
      key: "root-cert.pem"
    namespaceSelector:
      matchLabels:
        trust: synchat
```

Any namespace labelled `trust: synchat` gets a `ConfigMap` named `synchat-trust` holding the PEM bundle. You can also write JKS and PKCS#12 for the JVM crowd with `additionalFormats`, and target `Secret`s instead of `ConfigMap`s if you enable it at startup.

### `ca.crt` vs `tls.crt`

Two fields in a cert-manager Secret look relevant and they are not interchangeable:

- `tls.crt` often contains a **chain** - leaf plus intermediates. Don't use it as a trust source.
- `ca.crt` is the issuer's certificate, populated on a best-effort basis and only correct if the issuer is configured for it.

Prefer bundles built from **root certificates**, and pick whichever field holds exactly one root.

### The rotation trap

This is the mistake that causes real outages, and the docs call it out explicitly: **do not point a `Bundle` directly at the Secret cert-manager writes the CA into.**

Why: rotate the issuer and cert-manager rewrites that Secret. trust-manager sees the change and immediately swaps the bundle - instantly distrusting the old root. Anything still holding a certificate from the old root breaks at that exact moment.

The safe workflow is a deliberate rollover:

1. copy the current root out of the cert-manager Secret into a dedicated `ConfigMap`
2. point the `Bundle` at **that** copy
3. when a new root arrives, put **both** roots in the copy
4. rotate workloads and leaf certificates until nothing depends on the old root
5. only then remove the old root from the copy

That extra copy is what gives you control over when trust propagates. It feels redundant right up until the first rotation.

## How to Monitor and Alert

cert-manager exposes Prometheus metrics on port `9402`. Three of them matter:

```promql
# when does it expire (unix seconds)
certmanager_certificate_expiration_timestamp_seconds

# is it ready (1 = current condition)
certmanager_certificate_ready_status

# when is renewal scheduled
certmanager_certificate_renewal_timestamp_seconds
```

Alert on the **failure of the automation**, not just on expiry. By the time a certificate has expired the outage is already public:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cert-manager-alerts
  namespace: monitoring
spec:
  groups:
    - name: certificates
      interval: 30s
      rules:
        - alert: CertificateExpiresCritical
          expr: |
            ((certmanager_certificate_expiration_timestamp_seconds - time()) / 3600 < 24)
            and certmanager_certificate_expiration_timestamp_seconds > 0
          labels:
            severity: critical
          annotations:
            summary: "Certificate {{ $labels.name }} expires in under 24h"

        - alert: CertificateExpiresWarning
          expr: |
            ((certmanager_certificate_expiration_timestamp_seconds - time()) / 86400 < 7)
            and certmanager_certificate_expiration_timestamp_seconds > 0
          labels:
            severity: warning
          annotations:
            summary: "Certificate {{ $labels.name }} expires in under 7 days"

        - alert: CertificateNotReady
          expr: certmanager_certificate_ready_status{condition!="True"} == 1
          for: 10m
          labels:
            severity: critical
          annotations:
            summary: "Certificate {{ $labels.name }} has been not ready for 10m"

        - alert: CertificateRenewalOverdue
          expr: |
            ((time() - certmanager_certificate_renewal_timestamp_seconds) / 3600 > 6)
            and certmanager_certificate_renewal_timestamp_seconds > 0
            and ((certmanager_certificate_expiration_timestamp_seconds - time()) / 86400 < 30)
          for: 30m
          labels:
            severity: warning
          annotations:
            summary: "Certificate {{ $labels.name }} renewal is overdue"
```

`CertificateRenewalOverdue` is the important one. It fires when cert-manager **should** have renewed and didn't - the automation is broken while the certificate still looks fine. That's your warning window.

For a quick manual sweep there's `cmctl`:

```bash
cmctl check certificate --all-namespaces
```

## How to Harden and Run It HA

cert-manager holds private keys for everything in the cluster. Treat it accordingly.

**The root key does not belong in the cluster.** If you run your own PKI, keep the root offline and issue an **intermediate** CA into the cluster. Give that intermediate `pathLen: 0` (so anything it signs cannot itself be a CA) and **name constraints** restricting issuance to the hostnames you control. Then a compromise of the cluster cannot mint certificates for your other domains.

**Encrypt Secrets at rest.** etcd encryption is what protects `tls.key` from someone with disk access. Without it, a backup leak is a key leak.

**Make it highly available.** cert-manager is a controller, so only one replica is active at a time - but the webhook and cainjector take traffic. Run multiple replicas of all three, put a `PodDisruptionBudget` on them so a node drain doesn't take the whole thing out, and use anti-affinity so they don't land on the same node.

**Least privilege on the CA Secrets.** Only cert-manager's service account should read them. Everything else in the cluster reading those Secrets is a standing risk.

**Gate who can request certificates.** `approver-policy` lets you define which `CertificateRequest`s are allowed to be approved and by whom. Without it, anyone who can create a `Certificate` can get one signed.

**Rotate DNS credentials on a calendar.** An expired DNS API token is the single most common cause of DNS-01 renewal failure - and it fails silently, at 3am, months after whoever configured it moved on.

**One ACME account per cluster.** Sharing an account key across clusters causes order races and makes rate-limit attribution impossible.

**Özetlersek:** kök anahtarı cluster dışında tut, etcd'yi şifrele, üç controller'ı da yedekli çalıştır, CA Secret'larına RBAC kısı, DNS credential'ını takvimle döndür. Bunlar "sonra bakarım" listesi değil; sertifika otomasyonu tek başına çalışırken bunlar olmadan sessizce bozulur.

## Production Checklist

- [ ] Staging issuer tested end to end before touching production
- [ ] `email` is a team list, not a person
- [ ] Separate ACME account keys per cluster
- [ ] DNS-01 credentials scoped to one zone and rotated on schedule
- [ ] Wildcards via DNS-01; HTTP-01 only where port 80 is genuinely open
- [ ] `duration` / `renewBefore` set explicitly
- [ ] `privateKey.rotationPolicy: Always`
- [ ] ARI feature gate enabled
- [ ] No `subPath` mounts of TLS Secrets
- [ ] trust-manager bundles built from a **copied** root, not the live CA Secret
- [ ] Alerts on `CertificateNotReady` and `CertificateRenewalOverdue`
- [ ] etcd encryption at rest enabled
- [ ] CA Secrets readable only by cert-manager
- [ ] Controller, webhook and cainjector have replicas + PDB

## How to Test the Whole Thing

This needs a domain you control and a cluster the internet can reach. If you don't have one, still write the YAML and reason through where it would fail.

1. Write the staging/prod `ClusterIssuer` pair. Pick **either** HTTP-01 or DNS-01 based on whether your cluster exposes port 80 publicly.

2. Point a `Certificate` at the **staging** issuer first. Watch it, then read the failure if it fails:

```bash
kubectl describe certificate <name>
kubectl describe certificaterequest <name>
kubectl logs -n cert-manager deploy/cert-manager --tail=100
```

3. Once staging is `Ready: True`, switch `issuerRef` to the production issuer and confirm a browser trusts it. `openssl x509 -noout -issuer` makes the staging intermediate obvious.

4. If you went DNS-01, request a wildcard too: `*.yourdomain.com`. Watch cert-manager create the `_acme-challenge` TXT record and clean it up after.

5. Ship the `PrometheusRule` above. Without Prometheus, port-forward the metrics endpoint and evaluate the expressions by hand:

```bash
kubectl port-forward -n cert-manager deploy/cert-manager 9402:9402 &
curl -s localhost:9402/metrics | grep certmanager_certificate_
```

6. The exercise that actually teaches: delete the DNS API token Secret and wait. Watch the renewal fail **quietly**, then work out which alert would have caught it and at what time. That gap between "it broke" and "you found out" is the thing this chapter is really about.

> **Dikkat:** production'a geçerken ilk yaptığın şey production issuer'ı yaratmak olmasın. Staging'de tüm zinciri - DNS, routing, challenge, Secret yazımı - görmeden production issuer'a geçersen, hata ayıklamayı rate limit'lerle yarışarak yaparsın.

for more [prometheus-stack](../prometheus-stack/)
