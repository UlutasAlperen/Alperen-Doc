---
title: "docker-kurulum-hardening-debian"
weight: 1
---
### Debian 13 İçin Güvenli Docker ve Docker Compose Kurulum Dokümantasyonu

#### 1. Sistem Hazırlığı ve Bağımlılıklar

Kuruluma başlamadan önce sistemin güncel olduğundan ve paketleri HTTPS üzerinden alabilmek için gerekli araçların yüklendiğinden emin oluyoruz.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg
```
#### 2. Resmi Docker Deposu ve GPG Anahtarının Eklenmesi

Paketlerin bütünlüğünü doğrulamak için Docker'ın resmi GPG anahtarını eklemek güvenlik açısından zorunludur.

GPG anahtarını indirip yapılandırıyoruz:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo rm -f /etc/apt/keyrings/docker.gpg
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

> `rm -f` satırını ekledik çünkü anahtar daha önce indirilmişse `gpg --dearmor -o` komutu "File exists" hatası verir. Bu sayede komut güvenle tekrar çalıştırılabilir.

APT deposunu sisteme tanımlıyoruz. Bu komut, işletim sistemimizin kod adını (Debian 13 için `trixie`) otomatik olarak algılayıp ilgili depoyu ekler:

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

#### 3. Kurulum İşlemi

Depoları güncelleyip Docker Engine, Containerd ile Compose eklentisini kuruyoruz. Docker Compose, ayrı bir binary (`docker-compose`) yerine artık bir CLI eklentisi (`docker compose`) olarak dağıtılmaktadır.

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Servisin çalıştığını doğruluyoruz:

```bash
sudo systemctl enable --now docker
sudo systemctl status docker --no-pager
```

> Not: `docker-ce` paketi servisi genellikle kurulum sırasında otomatik olarak etkinleştirir; `enable --now` komutu bunun garantilenmesi içindir (etkinleştirilmişse sorun çıkmaz).
>
> Hata alırsak servisi son log satırlarıyla birlikte şöyle kontrol ederiz (Sonuçta bu bir servis):

```bash
sudo journalctl -u docker -e --no-pager
```

#### 4. Güvenlik Sıkılaştırma (Hardening) Prosedürleri

Docker'ı kurduğumuzda varsayılan olarak root yetkileriyle çalışır ve ciddi güvenlik riskleri barındırır.

**A. `docker` Grubuna Kullanıcı Ekleme Tehlikesi**

Bir kullanıcıyı `docker` grubuna eklemek, o kullanıcıya parolasız `sudo` yetkisi vermekle aynıdır. Kullanıcı, container içinden host dosya sistemini mount ederek root erişimi elde edebilir. (Aynı zamanda standart kullanıcıları bu gruba eklemiyoruz.)

**B. Daemon Yapılandırması (`daemon.json`)**

Varsayılan yapılandırma güvenlik açıklarına müsaittir. Container arası iletişimi kısıtlamak, logları sınırlamak ve ayrıcalık yükseltmeyi engellemek için `/etc/docker/daemon.json` dosyasını oluşturuyoruz veya düzenliyoruz.


```bash
sudo nano /etc/docker/daemon.json
```

Aşağıdaki yapılandırmayı dosyaya ekliyoruz:

```json
{
  "icc": false,
  "userns-remap": "default",
  "live-restore": true,
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "3"
  },
  "no-new-privileges": true
}
```

- `icc: false`: Container'ların varsayılan ağ üzerinde birbirleriyle iletişim kurmasını engeller. İletişim, özel olarak tanımlanmış ağlar üzerinden yapılmalıdır. Not: Bu ayar sadece varsayılan köprü (`bridge`) ağında geçerlidir; özel ağlarda container'lar zaten yalnızca aynı ağa bağlı oldukları için izole edilmiştir, bu yüzden `icc` aslında legacy bir bayraktır.

- `userns-remap: default`: Container içindeki root kullanıcısını, host üzerindeki yetkisiz bir kullanıcıya (dockremap) eşler. Container kırılsa bile saldırgan host üzerinde root yetkisine sahip olmaz.

- `live-restore: true`: Docker daemon'ı yeniden başlatıldığında veya güncellenirken container'ların çalışmaya devam etmesini sağlar. Üretim ortamında daemon bakımı sırasında servis kesintisi yaşanmaması için önemli bir ayardır.

- `no-new-privileges: true`: Container içindeki işlemlerin ek yetki kazanmasını (`setuid`/`setgid`) engeller.

Bu dosya oluşturulduktan sonra servisi yeniden başlatıyoruz; `userns-remap: default` ayarı ilk başlatmada `dockremap` kullanıcısını otomatik olarak oluşturur.

```bash
sudo systemctl restart docker
```

> **Dikkat:** `userns-remap` etkinleştirildiğinde container image ve volume'ların sahiplik UID'leri yeniden eşlenir (165536+ aralığına) ve **mevcut image/volume'lar erişilemez hale gelir**. Yeni kurulumda bu bir sorun değildir; ancak üzerinde veri bulunan bir sistemde bu değişiklikten sonra image'ların yeniden pull edilmesi, volume'ların yeniden oluşturulması ve host dizinlerinin bind mount'larda sahiplik uyumunun gözden geçirilmesi gerekir.

**C. Güvenlik Duvarı (iptables) Davranışı**

Docker, port yönlendirmeleri için doğrudan iptables kurallarına müdahale eder ve UFW gibi araçların kurallarını bypass eder.

UFW kullanıyorsanız ve `docker run -p 8080:80 ...` komutunu çalıştırırsanız, UFW'de port 8080 kapalı olsa bile Docker bu portu tüm dünyaya (0.0.0.0) açar. Bunu engellemek için portları yayınlarken her zaman loopback'e bağlıyoruz. Önüne bir reverse proxy / web sunucu servisi koyarak bu riskin önüne geçmiş oluyoruz.

Güvenli port yayınlama yöntemi:

```bash
docker run -p 127.0.0.1:8080:80 nginx
```

Aynı mantık `docker-compose.yml` dosyaları için de geçerlidir:


```yaml
services:
  web:
    image: nginx
    ports:
      - "127.0.0.1:8080:80"
