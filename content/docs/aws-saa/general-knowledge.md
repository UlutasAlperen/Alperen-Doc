---
title: "general-knowledge"
weight: 100
---
# AWS SAA-C03 - General Knowledge: Tek Satırlık Özet ve Sınav Rehberi

> **Amaç:** Sınav öncesi hızlı tekrar. Her servisin ne yaptığını, hangi senaryoda seçileceğini ve benzer servislerden nasıl ayrıldığını hatırlatır. Ayrıntılı açıklamalar için ilgili konu dokümanlarına bak.

---

## 1. Compute - İşlem ve Uygulama Çalıştırma

- **EC2 (Elastic Compute Cloud):** AWS üzerinde sanal sunucu; işletim sistemi, instance tipi, disk ve ağ yapılandırması üzerinde kontrol sağlar.\
  **Kullanım alanları:** Web uygulamaları ve API sunucuları; özel işletim sistemi veya yapılandırma gerektiren iş yükleri; veritabanı ve orta katman sunucuları.\
  **Avantajlar:** Instance tipi, işletim sistemi, disk ve ağ üzerinde tam kontrol; yüzlerce instance tipi ve satın alma seçeneği.\
  **Sınavda:** Özel işletim sistemi yapılandırması, uzun süre çalışan uygulama veya tam sunucu kontrolü isteniyorsa EC2.

- **Lambda:** Sunucu yönetmeden olay tetiklemeli kod çalıştırır; kullanım ve çalışma süresine göre ücretlendirilir.\
  **Kullanım alanları:** Olay tetiklemeli işlemler (dosya yükleme, kuyruk mesajı); kısa süreli API backend'leri; zamanlanmış bakım işleri.\
  **Avantajlar:** Sunucu yönetimi yok; istek başına ödeme; trafiğe göre otomatik ölçekleme.\
  **Sınavda:** S3 (Simple Storage Service)'e dosya yüklenince işlem başlatma, SQS (Simple Queue Service) mesajı işleme veya kısa süreli API backend'i oluşturma.

- **ECS (Elastic Container Service):** AWS'nin container orkestrasyon servisidir; Docker container'larını görev ve servis olarak çalıştırır.\
  **Kullanım alanları:** Docker tabanlı mikroservisler; konteynerli batch işleri; mevcut konteyner iş yüklerini AWS'te çalıştırma.\
  **Avantajlar:** IAM, ALB ve CloudWatch ile yerel entegrasyon; Kubernetes karmaşıklığı olmadan konteyner yönetimi.\
  **Sınavda:** Kubernetes karmaşıklığı olmadan AWS üzerinde container yönetmek.

- **EKS (Elastic Kubernetes Service):** AWS tarafından yönetilen Kubernetes control plane sunar; worker node'lar EC2 veya desteklenen iş yüklerinde Fargate üzerinde çalışabilir.\
  **Kullanım alanları:** Kubernetes tabalı uygulamalar; çoklu bulut Kubernetes stratejileri; mevcut Kubernetes araçlarıyla iş yükleri.\
  **Avantajlar:** Yönetilen Kubernetes control plane; AWS hizmetleriyle yerel entegrasyon; Fargate ile sunucusuz worker seçeneği.\
  **Sınavda:** Kubernetes API'si, mevcut Kubernetes araçları ve ekosistemi gerekiyorsa EKS.

- **Fargate:** ECS veya EKS container'ları için sunucu/worker node yönetme ihtiyacını azaltan serverless compute altyapısıdır.\
  **Kullanım alanları:** Sunucusuz konteyner iş yükleri; düzensiz veya az sayıda çalışan servisler; node yönetmeden mikroservisler.\
  **Avantajlar:** Sunucu/altyapı yönetimi yok; konteyner başına ödeme; ECS ve EKS ile kullanım.\
  **Sınavda:** Container çalıştırılacak ancak EC2 instance'ları ve node kapasitesi yönetilmek istenmiyorsa Fargate.

- **Lightsail:** Basitleştirilmiş sanal sunucu, disk, ağ ve uygulama altyapısı sunar.\
  **Kullanım alanları:** Küçük web siteleri ve bloglar; basit uygulamalar; geliştirme ve test ortamları.\
  **Avantajlar:** Sabit ve düşük fiyatlı paketler; kurulumu kolay VPS benzeri deneyim.\
  **Sınavda:** Küçük web sitesi, blog veya kolay kurulabilen VPS benzeri ortam.

- **AWS Batch:** Batch işlerini kuyruklar, zamanlar ve uygun işlem kapasitesinde çalıştırır.\
  **Kullanım alanları:** Bilimsel simülasyonlar; toplu veri işleme; model eğitimi gibi paralel işler.\
  **Avantajlar:** İşleri otomatik planlar ve uygun kapasitede çalıştırır; Spot kullanımıyla maliyet optimizasyonu.\
  **Sınavda:** Büyük veri setleri üzerinde toplu hesaplama, simülasyon veya çok sayıda bağımsız iş.

- **Elastic Beanstalk:** Uygulama dağıtımı, kapasite, yük dengeleme ve izleme gibi altyapı işlerini yönetir; EC2 tabanlı kaynaklara erişim imkânı da vardır.\
  **Kullanım alanları:** Web uygulaması dağıtımı (Java, .NET, Python, Node.js vb.); hızlı geliştirme ortamları; altyapı ayrıntısıyla uğraşmadan yayınlama.\
  **Avantajlar:** Kod yükleyince altyapıyı otomatik kurar; kaynaklara erişim ve kontrol kullanıcıda kalır.\
  **Sınavda:** Uygulama kodunu dağıtmak, altyapı ayrıntılarını daha az yönetmek.

- **Auto Scaling:** EC2 instance sayısını talebe veya politikalara göre artırır ve azaltır.\
  **Kullanım alanları:** Trafik dalgalanan web uygulamaları; maliyeti talebe göre optimize etme; hata toleranslı servisler.\
  **Avantajlar:** Otomatik kapasite ayarı; sağlık kontrolleriyle bozuk instance'ları değiştirme; maliyet kontrolü.\
  **Sınavda:** Trafik değişimlerine otomatik uyum ve kapasite maliyetini kontrol etme.

- **EC2 Placement Groups:** Instance'ların fiziksel yerleşimini kontrol ederek düşük gecikme, yüksek ağ performansı veya donanım arızalarına karşı ayrışma sağlar.\
  **Kullanım alanları:** HPC ve düşük gecikmeli kümeler (cluster); tek donanım arızasından etkilenmemesi gereken kritik instance'lar (spread); büyük dağıtık sistemler (partition).\
  **Avantajlar:** Fiziksel yerleşim üzerinde kontrol; performans veya dayanıklılık ihtiyacına göre ayrışma.\
  **Sınavda:** Cluster = düşük gecikme; Spread = instance'ları ayrı donanımlara yayma; Partition = büyük dağıtık sistemlerde bölüm bazlı ayrışma.

**Karıştırma:**
- ECS = container orkestrasyonu; Fargate = container çalıştırma altyapısı.
- EKS = yönetilen Kubernetes; ECS = AWS'nin kendi container orkestrasyonu.
- Lambda = fonksiyon çalıştırma; Fargate = container çalıştırma.
- Auto Scaling = kapasiteyi dinamik değiştirme; ELB (Elastic Load Balancing) = gelen trafiği hedeflere dağıtma.

## 2. Storage - Depolama

- **S3:** Nesne depolama; dosyalar bucket içinde object olarak saklanır. Çok yüksek dayanıklılık, sürümleme, lifecycle ve farklı depolama sınıfları sunar.\
  **Kullanım alanları:** Statik web içeriği ve medya; yedekleme ve arşiv; data lake; veri paylaşımı ve dağıtımı.\
  **Avantajlar:** Yüksek dayanıklılık; sınırsız ölçek; sürümleme, lifecycle ve geniş depolama sınıfı seçenekleri.\
  **Sınavda:** Yedekler, medya dosyaları, statik web içeriği, data lake ve arşiv.

- **S3 Storage Classes:** Veriye erişim sıklığı ve erişim süresine göre maliyeti optimize eder.\
  **Kullanım alanları:** Sık erişilen veriler (Standard); seyrek erişilen veriler (Standard-IA); erişim deseni belirsiz veriler (Intelligent-Tiering); arşiv (Glacier sınıfları).\
  **Avantajlar:** Erişim sıklığına göre maliyet optimizasyonu; lifecycle ile otomatik geçiş.\
  **Sınavda:** Sık erişim = S3 Standard; öngörülebilir seyrek erişim = Standard-IA; erişim deseni belirsiz = Intelligent-Tiering; arşiv = Glacier sınıfları.

- **EBS (Elastic Block Store):** EC2'ye bağlanan block storage; işletim sistemi diski ve veritabanı diskleri için kullanılır.\
  **Kullanım alanları:** EC2 işletim sistemi diskleri; ilişkisel veritabanı depolaması; düşük gecikmeli kalıcı blok depolama.\
  **Avantajlar:** Anlık snapshot alma; gp3/io2 gibi türlerle performans-fiyat seçeneği; AZ içinde dayanıklı.\
  **Sınavda:** Kalıcı block disk, düşük gecikmeli disk I/O'su ve belirli performans gereksinimleri.

