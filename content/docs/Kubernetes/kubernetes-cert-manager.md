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

cert-manager'ın tamamı üç nesne etrafında döner:

- **`Issuer` / `ClusterIssuer`** — _kim_ imzalar. `Issuer` tek namespace'te geçerli, `ClusterIssuer` tüm cluster'da. İkisinin spec'i aynı; tek fark kapsamı.
- **`Certificate`** — _ne_ isteriz: hangi isimler (`dnsNames`), hangi imzalayıcıdan (`issuerRef`), hangi `Secret`'a yazılsın (`secretName`).
- **`Secret`** — sonuç. `tls.crt`, `tls.key`, genelde `ca.crt`. Gateway'in ve Ingress'in gerçekten tükettiği tek şey bu.

Rol dağılımı net: Issuer politika, Certificate istek, Secret sonuç. Gateway araya girip o Secret'ı son kullanıcıya TLS olarak sunar.

# Installing cert-manager

Kurs boyunca her şeyi `kubectl apply` ile kuruyoruz - Helm'i [ileride](../../kubernetes_v2/helm/) konuşacağız. Aynı yaklaşım burada da geçerli:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.21.0/cert-manager.yaml
```

Kurulum üç parçadan oluşur: CRD'lar (`Certificate`, `Issuer`, `CertificateRequest`...), `cert-manager` namespace'i, ve controller'ın kendisi. Birkaç saniye sonra hazır olduğunu doğrula:

```bash
kubectl get pods -n cert-manager
kubectl get crds | grep cert-manager
```

Üç pod görmelisin: `cert-manager`, `cert-manager-cainjector`, `cert-manager-webhook`. Hepsi `Running` olana kadar `Certificate` oluşturma - webhook hazır değilse isteklerin reddedilir.

> **Dikkat:** Gateway API CRD'ları cert-manager'dan **önce** kurulmuş olmalı, ya da cert-manager'ı restart etmelisin. Bazı bileşenler bu kontrolü sadece startup'ta yapar. Önceki bölümde Envoy Gateway'i kurduğumuz için CRD'lar zaten orada - ama sıralamayı bil. Çözümü tek satır: `kubectl rollout restart deployment cert-manager -n cert-manager`

# Path A: Explicit Certificate

Önce mekanizmayı elle gösterelim - bu yol hiçbir ek flag istemez.

## The SelfSigned issuer

Local'de `synchat.internal` gibi bir domain gerçek değildir; Let's Encrypt'e imzalatamazsın. Bu yüzden kendi kendini imzalayan bir imzalayıcı kullanırız:

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

`READY` kolonu `True` olmalı. `SelfSigned` imzalayıcısının hiçbir dış bağımlılığı yoktur - sertifika kendi private key'iyle kendini imzalar, ortada ayrı bir CA yoktur.

## The Certificate

Şimdi ne istediğimizi yazıyoruz:

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

`subject` satırı süs değil. SelfSigned sertifikalarda Subject DN = Issuer DN olduğundan, subject'i boş bırakırsan Issuer DN'i de boş kalır ve X.509 spesifikasyonu bunu teknik olarak geçersiz sayar. cert-manager bu durumda `BadConfig` diye bir event basar. Bir satırla kurtuluyorsun, ekle.

```bash
kubectl get certificate synchat-tls
kubectl describe certificate synchat-tls
```

`READY` `True` olduğunda `Secret` da oluşmuştur:

```bash
kubectl get secret synchat-tls
kubectl get secret synchat-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text
```

`Subject:` ve `X509v3 Subject Alternative Name` satırlarında iki domain'i de görmelisin.

## Wiring it into the Gateway

Şimdi [gateway](../kubernetes-gateway-minikube/) bölümündeki `app-gateway`'i HTTPS'e çevirelim. Listener'lara `tls` bloğu ekliyoruz:

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

İki listener, **aynı** `certificateRefs[].name`. Bu bilerek yapıldı: cert-manager isimleri birleştirip tek `Certificate` üretir ve `dnsNames` içinde tekrarları ezer. İki hostname, tek sertifika, tek Secret. Hostname'leri ayrı ayrı sertifikalamak da mümkün ama neden?

> `tls.mode: Terminate` trafikte TLS'i sonlandırır - sertifika burada, Envoy'un içinde açılır. `Passthrough` sertifikayı sonlandırmaz, kriptoyu doğrudan backend'e iletir ve bu modelde desteklenmez.

`certificateRefs` yalnızca bir isimdir; Secret'ın nerede olduğunu Gateway'in **kendi namespace'i** varsayar (`app-gateway` için `default`). Certificate'ı da oraya yazdık.

HTTPRoute'ların hangi listener'a takılacağını `sectionName` ile açıkça söylemek iyi bir alışkanlıktır:

```yaml
spec:
  parentRefs:
    - name: app-gateway
      sectionName: web-https
