---
title: "docker-distroless-container-images"
weight: 14
---
### Distroless Container Image'leri: İçlerinde Ne Var, Neden Kullanılır?

#### 1. Distroless Nedir, Neden Var?

`debian`, `ubuntu` veya türevlerinden (`node:lts`, `python:3` gibi) `FROM` ile başlayan container image'leri, genellikle dolu dolu bir Linux dağıtımıyla birlikte gelir. Gündelik işler için harikadır ama çoğu container uygulaması çalışma zamanında bu araçların ve kütüphanelerin neredeyse hiçbirine ihtiyaç duymaz. Sonuç? Teknik bir gerekçe olmaksızın daha büyük image boyutu ve yönetmesi gereken daha fazla güvenlik açığı (CVE).

Küçük, güvenli image'ler üretme isteği gayet doğal. Bunun en ekstrem yolu `FROM scratch` ile başlamaktır; yani boş bir base image'e sadece uygulamamızın gerçekten ihtiyaç duyduğu dosyaları eklemek. Ancak `FROM scratch` ile başlayan container'lar varsayılan olarak şunları içermez:

- `/tmp`, `/home`, `/var` gibi sistem dizinleri.
- HTTPS için CA sertifikaları.
- Kullanıcı yönetim dosyaları (`/etc/passwd`, `/etc/group`).
- Dinamik bağlanan uygulamalar için paylaşılan kütüphaneler.
- Saat dilimi bilgisi (tzdata).

...ve ihtiyaca göre daha fazlası.

> **Önemli:** `FROM scratch` container'ları temiz bir başlangıç sunar ama bu boşlukları elle doldurmak ciddi manuel emek ister; bu haliyle çoğu zaman üretim için eksik ve sorunludur.

İşte tam bu noktada distroless image'ler sahneye çıkar! [GoogleContainerTools/distroless](https://github.com/GoogleContainerTools/distroless) projesi, `scratch`'e mümkün olduğunca yakın ama gerekli sistem dosyaları ve dizinleri yerinde olan hazır minimal base image'ler sağlar. Yapmamız gereken tek şey hiyerarşilerini anlamak ve uygulamamıza en uygun olanını seçmek.

> İlgili doküman: Multi-stage build zaten image küçültmenin bir yolu olarak [optimize-container-images-with-multi-stage-builds](../optimize-container-images-with-multi-stage-builds/) dokümanında anlatılmıştı; distroless de aynı hedefe giden ama tamamen farklı bir yoldur.

#### 2. Birinci Katman: `gcr.io/distroless/static`

Distroless ailesiyle tanışmak için iyi bir başlangıç noktası `gcr.io/distroless/static` image'idir:

```bash
docker pull gcr.io/distroless/static
docker images
```

Çıktı:

```
REPOSITORY                  TAG       IMAGE ID       SIZE
gcr.io/distroless/static    latest    5d7d2b425607   1.99MB
```

Dosya sistemini incelediğimizde şunları görürüz:

