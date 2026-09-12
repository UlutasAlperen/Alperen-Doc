---
title: "github_workflow"
weight: 3
---
001 - [continuous_integration](continuous_integration/)
002 - [continuous_deployment](continuous_deployment/)

# Hugo-Book Teması ve GitHub Actions(workflow) ile VPS'e CD

Hugo,Markdown dosyalarını statik bir siteye dönüştüren bir araç. Bu yazıda, suan bulundugunuz siteyi nasil _hugo-book_ temasını kullanırken aynı zamanda GitHub Workflow kullanarak her değişikliği debian tabanli uzak sunucuma nasıl otomatik olarak deploy edeceğimi anlatmak istedim.

## 1. VPS ve Kullanıcı Yapılandırması

İlk başta sistemi `--system` parametresiyle kurmayı düşünmüştüm. Sistem kullanıcısı oluşturmak mantıklı çünkü bu kullanıcının varsayılan olarak bir şifresi ve interaktif bir shell oturumu olmaz (tabi belirli komutlarla yapilabilir). Bu da sunucunun guvenligini arttirir.

Ancak dikkat edilmesi gereken önemli bir detay var:

- **Shell Durumu:** `rsync` işleminin SSH üzerinden çalışabilmesi için kullanıcının geçici de olsa geçerli bir shell'e (`/bin/bash` veya `/bin/sh`) ihtiyacı var. Sunucuya dosya taşımak için rsync kullanacağımdan, shell'i `/usr/sbin/nologin` veya `/bin/false` yaparsam SSH bağlantısı kurulamazdim.

- **Çözüm:** Güvenliği shell'i tamamen kapatarak değil, SSH anahtarı kullanarak (ve gerekirse sadece rsync komutuna izin vererek) kullanacagim.

### Kullanıcı ve Dizin Hazırlığı (Debian VPS):


```bash
# 1. Sistem kullanıcısını oluşturuyorum (home dizini /var/www/kole olacak şekilde)
sudo useradd -r -m -d /var/www/kole -s /bin/bash kole

# 2. Web dizinini oluşturup Caddy'nin de okuyabilmesi için izinleri ayarlıyorum
sudo mkdir -p /var/www/kole/public
sudo chown -R kole:caddy /var/www/kole

# İzinleri dizinler için 750 (rwxr-x---), dosyalar için ise 640 (rw-r-----) şeklinde yapılandıracağım.
```

## 2. SSH ve Güvenlik (Şifresiz SSH & Secrets)

Sürecin tamamen şifresiz ve güvenli çalışması için bir SSH Key çifti üretiyorum; çünkü otomasyonu bu anahtarla sağlayacağım.

### Adım A: SSH Key Üretimi


```bash
ssh-keygen -t ed25519 -f id_github_deploy -C "github-actions-deploy"
```

Bu işlem bize iki adet dosya verir:

1. `id_github_deploy` (Private Key): GitHub'da Secrets kısmına ekleyeceğim. Eğer birileri bu key'e erişirse işletim sistemime sızabilir. GitHub patlamadığı sürece sorun yok, patlarsa who cares.

2. `id_github_deploy.pub` (Public Key): VPS tarafına ekleyeceğim anahtar.

### Adım B: Public Key'i VPS'e Ekleme

VPS üzerinde yeni oluşturduğum `kole` kullanıcısının SSH yetkilendirmesini yapıyorum:

```bash
# kole kullanıcısının SSH dizinini oluşturuyorum
sudo mkdir -p /var/www/kole/.ssh
sudo touch /var/www/kole/.ssh/authorized_keys

# Ürettiğim PUBLIC key içeriğini bu dosyaya yazdırıyorum
echo "pub-key" > /var/www/kole/.ssh/authorized_keys

# Klasör ve dosya izinlerini sıkılaştırıyorum
sudo chmod 700 /var/www/kole/.ssh
sudo chmod 600 /var/www/kole/.ssh/authorized_keys
sudo chown -R kole:kole /var/www/kole/.ssh
#zaten ssh baglantisi kole uzerinden ve  caddy agent'in ssh dizinine erisimine gerek yok
```

## 3. Caddy Yapılandırması

Caddy ile statik dosyalarımı web'e sunacağım. `/etc/caddy/Caddyfile` (temsili olarak paylayisorum prodda biraz daha farkli conf) dosyasını şu şekilde yapılandırıyorum:


```toml
dev.ulutasalperen.com {
    # Statik dosyaların konumunu belirtiyorum
    root * /var/www/kole/public 
    
    # Dosyaları statik olarak serve etmesini söylüyorum
    file_server 
    
    # Statik dosyaları sıkıştırıp göndermek performansı ciddi artırır
    encode gzip zstd 
}
```

> **Not:** Caddy'nin `/var/www/kole/public` altındaki dosyaları okuyup web'e sunabilmesi için dizin geçiş izinlerinin en az `5` (r-x) ve dosyaların okuma izinlerinin en az `4` (r--) olması gerekir. `rsync` deploy aşamasında bu izinleri otomatik olarak ayarlayacak seklinde yml yazacagim.

## 4. GitHub Repository Secrets Tanımlama

GitHub deponuzun **Settings > Secrets and variables > Actions** kısmına giderek aşağıdaki **Repository secrets** değerlerini ekleyin:

- `SSH_PRIVATE_KEY`: Ürettiğiniz `id_github_deploy` dosyasının _tüm_ içeriği.
- `SSH_HOST`: VPS sunucunuzun IP adresi.
- `SSH_USER`: `kole`
- `SSH_PORT`: Güvenlik amacıyla değiştirdiyseniz yeni SSH port numaranız (varsayılan 22).

## 5. GitHub Actions Workflow Dosyası (`.github/workflows/deploy.yml`)

Deponuzun kök dizininde `.github/workflows/deploy.yml` dosyasını oluşturun ve aşağıdaki içeriği ekleyin. Bu workflow sırasıyla şunları yapar:

1. Hugo kurulumunu gerçekleştirir.
2. Siteyi build eder.
3. SSH Private Key'i geçici olarak runner'a yükler.
4. `rsync` ile sadece değişen dosyaları güvenli bir şekilde VPS'e aktarır ve izinleri sunucu üzerinde yeniden yazar.

```yaml
name: Deploy Hugo Site

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Source Code
        uses: actions/checkout@v4
        with:
          submodules: recursive # Hugo temaları (hugo-book) submodule icin gerekli
          fetch-depth: 0

      # 2. Hugo Kurulumu
      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: "latest"
          extended: true # hugo-book veya temalar sass/scss kullanıyorum o yuzden true

      # 3. Hugo Build
      - name: Build Hugo Site
        run: hugo --minify

      # 4. SSH Key Yapılandırması
      - name: Install SSH Key
        uses: shimataro/ssh-key-action@v2
        with:
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          known_hosts: "unnecessary"
          if_key_exists: replace

      # 5. VPS IP'sini bilinen hostlar listesine ekliyorum hem sormasin hemde guvenlik icin
      - name: Adding Known Hosts
        run: |
          mkdir -p ~/.ssh
          ssh-keyscan -p ${{ secrets.SSH_PORT }} -H ${{ secrets.SSH_HOST }} >> ~/.ssh/known_hosts

      # 6. Rsync ile VPS'e gönderim
      - name: Deploy Files with Rsync
        run: |
          rsync -avz --delete \
            --no-perms --no-owner --no-group \
            --chmod=D2750,F640 \
            -e "ssh -p ${{ secrets.SSH_PORT }}" \
            public/ ${{ secrets.SSH_USER }}@${{ secrets.SSH_HOST }}:/var/www/kole/public/
```
### Rsync Parametrelerinin Detayları:

- `-a` (archive): Dosya izinlerini, sahipliklerini ve tarihlerini korur.
    
- `-v` (verbose): İşlem detaylarını loglara yazar.
    
- `-z` (compress): Transfer sırasında veriyi sıkıştırarak hızı artırır.
    
- `--delete`: Eğer lokalde (git üzerinde) bir dosyayı sildiyseniz, VPS tarafındaki `public` klasöründen de silinmesini sağlar. Statik sitelerde eski ve çöp dosyaların birikmesini önlemek için kritiktir.
    
- `public/` (Sondaki eğik çizgi önemlidir): Sadece `public` klasörünün **içeriğini** karşıya yükler. Eğer eğik çizgi koymazsanız, karşı tarafta `/var/www/kole/public/public/` şeklinde iç içe gereksiz bir klasör yapısı oluşur.
    
- `--no-perms`, `--no-owner`, `--no-group`: GitHub runner üzerindeki lokal kullanıcı izinlerinin ve sahiplik bilgilerinin hedef sunucuya olduğu gibi taşınmasını engeller.
    
