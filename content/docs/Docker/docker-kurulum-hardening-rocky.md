---
title: "docker-kurulum-hardening-rocky"
weight: 13
---
### Rocky Linux İçin Güvenli Docker ve Docker Compose Kurulum Dokümantasyonu

#### 1. Sistem Hazırlığı ve Bağımlılıklar

Kuruluma başlamadan önce sistemin güncel olduğundan ve Docker deposunu yönetmemizi sağlayacak araçların yüklendiğinden emin oluyoruz.

```bash
sudo dnf update -y
sudo dnf install -y dnf-plugins-core
```

#### 2. Resmi Docker Deposunun Eklenmesi

Docker, Rocky için ayrı bir depo yayınlamıyor; RHEL türevleri için tek bir depo tutuyor ve bu depo **CentOS** olarak adlandırılıyor. Rocky'de de bu depoyu kullanıyoruz:

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

> Rocky 10'da paket yöneticisi dnf5'e geçti; orada komut biraz farklı: `sudo dnf config-manager addrepo --from-repofile=https://download.docker.com/linux/centos/docker-ce.repo`. Rocky 9'da (dnf4) yukarıdaki komut olduğu gibi çalışır.

Depo doğru eklendiyse şöyle kontrol edebiliriz:

```bash
dnf repolist | grep docker
```

#### 3. Kurulum İşlemi

Docker Engine, Containerd ile Compose eklentisini kuruyoruz. Docker Compose, ayrı bir binary (`docker-compose`) yerine artık bir CLI eklentisi (`docker compose`) olarak dağıtılmaktadır.

```bash
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Servisi etkinleştirip çalıştığını doğruluyoruz:

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

**C. Güvenlik Duvarı (firewalld) Davranışı**

Rocky'de güvenlik duvarı olarak `firewalld` gelir ve UFW benzeri bir davranış vardır. Docker, port yönlendirmeleri için doğrudan iptables kurallarına müdahale eder ve firewalld'ın kurallarını bypass eder.

Yani `docker run -p 8080:80 ...` komutunu çalıştırırsanız, firewalld'de port 8080 kapalı olsa bile Docker bu portu tüm dünyaya (0.0.0.0) açar. Bunu engellemek için portları yayınlarken her zaman loopback'e bağlıyoruz. Önüne bir reverse proxy / web sunucu servisi koyarak bu riskin önüne geçmiş oluyoruz.

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

> Not: `DOCKER-USER` kuralları Docker tarafından yeniden başlatmada silinmez, ama sistem yeniden başlatıldığında kaybolurlar. Kalıcı hale getirmek için `iptables-services` paketiyle `iptables-save > /etc/sysconfig/iptables` yöntemini ya da kuralı systemd unit'ine ekleyen küçük bir script kullanabiliriz.
>
> Bu tek kural yeterlidir, çünkü container'dan giden trafiğin dönüş paketleri hedef port olarak `8080` içermez (`DOCKER-USER` zinciri FORWARD trafiği için değerlendirilir).

**D. SELinux Notları (Rocky'ye Özel)**

Rocky, SELinux'u `enforcing` modda kurar ve container'larla en çok takılınan yer de tam burasıdır. Docker'ın image'ları `container-selinux` ile hazır politikalarla gelir; volume kullanırken sorun çıkmaz. Ancak **host'tan bind mount** yaptığımızda klasik hata şudur:

```
permission denied while trying to bind mount ...
```

Çünkü host'taki dizin `usr_t` veya benzeri bir context'e sahiptir, container içinden okunamaz. Test amaçlı `setenforce 0` ile geçici olarak kapatmak tanıyı kolaylaştırır ama asla kalıcı çözüm değildir. Kalıcı çözüm, dizine container'ın okuyup yazabileceği bir context atamaktır:

```bash
sudo chcon -Rt container_file_t /data/myapp
```

Bu yöntem dosya system'i yeniden etiketlenince (relabel) kaybolur. Kalıcı istiyorsak policy tabanlı etiketleme yaparız:

```bash
sudo semanage fcontext -a -t container_file_t '/data/myapp(/.*)?'
sudo restorecon -Rv /data/myapp
```

>`semanage` komutu `policycoreutils-python-utils` paketinde gelir; kurulu değilse `sudo dnf install -y policycoreutils-python-utils` ile ekleriz.

Bind mount'ı compose veya `docker run` sırasında da etiketleyebiliriz; `z` (paylaşımlı) ve `Z` (özel) flag'leri tam bu işe yarar:

```bash
docker run -v /data/myapp:/data:z nginx
```

> `Z` flag'i mount edilen dizini yalnızca o container'a özel etiketler; `z` ise birden fazla container aynı volume'ü paylaşıyorsa kullanılır.

SELinux'un engellediği bir işlemi bulmak için denetim loglarına bakarız:

```bash
sudo ausearch -m avc -ts recent
```

**E. Podman Alternatifi (Neden Rootless Podman Daha İyi?)**

Rocky, Docker'a alternatif olarak kendi container ekosisteminde **Podman**'ı öneriyor. Kurulumu kolaydır ve aynı komut satırı arayüzünü kullanır:

```bash
sudo dnf install -y podman podman-compose
```

`docker` komutunu birebir kullanan bir alışkanlığımız varsa uyumluluk katmanını da ekleyebiliriz:

```bash
sudo dnf install -y podman-docker
```

Neden rootless Podman daha iyi bir seçenek?

- **Daemon yok.** Docker'da container'ları yöneten her zaman çalışan, root yetkisiyle koşan bir daemon (`dockerd`) vardır. Bu daemon exploite edilirse saldırgan doğrudan host root'una geçer. Podman daemonless çalışır; container'lar, kullanıcı process'inin fork/exec çocuğu olarak başlar. Sürekli koşan bir privileged süreç olmadığından saldırı yüzeyi küçülür.
- **Rootless tasarımı default.** Docker'da rootless mode sonradan eklenen bir özellik ve kurulumda ek adım gerektiriyor. Podman ise baştan rootless'a göre tasarlandı; tek `sudo dnf install podman` komutu ile root olmayan kullanıcı bile container çalıştırabilir.
- **systemd entegrasyonu.** Podman'ın systemd ile native entegrasyonu (Quadlet) sayesinde container'ları kullanıcı seviyesinde systemd unit'leri gibi yönetebiliriz; `systemctl --user` komutlarıyla reboot sonrası otomatik başlatma root yetkisi olmadan yapılır.
- **SELinux entegrasyonu.** Rocky ve RHEL ekosisteminde Podman'ın SELinux politikaları daha kusursuz çalışır; RHEL'in kendi container araçları olduğundan SELinux ile uyum testleri Docker'a göre daha kapsamlıdır.
- **Docker CLI uyumluluğu.** `podman-docker` paketi sayesinde mevcut scriptler ve `docker` komutları değişiklik olmadan çalışır; Docker Compose yerine `podman-compose` kullanılır.

> Not: Üretimde kritik izolasyon gerekiyorsa, Docker rootless modu (önceki bölümlerde anlatılan hardening'le birlikte) da geçerli bir mimaridir. Ancak Rocky'de **rootless Podman** varsayılan davranış ve daha küçük saldırı yüzeyi nedeniyle genelde daha temiz bir tercihtir.