- **EBS Snapshots:** EBS volume verisinin point-in-time yedeğidir; artımlı snapshot mekanizması kullanır.\
  **Kullanım alanları:** Volume yedekleme ve felaket kurtarma; AMI oluşturma; veriyi bölge/region kopyalama.\
  **Avantajlar:** Artımlı ve maliyeti verimli yedekleme; S3 üzerinde depolama; anlık geri yükleme.\
  **Sınavda:** Volume yedeği, başka bir AZ (Availability Zone) veya Region'a kopyalama ve kurtarma.

- **Instance Store:** Fiziksel sunucuya bağlı geçici block storage; kalıcı veri için uygun değildir.\
  **Kullanım alanları:** Geçici cache ve buffer; yeniden oluşturulabilir veriler; yüksek IOPS gerektiren geçici iş yükleri.\
  **Avantajlar:** Çok yüksek yerel disk performansı; instance'a dahil olduğu için ek disk maliyeti yok.\
  **Sınavda:** Geçici cache, buffer ve yeniden oluşturulabilir yüksek performanslı veriler.

- **EFS (Elastic File System):** Birden fazla Linux/Unix istemcisinin aynı dosya sistemine erişmesini sağlayan yönetilen NFS dosya depolamasıdır.\
  **Kullanım alanları:** Birden fazla instance'ın paylaştığı dosya sistemleri; içerik yönetimi; Linux uygulamaları için paylaşımlı depolama.\
  **Avantajlar:** Otomatik ölçeklenir; çoklu AZ'den erişim; NFS uyumlu.\
  **Sınavda:** Birden fazla EC2 instance'ının aynı dosyaları eş zamanlı okuması ve yazması.

- **FSx:** Windows File Server, Lustre, NetApp ONTAP ve OpenZFS gibi yönetilen dosya sistemleri sunar.\
  **Kullanım alanları:** Windows uygulamaları ve SMB paylaşımları; HPC ve ML için Lustre; NetApp ONTAP gerektiren kurumsal iş yükleri.\
  **Avantajlar:** Yönetilen dosya sistemleri; yüksek performans; mevcut protokol ve araçlarla uyumluluk.\
  **Sınavda:** Windows SMB/Active Directory, HPC için Lustre veya belirli kurumsal dosya sistemi özellikleri.

- **Storage Gateway:** On-premises ortam ile AWS depolama servisleri arasında hibrit entegrasyon sağlar.\
  **Kullanım alanları:** On-premises uygulamaların bulut yedeklemesi; hibrit depolama uzantısı; dosya/blok/tape iş yükleri.\
  **Avantajlar:** Yerel erişim hızı + bulut depolama maliyeti; VMware/Hyper-V uyumlu.\
  **Sınavda:** Yerel uygulamaların bulut depolamasına geçişini kolaylaştırma.

- **AWS Snow Family:** Büyük miktarda veriyi fiziksel cihazlarla taşıma ve uygun modellerde edge computing sağlama yaklaşımıdır.\
  **Kullanım alanları:** TB-PB ölçeğinde veri taşıma; sınırlı bant genişliği olan lokasyonlar; uç (edge) işleme senaryoları.\
  **Avantajlar:** Fiziksel transferle ağ maliyeti ve süresi ortadan kalkar; şifreli ve dayanıklı cihazlar.\
  **Sınavda:** Sınırlı ağ bant genişliği, yüksek veri aktarım süresi veya bağlantı kısıtları.

- **AWS DataSync:** NFS, SMB ve desteklenen depolama sistemleri arasında veriyi otomatik ve verimli biçimde taşır.\
  **Kullanım alanları:** On-premises veriyi S3/EFS/FSx'e düzenli taşıma; bulutlar arası kopyalama; göç öncesi sürekli senkronizasyon.\
  **Avantajlar:** Otomatik şifreleme ve veri doğrulama; ağ optimizasyonu; zamanlanmış otomatik transfer.\
  **Sınavda:** On-premises dosyaları S3 veya EFS'ye düzenli ya da toplu taşıma.

- **AWS Backup:** Desteklenen AWS kaynakları için merkezi yedekleme planları, politikalar ve saklama yönetimi sunar.\
  **Kullanım alanları:** RDS, EBS, EFS, S3 ve DynamoDB için merkezi yedekleme; uyumluluk amaçlı saklama; çoklu hesap yedek yönetimi.\
  **Avantajlar:** Tek panelden politika yönetimi; otomatik saklama ve silmeye karşı koruma.\
  **Sınavda:** Birden fazla AWS servisinin yedeklerini merkezi yönetmek.

**Karıştırma:**
- S3 = object; EBS = block; EFS = shared file.
- EBS genellikle bir AZ'ye bağlıdır; EFS birden fazla AZ'den erişilebilir şekilde yapılandırılabilir.
- Instance Store geçicidir; EBS kalıcı disk olarak kullanılabilir.
- DataSync = veri taşıma; Storage Gateway = hibrit depolama entegrasyonu; Snow Family = fiziksel veri aktarımı.

## 3. Database - Veritabanları

- **RDS (Relational Database Service):** MySQL, PostgreSQL, MariaDB, Oracle ve SQL Server gibi ilişkisel veritabanlarını yönetilen hizmet olarak çalıştırır.\
  **Kullanım alanları:** İlişkisel uygulamalar (MySQL, PostgreSQL, Oracle vb.); ACID gerektiren iş yükleri; yönetilen veritabanı ihtiyacı.\
  **Avantajlar:** Otomatik yedekleme, patch ve failover; Multi-AZ ve Read Replica seçenekleri.\
  **Sınavda:** SQL, ilişkisel veri, otomatik yedekleme ve yönetilen veritabanı.

- **RDS Multi-AZ:** Veritabanı altyapısının farklı AZ'lerde yüksek erişilebilirlik için yedeklenmesini sağlar.\
  **Kullanım alanları:** Yüksek erişilebilirlik gerektiren üretim veritabanları; planlanmamış arızalarda kesintisiz devam.\
  **Avantajlar:** Otomatik failover; senkron replikasyon; uygulama tarafında değişiklik gerekmez.\
  **Sınavda:** Veritabanı instance'ı veya AZ arızasında otomatik failover.

- **RDS Read Replica:** Okuma sorgularını ana veritabanından ayırmak için asenkron replika oluşturur.\
  **Kullanım alanları:** Raporlama ve analitik okumalar; okuma trafiğini ölçekleme; bölge dışı kopya.\
  **Avantajlar:** Okuma kapasitesini artırır; asenkron replikasyon; birden fazla kopya.\
  **Sınavda:** Okuma ağırlıklı iş yüklerinde performans artırma.

- **Amazon Aurora:** AWS tarafından tasarlanmış, MySQL ve PostgreSQL uyumlu ilişkisel veritabanıdır. Ayrıştırılmış, dağıtık depolama mimarisi ve yönetilen failover özellikleri sunar.\
  **Kullanım alanları:** Yüksek performanslı ilişkisel uygulamalar; oyun, medya ve içerik yönetimi, gerçek zamanlı analitik; MySQL/PostgreSQL iş yükleri.\
  **Avantajlar:** Standart motorlara göre yüksek performans; otomatik ölçeklenen depolama; 15 Read Replica'ya kadar.\
  **Sınavda:** Yüksek performans, yüksek erişilebilirlik ve ilişkisel veri.

- **Aurora Serverless:** Aurora kapasitesini desteklenen yapılandırmalarda talebe göre otomatik ölçekler.\
  **Kullanım alanları:** Düzensiz veya tahmin edilemeyen trafikli uygulamalar; geliştirme/test ortamları; sezonluk iş yükleri.\
  **Avantajlar:** Kapasite otomatik ölçeklenir; kullanılmadığında durabilir; saniye bazlı faturalandırma.\
  **Sınavda:** Değişken veya öngörülemeyen veritabanı yükü.

- **DynamoDB:** Yönetilen, dağıtık NoSQL key-value ve document veritabanıdır.\
  **Kullanım alanları:** Oyun skorları ve profilleri; sepet ve oturum verileri; IoT ve mobil uygulama verileri; anahtar-değer erişimi.\
  **Avantajlar:** Tek haneli milisaniye gecikme; sunucusuz ve sınırsız ölçek; TTL ve Global Tables.\
  **Sınavda:** Çok yüksek ölçek, düşük gecikmeli anahtar erişimi ve sunucu yönetimi olmadan kapasite yönetimi.

- **DynamoDB DAX (DynamoDB Accelerator):** DynamoDB okumaları için bellek içi cache katmanıdır.\
  **Kullanım alanları:** Yoğun okunan DynamoDB tabloları; oyun panoları; flash-sale gibi ani yük senaryoları.\
  **Avantajlar:** Mikrosaniye seviyesinde okuma; write-through cache; uygulama kodunda değişiklik gerekmez.\
  **Sınavda:** Sık okunan verilerde mikro saniye düzeyinde okuma gecikmesi hedefi.

- **ElastiCache:** Redis OSS veya Memcached tabanlı yönetilen cache çözümleri sunar.\
  **Kullanım alanları:** Veritabanı önü cache; oturum yönetimi; liderlik panoları (Redis); gerçek zamanlı analitik.\
  **Avantajlar:** Milisaniye altı gecikme; veritabanı yükünü azaltır; Redis ve Memcached seçenekleri.\
  **Sınavda:** Veritabanı üzerindeki okuma yükünü azaltma ve uygulama gecikmesini düşürme.

