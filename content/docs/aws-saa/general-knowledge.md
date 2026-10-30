---
title: "general-knowledge"
weight: 100
---
# AWS SAA-C03 - General Knowledge: Tek Satırlık Özet ve Sınav Rehberi

> **Amaç:** Sınav öncesi hızlı tekrar. Her servisin ne yaptığını, hangi senaryoda seçileceğini ve benzer servislerden nasıl ayrıldığını hatırlatır. Ayrıntılı açıklamalar için ilgili konu dokümanlarına bak.

---

## 1. Compute - İşlem ve Uygulama Çalıştırma

- **EC2 (Elastic Compute Cloud):** AWS üzerinde sanal sunucu; işletim sistemi, instance tipi, disk ve ağ yapılandırması üzerinde kontrol sağlar. **Sınavda:** Özel işletim sistemi yapılandırması, uzun süre çalışan uygulama veya tam sunucu kontrolü isteniyorsa EC2.
- **Lambda:** Sunucu yönetmeden olay tetiklemeli kod çalıştırır; kullanım ve çalışma süresine göre ücretlendirilir. **Sınavda:** S3 (Simple Storage Service)'e dosya yüklenince işlem başlatma, SQS (Simple Queue Service) mesajı işleme veya kısa süreli API backend'i oluşturma.
- **ECS (Elastic Container Service):** AWS'nin container orkestrasyon servisidir; Docker container'larını görev ve servis olarak çalıştırır. **Sınavda:** Kubernetes karmaşıklığı olmadan AWS üzerinde container yönetmek.
- **EKS (Elastic Kubernetes Service):** AWS tarafından yönetilen Kubernetes control plane sunar; worker node'lar EC2 veya desteklenen iş yüklerinde Fargate üzerinde çalışabilir. **Sınavda:** Kubernetes API'si, mevcut Kubernetes araçları ve ekosistemi gerekiyorsa EKS.
- **Fargate:** ECS veya EKS container'ları için sunucu/worker node yönetme ihtiyacını azaltan serverless compute altyapısıdır. **Sınavda:** Container çalıştırılacak ancak EC2 instance'ları ve node kapasitesi yönetilmek istenmiyorsa Fargate.
- **Lightsail:** Basitleştirilmiş sanal sunucu, disk, ağ ve uygulama altyapısı sunar. **Sınavda:** Küçük web sitesi, blog veya kolay kurulabilen VPS benzeri ortam.
- **AWS Batch:** Batch işlerini kuyruklar, zamanlar ve uygun işlem kapasitesinde çalıştırır. **Sınavda:** Büyük veri setleri üzerinde toplu hesaplama, simülasyon veya çok sayıda bağımsız iş.
- **Elastic Beanstalk:** Uygulama dağıtımı, kapasite, yük dengeleme ve izleme gibi altyapı işlerini yönetir; EC2 tabanlı kaynaklara erişim imkânı da vardır. **Sınavda:** Uygulama kodunu dağıtmak, altyapı ayrıntılarını daha az yönetmek.
- **Auto Scaling:** EC2 instance sayısını talebe veya politikalara göre artırır ve azaltır. **Sınavda:** Trafik değişimlerine otomatik uyum ve kapasite maliyetini kontrol etme.
- **EC2 Placement Groups:** Instance'ların fiziksel yerleşimini kontrol ederek düşük gecikme, yüksek ağ performansı veya donanım arızalarına karşı ayrışma sağlar. **Sınavda:** Cluster = düşük gecikme; Spread = instance'ları ayrı donanımlara yayma; Partition = büyük dağıtık sistemlerde bölüm bazlı ayrışma.

**Karıştırma:**
- ECS = container orkestrasyonu; Fargate = container çalıştırma altyapısı.
- EKS = yönetilen Kubernetes; ECS = AWS'nin kendi container orkestrasyonu.
- Lambda = fonksiyon çalıştırma; Fargate = container çalıştırma.
- Auto Scaling = kapasiteyi dinamik değiştirme; ELB (Elastic Load Balancing) = gelen trafiği hedeflere dağıtma.

## 2. Storage - Depolama