- `--chmod=D2750,F640`: Klasörleri (`D`) 2750 (`rwxr-s---`), dosyaları (`F`) ise 640 (`rw-r-----`) yetkisiyle hedefe yazar. Baştaki `2` = `SetGID` bitidir. Böylece her yeni klasör `caddy` grubunu otomatik miras alır, Caddy sorunsuz okurken `other=0` olduğu için dışarıdan erişim engellenir. `D750` kullanılsaydı yeni klasörler `s` olmadan oluşup grup mirası kırılırdı.

> **Güncelleme:** Yukarıdaki varsayımın canlıda her zaman çalışmadığı bir vaka yaşadım; detaylı analiz için dokümanın sonundaki **"Güncelleme: Birincil/İkincil Grup ve Bir Deploy İzni Vakası"** bölümüne bakın.
    

### SetGID Bitinin Sunucu Tarafında Tanımlanması

SetGID'nin hedef klasörde (`/var/www/kole/public`) aktif olduğunu garanti altına almak için VPS üzerinde şu komutları bir kez çalıştırmamız gerekiyor:


```bash
# Klasörün grubunu caddy olarak ayarlıyoruz
sudo chown -R kole:caddy /var/www/kole/public

# Mevcut ve yeni açılacak tüm alt klasörlere SetGID yetkisi veriyoruz (g+s)
sudo find /var/www/kole/public -type d -exec chmod g+s {} \;
```

> **SetGID Nasıl Çalışır?** `g+s` izni alan bir klasörün altında oluşturulan her yeni dosya veya alt klasör, onu oluşturan kullanıcının (bu senaryoda `kole`) birincil grubuna bakmaksızın, otomatik olarak üst klasörün grubunu (`caddy`) miras alır. Bu sayede her deploy sonrasında Caddy'nin dosyaları okuyamama sorunu  ortadan kalkar. Artık rsync her deploy'da `D2750` ile geldiği için bu işlem kalıcı hale gelir.

## Güncelleme: Birincil/İkincil Grup ve Bir Deploy İzni hatasi (403/404)

Bu dokümanı yazdıktan bir süre sonra, iki yeni sayfayı deploy ettiğimde canlıda ilginç bir sorunla karşılaştım: sayfalar build ediliyor, GitHub Actions deploy'u başarılı görünüyordu, ama sunucuda iki sayfa 404 döndürüyordu. İşte bu vaka, yukarıdaki anlatımdaki varsayımların gerçekte her zaman geçerli olmadığını öğretti.

### Vakanın Özeti

Deploy sonrası durum:

| URL | Sonuç |
|---|---|
| Eski bir sayfa (`/docs/docker/docker-logs/`) | `200` |
| Yeni eklenen sayfa (`/docs/docker/docker-distroless-container-images/`) | `404` |
| Aynı sayfanın dosyası (`.../index.html`) | `403` |
| Hiç var olmayan bir sayfa | `404` |

Ayrıca Actions'un rsync adımının logu, ilgili `index.html` dosyalarının sunucuya aktarıldığını açıkça gösteriyordu. Yani Hugo build'i ve deploy'un kendisi sağlamdı; sorun **sunucu tarafındaki dosya izinlerindeydi**.

Sunucuya girip baktığımda tablo şuydu:

```bash
$ ls -la /var/www/kole/public/docs/docker/
drwxr-s--- 2 kole caddy docker-logs/                              # eski: setgid VAR
drwxr-x--- 2 kole caddy docker-distroless-container-images/      # yeni: setgid YOK (750)
drwxr-x--- 2 kole caddy docker-kurulum-hardening-rocky/          # yeni: setgid YOK (750)

$ ls -la /var/www/kole/public/docs/docker/docker-distroless-container-images/
-rw-r----- 1 kole kole index.html   # grup caddy DEĞİL!
```

### Kök Neden

403/404 ayrımı klasik bir izin semptomudur: dosya **sunucuda duruyor** ama Caddy süreci (grup `caddy` üzerinden erişiyor) onu **okuyamıyor** → dosyaya direkt istek `403`; dizin URL'sinde ise Caddy, okunamayan `index.html`'i "yok" sayıp `404` döndürüyor.

Peki zincir nasıl işledi?