- **Redshift:** Analitik sorgular ve büyük ölçekli veri ambarı iş yükleri için kullanılır.\
  **Kullanım alanları:** Büyük ölçekli veri ambarı; iş zekâsı raporları; yılların satış verisi üzerinde analitik sorgular.\
  **Avantajlar:** Kolonlu depolama ile hızlı sorgu; SQL uyumlu; petabayt ölçek.\
  **Sınavda:** Büyük veri üzerinde SQL raporlama, BI ve analitik.

- **Neptune:** İlişkileri ve bağlantıları grafik modeliyle tutar.\
  **Kullanım alanları:** Sosyal ağlar; öneri motorları; dolandırıcılık tespiti; graf verisi gereken uygulamalar.\
  **Avantajlar:** Gremlin ve SPARQL desteği; yüksek dayanıklılık; hızlı graf sorguları.\
  **Sınavda:** Sosyal ağ ilişkileri, fraud bağlantıları ve graph traversal.

- **DocumentDB:** MongoDB ile uyumlu API'ler sunan yönetilen document database hizmetidir.\
  **Kullanım alanları:** MongoDB uyumlu uygulamalar; içerik katalogları; kullanıcı profilleri.\
  **Avantajlar:** MongoDB uyumluluğu; yönetilen yedekleme ve failover; otomatik ölçeklenen depolama.\
  **Sınavda:** Belge tabanlı veri modeli ve MongoDB uyumlu uygulama gereksinimleri.

- **OpenSearch Service:** Tam metin arama, log analizi ve arama odaklı analiz sağlar.\
  **Kullanım alanları:** Uygulama içi arama; log analizi; güvenlik olayı izleme.\
  **Avantajlar:** Tam metin arama ve analitik; Kibana görselleştirme; gerçek zamanlı sorgular.\
  **Sınavda:** Uygulama arama kutusu, log arama ve indekslenmiş metin sorguları.

- **Timestream:** Zaman damgalı veriler ve zaman serisi analizi için tasarlanmıştır.\
  **Kullanım alanları:** IoT sensör verileri; DevOps metrikleri; zaman serisi analitiği.\
  **Avantajlar:** Zaman serisine özel depolama; otomatik katmanlandırma; SQL sorgu desteği.\
  **Sınavda:** IoT telemetrisi, sensör verileri ve zaman serisi metrikleri.

- **QLDB (Quantum Ledger Database):** Değişiklik geçmişini kriptografik olarak doğrulanabilir bir ledger yapısında tutmak için tasarlanmıştı.\
  **Kullanım alanları:** Denetlenebilir işlem geçmişi; tedarik zinciri kayıtları; sistem-of-record doğrulama.\
  **Avantajlar:** Değiştirilemez ledger; kriptografik doğrulama; merkezi ve yönetilen altyapı.\
  **Sınavda:** Eski kaynaklarda karşılaşabilirsin; AWS, QLDB desteğini 31 Temmuz 2025'te sona erdirdiğinden güncel mimariler için aktif bir seçenek olarak değerlendirme.

**Karıştırma:**
- Multi-AZ = yüksek erişilebilirlik ve failover; Read Replica = okuma kapasitesini artırma.
- RDS = ilişkisel veritabanı; DynamoDB = NoSQL; Redshift = analitik veri ambarı.
- ElastiCache = uygulama cache'i; DAX = DynamoDB'ye özel cache.
- OpenSearch = arama ve indeksleme; Neptune = grafik ilişkileri.

## 4. Networking & CDN - Ağ ve Trafik Yönetimi

- **VPC (Virtual Private Cloud):** AWS kaynaklarının IP aralıklarını, subnet'lerini, route table'larını ve ağ güvenliğini tanımlayan izole sanal ağdır.\
  **Kullanım alanları:** İzole bulut ağı; çok katmanlı uygulama mimarileri; özel subnet'li iş yükleri.\
  **Avantajlar:** Subnet, route ve gateway üzerinde tam kontrol; katmanlı güvenlik yapılandırması.\
  **Sınavda:** Kaynakları özel ağda çalıştırma ve ağ erişimini kontrol etme.

- **Subnet:** VPC içindeki belirli bir Availability Zone'a ait IP aralığıdır.\
  **Kullanım alanları:** Genel/özel katman ayrımı; erişim kontrolü; AZ bazlı kaynak yerleşimi.\
  **Avantajlar:** Kaynakları mantıksal olarak gruplama; NACL ile katman güvenliği.\
  **Sınavda:** Kaynakları AZ'lere ve public/private ağ katmanlarına ayırma.

- **Internet Gateway (IGW):** VPC ile internet arasında internet yönlendirmesini sağlar.\
  **Kullanım alanları:** VPC'nin internete çıkması; genel IP'li kaynakların internet erişimi.\
  **Avantajlar:** Çift yönlü internet erişimi; yüksek bant genişliği; ek ücret yok.\
  **Sınavda:** Public subnet'teki kaynağın internete erişmesi; bunun için route, public IP ve güvenlik kuralları da gerekir.

- **NAT Gateway:** Private subnet'teki IPv4 kaynaklarının internete veya dış ağlara bağlantı başlatmasını sağlar; internetten doğrudan başlatılan bağlantıları kabul etmez.\
  **Kullanım alanları:** Özel subnet'lerden internete çıkış; özel instance'ların güncelleme ve paket erişimi.\
  **Avantajlar:** Gelen trafiğe kapalı güvenlik; yönetilen ve otomatik ölçeklenen servis.\
  **Sınavda:** Private EC2'nin yazılım güncellemesi indirmesi.

- **Security Group:** ENI (Elastic Network Interface)/instance düzeyinde stateful sanal güvenlik duvarıdır.\
  **Kullanım alanları:** Instance seviyesinde trafik filtreleme; port bazlı erişim izinleri.\
  **Avantajlar:** Stateful yapı (izin verilen dönüş trafiği otomatik); kural yönetimi kolay.\
  **Sınavda:** Yalnızca belirli kaynaklardan belirli portlara izin verme.

- **Network ACL (NACL):** Subnet düzeyinde stateless filtreleme yapar; inbound ve outbound kuralları ayrı değerlendirilir.\
  **Kullanım alanları:** Subnet seviyesinde trafik kontrolü; belirli IP adreslerini engelleme.\
  **Avantajlar:** Explicit deny desteği; stateless yapı; subnet genelinde koruma.\
  **Sınavda:** Subnet düzeyinde ek filtreleme veya belirli IP aralıklarını engelleme.

- **Route Table:** Hedef IP aralığına göre trafiğin IGW, NAT, peering veya başka bir hedefe yönlendirilmesini belirler.\
  **Kullanım alanları:** Trafik yönlendirme kuralları; internet veya VPC endpoint yönlendirmesi; VPN/Direct Connect trafiği.\
  **Avantajlar:** Subnet bazlı hedef kontrolü; basit ve esnek yönlendirme.\
  **Sınavda:** Public/private subnet erişim sorunları ve ağ yolları.

- **Route 53:** DNS, health check ve routing policy hizmetidir.\
  **Kullanım alanları:** DNS yönetimi; bölge bazlı yönlendirme; sağlık kontrollü failover; domain kayıt.\
  **Avantajlar:** Yüksek erişilebilirlik; latency/weighted/geolocation politikaları; alias ile ücretsiz sorgular.\
  **Sınavda:** DNS yönlendirme, failover, latency-based veya geolocation routing.

- **CloudFront:** İçeriği edge location'larda cache ederek kullanıcılara daha düşük gecikmeyle ulaştıran CDN'dir.\
  **Kullanım alanları:** Statik içerik ve medya dağıtımı; API hızlandırma; DDoS/bot koruması katmanı.\
  **Avantajlar:** Kenar (edge) önbellekleme ile düşük gecikme; S3 internet çıkış ücretlerini düşürür.\
  **Sınavda:** Global kullanıcılar, statik içerik hızlandırma ve origin yükünü azaltma.

- **Elastic Load Balancing (ELB):** Trafiği sağlıklı hedeflere dağıtır.\
  **Kullanım alanları:** Gelen trafiği birden fazla hedefe dağıtma; HTTPS sonlandırma; sağlık kontrollü yönlendirme.\
  **Avantajlar:** Otomatik ölçeklenen yönetilen servis; ALB/NLB/GWLB seçenekleri; AZ'ler arası dağıtım.\
  **Sınavda:** ALB (Application Load Balancer) = HTTP/HTTPS ve katman 7 yönlendirme; NLB (Network Load Balancer) = katman 4, TCP/UDP ve yüksek performans; Gateway Load Balancer = sanal ağ güvenlik cihazları.

- **AWS Global Accelerator:** Anycast statik IP'ler ve AWS global ağı üzerinden trafiği uygun bölgesel endpoint'lere yönlendirir.\
  **Kullanım alanları:** Global kullanıcılar için düşük gecikmeli uygulamalar; tek IP ile çoklu bölge failover.\
  **Avantajlar:** AWS omurgası üzerinden yönlendirme; sabit giriş IP'leri; hızlı bölgesel geçiş.\
  **Sınavda:** Global uygulamalar, hızlı failover ve TCP/UDP trafiğinin AWS ağı üzerinden yönlendirilmesi.

- **AWS Direct Connect:** On-premises veri merkezi ile AWS arasında özel ağ bağlantısı sağlar.\
  **Kullanım alanları:** On-premises'ten AWS'e özel bağlantı; yüksek hacimli veri aktarımı; gecikme duyarlı hibrit mimariler.\
  **Avantajlar:** İnternet dışı özel hat; tutarlı bant genişliği ve gecikme; ağ maliyetinde tasarruf.\
  **Sınavda:** Daha tutarlı ağ performansı, yüksek bant genişliği ve özel bağlantı ihtiyacı.