- **S3:** Nesne depolama; dosyalar bucket içinde object olarak saklanır. Çok yüksek dayanıklılık, sürümleme, lifecycle ve farklı depolama sınıfları sunar. **Sınavda:** Yedekler, medya dosyaları, statik web içeriği, data lake ve arşiv.
- **S3 Storage Classes:** Veriye erişim sıklığı ve erişim süresine göre maliyeti optimize eder. **Sınavda:** Sık erişim = S3 Standard; öngörülebilir seyrek erişim = Standard-IA; erişim deseni belirsiz = Intelligent-Tiering; arşiv = Glacier sınıfları.
- **EBS (Elastic Block Store):** EC2'ye bağlanan block storage; işletim sistemi diski ve veritabanı diskleri için kullanılır. **Sınavda:** Kalıcı block disk, düşük gecikmeli disk I/O'su ve belirli performans gereksinimleri.
- **EBS Snapshots:** EBS volume verisinin point-in-time yedeğidir; artımlı snapshot mekanizması kullanır. **Sınavda:** Volume yedeği, başka bir AZ (Availability Zone) veya Region'a kopyalama ve kurtarma.
- **Instance Store:** Fiziksel sunucuya bağlı geçici block storage; kalıcı veri için uygun değildir. **Sınavda:** Geçici cache, buffer ve yeniden oluşturulabilir yüksek performanslı veriler.
- **EFS (Elastic File System):** Birden fazla Linux/Unix istemcisinin aynı dosya sistemine erişmesini sağlayan yönetilen NFS dosya depolamasıdır. **Sınavda:** Birden fazla EC2 instance'ının aynı dosyaları eş zamanlı okuması ve yazması.
- **FSx:** Windows File Server, Lustre, NetApp ONTAP ve OpenZFS gibi yönetilen dosya sistemleri sunar. **Sınavda:** Windows SMB/Active Directory, HPC için Lustre veya belirli kurumsal dosya sistemi özellikleri.
- **Storage Gateway:** On-premises ortam ile AWS depolama servisleri arasında hibrit entegrasyon sağlar. **Sınavda:** Yerel uygulamaların bulut depolamasına geçişini kolaylaştırma.
- **AWS Snow Family:** Büyük miktarda veriyi fiziksel cihazlarla taşıma ve uygun modellerde edge computing sağlama yaklaşımıdır. **Sınavda:** Sınırlı ağ bant genişliği, yüksek veri aktarım süresi veya bağlantı kısıtları.
- **AWS DataSync:** NFS, SMB ve desteklenen depolama sistemleri arasında veriyi otomatik ve verimli biçimde taşır. **Sınavda:** On-premises dosyaları S3 veya EFS'ye düzenli ya da toplu taşıma.
- **AWS Backup:** Desteklenen AWS kaynakları için merkezi yedekleme planları, politikalar ve saklama yönetimi sunar. **Sınavda:** Birden fazla AWS servisinin yedeklerini merkezi yönetmek.

**Karıştırma:**
- S3 = object; EBS = block; EFS = shared file.
- EBS genellikle bir AZ'ye bağlıdır; EFS birden fazla AZ'den erişilebilir şekilde yapılandırılabilir.
- Instance Store geçicidir; EBS kalıcı disk olarak kullanılabilir.
- DataSync = veri taşıma; Storage Gateway = hibrit depolama entegrasyonu; Snow Family = fiziksel veri aktarımı.

## 3. Database - Veritabanları