```

# Path B: The Annotation

Path A'da `Certificate`'ı elle yazdık. Asıl üretkenlik, onu Gateway'in kendisinden türetmekte:

```yaml
metadata:
  name: app-gateway
  annotations:
    cert-manager.io/cluster-issuer: selfsigned
```

Bu annotation'ı ekleyince cert-manager Gateway'i izlemeye başlar, HTTPS listener'larından `Certificate`'ları kendisi üretir. Elle `Certificate` yazmayı tamamen bırakabilirsin.

Ama bu yolun bir bedeli var: Gateway desteğini **açman** gerekir. Raw manifest ile kurduysan (ki öyle yaptık) bu bir flag'dir:

```bash
kubectl patch deployment cert-manager -n cert-manager --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--enable-gateway-api"}]'
kubectl rollout restart deployment cert-manager -n cert-manager
```

> Helm kullanıyorsan karşılığı `--set config.gatewayAPI.enabled=true`. Raw manifest'in values.yaml'ı olmadığı için flag doğru araç.

## Which listeners get a Certificate?

cert-manager her listener'ı tek tek değerlendirir ve işe yaramayanları **sessizce atlar**. Certificate bir türlü oluşmuyorsa sebebi neredeyse her zaman bu tablodur:

| Alan | Gereksinim |
|---|---|
| `hostname` | boş olamaz |
| `tls.mode` | `Terminate` yazılmalı; `Passthrough` desteklenmiyor |
| `tls.certificateRefs[].name` | zorunlu |
| `tls.certificateRefs[].kind` | yazılacaksa `Secret` |
| `tls.certificateRefs[].group` | yazılacaksa `""` |
| `tls.certificateRefs[].namespace` | yazılacaksa Gateway'inkiyle **aynı** olmalı |

> **Dikkat:** `dnsNames` HTTPRoute'un `hostnames` alanından **değil**, listener'ın `hostname` alanından gelir. HTTPRoute'ta yazdığın isimler yönlendirme içindir; TLS isimleri listener'da yaşar. İkisini karıştırmak, "Certificate oluştu ama yanlış isimlere" probleminin bir numaralı sebebi.

**Özetlersek:** Path A = her şeyi sen yaz, her yerde çalışır. Path B = annotation yaz, cert-manager üretir, ama `--enable-gateway-api` ve listener kısıtları gerekir. İkisi aynı sonucu verir; fark YAML'ı kimin yazdığı.

# Trust: A Real CA Instead

Şu ana kadar `SelfSigned` kullandık ve çalıştık - ama tarayıcıya `https://synchat.internal` yazdığında "Your connection is not private" ikazını görürsün. Sebebi önemli: her self-signed sertifika **kendi root'udur**. Güvenmek istersen o sertifikayı tek tek trust store'a eklemen gerekir. İki hostname, iki ayrı güven problemi.

Gerçek dünyada böyle çalışmaz. Let's Encrypt ve kurumsal PKI'lerde bir **root CA** vardır, tüm yaprak sertifikalar ondan imzalanır ve o root'a bir kez güvenilir. Aynı kalıbı local'de kurabiliriz - cert-manager'ın da `SelfSigned`'ı tam olarak bunun için önerdiği şey budur:

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

Zinciri okumak önemli, çünkü her satırın bir sebebi var:

1. `selfsigned` yalnızca **ilk adımı** atar - tek işi root'u bootstrap etmek
2. `synchat-root-ca` bir sertifika ama `isCA: true` ile işaretli; kendi kendine imzalanır ve `cert-manager` namespace'indeki bir `Secret`'a yazılır
3. `synchat-ca` adında yeni bir `ClusterIssuer` o `Secret`'ı CA olarak kullanır
4. `synchat-tls` artık `selfsigned`'dan değil `synchat-ca`'dan ister - yani root tarafından **imzalanır**