- **Site-to-Site VPN:** On-premises ağ ile AWS arasında internet üzerinden IPsec tünelleri oluşturur.\
  **Kullanım alanları:** On-premises ağdan VPC'ye güvenli tünel; geçici veya yedek bağlantı; küçük ofis bağlantıları.\
  **Avantajlar:** Hızlı kurulum; IPSec şifreleme; Direct Connect'e ucuz yedek.\
  **Sınavda:** Hızlı hibrit bağlantı veya Direct Connect'e yedek bağlantı.

- **VPC Peering:** İki VPC arasında doğrudan özel IP yönlendirmesi sağlar; transit yönlendirme desteklemez.\
  **Kullanım alanları:** İki VPC arasında özel iletişim; farklı hesap veya bölgeler arası erişim.\
  **Avantajlar:** Ağ geçidi gerektirmez; düşük gecikme; basit yapılandırma.\
  **Sınavda:** Az sayıda VPC'yi birbirine bağlama.

- **Transit Gateway:** Çok sayıda VPC ve on-premises ağı merkezi hub üzerinden bağlar.\
  **Kullanım alanları:** Çok sayıda VPC'yi merkezi bağlama; hub-and-spoke ağ; VPN/Direct Connect'i merkezileştirme.\
  **Avantajlar:** Transitif yönlendirme; merkezi kontrol; peering mesh karmaşıklığını ortadan kaldırır.\
  **Sınavda:** Çoklu hesap ve VPC'lerde merkezi ağ topolojisi.

- **VPC Endpoints:** AWS servislerine özel ağ üzerinden erişim sağlar.\
  **Kullanım alanları:** Özel subnet'ten AWS servislerine internetsiz erişim; S3/DynamoDB erişiminde NAT maliyetini kesme.\
  **Avantajlar:** Trafik AWS ağında kalır; gateway endpoint ücretsiz; politika ile erişim kontrolü.\
  **Sınavda:** Private subnet'teki EC2'nin internet/NAT kullanmadan S3 veya DynamoDB'ye erişmesi için Gateway Endpoint; birçok diğer servis için uygun Interface Endpoint.

- **AWS PrivateLink:** Bir servise VPC endpoint üzerinden özel erişim sağlar; servis sağlayıcı ve tüketici ağlarını doğrudan birbirine açmadan bağlantı kurabilir.\
  **Kullanım alanları:** Üçüncü parti SaaS'a özel erişim; VPC'ler arası servis paylaşımı; özel servis yayınlama.\
  **Avantajlar:** Public IP'siz ve internetsiz bağlantı; tek yönlü güvenli erişim; VPC dışına sızıntı yok.\
  **Sınavda:** Başka bir VPC'deki veya üçüncü taraf sağlayıcıdaki servise özel erişim.

**Karıştırma:**
- Public subnet olmak tek başına internete erişim sağlamaz: IGW route'u ve uygun public IP gerekir.
- NAT Gateway = private kaynakların outbound IPv4 internet erişimi; IGW = VPC'nin internet bağlantısı.
- Security Group = stateful; NACL = stateless.
- VPC Peering = transit yok; Transit Gateway = merkezi transit yönlendirme.
- CloudFront = içerik cache/CDN; Global Accelerator = global ağ yönlendirme.
- Route 53 = DNS; ELB = uygulama trafiğini hedeflere dağıtma.

## 5. Integration & Messaging - Mesajlaşma ve İş Akışları

- **SQS:** Dağıtık uygulamalar arasında mesajları kuyrukta tutarak üretici ve tüketiciyi birbirinden ayırır.\
  **Kullanım alanları:** Servisleri birbirinden ayırma (decoupling); yük patlaması tamponlama; iş kuyrukları.\
  **Avantajlar:** Sunucusuz ve sınırsız ölçek; DLQ ile başarısız mesaj yönetimi; at-least-once teslimat.\
  **Sınavda:** Asenkron işleme, yük patlamalarını tamponlama ve başarısız mesajları DLQ (dead-letter queue)'ya yönlendirme.

- **SQS FIFO:** Mesaj sıralamasını ve uygun kullanımda deduplication desteğini sağlar.\
  **Kullanım alanları:** Sipariş işleme; bankacılık işlemleri; tekilleştirme gerektiren iş akışları.\
  **Avantajlar:** Sıralı teslimat; exactly-once processing; grup bazlı paralellik.\
  **Sınavda:** Sıralama veya mesajların tekilleştirilmesi gerektiğinde FIFO; aksi durumda Standard daha yüksek esneklik ve ölçek sunar.

- **SNS (Simple Notification Service):** Bir mesajı birden fazla aboneye yayınlayan pub/sub hizmetidir.\
  **Kullanım alanları:** Olay bildirimleri; e-posta/SMS/Lambda bildirimi; fan-out ile çok tüketiciye yayın.\
  **Avantajlar:** Push tabanlı; çoklu abonelik protokolü; SQS ile fan-out deseni.\
  **Sınavda:** Tek olayın birden fazla SQS kuyruğuna, Lambda fonksiyonuna veya başka desteklenen abonelere iletilmesi.

- **EventBridge:** Servislerden gelen olayları kurallara göre hedeflere yönlendirir.\
  **Kullanım alanları:** Uygulama ve SaaS olaylarını filtreleyip yönlendirme; zamanlanmış kurallar; olay tabanlı mimari.\
  **Avantajlar:** İçerik bazlı filtreleme; olay arşivi ve replay; çoklu hedef desteği.\
  **Sınavda:** Bir AWS kaynağındaki olay gerçekleşince Lambda, Step Functions veya başka bir hedefi tetikleme.

- **Step Functions:** Birden fazla adımı, dallanmayı, tekrar denemeyi ve hata yönetimini state machine ile orkestre eder.\
  **Kullanım alanları:** Çok adımlı iş akışları; onay süreçleri; hata kurtarmalı orkestrasyon.\
  **Avantajlar:** Görsel akış yönetimi; otomatik retry; insan onayı (wait) adımları; sunucusuz.\
  **Sınavda:** Birden fazla servisten oluşan iş akışı ve kontrollü hata yönetimi.

- **Kinesis Data Streams:** Gerçek zamanlı veri akışlarını toplar ve tüketicilere sunar.\
  **Kullanım alanları:** Gerçek zamanlı veri akışı (log, clickstream, IoT); veriyi birden fazla tüketiciye dağıtmak.\
  **Avantajlar:** Saniyede binlerce olay; olayları yeniden oynatma; shard ile ölçekleme.\
  **Sınavda:** Clickstream, IoT veya sürekli gelen logların düşük gecikmeyle işlenmesi.

- **Amazon Data Firehose:** Akış verisini desteklenen hedeflere teslim eder; tamponlama ve dönüşüm seçenekleri sunar.\
  **Kullanım alanları:** Veriyi S3/Redshift/OpenSearch'e otomatik yükleme; near real-time veri ambarı besleme.\
  **Avantajlar:** Tam yönetilen; otomatik ölçekleme; dönüşüm ve sıkıştırma seçenekleri.\
  **Sınavda:** Streaming verisini S3, Redshift veya desteklenen hedeflere yönetilen biçimde aktarma.

- **Managed Service for Apache Flink:** Akış verisi üzerinde durum bilgili gerçek zamanlı analiz çalıştırır.\
  **Kullanım alanları:** Akış verisi üzerinde gerçek zamanlı analitik; pencereleme ve agregasyon işlemleri.\
  **Avantajlar:** Yönetilen Flink; SQL ve Java desteği; otomatik ölçekleme.\
  **Sınavda:** Gelen olaylar üzerinde sürekli hesaplama ve pencere bazlı analiz. Eski kaynaklarda Kinesis Data Analytics adıyla geçebilir.

- **API Gateway:** Yönetilen API endpoint'leri oluşturur; kimlik doğrulama, throttling ve entegrasyon özellikleri sunar.\
  **Kullanım alanları:** REST/WebSocket API yayınlama; Lambda backend'leri; API anahtarı ve throttling.\
  **Avantajlar:** Yönetilen API katmanı; istek başı ödeme; cache ve sürümleme.\
  **Sınavda:** Lambda veya backend servisleri için yönetilen HTTP API katmanı.

- **AWS AppSync:** GraphQL API'leri ve desteklenen veri kaynaklarıyla entegrasyon sağlar.\
  **Kullanım alanları:** GraphQL API'ler; mobil/web için senkron veri; gerçek zamanlı uygulamalar.\
  **Avantajlar:** Otomatik veri senkronizasyonu; offline destek; çoklu veri kaynağı.\
  **Sınavda:** İstemcinin ihtiyacı olan alanları tek GraphQL API üzerinden sorgulaması.

- **Amazon MQ:** ActiveMQ veya RabbitMQ gibi mesajlaşma sistemleriyle uyumluluk sağlar.\
  **Kullanım alanları:** ActiveMQ/RabbitMQ tabanlı mevcut uygulamalar; kurumsal mesajlaşma geçişleri.\
  **Avantajlar:** Standart protokoller (JMS, AMQP); yönetilen broker; on-premises uyumluluk.\
  **Sınavda:** Mevcut uygulamanın geleneksel broker protokollerini ve özelliklerini koruması gerekiyorsa.