- **RDS (Relational Database Service):** MySQL, PostgreSQL, MariaDB, Oracle ve SQL Server gibi ilişkisel veritabanlarını yönetilen hizmet olarak çalıştırır. **Sınavda:** SQL, ilişkisel veri, otomatik yedekleme ve yönetilen veritabanı.
- **RDS Multi-AZ:** Veritabanı altyapısının farklı AZ'lerde yüksek erişilebilirlik için yedeklenmesini sağlar. **Sınavda:** Veritabanı instance'ı veya AZ arızasında otomatik failover.
- **RDS Read Replica:** Okuma sorgularını ana veritabanından ayırmak için asenkron replika oluşturur. **Sınavda:** Okuma ağırlıklı iş yüklerinde performans artırma.
- **Amazon Aurora:** AWS tarafından tasarlanmış, MySQL ve PostgreSQL uyumlu ilişkisel veritabanıdır. Ayrıştırılmış, dağıtık depolama mimarisi ve yönetilen failover özellikleri sunar. **Sınavda:** Yüksek performans, yüksek erişilebilirlik ve ilişkisel veri.
- **Aurora Serverless:** Aurora kapasitesini desteklenen yapılandırmalarda talebe göre otomatik ölçekler. **Sınavda:** Değişken veya öngörülemeyen veritabanı yükü.
- **DynamoDB:** Yönetilen, dağıtık NoSQL key-value ve document veritabanıdır. **Sınavda:** Çok yüksek ölçek, düşük gecikmeli anahtar erişimi ve sunucu yönetimi olmadan kapasite yönetimi.
- **DynamoDB DAX (DynamoDB Accelerator):** DynamoDB okumaları için bellek içi cache katmanıdır. **Sınavda:** Sık okunan verilerde mikro saniye düzeyinde okuma gecikmesi hedefi.
- **ElastiCache:** Redis OSS veya Memcached tabanlı yönetilen cache çözümleri sunar. **Sınavda:** Veritabanı üzerindeki okuma yükünü azaltma ve uygulama gecikmesini düşürme.
- **Redshift:** Analitik sorgular ve büyük ölçekli veri ambarı iş yükleri için kullanılır. **Sınavda:** Büyük veri üzerinde SQL raporlama, BI ve analitik.
- **Neptune:** İlişkileri ve bağlantıları grafik modeliyle tutar. **Sınavda:** Sosyal ağ ilişkileri, fraud bağlantıları ve graph traversal.
- **DocumentDB:** MongoDB ile uyumlu API'ler sunan yönetilen document database hizmetidir. **Sınavda:** Belge tabanlı veri modeli ve MongoDB uyumlu uygulama gereksinimleri.
- **OpenSearch Service:** Tam metin arama, log analizi ve arama odaklı analiz sağlar. **Sınavda:** Uygulama arama kutusu, log arama ve indekslenmiş metin sorguları.
- **Timestream:** Zaman damgalı veriler ve zaman serisi analizi için tasarlanmıştır. **Sınavda:** IoT telemetrisi, sensör verileri ve zaman serisi metrikleri.
- **QLDB (Quantum Ledger Database):** Değişiklik geçmişini kriptografik olarak doğrulanabilir bir ledger yapısında tutmak için tasarlanmıştı. **Sınavda:** Eski kaynaklarda karşılaşabilirsin; AWS, QLDB desteğini 31 Temmuz 2025'te sona erdirdiğinden güncel mimariler için aktif bir seçenek olarak değerlendirme.

**Karıştırma:**
- Multi-AZ = yüksek erişilebilirlik ve failover; Read Replica = okuma kapasitesini artırma.
- RDS = ilişkisel veritabanı; DynamoDB = NoSQL; Redshift = analitik veri ambarı.
- ElastiCache = uygulama cache'i; DAX = DynamoDB'ye özel cache.
- OpenSearch = arama ve indeksleme; Neptune = grafik ilişkileri.

## 4. Networking & CDN - Ağ ve Trafik Yönetimi