1. rsync komutunda `--no-perms` vardı. rsync'in man sayfasına göre `--chmod` bu durumda **mevcut dosyalara hiç uygulanmaz**; yeni öğelerde de sunucu umask'i kazanabiliyor.
2. rsync yeni dizini `mkdir` ile yarattığında kernel setgid bitini aslında kopyalar; ama `--no-perms` yüzünden rsync sonradan klasik bir `chmod` yapar (umask ile maskeleme → `750`) ve **chmod çağrısı setgid bitini siler**.
3. Setgid'siz bir dizinde yaratılan dosyalar, oluşturan sürecin **birincil grubunu** alır → `index.html` `kole:kole 640` oldu.
4. Caddy, `caddy` grubuyla okumaya çalıştığı için `EACCES` → `403/404`.

Eski dosyaların çalışması ise kurulum sırasında manuel yaptığım `chgrp`/`chmod g+s` komutlarının kalıntısıydı; her deploy'da **yeni** dosyalara bu düzeltmeler otomatik uygulanmıyordu. Yani "rsync `D2750` ile geldiği için setgid kalıcıdır" varsayımım yanlıştı.

### Birincil ve İkincil Grup Kavramı

Bu vakanın merkezinde, Linux'un grup kavramının bir detayı var:

- **Birincil grup (primary group):** `/etc/passwd`'de kullanıcının satırında görünen **tek** GID'dir. Bir süreç yeni bir dosya yarattığında dosyanın grubu **buradan** gelir - tek istisna: dosyanın yaratıldığı dizin **setgid** (`g+s`) bitine sahipse dosya üst dizinin grubunu miras alır.
- **İkincil gruplar (secondary groups):** `groups` veya `id` komutuyla listelenirler. Dosya erişim denetiminde (izin hesabında) geçerlidirler; ama **yeni yaratılan dosyaların grubunu etkilemezler.**

Bu yüzden şu iki komut birbiriyle karıştırılmamalı:

```bash
sudo usermod -aG caddy kole   # ikincil gruba ekler → YENİ dosyaların grubu DEĞİŞMEZ, sorunu çözmez
sudo usermod -g caddy kole    # birincil grubu değiştirir → YENİ dosyalar otomatik grup caddy olur
```

### Kalıcı Çözümler

**1. ACL:**

```bash
sudo setfacl -R -m g:caddy:rX,d:g:caddy:rX /var/www/kole/public
```

`d:` (default ACL) bu ağaç altında bundan sonra yaratılacak **her** yeni dosya ve dizine, kim yaratırsa yaratsın (rsync dahil), grup `caddy`'ye okuma iznini kernel seviyesinde verir; `--no-perms` ve umask oyunları bu mirası ezip geçemez. `rsync --delete` ile silinip yeniden açılan dizinler bile üst dizinden default ACL'yi devralır. En büyük avantajı: etkisi yalnızca webroot ile sınırlıdır, kullanıcı/grup yapılandırmasına dokunmaz. Doğrulama için `getfacl /var/www/kole/public/docs/docker` yeterli.

**2. Birincil grup değişikliği:**

```bash
sudo usermod -g caddy kole
```

Çalışır ve tek komuttur; ama `kole` bundan sonra yaratacağı **tüm** dosyalara grup `caddy` verecek. `/home/kole` 750 `kole:kole` kaldığı için büyük bir risk yok; yine de `/tmp` gibi herkesin travers edebildiği dizinlerde yarattığı dosyalar Caddy süreci tarafından okunabilir hale gelir - düşük ama sıfır olmayan bir risk.

**3. Workflow tarafı:**

```yaml
rsync -avz --delete \
  --chmod=D2755,F644 \
  ...
```

`--no-perms` kaldırılır; böylece `--chmod` her transferde deterministik uygulanır. Sunucuya hiç dokunmazsınız ama dosyalar world-readable (644) olur - herkese açık bir statik site için zararsızdır.

### Anlık Onarım

Mevcut dosyaları hemen düzeltmek için:

```bash
sudo chgrp -R caddy <etkilenen-dizinler>
sudo chmod -R g+rX,o-rwx <etkilenen-dizinler>
sudo chmod g+s <etkilenen-dizinler>   # bir daha aynı sorun yaşanmasın diye setgid
```

Ve unutmayın: bu komutların `usermod -aG caddy kole` ile ikame edilebileceğini düşünmeyin - ikincil grup eklemek yeni dosyaların grubunu değiştirmez. Bu döngüden kalıcı çıkmak için yukarıdaki üç çözümden birini uygulamak gerekir; ben 2. yolu yaptim.

Okudugunuz icin tesekkur ederim.