**Karıştırma:**
- SQS = mesajı kuyrukta tutar; SNS = mesajı abonelere yayınlar.
- EventBridge = olay bazlı yönlendirme; SQS = asenkron kuyruk.
- Step Functions = iş akışının adımlarını yönetir; EventBridge = olayları kurallara göre yönlendirir.
- Kinesis Data Streams = akışı toplama ve tüketme; Firehose = akışı hedefe teslim etme.
- API Gateway = API katmanı; AppSync = GraphQL API.

## 6. Security & Identity - Güvenlik ve Kimlik

- **IAM (Identity and Access Management):** AWS kullanıcılarının, rollerinin ve servislerin hangi işlemleri yapabileceğini policy'lerle belirler.\
  **Kullanım alanları:** Kullanıcı ve servis yetkilendirmesi; politika tabanlı erişim kontrolü; çoklu hesap yönetimi.\
  **Avantajlar:** Merkezi erişim kontrolü; least privilege; ayrıntılı politika dili.\
  **Sınavda:** Least privilege, IAM role, cross-account access ve servisler arası yetkilendirme.

- **IAM Role:** Uzun ömürlü erişim anahtarları yerine geçici kimlik bilgileriyle AWS kaynaklarına erişim sağlar.\
  **Kullanım alanları:** EC2/Lambda'ya geçici kimlik verme; çapraz hesap erişimi; servisler arası yetkilendirme.\
  **Avantajlar:** Uzun ömürlü anahtar yok; geçici kimlik bilgileri; güvenli çapraz hesap erişimi.\
  **Sınavda:** EC2'nin S3'e veya Lambda'nın DynamoDB'ye güvenli erişmesi.

- **AWS KMS (Key Management Service):** Şifreleme anahtarlarını ve bunların kullanım izinlerini yönetir.\
  **Kullanım alanları:** Şifreleme anahtarı yönetimi; S3/EBS/RDS şifreleme; anahtar rotasyonu.\
  **Avantajlar:** Denetlenebilir anahtar kullanımı (CloudTrail); servislerle entegre şifreleme; FIPS uyumlu.\
  **Sınavda:** S3, EBS, RDS veya uygulama verisini şifreleme ve anahtar erişimini kontrol etme.

- **AWS Secrets Manager:** Veritabanı parolaları, API anahtarları ve diğer gizli bilgileri saklar; desteklenen entegrasyonlarda rotasyonu yönetebilir.\
  **Kullanım alanları:** Veritabanı kimlik bilgileri saklama; otomatik rotasyon; API anahtarları.\
  **Avantajlar:** Otomatik sır rotasyonu; çapraz hesap erişimi; KMS ile şifreleme.\
  **Sınavda:** Veritabanı parolasını güvenli saklama ve otomatik döndürme.

- **SSM (Systems Manager) Parameter Store:** Konfigürasyon ve parametreleri merkezi saklar; SecureString ile KMS destekli şifreleme yapabilir.\
  **Kullanım alanları:** Uygulama konfigürasyonları; ortam değişkenleri; basit parametre yönetimi.\
  **Avantajlar:** Standart katman ücretsiz; hierarşik yapı; sürüm geçmişi.\
  **Sınavda:** Uygulama ayarları ve basit parametre yönetimi.

- **Amazon Cognito:** Uygulama kullanıcılarının kayıt, giriş ve token tabanlı kimlik doğrulama ihtiyaçlarını karşılar.\
  **Kullanım alanları:** Mobil/web uygulamalarında kullanıcı girişi; sosyal login (Facebook/Google); misafir erişimi.\
  **Avantajlar:** Hazır UI ve SDK'lar; kimlik havuzu ile AWS erişimi; otomatik ölçekleme.\
  **Sınavda:** Web veya mobil uygulama kullanıcılarını AWS IAM kullanıcılarından ayrı yönetme.

- **AWS WAF (Web Application Firewall):** HTTP/HTTPS isteklerini web ACL ve kurallarla filtreler.\
  **Kullanım alanları:** SQL injection ve XSS koruması; bot trafiğini filtreleme; oran bazlı istek sınırlama.\
  **Avantajlar:** Katman 7 filtreleme; yönetilen kural kümeleri; CloudFront/ALB ile entegrasyon.\
  **Sınavda:** SQL injection, XSS, IP tabanlı filtreleme ve istek hız sınırlandırma.

- **AWS Shield:** DDoS saldırılarına karşı koruma sağlar; Standard temel korumayı otomatik sunar, Advanced ek özellikler sağlar.\
  **Kullanım alanları:** DDoS koruması; yüksek trafikli web servisleri; saldırı maliyeti koruması (Advanced).\
  **Avantajlar:** Standart ücretsiz; Advanced'te 7/24 DDoS Response Team ve maliyet tazminatı.\
  **Sınavda:** DDoS riskini azaltma.

- **Amazon GuardDuty:** CloudTrail, VPC Flow Logs ve diğer desteklenen veri kaynaklarından tehdit göstergeleri çıkarır.\
  **Kullanım alanları:** Hesap güvenliği izleme; tehdit tespiti (kompromize kimlik, kötü amaçlı trafik); CloudTrail/VPC log analizi.\
  **Avantajlar:** Kurulum gerektirmez; makine öğrenmesi tabanlı bulgular; risk önceliklendirmesi.\
  **Sınavda:** Şüpheli API çağrıları, olası hesap ihlali ve kötü amaçlı ağ etkinliği.

- **Amazon Inspector:** Desteklenen kaynaklarda yazılım zafiyetlerini ve belirli güvenlik risklerini değerlendirir.\
  **Kullanım alanları:** EC2/EKS/ECR'de yazılım açıklığı taraması; benchmark uyumluluğu; sürekli güvenlik değerlendirmesi.\
  **Avantajlar:** Otomatik bulgu yönetimi; risk önceliklendirmesi; CI/CD entegrasyonu.\
  **Sınavda:** EC2, container image ve desteklenen compute kaynaklarında vulnerability management.

- **Amazon Macie:** S3'teki hassas verileri, makine öğrenmesi ve eşleştirme yöntemleriyle keşfetmeye yardımcı olur.\
  **Kullanım alanları:** S3'de hassas veri (PII, kredi kartı) keşfi; veri sınıflandırma; uyumluluk raporları.\
  **Avantajlar:** Makine öğrenmesi ile otomatik tespit; ayrıntılı bulgular; S3 genelinde görünürlük.\
  **Sınavda:** S3 içindeki PII ve benzeri hassas bilgileri tespit etme.

- **AWS Security Hub:** Birden fazla güvenlik kaynağından gelen bulguları merkezileştirir ve güvenlik duruşunu değerlendirmeye yardımcı olur.\
  **Kullanım alanları:** Çoklu hesap güvenlik duruşu; bulguları merkezileştirme; uyumluluk kontrolleri.\
  **Avantajlar:** Tek panelden görünürlük; standart bazlı otomatik kontroller; entegre bulgu formatı.\
  **Sınavda:** Birden fazla güvenlik hizmetinin bulgularını tek yerde toplama.

- **AWS CloudTrail:** AWS hesap etkinliğini ve API çağrılarını kaydeder.\
  **Kullanım alanları:** API çağrı denetimi; 'kim ne yaptı' sorusu; uyumluluk kayıtları.\
  **Avantajlar:** Hesaplar arası aktivite kaydı; S3'e log aktarımı; CloudTrail Lake ile sorgulama.\
  **Sınavda:** Kim, ne zaman, hangi API işlemini yaptı sorusunun cevabı.

- **AWS Artifact:** AWS uyumluluk raporlarına ve ilgili anlaşma belgelerine erişim sağlar.\
  **Kullanım alanları:** SOC/PCI gibi uyumluluk raporlarına erişme; denetim kanıtları; iş ortağı soru formları.\
  **Avantajlar:** Tek noktadan rapor indirme; NDA otomasyonu; güncel belgeler.\
  **Sınavda:** AWS'nin SOC, ISO ve PCI gibi uyumluluk belgelerini edinme.

- **AWS Certificate Manager (ACM):** Desteklenen AWS servislerinde TLS sertifikalarının sağlanması ve yenilenmesini yönetir.\
  **Kullanım alanları:** Ücretsiz SSL/TLS sertifikaları; CloudFront/ALB'de HTTPS; otomatik sertifika yenileme.\
  **Avantajlar:** Ücretsiz sertifika; otomatik yenileme ve dağıtım; yönetilen doğrulama.\
  **Sınavda:** ALB veya CloudFront üzerinde HTTPS sertifikası kullanma.

- **AWS Directory Service:** Microsoft Active Directory ile ilgili yönetilen dizin ihtiyaçlarını karşılar.\
  **Kullanım alanları:** AWS'de Active Directory; Microsoft uygulamaları; Windows tabanlı iş yükleri.\
  **Avantajlar:** Yönetilen AD; on-premises AD ile güven ilişkisi; Kerberos/LDAP desteği.\
  **Sınavda:** Windows iş yükleri, domain join ve Active Directory entegrasyonu.