- **VPC (Virtual Private Cloud):** AWS kaynaklarının IP aralıklarını, subnet'lerini, route table'larını ve ağ güvenliğini tanımlayan izole sanal ağdır. **Sınavda:** Kaynakları özel ağda çalıştırma ve ağ erişimini kontrol etme.
- **Subnet:** VPC içindeki belirli bir Availability Zone'a ait IP aralığıdır. **Sınavda:** Kaynakları AZ'lere ve public/private ağ katmanlarına ayırma.
- **Internet Gateway (IGW):** VPC ile internet arasında internet yönlendirmesini sağlar. **Sınavda:** Public subnet'teki kaynağın internete erişmesi; bunun için route, public IP ve güvenlik kuralları da gerekir.
- **NAT Gateway:** Private subnet'teki IPv4 kaynaklarının internete veya dış ağlara bağlantı başlatmasını sağlar; internetten doğrudan başlatılan bağlantıları kabul etmez. **Sınavda:** Private EC2'nin yazılım güncellemesi indirmesi.
- **Security Group:** ENI (Elastic Network Interface)/instance düzeyinde stateful sanal güvenlik duvarıdır. **Sınavda:** Yalnızca belirli kaynaklardan belirli portlara izin verme.
- **Network ACL (NACL):** Subnet düzeyinde stateless filtreleme yapar; inbound ve outbound kuralları ayrı değerlendirilir. **Sınavda:** Subnet düzeyinde ek filtreleme veya belirli IP aralıklarını engelleme.
- **Route Table:** Hedef IP aralığına göre trafiğin IGW, NAT, peering veya başka bir hedefe yönlendirilmesini belirler. **Sınavda:** Public/private subnet erişim sorunları ve ağ yolları.
- **Route 53:** DNS, health check ve routing policy hizmetidir. **Sınavda:** DNS yönlendirme, failover, latency-based veya geolocation routing.
- **CloudFront:** İçeriği edge location'larda cache ederek kullanıcılara daha düşük gecikmeyle ulaştıran CDN'dir. **Sınavda:** Global kullanıcılar, statik içerik hızlandırma ve origin yükünü azaltma.
- **Elastic Load Balancing (ELB):** Trafiği sağlıklı hedeflere dağıtır. **Sınavda:** ALB (Application Load Balancer) = HTTP/HTTPS ve katman 7 yönlendirme; NLB (Network Load Balancer) = katman 4, TCP/UDP ve yüksek performans; Gateway Load Balancer = sanal ağ güvenlik cihazları.
- **AWS Global Accelerator:** Anycast statik IP'ler ve AWS global ağı üzerinden trafiği uygun bölgesel endpoint'lere yönlendirir. **Sınavda:** Global uygulamalar, hızlı failover ve TCP/UDP trafiğinin AWS ağı üzerinden yönlendirilmesi.
- **AWS Direct Connect:** On-premises veri merkezi ile AWS arasında özel ağ bağlantısı sağlar. **Sınavda:** Daha tutarlı ağ performansı, yüksek bant genişliği ve özel bağlantı ihtiyacı.
- **Site-to-Site VPN:** On-premises ağ ile AWS arasında internet üzerinden IPsec tünelleri oluşturur. **Sınavda:** Hızlı hibrit bağlantı veya Direct Connect'e yedek bağlantı.
- **VPC Peering:** İki VPC arasında doğrudan özel IP yönlendirmesi sağlar; transit yönlendirme desteklemez. **Sınavda:** Az sayıda VPC'yi birbirine bağlama.
- **Transit Gateway:** Çok sayıda VPC ve on-premises ağı merkezi hub üzerinden bağlar. **Sınavda:** Çoklu hesap ve VPC'lerde merkezi ağ topolojisi.
- **VPC Endpoints:** AWS servislerine özel ağ üzerinden erişim sağlar. **Sınavda:** Private subnet'teki EC2'nin internet/NAT kullanmadan S3 veya DynamoDB'ye erişmesi için Gateway Endpoint; birçok diğer servis için uygun Interface Endpoint.
- **AWS PrivateLink:** Bir servise VPC endpoint üzerinden özel erişim sağlar; servis sağlayıcı ve tüketici ağlarını doğrudan birbirine açmadan bağlantı kurabilir. **Sınavda:** Başka bir VPC'deki veya üçüncü taraf sağlayıcıdaki servise özel erişim.

**Karıştırma:**
- Public subnet olmak tek başına internete erişim sağlamaz: IGW route'u ve uygun public IP gerekir.
- NAT Gateway = private kaynakların outbound IPv4 internet erişimi; IGW = VPC'nin internet bağlantısı.
- Security Group = stateful; NACL = stateless.
- VPC Peering = transit yok; Transit Gateway = merkezi transit yönlendirme.
- CloudFront = içerik cache/CDN; Global Accelerator = global ağ yönlendirme.
- Route 53 = DNS; ELB = uygulama trafiğini hedeflere dağıtma.

## 5. Integration & Messaging - Mesajlaşma ve İş Akışları

