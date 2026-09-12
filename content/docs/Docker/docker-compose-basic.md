---
title: "docker-compose-basic"
weight: 10
---
docker compose ne isimize yariyor dersek, birden fazla konteyneri tek bir yaml dosyasindan yonetmemize yariyor, tek tek docker run ile ugrasmiyoruz ve ayni zamanda restart sayesinde sistem acilinca uygulamayi otomatik ayakta tutabiliyoruz

### temel calisma mantigi

proje klasorunde docker-compose.yml diye bir dosya aciyoruz, icine servisleri, networkleri ve volumeleri yazip sonra compose bunlari otomatik kurup birbirine bagliyor ve dayyum! uygulamalar ayakta oluyor, istegimize gore state'ini ayarlayabiliyoruz

### ornek 1 - tier1 (docker-compose.yml)

asagidaki ornekte bir nginx web sunucusu ve bir postgres veritabani var, en cok kullandigimiz temel seyler icinde.

```yaml
services:
  web:
    image: nginx:latest
    container_name: my-web
    build: ./web
    ports:
      - "8080:80"
    restart: unless-stopped
    env_file:
      - ./web.env
    command: ["nginx", "-g", "daemon off;"]
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app_net

  db:
    image: postgres:15
    container_name: my-db
    restart: always
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secretpassword
    env_file:
      - ./db.env
    volumes:
      - db_data:/var/lib/postgresql/data
    networks:
      - app_net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin"]
      interval: 10s
      timeout: 5s
      retries: 5

networks:
  app_net:
    driver: bridge

volumes:
  db_data:
```

### tier1 variable anlamlari

- **`image`**: burada kullanacak oldugumuz uygulamanin build edilmis image'ini (docker hub, github, etc...) ve versiyonunu yaziyoruz
- **`ports`**: burada kullandigimiz image'in hangi port uzerinden calisacagini belirtiyoruz, 8080 host portu 80 de zaten nginx'in klasik http portu
- **`environment`**: burada developer tarafinda tanimlanan, uygulamamiz icin gerekli olan env varlari yazmak icin kullaniyoruz
- **`volumes`**: burada yaptigimiz islemlerin kaybolmamasi icin volume ayarliyoruz, konteyner silinse bile veri duruyor
- **`depends_on`**: burada hangi servisin once baslamasi gerektigini yaziyoruz, mesela web db'den once baslamasin diye kullaniyoruz
- **`build`**: burada hazir image kullanmak yerine kendi Dockerfile'imizdan build almak icin path yaziyoruz
- **`restart`**: burada konteyner kapanirsa ne olacagini yaziyoruz, mesela always yazarsak hep yeniden baslatiyor
- **`networks`**: burada konteynerlerin hangi network uzerinden konusacagini yaziyoruz, servisleri birbirine baglamak icin kullaniyoruz
- **`env_file`**: burada env varlari tek tek yazmak yerine .env dosyasi yolunu veriyoruz
- **`container_name`**: burada konteynere kendimiz isim veriyoruz, yazmazsak compose kendi ismini veriyor
- **`command`**: burada image'in default komutunu ezmek icin kendi calistiracagimiz komutu yaziyoruz
- **`healthcheck`**: burada konteynerin saglikli olup olmadigini anlamak icin kontrol yaziyoruz, hazir olmadan diger servis gecmesin diye kullaniyoruz

### ornek 2 - tier2 (docker-compose.yml)

bu ornekte tier2'deki daha az kullandigimiz ama isimize yarayan ayarlar var, yine ayni web + db uzerinden gidiyoruz.

```yaml
services:
  web:
    image: nginx:latest
    expose:
      - "80"
    pull_policy: always
    profiles: ["frontend"]
    working_dir: /usr/share/nginx/html
    entrypoint: ["/docker-entrypoint.sh"]
    networks:
      - app_net
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    deploy:
      resources:
        limits:
          cpus: "0.50"
          memory: 512M

  db:
    image: postgres:15
    user: "999:999"
    networks:
      - app_net
    stop_grace_period: 30s
    secrets:
      - db_password

networks:
  app_net:
    driver: bridge

secrets:
  db_password:
    file: ./db_password.txt
```

### tier2 variable anlamlari
- **`expose`**: burada portu sadece compose icindeki servislere aciyoruz, host makineye acmiyoruz, ports gibi disari vermiyor
- **`entrypoint`**: burada konteyner acilirken calisacak ana komutu ezmek icin yaziyoruz, command'dan daha oncelikli calisiyor
- **`working_dir`**: burada konteynerin icinde hangi klasorde calisacagimizi yaziyoruz
- **`user`**: burada konteynerin hangi kullanici ile calisacagini yaziyoruz, root yerine guvenlik icin kullaniyoruz
- **`deploy`**: burada resources limit ve replica yaziyoruz, mesela cpus memory ve kac tane calisacak diye belirtiyoruz
- **`logging`**: burada loglarin nasil tutulacagini yaziyoruz, mesela max-size verip loglarin sismesini engelliyoruz
- **`secrets`**: burada password gibi hassas seyleri env yerine secret olarak veriyoruz, daha guvenli oluyor
- **`profiles`**: burada hangi servisin ne zaman acilacagini yaziyoruz, mesela frontend yazarsak sadece --profile frontend deyince aciliyor
- **`pull_policy`**: burada image'in ne zaman cekilecegini yaziyoruz, mesela always yazarsak hep en gunceli cekiyor
- **`stop_grace_period`**: burada konteyner kapanirken ne kadar bekleyecegini yaziyoruz, db gibi seyler icin lazim oluyor

### temel komutlar

komutlari `docker-compose.yml` dosyasinin oldugu klasorde calistiriyoruz, yeni dockerlarda `docker-compose` degil `docker compose` diye kullaniyoruz.

- **`docker compose up`**: burada dosyada yazdigimiz tum servisleri olusturup baslatiyoruz, loglar ekrana akiyor
- **`docker compose up -d`**: burada servisleri arkada baslatiyoruz, terminal bize kaliyor
- **`docker compose down`**: burada calisan servisleri durdurup konteynerleri ve networkleri siliyoruz, volumeler duruyor
- **`docker compose down -v`**: burada servisleri durdurup volumeleri de siliyoruz, db gibi veriler gidiyor dikkat et
- **`docker compose ps`**: burada bu projeye ait konteynerlerin durumuna bakiyoruz
- **`docker compose logs`**: burada servislerin loglarini goruyoruz, canli izlemek icin `docker compose logs -f` yaziyoruz
- **`docker compose build`**: burada Dockerfile'dan image build ediyoruz, kod degisince yeniden build icin kullaniyoruz
- **`docker compose exec [servis_adi] [komut]`**: burada calisan konteynerin icine girip komut calistiriyoruz, mesela `docker compose exec db psql -U admin` gibi
