---
title: "var-olan-volume-uzerine-docker-compose"
weight: 11
---
# ilk olarak ben ornek olarak verecegim


## kullanmak istedigim uygulama (vault) kurulum yaparken admin kullanicinin yaninda rastgele encyprepted sifre olusuruyor ve compose dosyasi olusturuken 

```bash
docker compose up -d # -d flag arka planda calistir demek
```

> boyle yaptigimdan dolayi arkaplanda calisttirdigimdan dolayi `output` olarak yazdi ama ben gormedim direk aslinda 

```bash
docker compose up 
```
> bana sifreyi verecekti ama ben coktan is isten gecti :D sifreyide hashli olarak sakladigindan dosyara girsem bile fayda vermeyecekti 


```bash
docker run \
    -v filebrowser_data:/srv \
    -v filebrowser_database:/database \
    -v filebrowser_config:/config \
    -p 8080:80 \
    filebrowser/filebrowser
```

>`docker run ` ile volumlu bir sekilde olusturmustum ve direk bunu rundan volumeleri kullanarak nasil compose haline getiririm onu halledecegim  burada bana kullanici olarak *admin* parola olarakda *rastgele encyprepted parola* verdi bu parola aklimda ve volumde databasede duruyor

```bash
docker -ps -a
dokcer rm -f $(docker id)
```
>calisan docker uygulamami kapattim sildim sadece volume duruyor 

## en bastan yaml dosyami yaziyorum
```yaml
name: filebrowser

services:
  filebrowser:
    container_name: filebrowser
    image: filebrowser/filebrowser:latest
    ports:
      - "127.0.0.1:8000:80"
    volumes:
      - filebrowser_data:/srv
      - filebrowser_database:/database
      - filebrowser_config:/config
    restart: unless-stopped	  

volumes:
  filebrowser_data:
    external: true
  filebrowser_database:
    external: true
  filebrowser_config:
    external: true
```

> burada tekrardan olusturmasini engellemek icin `external` olarak volumeleri belirttim  artik hazir 

```bash
docker compose up -d
```


## basta normal ayarlayip compose up ile ciktiyi gorup yapabilirdim

```yaml
name: filebrowser

services:
  filebrowser:
    container_name: filebrowser
    image: filebrowser/filebrowser:latest
    ports:
      - "127.0.0.1:8000:80"
    volumes:
      - filebrowser_data:/srv
      - filebrowser_database:/database
      - filebrowser_config:/config 
    restart: unless-stopped	  

volumes:
  filebrowser_data:
  filebrowser_database:
  filebrowser_config:
```

> `restart: unless-stopped` burada ben durdurmadigim surece sistem kapanip acilsa bile tekrar calisir