- **SQS:** Dağıtık uygulamalar arasında mesajları kuyrukta tutarak üretici ve tüketiciyi birbirinden ayırır. **Sınavda:** Asenkron işleme, yük patlamalarını tamponlama ve başarısız mesajları DLQ (dead-letter queue)'ya yönlendirme.
- **SQS FIFO:** Mesaj sıralamasını ve uygun kullanımda deduplication desteğini sağlar. **Sınavda:** Sıralama veya mesajların tekilleştirilmesi gerektiğinde FIFO; aksi durumda Standard daha yüksek esneklik ve ölçek sunar.
- **SNS (Simple Notification Service):** Bir mesajı birden fazla aboneye yayınlayan pub/sub hizmetidir. **Sınavda:** Tek olayın birden fazla SQS kuyruğuna, Lambda fonksiyonuna veya başka desteklenen abonelere iletilmesi.
- **EventBridge:** Servislerden gelen olayları kurallara göre hedeflere yönlendirir. **Sınavda:** Bir AWS kaynağındaki olay gerçekleşince Lambda, Step Functions veya başka bir hedefi tetikleme.
- **Step Functions:** Birden fazla adımı, dallanmayı, tekrar denemeyi ve hata yönetimini state machine ile orkestre eder. **Sınavda:** Birden fazla servisten oluşan iş akışı ve kontrollü hata yönetimi.
- **Kinesis Data Streams:** Gerçek zamanlı veri akışlarını toplar ve tüketicilere sunar. **Sınavda:** Clickstream, IoT veya sürekli gelen logların düşük gecikmeyle işlenmesi.
- **Amazon Data Firehose:** Akış verisini desteklenen hedeflere teslim eder; tamponlama ve dönüşüm seçenekleri sunar. **Sınavda:** Streaming verisini S3, Redshift veya desteklenen hedeflere yönetilen biçimde aktarma.
- **Managed Service for Apache Flink:** Akış verisi üzerinde durum bilgili gerçek zamanlı analiz çalıştırır. **Sınavda:** Gelen olaylar üzerinde sürekli hesaplama ve pencere bazlı analiz. Eski kaynaklarda Kinesis Data Analytics adıyla geçebilir.
- **API Gateway:** Yönetilen API endpoint'leri oluşturur; kimlik doğrulama, throttling ve entegrasyon özellikleri sunar. **Sınavda:** Lambda veya backend servisleri için yönetilen HTTP API katmanı.
- **AWS AppSync:** GraphQL API'leri ve desteklenen veri kaynaklarıyla entegrasyon sağlar. **Sınavda:** İstemcinin ihtiyacı olan alanları tek GraphQL API üzerinden sorgulaması.
- **Amazon MQ:** ActiveMQ veya RabbitMQ gibi mesajlaşma sistemleriyle uyumluluk sağlar. **Sınavda:** Mevcut uygulamanın geleneksel broker protokollerini ve özelliklerini koruması gerekiyorsa.

**Karıştırma:**
- SQS = mesajı kuyrukta tutar; SNS = mesajı abonelere yayınlar.
- EventBridge = olay bazlı yönlendirme; SQS = asenkron kuyruk.
- Step Functions = iş akışının adımlarını yönetir; EventBridge = olayları kurallara göre yönlendirir.
- Kinesis Data Streams = akışı toplama ve tüketme; Firehose = akışı hedefe teslim etme.
- API Gateway = API katmanı; AppSync = GraphQL API.

## 6. Security & Identity - Güvenlik ve Kimlik

- **IAM (Identity and Access Management):** AWS kullanıcılarının, rollerinin ve servislerin hangi işlemleri yapabileceğini policy'lerle belirler. **Sınavda:** Least privilege, IAM role, cross-account access ve servisler arası yetkilendirme.
- **IAM Role:** Uzun ömürlü erişim anahtarları yerine geçici kimlik bilgileriyle AWS kaynaklarına erişim sağlar. **Sınavda:** EC2'nin S3'e veya Lambda'nın DynamoDB'ye güvenli erişmesi.
- **AWS KMS (Key Management Service):** Şifreleme anahtarlarını ve bunların kullanım izinlerini yönetir. **Sınavda:** S3, EBS, RDS veya uygulama verisini şifreleme ve anahtar erişimini kontrol etme.
- **AWS Secrets Manager:** Veritabanı parolaları, API anahtarları ve diğer gizli bilgileri saklar; desteklenen entegrasyonlarda rotasyonu yönetebilir. **Sınavda:** Veritabanı parolasını güvenli saklama ve otomatik döndürme.
- **SSM (Systems Manager) Parameter Store:** Konfigürasyon ve parametreleri merkezi saklar; SecureString ile KMS destekli şifreleme yapabilir. **Sınavda:** Uygulama ayarları ve basit parametre yönetimi.
- **Amazon Cognito:** Uygulama kullanıcılarının kayıt, giriş ve token tabanlı kimlik doğrulama ihtiyaçlarını karşılar. **Sınavda:** Web veya mobil uygulama kullanıcılarını AWS IAM kullanıcılarından ayrı yönetme.
- **AWS WAF (Web Application Firewall):** HTTP/HTTPS isteklerini web ACL ve kurallarla filtreler. **Sınavda:** SQL injection, XSS, IP tabanlı filtreleme ve istek hız sınırlandırma.
- **AWS Shield:** DDoS saldırılarına karşı koruma sağlar; Standard temel korumayı otomatik sunar, Advanced ek özellikler sağlar. **Sınavda:** DDoS riskini azaltma.
- **Amazon GuardDuty:** CloudTrail, VPC Flow Logs ve diğer desteklenen veri kaynaklarından tehdit göstergeleri çıkarır. **Sınavda:** Şüpheli API çağrıları, olası hesap ihlali ve kötü amaçlı ağ etkinliği.
- **Amazon Inspector:** Desteklenen kaynaklarda yazılım zafiyetlerini ve belirli güvenlik risklerini değerlendirir. **Sınavda:** EC2, container image ve desteklenen compute kaynaklarında vulnerability management.
- **Amazon Macie:** S3'teki hassas verileri, makine öğrenmesi ve eşleştirme yöntemleriyle keşfetmeye yardımcı olur. **Sınavda:** S3 içindeki PII ve benzeri hassas bilgileri tespit etme.
- **AWS Security Hub:** Birden fazla güvenlik kaynağından gelen bulguları merkezileştirir ve güvenlik duruşunu değerlendirmeye yardımcı olur. **Sınavda:** Birden fazla güvenlik hizmetinin bulgularını tek yerde toplama.
- **AWS CloudTrail:** AWS hesap etkinliğini ve API çağrılarını kaydeder. **Sınavda:** Kim, ne zaman, hangi API işlemini yaptı sorusunun cevabı.
- **AWS Artifact:** AWS uyumluluk raporlarına ve ilgili anlaşma belgelerine erişim sağlar. **Sınavda:** AWS'nin SOC, ISO ve PCI gibi uyumluluk belgelerini edinme.
- **AWS Certificate Manager (ACM):** Desteklenen AWS servislerinde TLS sertifikalarının sağlanması ve yenilenmesini yönetir. **Sınavda:** ALB veya CloudFront üzerinde HTTPS sertifikası kullanma.
- **AWS Directory Service:** Microsoft Active Directory ile ilgili yönetilen dizin ihtiyaçlarını karşılar. **Sınavda:** Windows iş yükleri, domain join ve Active Directory entegrasyonu.