**Karıştırma:**
- IAM = AWS kaynaklarına erişim yetkisi; Cognito = uygulama kullanıcı kimliği.
- KMS = anahtar yönetimi; Secrets Manager = gizli bilgilerin saklanması ve rotasyonu.
- CloudTrail = API denetim kaydı; CloudWatch = metrik, log ve alarm.
- GuardDuty = tehdit tespiti; Inspector = zafiyet taraması; Macie = S3 hassas veri keşfi.
- WAF = web istek filtreleme; Shield = DDoS koruması; Security Hub = güvenlik bulgularını birleştirme.

## 7. Management & Monitoring - Yönetim ve İzleme

- **Amazon CloudWatch:** Metrik, log, alarm ve gözlemlenebilirlik özellikleri sağlar.\
  **Kullanım alanları:** Metrik izleme ve alarmlar; otomatik ölçekleme tetikleme; operasyon panoları.\
  **Avantajlar:** Servis metrikleri hazır gelir; alarmlarla otomasyon; metrik ve log birleştirme.\
  **Sınavda:** CPU kullanımı yükselince alarm oluşturma veya belirli bir metriğe göre aksiyon başlatma.

- **CloudWatch Logs:** Uygulama ve servis loglarını toplar, saklar ve sorgulamaya yardımcı olur.\
  **Kullanım alanları:** Uygulama ve sistem logları; log sorgulama (Logs Insights); hata ayıklama.\
  **Avantajlar:** Merkezi log toplama; metrik filtreleme; uzun süreli saklama.\
  **Sınavda:** Hata kayıtlarını inceleme, log grupları ve log tabanlı metrikler.

- **AWS X-Ray:** Dağıtık uygulamalarda isteklerin servisler arasındaki izini sürer.\
  **Kullanım alanları:** Dağıtık uygulamalarda istek takibi; darboğaz tespiti; mikroservis hata ayıklama.\
  **Avantajlar:** Servis haritası; gecikme analizi; SDK entegrasyonu.\
  **Sınavda:** Mikroservislerde gecikmenin hangi servisten kaynaklandığını bulma.

- **AWS Config:** Kaynak konfigürasyonlarını kaydeder ve kurallarla uyumluluk değerlendirmesi yapar.\
  **Kullanım alanları:** Kaynak değişiklik geçmişi; uyumluluk kontrolleri; yanlış yapılandırma tespiti.\
  **Avantajlar:** Zamanda geriye bakış; otomatik uyumluluk kuralları; kaynak ilişkileri görünümü.\
  **Sınavda:** Security Group'un zaman içinde değişimini bulma veya kurallara aykırı kaynakları tespit etme.