- Sadece ~2MB (alpine image'ının ~%25'i).
- Tipik bir Linux dağıtım dizin yapısına sahip.
- `/etc/passwd`, `/etc/group` hatta `/etc/nsswitch.conf` dosyaları yerinde.
- Sertifikalar ve saat dilimi bilgisi de mevcut.
- Image Debian bazlı (yani distroless image'inin içinde aslında bir dağıtım var, ama eti kemiklerine kadar soyulmuş).
- Lisanslar da korunmuş görünüyor (ben telif uzmanı değilim ama).

Yani %99.99 statik. Paket yok, paket yöneticisi yok, `libc` yok ve **0 CVE**:

```bash
trivy image gcr.io/distroless/static
```

```
gcr.io/distroless/static (debian 12.9)

Total: 0 (UNKNOWN: 0, LOW: 0, MEDIUM: 0, HIGH: 0, CRITICAL: 0)
```

> **Özet:** `gcr.io/distroless/static`, `scratch`'in daha pratik eşdeğeridir; gerekli sistem dosyalarını ve dizinlerini sağlayan tamamen minimal bir base'dir ve CVE tanımaz.

#### 3. Her Program Statik Bağlanmış Değildir

`FROM scratch` denemelerinin güzel bir yan faydası, bir programın çalışması için aslında neye ihtiyaç duyduğunu anlamamıza yardımcı olmasıdır. Statik bağlanmış (statically linked) bir executable için genellikle birkaç config dosyası ve düzgün bir rootfs dizin yapısı yeterlidir. Peki dinamik bağlanmış (dynamically linked) bir executable için?

Bunu görmek için CGO etkin şekilde derlenmiş bir Go programı hazırlıyoruz (Go'da bile CGO devreye girince `libc` gibi paylaşılan kütüphanelere muhtaç hale geliyoruz):

`~/Dockerfile.cgo`:

```dockerfile
# syntax=docker/dockerfile:1
# -=== Builder image ===-
FROM golang:1 AS builder
WORKDIR /app

COPY <<EOF main.go
package main

import (
  "fmt"
  "os/user"
)

func main() {
  u, err := user.Current()
  if err != nil {
    panic(err)
  }
  fmt.Println("Hello from", u.Username)
}
EOF

RUN CGO_ENABLED=1 go build main.go

# -=== Target image ===-
FROM ubuntu
COPY --from=builder /app/main /
CMD ["/main"]
```

```bash
docker build -f ~/Dockerfile.cgo -t go-cgo-ubuntu .
docker run --rm go-cgo-ubuntu
```

```
Hello from root
```

Program çalışıyor. Şimdi kutsal `ldd` ile hangi kütüphaneleri dinamik olarak yüklüyor bakalım:

```bash
docker run --rm go-cgo-ubuntu ldd /main
```

```
linux-vdso.so.1 (0x00007ffd9acd5000)
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fb862292000)
/lib64/ld-linux-x86-64.so.2 (0x00007fb8624a8000)
```

Standart bir dinamik Linux executable'ının ihtiyaç duyduğu paylaşılan kütüphane seti, dahil `libc`. Ama tabii ki bunların hiçbiri `gcr.io/distroless/static` image'inde bulunmuyor...

#### 4. İkinci Katman: `gcr.io/distroless/base` ve `base-nossl`

`gcr.io/distroless/static`, programınız statik bağlanmış bir Go binary'siyse mükemmel bir seçim. Peki CGO kullanmak zorundaysak ve bağlı olduğunuz kütüphaneler statik bağlanamıyorsa (`libc`'ye bakarım), ya da Rust, C veya statik build desteği Go kadar iyi olmayan başka derlenmiş bir dil yazıyorsanız? Karşınızda `gcr.io/distroless/base` ve `gcr.io/distroless/base-nossl` image'leri:

```bash
docker pull gcr.io/distroless/base
docker pull gcr.io/distroless/base-nossl
docker images
```

```
REPOSITORY                     TAG       IMAGE ID       SIZE
gcr.io/distroless/static       latest    5d7d2b425607   1.99MB
gcr.io/distroless/base-nossl   latest    ae4cc24e698d   14.8MB
gcr.io/distroless/base         latest    fab58a7ef52e   20.7MB
```

Daha küçük olan `gcr.io/distroless/base-nossl` dosya sistemini incelediğimizde:

- `gcr.io/distroless/static`'ten yaklaşık 7 kat büyük (ama hala sadece ~15MB).
- Tamamen `gcr.io/distroless/static` üzerine kurulu (rootfs düzeni, CA sertifikaları, tzdata vb. dahil).
- Ekstra katmanlar bir düzine paylaşılan kütüphane getiriyor; en önemlileri `libc`, `libresolv`, `libnss` ve `libpthread`.
- ...ve yine, tipik bir Linux dağıtımı kalabalığı yok.

Biraz daha büyük olan `gcr.io/distroless/base` ise tamamen `gcr.io/distroless/base-nossl` üzerine kurulu ve fazladan tek bir paylaşılan kütüphane getiriyor: `libssl` (ve bağımlılıkları).

`gcr.io/distroless/base`'in `libc` ve `libssl` kütüphanelerinden gelen birkaç CVE'si var, ama çoğu zaman sorun olmaz çünkü bu kütüphanelerdeki kritik CVE'ler nadir görülür ve hızla düzeltilir:

```bash
trivy image gcr.io/distroless/base
```

```
gcr.io/distroless/base (debian 12.9)

Total: 8 (UNKNOWN: 0, LOW: 7, MEDIUM: 1, HIGH: 0, CRITICAL: 0)
```

Go örneğimizi yeni distroless base ile çalışacak şekilde düzenliyoruz:

`~/Dockerfile.cgo` (target stage):

```dockerfile
...build stage aynı kalıyor...

# -=== Target image ===-
# 'FROM scratch' yerine 'FROM gcr.io/distroless/base-nossl'
FROM gcr.io/distroless/base-nossl

COPY --from=builder /app/main /
CMD ["/main"]
```

> **Özet:** `gcr.io/distroless/base-nossl` ve `gcr.io/distroless/base`, `glibc`'ye (ve isteğe bağlı olarak `libssl`'ye) bağımlı dinamik bağlanmış uygulamalar için `FROM scratch`'in daha pratik eşdeğerleridir.