**Karıştırma:**
- IAM = AWS kaynaklarına erişim yetkisi; Cognito = uygulama kullanıcı kimliği.
- KMS = anahtar yönetimi; Secrets Manager = gizli bilgilerin saklanması ve rotasyonu.
- CloudTrail = API denetim kaydı; CloudWatch = metrik, log ve alarm.
- GuardDuty = tehdit tespiti; Inspector = zafiyet taraması; Macie = S3 hassas veri keşfi.
- WAF = web istek filtreleme; Shield = DDoS koruması; Security Hub = güvenlik bulgularını birleştirme.

## 7. Management & Monitoring - Yönetim ve İzleme

- **Amazon CloudWatch:** Metrik, log, alarm ve gözlemlenebilirlik özellikleri sağlar. **Sınavda:** CPU kullanımı yükselince alarm oluşturma veya belirli bir metriğe göre aksiyon başlatma.
- **CloudWatch Logs:** Uygulama ve servis loglarını toplar, saklar ve sorgulamaya yardımcı olur. **Sınavda:** Hata kayıtlarını inceleme, log grupları ve log tabanlı metrikler.
- **AWS X-Ray:** Dağıtık uygulamalarda isteklerin servisler arasındaki izini sürer. **Sınavda:** Mikroservislerde gecikmenin hangi servisten kaynaklandığını bulma.
- **AWS Config:** Kaynak konfigürasyonlarını kaydeder ve kurallarla uyumluluk değerlendirmesi yapar. **Sınavda:** Security Group'un zaman içinde değişimini bulma veya kurallara aykırı kaynakları tespit etme.
- **AWS Systems Manager (SSM):** Desteklenen sunuculara uzaktan erişim, patch yönetimi, envanter ve otomasyon sağlar. **Sınavda:** SSH/RDP açmadan yönetim için Session Manager veya merkezi patch operasyonları.
- **AWS CloudFormation:** AWS kaynaklarını şablonlarla tanımlayıp yönetir. **Sınavda:** Tekrarlanabilir altyapı dağıtımı ve Infrastructure as Code.
- **AWS Service Catalog:** Kuruluşların onaylanmış altyapı ürünlerini kullanıcılarına sunmasını sağlar. **Sınavda:** Kullanıcıların yalnızca kurumsal olarak onaylanan yapılandırmaları dağıtması.
- **AWS Organizations:** Birden fazla AWS hesabını merkezi yönetir; SCP (service control policy) ve konsolide faturalandırma gibi özellikler sunar. **Sınavda:** Çoklu hesap yönetimi ve hesap seviyesinde izin sınırları.
- **AWS Control Tower:** Landing zone ve çoklu hesap ortamının kurulumunu ve yönetişimini kolaylaştırır. **Sınavda:** Yeni kurumsal AWS ortamı, merkezi guardrail ve hesap yaşam döngüsü.
- **AWS Trusted Advisor:** Desteklenen kontrol kategorilerinde maliyet, güvenlik, performans ve hata toleransı önerileri sunar. **Sınavda:** Kullanılmayan kaynakları veya önerilen iyileştirmeleri tespit etme.
- **AWS Compute Optimizer:** Desteklenen compute kaynakları için kullanım verisine dayalı boyutlandırma önerileri üretir. **Sınavda:** Fazla büyük veya yetersiz boyutlandırılmış EC2 ve diğer desteklenen kaynakları belirleme.
- **AWS Backup:** Desteklenen kaynaklar için merkezi yedekleme politikaları, saklama ve kurtarma yönetimi sunar. **Sınavda:** Birden fazla hesap veya serviste standart yedekleme stratejisi.

