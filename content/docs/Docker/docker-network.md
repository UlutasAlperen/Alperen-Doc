---
title: "docker-network"
weight: 7
---
# Docker-Network

## Offline
Now, let's force a container into _offline_ mode!

You might be thinking, "why would I want to turn off networking?!?" Well, usually it's for security reasons. You might want to remove the network connection from a container in one of these scenarios:

- You're running 3rd party code that you don't trust, and it shouldn't need network access
- You're building an e-learning site, and you're allowing students to execute code on your machines
- You know a container has a virus that's sending malicious requests over the internet, and you want to do an audit

## Network None

The `docker run` command has a `--network none` flag that makes it so that the container can't network with the outside world, which is super useful for isolating containers.


1. Use `docker ps`, `docker stop`, and `docker rm` to stop and remove the "getting started" container if it's running.
2. Start a new "Getting Started" container in `--network none` mode:

```bash
docker run -d --network none docker/getting-started
```

3. Run the `ping` command with a timeout of 2 seconds inside the container:

```bash
docker exec CONTAINER_ID ping google.com -W 2
```

If all goes well, the program should hang for 2 seconds, then **report an error message**, because you don't have internet access!

## Load Balancers

Let's try something a bit more complex: configuring a load balancer!

A load balancer behaves as advertised: it balances a load of network traffic across some number of servers. Think of a huge website like Google.com. There's no way that a single server (literally a single computer) could handle all of the Google searches for the entire world. Google uses load balancers to route requests to different servers.

A central server, called the "load balancer", receives traffic from users (aka clients), then routes those requests to different back-end application servers. In the case of Google, this splits the world's traffic across potentially many different thousands of computers.

A good load balancer sends new traffic to servers that have lower current resource utilization (CPU and memory). The goal is to "balance the load" so that no single backend server becomes overwhelmed. There are many strategies that load balancers use, but a simple strategy is the "round robin" where requests are simply routed one after the other to different back-end servers:

``` mysql
Request 1 -> Server 1
Request 2 -> Server 2
Request 3 -> Server 3
Request 4 -> Server 1
Request 5 -> Server 2
```

# Custom Network