#### 5. Üçüncü Katman: `gcr.io/distroless/cc`

Önceki bölümde Rust'tan bahsetmiştim; çünkü bu günlerde oldukça popüler. `gcr.io/distroless/base` image'iyle gerçekten çalışabilir mi? Basit bir hello-world programı yazalım:

`~/Dockerfile.rust`:

```dockerfile
# syntax=docker/dockerfile:1
# -=== Builder image ===-
FROM rust:1 AS builder
WORKDIR /app

COPY <<EOF Cargo.toml
[package]
name = "hello-world"
version = "0.0.1"
EOF

COPY <<EOF src/main.rs
fn main() {
  println!("Hello world! (Rust edition)");
}
EOF

RUN cargo install --path .

# -=== Target image ===-
FROM gcr.io/distroless/base
COPY --from=builder /usr/local/cargo/bin/hello-world /
CMD ["/hello-world"]
```

Çalıştıralım:

```bash
docker build -f ~/Dockerfile.rust -t distroless-base-rust .
docker run --rm distroless-base-rust
```

```
/hello-world: error while loading shared libraries:
libgcc_s.so.1: cannot open shared object file:
No such file or directory
```

Dayyum! Görünüşe göre `gcr.io/distroless/base` image'i tüm gerekli paylaşılan kütüphaneleri sağlamıyor. Rust'ın bir çalışma zamanı bağımlılığı var: `libgcc`. Ve bu, container'da yok.

Dinamik bağlanmış binary'ler için bu bağımlılık o kadar yaygın ki, ona özel bir distroless base image bile tanıtılmış: `gcr.io/distroless/cc`:

```bash
docker pull gcr.io/distroless/cc
docker images
```

```
REPOSITORY                     TAG       IMAGE ID       SIZE
gcr.io/distroless/static       latest    5d7d2b425607   1.99MB
gcr.io/distroless/base-nossl   latest    ae4cc24e698d   14.8MB
gcr.io/distroless/base         latest    fab58a7ef52e   20.7MB
gcr.io/distroless/cc           latest    6f09ff5d0af8   23.4MB
```

`gcr.io/distroless/cc` dosya sistemini incelediğimizde:

- Tamamen `gcr.io/distroless/base` üzerine kurulu.
- Yeni katmanlar image boyutuna sadece ~2MB ekliyor.
- Image, `libgcc`'yi (ve bağımlılıklarını) ve birkaç başka paylaşılan kütüphaneyi içeriyor.

Rust örneğini `gcr.io/distroless/cc` image'iyle düzeltiyoruz:

`~/Dockerfile.rust` (target stage):

```dockerfile
...build stage aynı kalıyor...

# -=== Target image ===-
# 'FROM gcr.io/distroless/base' yerine 'FROM gcr.io/distroless/cc'
FROM gcr.io/distroless/cc

COPY --from=builder /usr/local/cargo/bin/hello-world /
CMD ["/hello-world"]
```