```

**`DOCKER-USER` zinciri ile firewall kuralı koymak**

Portların bir kısmının container'a dışarıdan erişilebilir olması gerekiyorsa (loopback'e bağlamak yerine), firewall kuralı koymak için doğru yer Docker'ın kullandığı iptables zinciridir: `DOCKER-USER`. Docker, kendi yönettiği zincirlerden (örn. `DOCKER`) önce bu zincire bakar, bu yüzden buraya eklenen kurallar Docker tarafından silinmez ve bypass edilemez.

Örnek: `8080` portuna sadece belirli bir IP üzerinden erişim tanımlamak:

```bash
sudo iptables -I DOCKER-USER -p tcp --dport 8080 ! -s 203.0.113.10 -j DROP
```

> Not: `iptables` kuralları kalıcı değildir; sistem yeniden başlatıldığında kaybolurlar. Kalıcı hale getirmek için `iptables-persistent` paketi kullanılabilir (`sudo apt install iptables-persistent`).
>
> Bu tek kural yeterlidir, çünkü container'dan giden trafiğin dönüş paketleri hedef port olarak `8080` içermez (`DOCKER-USER` zinciri FORWARD trafiği için değerlendirilir).

**D. Rootless Mode (Alternatif Güvenlik Mimarisi)**

Sistem genelinde root bağımlılığını tamamen ortadan kaldırmak için, Docker daemon'ın root olmayan bir kullanıcı alanında çalıştırıldığı "Rootless Mode" tercih edilebilir. Bu kurulum mevcut daemon'ı durdurmayı ve belirli bağımlılıkları (`uidmap`, `dbus-user-session`) kurmayı gerektirir. Eğer üretim ortamında kritik izolasyon gerekiyorsa, resmi Docker Rootless kurulum dokümantasyonu referans alınarak standart kurulum yerine bu mimari uygulanmalıdır.