`namespace: cert-manager` satırı tesadüf değil: `ClusterIssuer` namespace'e ait değildir, o yüzden `ca.secretName`'in `cert-manager` namespace'ine bakacağı varsayılır. Root CA'yı oraya koy.

Sonucu görmek için:

```bash
kubectl get secret synchat-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -issuer -subject
```

Artık `issuer=` ve `subject=` **farklı** olmalı. Farklıysa zincir çalışıyor demektir.

Kök CA'yı bir kez trust store'a eklediğinde (macOS Keychain, Linux'ta `/usr/local/share/ca-certificates/`) iki hostname de ikazsız açılır. `ca.crt` zaten Secret'ın içinde:

```bash
kubectl get secret synchat-root-ca-secret -n cert-manager -o jsonpath='{.data.ca\.crt}' | base64 -d > synchat-root-ca.pem
```

| | `SelfSigned` tek başına | `isCA` + `CA` zinciri |
|---|---|---|
| Obje sayısı | 2 | 4 |
| Trust anchor | her sertifika ayrı | tek root, bir kez |
| İki hostname | iki kez güven | root'a bir güven |
| Issuer zinciri | yok | gerçek |
| Tarayıcı | ikaz | ikazsız |

# Ingress vs Gateway

cert-manager her iki API'de de çalışır, ama TLS'in **nereye yazıldığı** değişir. Ingress'ten geliyorsan en çok burada takılırsın:

| | Ingress | Gateway |
|---|---|---|
| Annotation | `Ingress`'in üstünde | `Gateway`'in üstünde |
| TLS nerede | `spec.tls[].secretName` | listener `tls.certificateRefs[].name` |
| İsimler nereden | `spec.tls[].hosts` | listener `hostname` |
| Kim yönetir | uygulama ekibi (kendi Ingress'i) | platform ekibi (ortak Gateway) |
| Alt nesne | yok | `sectionName` ile listener'a bağlanır |

Annotation'lar aynıdır - `cert-manager.io/issuer`, `cert-manager.io/cluster-issuer`, `cert-manager.io/duration`, `cert-manager.io/renew-before`. Sadece girdikleri nesne değişir.

> 2026'nın başından beri `ingress-nginx` resmi olarak EOL. Yeni kurulumda Gateway API doğru hedef; Ingress'i bil ama üzerine yeni bir şey inşa etme.

**Özetlersek:** Ingress'te TLS'i *sen* Ingress'in içinde tarif edersin, app ekibi kendi başına yönetir. Gateway'de TLS *Gateway*'in içinde yaşar ve genelde platform ekibinin kontrolündedir - ki bu yüzden `ListenerSet` diye bir kaynak geliştiriliyor.

# In Production: Let's Encrypt

Local'de `synchat.internal` hiçbir işe yaramaz çünkü gerçek bir domain değildir. Production'da ise Let's Encrypt bedava sertifika verir - ama kim olduğunu **kanıtlamanı** ister. Buna ACME challenge denir ve iki yolu vardır: HTTP-01 (domain'in 80 portunda seni doğrular) veya DNS-01 (DNS kaydını değiştirirsin).

Gateway API ile HTTP-01'in kurulumu şöyledir:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-account-key
    solvers:
      - http01:
          gatewayHTTPRoute:
            parentRefs:
              - name: app-gateway
                kind: Gateway
```

`gatewayHTTPRoute` solver'ı challenge süresince geçici `HTTPRoute`'lar üretir; bunun için Gateway'inde bir **HTTP:80 listener** yaşamalıdır. Sonrasında sadece `cert-manager.io/cluster-issuer: letsencrypt` annotation'ını ekle, gerisini cert-manager halleder.

Bu kursun kapsamında değil, çünkü iki şartı birden istiyor: **herkese açık bir domain** ve **o domain'in 80 portuna dışarıdan erişilebilmesi**. İkisi de minikube'ta yok. Buraya yazmamızın sebebi gerçek kurulumun neye benzediğini görmek.

> **Dikkat:** Certificate bir namespace'te, Gateway başka bir namespace'te ise `certificateRefs` cross-namespace olur ve Gateway API bunu varsayılan olarak reddeder. Çözüm `ReferenceGrant`:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-gateway-tls
  namespace: apps
spec:
  from:
    - group: gateway.networking.k8s.io
      kind: Gateway
      namespace: default
  to:
    - group: ""
      kind: Secret
```

# Troubleshooting

TLS'in çalışmaması bir hata mesajıyla gelmez, sessizce gelir. Sıra:

**`Certificate` `Ready: False`** — her zaman detayı oradadır:

```bash
kubectl describe certificate synchat-tls
kubectl get certificaterequest
kubectl describe certificaterequest <isim>
```

Conditions altındaki `message` alanı gerçek sebebi söyler (`referenced signer resource does not exist`, `secret "..." not found`...).

**`Secret` hiç oluşmuyor** — Certificate `Ready` mi? `certificateRefs[].namespace` Gateway'in namespace'iyle aynı mı? Cross-namespace yazdıysan sessizce atlanır.

**Certificate hiç türemiyor (Path B)** — üç şeyi sırayla kontrol et: `--enable-gateway-api` açık mı (`kubectl get deployment cert-manager -n cert-manager -o jsonpath='{.spec.template.spec.containers[0].args}'`), annotation ismi doğru mu, listener kısıtlar tablosuna uyuyor mu. `hostname` boşsa listener atlanır.

**HTTPS çalışıyor ama isim yanlış** — `dnsNames`'e bak. Yanlışsa `hostname`'i **listener'da** düzelt, HTTPRoute'ta değil.

**Tarayıcı hâlâ ikaz ediyor** — bu bir cert-manager sorunu değil, trust sorunudur. `SelfSigned` kullanıyorsan CA zincirine geç ya da kökü trust store'a ekle. `openssl x509 -noout -issuer -subject` ile zincirin gerçekten var olup olmadığını ayırt et.

**Eski sertifika hâlâ sunuluyor** — Secret güncellenmiş ama Envoy yeniden okumamıştır. Gateway'i restart etmek yerine beklemek genelde yeterli; controller'lar Secret değişikliğini yakalar.

# Assignment

Gateway bölümünde kurduğumuz `app-gateway`'i gerçekten HTTPS yapalım - ikaz almadan.

1. CA zincirini kur: `selfsigned` ClusterIssuer, `isCA: true` ile `synchat-root-ca` (`cert-manager` namespace'inde), `synchat-ca` ClusterIssuer. Hepsi tek dosyada olabilir.

2. `synchat-tls` Certificate'ını yaz. `secretName: synchat-tls`, `dnsNames` olarak `synchat.internal` **ve** `synchatapi.internal`, `issuerRef` → `synchat-ca`.

3. `app-gateway`'e `cert-manager.io/cluster-issuer: synchat-ca` annotation'ını ekle. Path B'yi kullanıyorsan `--enable-gateway-api`'ın açık olduğunu doğrula, sonra Certificate'ı elle yazdığın dosyayı sil - cert-manager üretecek. Hangi yolu seçersen seç, listener'ları `web-https` ve `api-https` olarak yeniden yaz, ikisi de `tls.mode: Terminate` ve aynı `certificateRefs[].name`'i kullansın.

4. HTTPRoute'lara `sectionName` eklemeyi unutma: web için `web-https`, api için `api-https`.

5. Doğrula:

```bash
kubectl get certificate
kubectl get secret synchat-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -issuer -subject -ext subjectAltName
curl -kv https://synchat.internal 2>&1 | grep -E "subject:|issuer:|SSL certificate"
```

İstediğin sonuç: `certificate` `READY` `True`, issuer ile subject **farklı**, `subjectAltName` iki ismi de içeriyor, ve `curl` çıktısında ikisi de görünüyor.

6. Son adım - asıl ödül. Kök CA'yı trust store'una ekle ve tarayıcıda `https://synchat.internal`'ı aç:

```bash
kubectl get secret synchat-root-ca-secret -n cert-manager -o jsonpath='{.data.tls\.crt}' | base64 -d > synchat-root-ca.pem
```

İkazsız açıldığında TLS'in aslında **çalıştığını** görmüş olursun. Bir sertifikayı elle hazırlayıp Gateway'e takmak kolay kısmıydı; asıl iş onu güvenilir ve kendini yenileyen hale getirmekti.

> **Dikkat:** `Certificate`'ı Gateway'in namespace'i dışında yazarsan sonuç sessizce bozulur. `Certificate` namespace'i, `certificateRefs`'un bakacağı namespace ile **aynı** olmalı - `app-gateway` için bu `default`.

for more [namespaces](../kubernetes-namespaces/)