create custom [bridge](https://docs.docker.com/network/bridge/) networks so that containers can communicate with each other if we want them to, but still otherwise remain isolated. Let's build a system where our application servers are hidden within a custom network, and only our load balancer is exposed to the host.

This is a very common setup in backend architecture. The load balancer is exposed to the public internet, but the application servers are only accessible _via_ the load balancer.

1. Let's create a custom bridge network called "caddytest".

```bash
docker network create caddytest
```

2. See if it worked by listing all the networks:

```bash
docker network ls
```

3. Stop and restart your caddy application servers, but this time, make sure you attach them to the `caddytest` network.
    - Use the `--network caddytest` flag to attach them to network
    - Use the `--name` flag to name them `caddy1` and `caddy2` respectively so it's easier to reference them later
    - Do _not_ use the `-p` flag to expose ports. We don't want these accessible from the host machine.

```bash
docker run -d --name caddy2 --network caddytest -v $PWD/index2.html:/usr/share/caddy/index.html caddy
```
4. Create another "getting started" container on the same network and start a shell session within it:

```bash
docker run -it --network caddytest docker/getting-started /bin/sh
```

By giving our containers some names, `caddy1` and `caddy2`, and providing a bridge network, Docker has set up name resolution for us! The container names resolve to the individual containers from all other containers on the network.

5. Within your `docker/getting-started` container shell, [curl](https://curl.se/) the first container:

```bash
curl caddy1
```

6. Also `curl` the second container:

```bash
curl caddy2
```

7. If you get the HTML responses that you expect, `exit` out of your shell session within the "getting started" container.

If you need to restart your caddy application servers after naming them, you can use: `docker start caddy1` and `docker start caddy2`.

# Configuring the Load Balancer

We've confirmed that we have 2 application servers (Caddy) working properly on a custom bridge network. Let's create a load balancer that balances network requests between the two! We'll use a round-robin balancing strategy, so each request should route back and forth between the servers.

## Caddyfiles

Caddy works great as a file server, which is what our little HTML servers are, but it also works great as a load balancer! To use Caddy as a load balancer we'll need to create a custom [Caddyfile](https://caddyserver.com/docs/caddyfile) to tell Caddy how we want it to balance the traffic. It's just a config file for Caddy.


1. Stop and remove any containers that aren't the 2 caddy servers we're working with currently.
2. Create a new file in your local directory called `Caddyfile`:

```Caddyfile
localhost:80

reverse_proxy caddy1:80 caddy2:80 {
	lb_policy       round_robin
}
```

This tells Caddy to run on `localhost:80`, and to round robin any incoming traffic to `caddy1:80` and `caddy2:80`. Remember, this only works because we're going to run the loadbalancer _on the same network_, so `caddy1` and `caddy2` will automatically resolve to our application server's containers.

3. Start the load balancer container on port `8880`. Instead of an `index.html`, give it our custom `Caddyfile`:

```bash
docker run -d --network caddytest -p 8880:80 -v $PWD/Caddyfile:/etc/caddy/Caddyfile caddy
```


4. Hit the load balancer on `http://localhost:8880/`! You should either get a response from server 1 or server 2, and if you hard refresh the page, it should swap back and forth.

If it's not swapping properly, try using `curl` instead. Your browser might be caching the HTML.

```bash
curl http://localhost:8880/
```

### docker network aslında nasıl çalışıyor

docker network ne işe yarıyor dersek, konteynerlerin birbirleriyle ve dışarıyla nasıl konuşacağını ayarlamamıza yarıyor, her konteynere ayrı bir network namespace açılıyor yani her konteynerin kendi IP'si kendi `eth0`'ı varmış gibi düşün

temel çalışma mantığı şöyle, konteynerin `eth0`'ı bir `veth` kablosuyla host'taki `docker0` bridge'ine bağlı, apartman koridoru gibi düşün herkes aynı koridora çıkıyor ama herkesin kendi odası var

- **`veth + bridge`**: burada konteynerin odasıyla host'un koridoru birbirine bağlanıyor, `docker0` gelen gideni yönlendiriyor
- **`NAT`**: burada konteynerin dışarı çıkışı host'un IP'si üzerinden maskeleniyor, konteyner google'a gider ama google host'u görüyor
- **`port yayınlama`**: burada dışarıdan içeri girişi biz açıyoruz, `8080:80` yazmazsan dışarıdan kimse odaya giremez
- **`embedded DNS`**: burada isimle konuşma oluyor, `127.0.0.11` adresinde koşuyor, sadece custom network'te düzgün çalışıyor

### bridge

default olarak zaten bridge kullanıyoruz, `docker run` deyince konteyner buraya düşüyor

- **`IP`**: burada konteynere ayrı fake IP veriyor, örn: `172.18.0.3`, kendi odası var
- **`NAT`**: burada host ile arasında NAT / port kapısı var, sen `8080:80` yazmazsan dışarıdan kimse giremez ama konteyner dışarı çıkabilir
- **`izolasyon`**: burada iki konteyner aynı bridge'deyse konuşur, değilse göremez
- **`DNS sorunu`**: burada default bridge'de isimle konuşma yok, IP ile uğraşıyorsun, `ping 172.18.0.3` çalışır ama `ping caddy1` çalışmaz

```bash
docker run -d --network bridge nginx
docker network inspect bridge
```

> `inspect` çıktısında `Containers` kısmına bak, hangi konteynerin hangi IP'yi aldığını oradan görüyoruz

### custom bridge vs default bridge

yukarıda caddy örneğinde zaten `caddytest` diye custom bridge kurduk, aslında olay bu, default bridge yetmiyor diye kendi networkümüzü açıyoruz

```bash
docker network create caddytest
docker run -d --name caddy1 --network caddytest caddy
docker run -d --name caddy2 --network caddytest caddy
```

- **`DNS`**: burada compose DNS var, `web -> db:5432` diye isimle konuşur, `curl caddy1` yazınca direkt o konteynere gider
- **`izolasyon`**: burada sadece bu network'tekiler birbirini görüyor, dışarı kapalı, load balancer'ı `8880:80` ile dışarı açtık ama app server'ları açmadık ya o mantık işte
- **`neden custom`**: burada default bridge'de `--link` gibi eski şeylerle uğraşmıyoruz, custom açınca DNS otomatik geliyor

> Caddyfile'da `reverse_proxy caddy1:80 caddy2:80` yazdık ya, o isimler sadece aynı custom network'te olduğumuz için çözülüyor

### host

- **`IP`**: burada ayrı IP yok, konteyner host'un IP'sini kullanıyor, oda yok aynı evde yaşıyoruz
- **`NAT`**: burada NAT yok kapı yok, uygulama `80` açtıysa host'un `80`'i doldu
- **`ports`**: burada `ports:` yazsan da takılmıyor, zaten host'un portundasın, `-p 8080:80` yazmanın anlamı yok
- **`konuşma`**: burada isimle konuşma yok, `localhost:port` ile konuşuyorsun, sanki host'a kurmuşsun gibi oluyor
- **`çakışma`**: burada hangi konteyner ne açtıysa hepsi birbirinin portunu görür, çakışır, iki tane `80` açamazsın

```bash
docker run -d --network host nginx
```

> burada performans iyi oluyor çünkü NAT yok ama güvenlik zayıf oluyor, bir de UFW'yi bypass ediyor, güvenli yayınlama için [docker-kurulum-hardening-debian](../docker-kurulum-hardening-debian/) sayfasındaki `127.0.0.1:8080:80` mantığına bak

### none

- **`durum`**: burada loopback'ten başka interface yok, offline kasa gibi düşün
- **`ne işe yarıyor`**: burada dışarıyla konuşmasını istemediğimiz işleri koşuyoruz, güvenmediğimiz 3rd party kodu çalıştırırken ya da virüslü konteyneri audit ederken kullanıyoruz
- **`deneme`**: burada yukarıdaki `Offline` bölümünde yaptığımız gibi `ping google.com -W 2` yazınca hata veriyor, internet yok çünkü

```bash
docker run -d --network none docker/getting-started
docker exec CONTAINER_ID ping google.com -W 2
```

### temel komutlar

komutlarla networkleri kurcalıyoruz, `docker network --help` yazınca hepsi çıkıyor zaten

- **`docker network ls`**: burada hangi networkler var ona bakıyoruz, `bridge host none` default geliyor
- **`docker network inspect caddytest`**: burada o networkün içine bakıyoruz, hangi konteyner hangi IP'yi almış görüyoruz
- **`docker network create my-net`**: burada kendi custom bridge'imizi açıyoruz, DNS otomatik geliyor
- **`docker network connect my-net konteyner-adi`**: burada çalışan konteyneri sonradan bir network'e sokuyoruz
- **`docker network disconnect my-net konteyner-adi`**: burada konteyneri networkten çıkarıyoruz
- **`docker network rm my-net`**: burada kullanmadığımız networkü siliyoruz
- **`docker network prune`**: burada boşta duran networkleri topluca temizliyoruz

### compose'da network

compose'da zaten `networks:` yazınca arka planda custom bridge açıyor, tek tek `docker network create` ile uğraşmıyoruz

```yaml
services:
  web:
    image: nginx:latest
    ports:
      - "8080:80"
    networks:
      - app_net
  db:
    image: postgres:15
    networks:
      - app_net

networks:
  app_net:
    driver: bridge
```

- **`networks`**: burada konteynerlerin hangi network üzerinden konuşacağını yazıyoruz, servisleri birbirine bağlıyoruz
- **`driver: bridge`**: burada tek makinede izole network açıyoruz, `web -> db:5432` diye isimle konuşuyorlar
- **`ports vs expose`**: `ports` yazarsak konteyner portunu host'a yayınlıyoruz, dışarıdan erişilebiliyor; `expose` ise sadece belgeleme amaçlı, ek izolasyon sağlamıyor — aynı network'teki servisler zaten birbirine tüm dinleyen portlardan erişebiliyor