**Karıştırma:**
- CloudWatch = çalışma zamanı metrik ve logları; Config = kaynak konfigürasyonu ve uyumluluk geçmişi.
- CloudTrail = API denetim izi; X-Ray = dağıtık istek izleme.
- Organizations = hesap ve politika yönetimi; Control Tower = yönetilen landing zone ve guardrail yaklaşımı.
- CloudFormation = altyapıyı tanımlama; Systems Manager = çalışan kaynakları yönetme.

## 8. Analytics & Machine Learning - Analitik ve ML

- **Amazon Athena:** S3'teki veriyi sunucu yönetmeden SQL ile sorgular. **Sınavda:** Data lake üzerinde seyrek veya ihtiyaca bağlı analitik sorgular.
- **AWS Glue:** Veri keşfi, Data Catalog ve ETL işlerini destekler. **Sınavda:** S3 verisini dönüştürüp analize hazırlama veya şemasını kataloglama.
- **Amazon Managed Service for Apache Flink:** Sürekli gelen akış verisini gerçek zamanlı işler ve analiz eder. **Sınavda:** Streaming verisi üzerinde sürekli hesaplama.
- **Amazon QuickSight:** Yönetilen BI ve dashboard hizmetidir. **Sınavda:** İş zekâsı raporları, görselleştirme ve dashboard.
- **Amazon EMR (Elastic MapReduce):** Hadoop, Spark ve desteklenen büyük veri çerçevelerini çalıştırır. **Sınavda:** Büyük ölçekli dağıtık veri işleme.
- **Amazon SageMaker AI:** Makine öğrenmesi modellerinin geliştirilmesi, eğitilmesi ve dağıtılmasına yönelik araçlar sağlar. **Sınavda:** Özel ML modeli oluşturma ve üretime alma.
- **Amazon Rekognition:** Görüntü ve videolardaki nesne, yüz ve diğer desteklenen özellikleri analiz eder. **Sınavda:** Görsel içerik analizi.
- **Amazon Comprehend:** Metin üzerinde doğal dil işleme görevleri gerçekleştirir. **Sınavda:** Duygu analizi, varlık çıkarımı ve metin sınıflandırma.
- **Amazon Translate:** Diller arasında metin çevirisi yapar. **Sınavda:** Uygulamalarda otomatik çok dilli içerik.
- **Amazon Polly:** Metni konuşmaya dönüştürür (text-to-speech). **Sınavda:** Metinden ses üretimi.
- **Amazon Transcribe:** Konuşmayı metne dönüştürür (speech-to-text). **Sınavda:** Ses kayıtlarının yazıya çevrilmesi.
- **Amazon Textract:** Belge ve formlardan metin, tablo ve form alanlarını çıkarır. **Sınavda:** Tarama veya PDF belgelerinden yapılandırılmış veri elde etme.

**Karıştırma:**
- Athena = S3 üzerinde SQL; Glue = katalog ve ETL.
- EMR = yönetilen büyük veri çerçeveleri; Redshift = analitik veri ambarı.
- Rekognition = görsel; Comprehend = metin; Transcribe = ses → metin; Polly = metin → ses; Textract = belge → yapılandırılmış veri.

## 9. Migration - Taşıma ve Geçiş

