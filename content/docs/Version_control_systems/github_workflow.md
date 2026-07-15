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
sudo chown -R kole:caddy /var/www/kole/.ssh
```

## 3. Caddy Yapılandırması

Caddy ile statik dosyalarımı web'e sunacağım. `/etc/caddy/Caddyfile` dosyasını şu şekilde yapılandırıyorum:


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
          submodules: recursive  # hugo-book teması submodule olarak ekliyse indirmek için şart
          fetch-depth: 0

      # 2. Hugo Kurulumu
      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: 'latest'
          extended: true # hugo-book veya kullandığınız temalar sass/scss derliyorsa true olmalı

      # 3. Hugo Build 
      - name: Build Hugo Site
        run: hugo --minify

      # 4. SSH Key Yapılandırması
      - name: Install SSH Key
        uses: shimataro/ssh-key-action@v2
        with:
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          known_hosts: 'unnecessary' 
          if_key_exists: replace

      # 5. VPS IP'sini bilinen hostlar listesine ekliyoruz (bağlantıda onay sormaması ve güvenlik için)
      - name: Adding Known Hosts
        run: |
          mkdir -p ~/.ssh
          ssh-keyscan -p ${{ secrets.SSH_PORT }} -H ${{ secrets.SSH_HOST }} >> ~/.ssh/known_hosts

      # 6. Rsync ile VPS'e gönderim
      - name: Deploy Files with Rsync
        run: |
          rsync -avz --delete \
            --no-perms --no-owner --no-group \
            --chmod=D750,F640 \
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
    
- `--chmod=D750,F640`: Klasörleri (`D`) 750 yetkisiyle, dosyaları (`F`) ise 640 yetkisiyle hedefe yazar. Böylece Caddy dosyaları sorunsuz okurken, dışarıdan yetkisiz yazma işlemlerinin önüne geçilir.
    

### SetGID Bitinin Sunucu Tarafında Tanımlanması

SetGID'nin hedef klasörde (`/var/www/kole/public`) aktif olduğunu garanti altına almak için VPS üzerinde şu komutları bir kez çalıştırmamız gerekiyor:


```bash
# Klasörün grubunu caddy olarak ayarlıyoruz
sudo chown -R kole:caddy /var/www/kole/public

# Mevcut ve yeni açılacak tüm alt klasörlere SetGID yetkisi veriyoruz (g+s)
sudo find /var/www/kole/public -type d -exec chmod g+s {} \;
```

> **SetGID Nasıl Çalışır?** `g+s` izni alan bir klasörün altında oluşturulan her yeni dosya veya alt klasör, onu oluşturan kullanıcının (bu senaryoda `kole`) birincil grubuna bakmaksızın, otomatik olarak üst klasörün grubunu (`caddy`) miras alır. Bu sayede her deploy sonrasında Caddy'nin dosyaları okuyamama sorunu  ortadan kalkar.

Okudugunuz icin tesekkur ederim.