- **AWS Systems Manager (SSM):** Desteklenen sunuculara uzaktan erişim, patch yönetimi, envanter ve otomasyon sağlar.\
  **Kullanım alanları:** Sunucu yönetimi ve patch; otomasyon (runbook'lar); oturum yönetimi (Session Manager).\
  **Avantajlar:** Ajan tabanlı merkezi yönetim; SSH anahtarı gerektirmez; envanter ve patch yönetimi.\
  **Sınavda:** SSH/RDP açmadan yönetim için Session Manager veya merkezi patch operasyonları.

- **AWS CloudFormation:** AWS kaynaklarını şablonlarla tanımlayıp yönetir.\
  **Kullanım alanları:** Altyapıyı kod olarak (IaC) yönetme; tekrarlanabilir ortamlar; yığın (stack) dağıtımı.\
  **Avantajlar:** Şablon tabanlı dağıtım; değişiklik önizleme; otomatik rollback.\
  **Sınavda:** Tekrarlanabilir altyapı dağıtımı ve Infrastructure as Code.

- **AWS Service Catalog:** Kuruluşların onaylanmış altyapı ürünlerini kullanıcılarına sunmasını sağlar.\
  **Kullanım alanları:** Onaylı hizmet portföyü sunma; kurumsal self-servis altyapı; standartlaştırılmış dağıtımlar.\
  **Avantajlar:** Kullanıcıya güvenli self-servis; merkezi politika kontrolü.\
  **Sınavda:** Kullanıcıların yalnızca kurumsal olarak onaylanan yapılandırmaları dağıtması.

- **AWS Organizations:** Birden fazla AWS hesabını merkezi yönetir; SCP (service control policy) ve konsolide faturalandırma gibi özellikler sunar.\
  **Kullanım alanları:** Çoklu hesap yönetimi; birleşik faturalandırma; servis kontrol politikaları (SCP).\
  **Avantajlar:** Merkezi hesap yönetimi; konsolide fatura ve indirim; otomatik hesap oluşturma.\
  **Sınavda:** Çoklu hesap yönetimi ve hesap seviyesinde izin sınırları.

- **AWS Control Tower:** Landing zone ve çoklu hesap ortamının kurulumunu ve yönetişimini kolaylaştırır.\
  **Kullanım alanları:** Çoklu hesaplı landing zone kurulumu; güvenlik temel çizgisi (guardrail); yeni hesap otomasyonu.\
  **Avantajlar:** Hazır en iyi uygulamalar; otomatik guardrail'ler; şablonlu hesap üretimi.\
  **Sınavda:** Yeni kurumsal AWS ortamı, merkezi guardrail ve hesap yaşam döngüsü.

- **AWS Trusted Advisor:** Desteklenen kontrol kategorilerinde maliyet, güvenlik, performans ve hata toleransı önerileri sunar.\
  **Kullanım alanları:** Maliyet optimizasyon önerileri; güvenlik açıkları; hizmet limiti kontrolleri.\
  **Avantajlar:** Gerçek zamanlı kontroller; kategorilere ayrılmış öneriler; maliyet tasarrufu ipuçları.\
  **Sınavda:** Kullanılmayan kaynakları veya önerilen iyileştirmeleri tespit etme.

- **AWS Compute Optimizer:** Desteklenen compute kaynakları için kullanım verisine dayalı boyutlandırma önerileri üretir.\
  **Kullanım alanları:** EC2/RDS/Lambda için doğru boyutlandırma; kullanılmayan kaynakları tespit; maliyet-performans dengesi.\
  **Avantajlar:** Makine öğrenmesi tabanlı öneriler; kullanım analiziyle instance seçimi; maliyet tasarrufu.\
  **Sınavda:** Fazla büyük veya yetersiz boyutlandırılmış EC2 ve diğer desteklenen kaynakları belirleme.

- **AWS Backup:** Desteklenen kaynaklar için merkezi yedekleme politikaları, saklama ve kurtarma yönetimi sunar.\
  **Kullanım alanları:** RDS, EBS, EFS, S3 ve DynamoDB için merkezi yedekleme; uyumluluk amaçlı saklama; çoklu hesap yedek yönetimi.\
  **Avantajlar:** Tek panelden politika yönetimi; otomatik saklama ve silmeye karşı koruma.\
  **Sınavda:** Birden fazla hesap veya serviste standart yedekleme stratejisi.

**Karıştırma:**
- CloudWatch = çalışma zamanı metrik ve logları; Config = kaynak konfigürasyonu ve uyumluluk geçmişi.
- CloudTrail = API denetim izi; X-Ray = dağıtık istek izleme.
- Organizations = hesap ve politika yönetimi; Control Tower = yönetilen landing zone ve guardrail yaklaşımı.
- CloudFormation = altyapıyı tanımlama; Systems Manager = çalışan kaynakları yönetme.

## 8. Analytics & Machine Learning - Analitik ve ML

- **Amazon Athena:** S3'teki veriyi sunucu yönetmeden SQL ile sorgular.\
  **Kullanım alanları:** S3 üzerinde SQL sorguları; log analizi; veri keşfi (ad-hoc sorgular).\
  **Avantajlar:** Sunucusuz; sorgu başı ödeme; standart SQL; Glue katalog entegrasyonu.\
  **Sınavda:** Data lake üzerinde seyrek veya ihtiyaca bağlı analitik sorgular.

- **AWS Glue:** Veri keşfi, Data Catalog ve ETL işlerini destekler.\
  **Kullanım alanları:** Veri keşfi ve kataloglama; ETL işleri; veri ambarı/gölü hazırlığı.\
  **Avantajlar:** Sunucusuz ETL; otomatik şema keşfi; Spark tabanlı dönüşüm.\
  **Sınavda:** S3 verisini dönüştürüp analize hazırlama veya şemasını kataloglama.

- **Amazon Managed Service for Apache Flink:** Sürekli gelen akış verisini gerçek zamanlı işler ve analiz eder.\
  **Kullanım alanları:** Akış verileri üzerinde gerçek zamanlı analitik ve zenginleştirme; SQL ile akış sorguları.\
  **Avantajlar:** Yönetilen altyapı; otomatik ölçekleme; kalıcı durum yönetimi.\
  **Sınavda:** Streaming verisi üzerinde sürekli hesaplama.

- **Amazon QuickSight:** Yönetilen BI ve dashboard hizmetidir.\
  **Kullanım alanları:** İş zekâsı panoları; görsel raporlama; uygulamalara gömülü analitik.\
  **Avantajlar:** Sunucusuz BI; SPICE motoru ile hızlı sorgu; çoklu veri kaynağı.\
  **Sınavda:** İş zekâsı raporları, görselleştirme ve dashboard.

- **Amazon EMR (Elastic MapReduce):** Hadoop, Spark ve desteklenen büyük veri çerçevelerini çalıştırır.\
  **Kullanım alanları:** Büyük veri işleme (Hadoop/Spark); veri dönüşümleri; ML pipeline'ları.\
  **Avantajlar:** Yönetilen big data kümeleri; Spot ile ucuz hesaplama; otomatik ölçekleme.\
  **Sınavda:** Büyük ölçekli dağıtık veri işleme.

- **Amazon SageMaker AI:** Makine öğrenmesi modellerinin geliştirilmesi, eğitilmesi ve dağıtılmasına yönelik araçlar sağlar.\
  **Kullanım alanları:** ML model geliştirme ve dağıtımı; model eğitimi ve ayarlama; uç (edge) dağıtımı.\
  **Avantajlar:** Uçtan uca ML platformu; yönetilen altyapı; hazır algoritmalar.\
  **Sınavda:** Özel ML modeli oluşturma ve üretime alma.

- **Amazon Rekognition:** Görüntü ve videolardaki nesne, yüz ve diğer desteklenen özellikleri analiz eder.\
  **Kullanım alanları:** Görüntü ve video analizi; nesne/yüz tespiti; içerik moderasyonu.\
  **Avantajlar:** Hazır bilgisayarlı görü modelleri; eğitim gerektirmez; video desteği.\
  **Sınavda:** Görsel içerik analizi.

- **Amazon Comprehend:** Metin üzerinde doğal dil işleme görevleri gerçekleştirir.\
  **Kullanım alanları:** Metin analizi; duygu analizi; varlık tespiti (NER).\
  **Avantajlar:** Doğal dil işleme; özel sınıflandırıcılar; çoklu dil desteği.\
  **Sınavda:** Duygu analizi, varlık çıkarımı ve metin sınıflandırma.

- **Amazon Translate:** Diller arasında metin çevirisi yapar.\
  **Kullanım alanları:** Uygulama içi çeviri; belge çevirisi; çok dilli içerik üretimi.\
  **Avantajlar:** Gerçek zamanlı çeviri; toplu çeviri; geniş dil desteği.\
  **Sınavda:** Uygulamalarda otomatik çok dilli içerik.

- **Amazon Polly:** Metni konuşmaya dönüştürür (text-to-speech).\
  **Kullanım alanları:** Metni sese çevirme (TTS); sesli asistanlar; erişilebilirlik uygulamaları.\
  **Avantajlar:** Gerçekçi sesler; SSML desteği; çoklu dil ve ses seçenekleri.\
  **Sınavda:** Metinden ses üretimi.

- **Amazon Transcribe:** Konuşmayı metne dönüştürür (speech-to-text).\
  **Kullanım alanları:** Konuşmayı metne çevirme; çağrı merkezi analizi; altyazı üretimi.\
  **Avantajlar:** Otomatik konuşma tanıma; özelleştirilebilir sözlük; çoklu dil.\
  **Sınavda:** Ses kayıtlarının yazıya çevrilmesi.

- **Amazon Textract:** Belge ve formlardan metin, tablo ve form alanlarını çıkarır.\
  **Kullanım alanları:** Belge verisi çıkarma; form ve tablo okuma; fatura/fiş işleme.\
  **Avantajlar:** OCR'dan öte yapılandırılmış veri; makine öğrenmesi tabanlı; hazır modeller.\
  **Sınavda:** Tarama veya PDF belgelerinden yapılandırılmış veri elde etme.

**Karıştırma:**
- Athena = S3 üzerinde SQL; Glue = katalog ve ETL.
- EMR = yönetilen büyük veri çerçeveleri; Redshift = analitik veri ambarı.
- Rekognition = görsel; Comprehend = metin; Transcribe = ses → metin; Polly = metin → ses; Textract = belge → yapılandırılmış veri.

## 9. Migration - Taşıma ve Geçiş

- **AWS Database Migration Service (DMS):** Veritabanları arasında veri taşımayı ve desteklenen senaryolarda değişiklikleri çoğaltmayı sağlar.\
  **Kullanım alanları:** Veritabanı göçü (homojen/heterojen); sürekli replikasyon; buluta geçiş.\
  **Avantajlar:** Minimum kesinti; kaynak çalışırken kopyalama; değişiklik veri yakalama (CDC).\
  **Sınavda:** Kaynak veritabanı çalışırken hedefe geçiş ve downtime'ı azaltma.

- **AWS Schema Conversion Tool (SCT):** Desteklenen heterojen veritabanı geçişlerinde şema ve kod dönüşümüne yardımcı olur.\
  **Kullanım alanları:** Motor değiştirmede şema dönüşümü (ör. Oracle→Aurora); uyumluluk raporu; kod dönüşümü.\
  **Avantajlar:** Otomatik şema dönüşümü; göç öncesi analiz; dönüşüm önerileri.\
  **Sınavda:** Oracle veya SQL Server'dan PostgreSQL gibi farklı motorlara geçiş.

- **AWS Migration Hub:** Birden fazla migrasyon projesinin ilerlemesini merkezi izlemeye yardımcı olur.\
  **Kullanım alanları:** Göç projelerini tek panelden izleme; envanter ve bağımlılık keşfi; ilerleme takibi.\
  **Avantajlar:** Merkezi görünürlük; çoklu araç entegrasyonu; strateji önerileri.\
  **Sınavda:** Büyük ölçekli sunucu ve uygulama taşıma süreçlerini takip etme.

- **AWS Application Migration Service (MGN):** Sunucuları AWS'ye lift-and-shift yaklaşımıyla taşımak için replikasyon ve geçiş süreçlerini yönetir.\
  **Kullanım alanları:** Sunucuları buluta taşıma (lift-and-shift); otomatik replikasyon; kesintisiz geçiş.\
  **Avantajlar:** Sürekli blok replikasyonu; test geçişleri; otomatik dönüştürme.\
  **Sınavda:** Uygulamayı büyük ölçüde yeniden tasarlamadan EC2'ye taşıma.

- **AWS DataSync:** Dosya verisini desteklenen on-premises ve AWS depolama sistemleri arasında taşır.\
  **Kullanım alanları:** On-premises veriyi S3/EFS/FSx'e düzenli taşıma; bulutlar arası kopyalama; göç öncesi sürekli senkronizasyon.\
  **Avantajlar:** Otomatik şifreleme ve veri doğrulama; ağ optimizasyonu; zamanlanmış otomatik transfer.\
  **Sınavda:** Büyük miktarda dosyayı verimli biçimde kopyalama.

- **AWS Transfer Family:** SFTP, FTPS, FTP ve AS2 gibi desteklenen protokollerle yönetilen dosya transferi sunar.\
  **Kullanım alanları:** SFTP/FTPS/FTP ile iş ortağı dosya transferi; mevcut transfer iş akışlarını S3/EFS'e bağlama.\
  **Avantajlar:** Yönetilen transfer uçları; kimlik entegrasyonu; sunucu yönetimi yok.\
  **Sınavda:** Müşterilerin veya iş ortaklarının mevcut dosya transfer protokolüyle S3/EFS'ye veri göndermesi.

- **AWS Snow Family:** Büyük veri setlerinin fiziksel cihazlarla taşınmasını destekler.\
  **Kullanım alanları:** TB-PB ölçeğinde veri taşıma; sınırlı bant genişliği olan lokasyonlar; uç (edge) işleme senaryoları.\
  **Avantajlar:** Fiziksel transferle ağ maliyeti ve süresi ortadan kalkar; şifreli ve dayanıklı cihazlar.\
  **Sınavda:** Ağ üzerinden transferin çok uzun sürmesi veya bağlantı kısıtları.

- **AWS Storage Gateway:** On-premises uygulamalarla AWS depolama hizmetleri arasında hibrit bağlantı sağlar.\
  **Kullanım alanları:** On-premises ortamla AWS depolama arasında hibrit köprü; bulut yedekleme; dosya/blok/tape uzantısı.\
  **Avantajlar:** Yerel erişim + bulut ekonomisi; standart protokoller (NFS/SMB/iSCSI); yönetilen servis.\
  **Sınavda:** Yerel sistemlerin bulut depolamasını entegre biçimde kullanması.

**Karıştırma:**
- DMS = veritabanı verisi; SCT = şema/kod dönüşümü.
- MGN = sunucu taşıma; DataSync = dosya taşıma.
- Transfer Family = SFTP/FTPS/FTP/AS2 gibi protokoller; Storage Gateway = hibrit depolama.

## 10. Cost Optimization - Maliyet Optimizasyonu

- **AWS Cost Explorer:** Geçmiş harcamaları, maliyet eğilimlerini ve tahminleri analiz eder.\
  **Kullanım alanları:** Maliyet ve kullanım analizi; gelecek dönem tahminleri; RI/SP kullanım ve kapsama raporları.\
  **Avantajlar:** Görsel raporlar; 12 aylık tahmin; özel görünümler ve filtreler.\
  **Sınavda:** Maliyet artışının kaynağını inceleme.

- **AWS Budgets:** Belirlenen bütçe veya kullanım eşikleri için uyarılar oluşturur; desteklenen senaryolarda aksiyonlarla entegre edilebilir.\
  **Kullanım alanları:** Bütçe eşikleri ve alarmlar; otomatik düzeltici eylemler (budget actions); planlanan maliyet takibi.\
  **Avantajlar:** Tahmin bazlı uyarılar; otomasyon eylemleri; ayrıntılı raporlama.\
  **Sınavda:** Harcama hedefini aşmadan önce bildirim alma.

- **Cost and Usage Report (CUR):** AWS kullanım ve maliyet verilerini ayrıntılı biçimde sunar.\
  **Kullanım alanları:** Kaynak seviyesinde fatura analizi; chargeback/showback; Athena ile sorgulama.\
  **Avantajlar:** En ayrıntılı fatura verisi; saatlik granülarite; QuickSight entegrasyonu.\
  **Sınavda:** Ayrıntılı fatura analizi ve Athena ile sorgulama.

- **AWS Pricing Calculator:** Mimari dağıtımdan önce tahmini maliyet hesaplamaya yardımcı olur.\
  **Kullanım alanları:** Mimari maliyet tahmini; satın alma öncesi hesaplama; senaryo karşılaştırma.\
  **Avantajlar:** Ücretsiz ve ayrıntılı tahmin; fiyat varyasyonlarını modelleme; paylaşılabilir raporlar.\
  **Sınavda:** Planlanan altyapının maliyetini önceden tahmin etme.

- **Savings Plans:** Belirli kullanım türleri için saatlik harcama taahhüdü karşılığında indirim sağlar.\
  **Kullanım alanları:** Sürekli iş yükleri için taahhütlü indirim; kullanım esnekliği gerektiren commit'ler.\
  **Avantajlar:** Yüksek indirim oranı; kullanımı kolay; Compute SP ile instance ailesi değişikliği serbest.\
  **Sınavda:** Uzun vadeli ve öngörülebilir compute kullanımını daha düşük maliyetle çalıştırma.

- **Reserved Instances (RI):** Uygun servislerde kullanım taahhüdü karşılığında indirim sağlar; bazı türlerde kapasite rezervasyonu da söz konusudur.\
  **Kullanım alanları:** Sabit iş yükleri için 1-3 yıllık taahhüt; kapasite rezervasyonu gereken durumlar.\
  **Avantajlar:** On-Demand'e göre büyük indirim; Standard/Convertible ve Scheduled seçenekleri.\
  **Sınavda:** Özellikle RDS veya belirli EC2 kullanım senaryolarında uzun vadeli maliyet optimizasyonu.

- **On-Demand Instances:** Uzun vadeli taahhüt olmadan kullandıkça ödeme modelidir.\
  **Kullanım alanları:** Kısa süreli veya tahmin edilemeyen iş yükleri; geliştirme/test; taahhütsüz kullanım.\
  **Avantajlar:** Taahhüt yok; anında kapasite; saatlik faturalandırma.\
  **Sınavda:** Kısa süreli veya belirsiz iş yükleri.

- **Spot Instances:** AWS'nin kullanılmayan EC2 kapasitesini indirimli sunar; kapasite geri alınabilir.\
  **Kullanım alanları:** Kesinti toleranslı işler (batch, CI/CD); büyük ölçekli hesaplama; maliyeti kritik iş yükleri.\
  **Avantajlar:** On-Demand'e göre çok yüksek indirim; ASG/Batch ile otomasyon; kesintiye dayanıklı işler için ideal.\
  **Sınavda:** Kesintiye dayanıklı batch, dağıtık işlem ve esnek iş yükleri.

- **S3 Lifecycle Policies:** Nesneleri zaman veya kurallara göre farklı depolama sınıflarına taşır ya da siler.\
  **Kullanım alanları:** Eski verileri ucuz depolama sınıflarına taşıma; süre sonunda silme; log/backup yönetimi.\
  **Avantajlar:** Otomatik sınıf geçişleri; maliyet optimizasyonu; kurallarla otomasyon.\
  **Sınavda:** Eski logları daha ucuz arşiv sınıfına taşıma.

- **S3 Intelligent-Tiering:** Erişim desenine göre nesneleri uygun erişim katmanlarında tutarak maliyet optimizasyonu sağlar.\
  **Kullanım alanları:** Erişim deseni belirsiz veriler; otomatik maliyet optimizasyonu; değişken iş yükleri.\
  **Avantajlar:** Otomatik sınıf geçişi; erişim ücreti yok; tahmin gerektirmez.\
  **Sınavda:** Erişim sıklığı önceden tahmin edilemiyorsa.

**Karıştırma:**
- Cost Explorer = maliyet analizi; Budgets = eşik ve uyarı; CUR = ayrıntılı maliyet verisi; Pricing Calculator = geleceğe yönelik tahmin.
- Spot = en ucuz seçeneklerden biri, fakat kesintiye dayanıklı iş yükleri için.
- Savings Plans/RI = taahhüt karşılığında indirim; On-Demand = taahhütsüz kullanım.
- S3 Lifecycle = kurallara göre geçiş; Intelligent-Tiering = erişim desenine göre otomatik katmanlama.

---

# 11. Sınavda Sık Karşılaşılan Senaryo → Servis Eşleştirmeleri

| Sorudaki gereksinim | Öncelikli servis / çözüm |
|---|---|
| S3'e yüklenen dosya otomatik işlensin | S3 Event Notification + Lambda |
| Uygulama bileşenleri birbirini beklemeden çalışsın | SQS |
| Bir olay birden fazla tüketiciye iletilsin | SNS veya EventBridge; olayın türüne göre |
| Birden fazla adımlı iş akışı, retry ve hata dalları olsun | Step Functions |
| Private EC2 internete çıkabilsin | NAT Gateway + uygun route |
| Private EC2, internet kullanmadan S3'e erişsin | S3 Gateway Endpoint |
| Birden fazla EC2 aynı Linux dosyalarını paylaşsın | EFS |
| EC2 için kalıcı block disk | EBS |
| Global kullanıcılar statik içeriğe hızlı erişsin | CloudFront |
| HTTP host/path bazlı yönlendirme | ALB |
| TCP/UDP yük dengeleme ve düşük seviyeli trafik | NLB |
| Çok sayıda VPC merkezi olarak bağlansın | Transit Gateway |
| İki VPC arasında basit özel bağlantı | VPC Peering |
| EC2, AWS API'lerine anahtar saklamadan erişsin | IAM Role |
| S3 içindeki hassas veriler bulunsun | Macie |
| Şüpheli AWS etkinliği tespit edilsin | GuardDuty |
| API çağrısının kim tarafından yapıldığı bulunsun | CloudTrail |
| Kaynak konfigürasyonu değişiklikleri takip edilsin | AWS Config |
| Birden fazla AWS hesabı merkezi yönetilsin | Organizations; kurulum ve guardrail için Control Tower |
| Büyük veriyi S3 üzerinde SQL ile sorgula | Athena |
| Veriyi keşfet, katalogla ve ETL uygula | Glue |
| SQL raporlama için analitik veri ambarı | Redshift |
| Veritabanı okuma yükü artırılsın | Read Replica; cache ihtiyacı varsa ElastiCache |
| Veritabanı arızasında otomatik failover | RDS Multi-AZ veya uygun Aurora yapılandırması |
| Veritabanı parolası otomatik döndürülsün | Secrets Manager |
| Çok büyük veriyi internet yerine fiziksel aktar | Snow Family |
| Dosya sunucularını AWS'ye verimli kopyala | DataSync |
| Mevcut sunucuyu minimum değişiklikle AWS'ye taşı | MGN |
| Harcamalar belirli eşiğe gelince uyarı al | AWS Budgets |
| Uygun, kesintiye dayanıklı işlem en düşük maliyetle çalışsın | Spot Instances |

---

# 12. Sınav Öncesi Son Kontrol

Soruyu cevaplamadan önce şu ayrımları kontrol et:

1. **Availability mi, scalability mi?** Arıza durumunda çalışmaya devam etmek ile artan trafiği karşılamak farklı gereksinimlerdir.
2. **Managed service mi, tam kontrol mü?** Yönetim yükünü azaltan çözüm çoğunlukla tercih edilir; fakat sorudaki özelleştirme gereksinimleri belirleyicidir.
3. **Block, file, object mı?** EBS, EFS/FSx ve S3 birbirinin yerine geçen depolama türleri değildir.
4. **Senkron mu, asenkron mu?** Üretici ve tüketiciyi ayırmak gerekiyorsa SQS; olay yayınlamak gerekiyorsa SNS veya EventBridge.
5. **Private erişim mi, internet erişimi mi?** VPC Endpoint, NAT Gateway ve IGW farklı ihtiyaçları karşılar.
6. **Veritabanı HA (high availability) mı, okuma ölçeklendirmesi mi?** Multi-AZ ve Read Replica aynı problem için tasarlanmamıştır.
7. **Güvenlikte hangi katman?** IAM yetkilendirme, KMS şifreleme anahtarı, Secrets Manager gizli bilgi, GuardDuty tehdit tespiti ve Inspector zafiyet değerlendirmesidir.
8. **En az operasyonel yük mü, en düşük maliyet mi?** Bunlar her zaman aynı çözüm değildir. Gereksinimlerin tamamını karşılayan seçeneği tercih et.
9. **Kesinti kabul ediliyor mu?** Spot Instances gibi kesintiye açık seçenekleri, kesintisiz çalışması gereken kritik işler için otomatik olarak seçme.
10. **Sorudaki kesin ifadeleri kontrol et.** “Automatically”, “without managing servers”, “near real-time”, “private connectivity”, “minimal operational overhead” ve “least cost” gibi ifadeler çözümün seçiminde belirleyicidir.

> **Son kural:** Servis isimlerini ezberlemek tek başına yeterli değildir. AWS SAA-C03 sorularında genellikle ihtiyacı en az operasyonel yükle, gerekli erişilebilirlik ve güvenlik düzeyini koruyarak karşılayan mimari aranır. Her soruda önce gereksinimi, sonra kısıtları, en son servisi belirle.