Ara sonuç artık net olmalı:

> **Özet:** `gcr.io/distroless/cc`, `libgcc`'ye ekstra bir çalışma zamanı bağımlılığı olan dinamik bağlanmış uygulamalar için `FROM scratch`'in daha pratik eşdeğeridir.

#### 6. Yorumlanan veya VM Tabanlı Diller İçin Distroless Image'leri

Bazı diller (Python gibi) script'in çalışması için bir interpreter ister. Bazıları (JavaScript veya Java gibi) tam teşekküllü bir runtime ister (Node.js veya JVM gibi). Şimdiye kadar ele aldığımız distroless image'leri paket yöneticisi olmadığı için bunları eklemek sorunlu olabilir (kendi türetilmiş distroless image'inizi üretmek için [Bazel öğrenmeniz](https://github.com/GoogleContainerTools/rules_distroless) gerekir).

Neyse ki distroless projesi en popüler runtime'ları hazır olarak destekliyor:

- [gcr.io/distroless/java](https://github.com/GoogleContainerTools/distroless/blob/dca9008b864a381b5ce97196a4d8399ac3c2fa65/java/README.md) - Java 17 ve 21 (bu yazının yazıldığı tarihte)
- [gcr.io/distroless/nodejs](https://github.com/GoogleContainerTools/distroless/blob/dca9008b864a381b5ce97196a4d8399ac3c2fa65/nodejs/README.md) - Node.js 18, 20 ve 22
- [gcr.io/distroless/python3](https://github.com/GoogleContainerTools/distroless/tree/dca9008b864a381b5ce97196a4d8399ac3c2fa65/python3) - Python 3

Python ve Node.js image'leri `gcr.io/distroless/cc` üzerine, Java image'i ise daha küçük olan `gcr.io/distroless/base-nossl` üzerine kurulu; hepsi runtime ve/veya interpreter'ı içeren bir-iki ekstra katman daha getiriyor.

Hiyerarşinin son hali şöyle:

```
gcr.io/distroless/static
└── base-nossl (+ libc, libpthread, ...)
    ├── base (+ libssl)
    │   └── cc (+ libgcc)
    │       ├── python3 (+ Python interpreter)
    │       └── nodejs (+ Node.js runtime)
    └── java (+ JVM / OpenJDK)
```

#### 7. Distroless Üzerine Nasıl Build Edilir?

Tüm distroless image'leri bilinçli olarak shell ve paket yöneticisi olmadan build edilir. Bu, onları üretim için daha güvenli yapar; ama Dockerfile'ınızı distroless'a dayandırdığınızda `RUN` komutu çalışmaz.

`RUN` genellikle şu amaçlarla kullanılır:

- Ek OS-seviyesi paketler kurmak.
- Uygulama bağımlılıklarını kurmak.
- Uygulamayı build etmek ve/veya paketlemek.

> Teknik olarak shell'i olan (`busybox` üzerinden) distroless varyantları vardır:
> - `gcr.io/distroless/static:debug`
> - `gcr.io/distroless/base:debug`
> - `gcr.io/distroless/cc:debug`
> - `gcr.io/distroless/java:debug`
> - vb.
>
> ...ama bunları üretimde kullanmak istemezsiniz (ve bu debug varyantlarında yine paket yöneticisi yoktur).

Uygulama bağımlılıklarını kurmak veya uygulamayı build etmek gerekiyorsa, bunu daha developer-friendly bir image'e dayanan [ayrı bir build stage](https://labs.iximiuz.com/tutorials/docker-multi-stage-builds)'de yapıp, build edilmiş uygulamayı distroless tabanlı runtime image'ine kopyalayabiliriz. (Kulağa tanıdık geldiyse çünkü bu, multi-stage build dokümanındaki aynı stratejidir.)

Örneğin bir Node.js uygulaması için şöyle yapabiliriz:

```dockerfile
FROM node:22 AS build
COPY . /app
WORKDIR /app
RUN npm ci --omit=dev

FROM gcr.io/distroless/nodejs22:nonroot
COPY --from=build /app /app
WORKDIR /app
CMD ["hello.js"]
```

> Not: Yukarıdaki örnekteki `:nonroot` tag'ine dikkatinizi çekerim. Her distroless image'inin birkaç "çeşidi" vardır:
> - `:latest` - root kullanıcı, shell yok.
> - `:nonroot` - non-root kullanıcı, shell yok.
> - `:debug` - root kullanıcı, shell var.
> - `:debug-nonroot` - non-root kullanıcı, shell var.
>
> ...ve `gcr.io/distroless/<name>` aslında `gcr.io/distroless/<name>-debian<version>` için bir takma addır. Build alırken muhtemelen spesifik olmayı ve Debian sürümü ile tag'i sabitlemeyi (pin) isteriz.

Buna karşılık, distroless base image'ine ekstra bir OS-seviyesi paket eklemek çok daha karmaşıktır. Bu image'ler Bazel ile build edilir ve onlardan yeni bir image türetmek için bazı ekstra [Bazel kuralları](https://github.com/GoogleContainerTools/rules_distroless) yazmamız gerekir. Build stage'den paket dosyalarını kopyalamak teorik olarak da mümkün, ama süreç biraz kırılgandır.

#### 8. Kim Kullanıyor Bu Distroless Image'leri?

Ben kullanıyorum! Özellikle `gcr.io/distroless/static`; favori `FROM scratch` alternatifim. Daha ciddi konuşacak olursak, distroless image'lerinin kayda değer bir takım kullanıcıları var:

- [Kubernetes](https://github.com/kubernetes/enhancements/blob/25fe46543fc69838b0ae8004830149372b5b5c72/keps/sig-release/1729-rebase-images-to-distroless/README.md)
- Knative
- [Kubebuilder](https://github.com/kubernetes-sigs/kubebuilder)
- [Tekton](https://github.com/tektoncd/operator)
- [Teleport](https://github.com/gravitational/teleport)
- ~~[ko](https://github.com/google/ko)~~ ([cgr.dev/chainguard/static'e geçti](https://github.com/ko-build/ko/blob/79df6d5691992cb52e3f159f1661ce00c341e570/docs/configuration.md))
- ~~[Jib](https://github.com/GoogleContainerTools/jib)~~ ([eclipse-temurin'e geçti](https://github.com/GoogleContainerTools/jib/blob/5d9beaee002e21fdeeb868edbc09e3727696937c/docs/default_base_image.md))

...ve GitHub code search'te `FROM gcr.io/distroless` için [40.000+ eşleşme](https://github.com/search?q=%22FROM+gcr.io%2Fdistroless%22&type=code).

#### 9. Artıları, Eksileri ve Alternatifleri

**GoogleContainerTools distroless image'leri küçük, hızlı ve daha güvenlidir.** `FROM scratch` benzeri senaryolar için `gcr.io/distroless/static` kullanmak düşünülmesine bile gerek olmayan bariz bir tercihtir; Node.js, Python ve Java uygulamaları da en azından distroless runtime image'lerini değerlendirmelidir.

Proje, upstream Debian sürümlerini otomatik takip eder; CVE düzeltmelerini Debian'ın yaptığıyla aynı hızda alır ve distro'daki kadar iyi yamar .

Buna karşılık:

- **Distroless base'e yeni OS paketi eklemek zordur.** Base'i değiştirmek Bazel bilmeyi (ve fork maintainer'ı olmayı?) gerektirir; sonradan bir şey eklemek paket yöneticisi olmadığından uğraştırır.
- **Base image seçimi kısıtlıdır.** Uygulamanız desteklenen runtime'lara uymuyorsa, distroless'tan yararlanamazsınız.
- **Operasyonel yük getirir.** Shell'i ve paket yöneticisi olmayan bir image'de üretim workload'ünü debug etmek, kendine has bir meydan okumadır.

Minimal bir base'den image'leri özenle üretme fikrine aşık olduysak, alternatiflere de göz atabiliriz:

- [Chainguard Images](https://github.com/chainguard-images) - Wolfi'yi minimal ve güvenli base olarak kullanır; [apko](https://github.com/chainguard-dev/apko) ve [melange](https://github.com/chainguard-dev/melange) araçlarıyla uygulamanıza özel, sadece (çoğunlukla?) gerekli bağımlılıkları içeren image'ler build etmeyi sağlar. Başlıca eksiği: **kapalı kaynak ve epey pahalı** 🙈
- [Chisel](https://github.com/canonical/chisel) - Yukarıdakiyle benzer bir fikir, ama Canonical'dan; dolayısıyla Ubuntu tabanlı. Proje çok yeni değil ve ciddi kullanıcıları var ([Microsoft, .NET runtime image'leri için kullanıyor](https://devblogs.microsoft.com/dotnet/dotnet-6-is-now-in-ubuntu-2204/)), ama **benimseme oranları hala çok yüksek değil**.
- [Buildah](https://github.com/containers/buildah) - Container image'leri build etmek için güçlü bir araç; özellikle `FROM scratch`'ten başlayıp container'a build araçlarını kurmadan host'un build araçlarından yararlanarak image üretmeye izin verir.
- [Multi-stage builds](https://labs.iximiuz.com/tutorials/docker-multi-stage-builds) - Şaka bir yana! `FROM scratch` veya slim bir runtime base'den başlayıp, build stage'lerden sadece gereken bitleri dikkatle kopyalayabiliriz. (Bu zaten bizim multi-stage dokümanındaki ana strateji.)

Hala minimalistic container image'leri istiyorsunuz ama yukarıdakilar için vaktiniz yok mu? O zaman size bu var:

- [minT(oolkit)](https://labs.iximiuz.com/playgrounds/mintoolkit) (eski adıyla DockerSlim) - Hedef container'ı çalışma zamanında analiz edip gereksiz her şeyi atarak "şişkin" bir container image'ini otomatik olarak "slim" bir hale dönüştüren bir CLI aracı.

#### 10. Sonuç: Ne Zaman Hangisi?

Distroless image'leri **küçük, hızlı ve daha güvenlidir**; ama etkili kullanımları için ekstra bilgi ister. Mevcut distroless base image'leri arasındaki farkı anlamak ve uygulamaya özel gereksinimlere göre optimal varyantı seçmek gerekir. Ayrıca distroless image'lerini ekstra OS paketleriyle genişletmek kolay değildir ve desteklenen dil runtime'larının seçimi GoogleContainerTools projesi maintainers'larının tercihleriyle sınırlıdır.

Peki distroless image'lerini ne zaman kullanmalı? İşte benim pratik kuralım:

- `FROM scratch` ile image build etmek istediğiniz **her** seferinde, `gcr.io/distroless/static` image'ini daha iyi bir alternatif olarak düşünün.
- Binary'niz statik bağlanamıyorsa - yani `FROM scratch` image'ine `libc`, `libssl`, `libgcc` gibi paylaşılan kütüphaneler eklemeye başladıysanız - `gcr.io/distroless/base` (ya da `base-nossl`) veya `gcr.io/distroless/cc` size biçilmiş kaftandır.
- Yüksek regülasyona tabi bir ortamda çalışıyorsanız ve güvenlik/compliance en öncelikli konuysa, `gcr.io/distroless/{java,nodejs,python}` image'leri denemeye değerdir.
- Ekstra OS-seviyesi paketler kurmanız gerekiyor ve Bazel öğrenmek fazla yorucu geliyorsa? Alternatiflere bakın: [multi-stage build](https://labs.iximiuz.com/tutorials/docker-multi-stage-builds) + slim bir runtime base, [Chainguard](https://github.com/chainguard-images), [Chisel](https://github.com/canonical/chisel) ve [minT(oolkit)](https://labs.iximiuz.com/playgrounds/mintoolkit).

Bu dokümanı hazırlarken [iximiuz.com](https://labs.iximiuz.com)'dan bolca yararlandım - kendilerine teşekkür ederim.

Okuduğunuz için teşekkür ederim. İyi build'ler!
