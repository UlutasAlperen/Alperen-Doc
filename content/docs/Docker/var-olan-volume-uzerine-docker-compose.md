---
title: "var-olan-volume-uzerine-docker-compose"
weight: 11
---
# İlk olarak ben örnek olarak vereceğim


## kullanmak istediğim uygulama (filebrowser) kurulum yaparken admin kullanıcının yanında rastgele encrypted şifre oluşturuyor ve compose dosyası oluştururken

```bash
docker compose up -d # -d flag arka planda çalıştır demek
```

> böyle yaptığımdan dolayı arka planda çalıştırdığımdan dolayı `output` olarak yazdı ama ben görmedim direkt aslında

```bash
docker compose up
```
> bana şifreyi verecekti ama ben çoktan iş işten geçti :D şifreyi de hash'li olarak sakladığından dosyalara girsem bile fayda vermeyecekti


```bash
docker run \
    --name filebrowser \
    -v filebrowser_data:/srv \
    -v filebrowser_database:/database \
    -v filebrowser_config:/config \
    -p 127.0.0.1:8000:80 \
    filebrowser/filebrowser
```

>`docker run ` ile volume'lü bir şekilde oluşturmuştum ve direkt bunu run'dan volume'leri kullanarak nasıl compose haline getiririm onu halledeceğim  burada bana kullanıcı olarak *admin* parola olarak da *rastgele encrypted parola* verdi bu parola aklımda ve volume'de database'de duruyor

```bash
docker ps -a
docker rm -f filebrowser
```
> çalışan docker uygulamamı kapattım sildim sadece volume duruyor

## en baştan yaml dosyamı yazıyorum
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

> burada tekrardan oluşturmasını engellemek için `external` olarak volume'leri belirttim  artık hazır

```bash
docker compose up -d
```


## başta normal ayarlayıp compose up ile çıktıyı görüp yapabilirdim

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

> `restart: unless-stopped` burada ben durdurmadığım sürece sistem kapanıp açılsa bile tekrar çalışır

> Not: bu ikinci dosya sıfırdan kurulum içindir, `external` olmadığı için compose `filebrowser_filebrowser_data` gibi prefix'li yeni volume yaratır, eski volume'lerdeki veriyi kullanmaz