- **AWS Database Migration Service (DMS):** Veritabanları arasında veri taşımayı ve desteklenen senaryolarda değişiklikleri çoğaltmayı sağlar. **Sınavda:** Kaynak veritabanı çalışırken hedefe geçiş ve downtime'ı azaltma.
- **AWS Schema Conversion Tool (SCT):** Desteklenen heterojen veritabanı geçişlerinde şema ve kod dönüşümüne yardımcı olur. **Sınavda:** Oracle veya SQL Server'dan PostgreSQL gibi farklı motorlara geçiş.
- **AWS Migration Hub:** Birden fazla migrasyon projesinin ilerlemesini merkezi izlemeye yardımcı olur. **Sınavda:** Büyük ölçekli sunucu ve uygulama taşıma süreçlerini takip etme.
- **AWS Application Migration Service (MGN):** Sunucuları AWS'ye lift-and-shift yaklaşımıyla taşımak için replikasyon ve geçiş süreçlerini yönetir. **Sınavda:** Uygulamayı büyük ölçüde yeniden tasarlamadan EC2'ye taşıma.
- **AWS DataSync:** Dosya verisini desteklenen on-premises ve AWS depolama sistemleri arasında taşır. **Sınavda:** Büyük miktarda dosyayı verimli biçimde kopyalama.
- **AWS Transfer Family:** SFTP, FTPS, FTP ve AS2 gibi desteklenen protokollerle yönetilen dosya transferi sunar. **Sınavda:** Müşterilerin veya iş ortaklarının mevcut dosya transfer protokolüyle S3/EFS'ye veri göndermesi.
- **AWS Snow Family:** Büyük veri setlerinin fiziksel cihazlarla taşınmasını destekler. **Sınavda:** Ağ üzerinden transferin çok uzun sürmesi veya bağlantı kısıtları.
- **AWS Storage Gateway:** On-premises uygulamalarla AWS depolama hizmetleri arasında hibrit bağlantı sağlar. **Sınavda:** Yerel sistemlerin bulut depolamasını entegre biçimde kullanması.

**Karıştırma:**
- DMS = veritabanı verisi; SCT = şema/kod dönüşümü.
- MGN = sunucu taşıma; DataSync = dosya taşıma.
- Transfer Family = SFTP/FTPS/FTP/AS2 gibi protokoller; Storage Gateway = hibrit depolama.

## 10. Cost Optimization - Maliyet Optimizasyonu

- **AWS Cost Explorer:** Geçmiş harcamaları, maliyet eğilimlerini ve tahminleri analiz eder. **Sınavda:** Maliyet artışının kaynağını inceleme.
- **AWS Budgets:** Belirlenen bütçe veya kullanım eşikleri için uyarılar oluşturur; desteklenen senaryolarda aksiyonlarla entegre edilebilir. **Sınavda:** Harcama hedefini aşmadan önce bildirim alma.
- **Cost and Usage Report (CUR):** AWS kullanım ve maliyet verilerini ayrıntılı biçimde sunar. **Sınavda:** Ayrıntılı fatura analizi ve Athena ile sorgulama.
- **AWS Pricing Calculator:** Mimari dağıtımdan önce tahmini maliyet hesaplamaya yardımcı olur. **Sınavda:** Planlanan altyapının maliyetini önceden tahmin etme.
- **Savings Plans:** Belirli kullanım türleri için saatlik harcama taahhüdü karşılığında indirim sağlar. **Sınavda:** Uzun vadeli ve öngörülebilir compute kullanımını daha düşük maliyetle çalıştırma.
- **Reserved Instances (RI):** Uygun servislerde kullanım taahhüdü karşılığında indirim sağlar; bazı türlerde kapasite rezervasyonu da söz konusudur. **Sınavda:** Özellikle RDS veya belirli EC2 kullanım senaryolarında uzun vadeli maliyet optimizasyonu.
- **On-Demand Instances:** Uzun vadeli taahhüt olmadan kullandıkça ödeme modelidir. **Sınavda:** Kısa süreli veya belirsiz iş yükleri.
- **Spot Instances:** AWS'nin kullanılmayan EC2 kapasitesini indirimli sunar; kapasite geri alınabilir. **Sınavda:** Kesintiye dayanıklı batch, dağıtık işlem ve esnek iş yükleri.
- **S3 Lifecycle Policies:** Nesneleri zaman veya kurallara göre farklı depolama sınıflarına taşır ya da siler. **Sınavda:** Eski logları daha ucuz arşiv sınıfına taşıma.
- **S3 Intelligent-Tiering:** Erişim desenine göre nesneleri uygun erişim katmanlarında tutarak maliyet optimizasyonu sağlar. **Sınavda:** Erişim sıklığı önceden tahmin edilemiyorsa.

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
