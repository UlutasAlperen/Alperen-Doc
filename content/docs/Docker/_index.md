---
title: "Docker"
weight: 2
bookCollapseSection: true
---
[Docker](https://docs.docker.com/) is _the_ container platform. Nearly every DevOps workflow uses it to package, ship and run applications. I use Docker to:

- Package apps with their dependencies into portable images
- Run isolated containers across homelab and production
- Persist state with volumes and connect containers with networks
- Ship images via registries (Docker Hub) into deployment pipelines
- And much more

## Docker konular

1. [docker-kurulum-hardening-debian](docker-kurulum-hardening-debian/)
2. [manage-docker-as-non-root-user](manage-docker-as-non-root-user/)
3. [docker-containers-images](docker-containers-images/)
4. [docker-volumes](docker-volumes/)
5. [docker-build](docker-build/)
6. [optimize-container-images-with-multi-stage-builds](optimize-container-images-with-multi-stage-builds/)
7. [docker-network](docker-network/)
8. [docker-exec-shell](docker-exec-shell/)
9. [docker-logs](docker-logs/)
10. [docker-compose-basic](docker-compose-basic/)
11. [var-olan-volume-uzerine-docker-compose](var-olan-volume-uzerine-docker-compose/)
12. [docker-publish](docker-publish/)
13. [docker-kurulum-hardening-rocky](docker-kurulum-hardening-rocky/)
14. [docker-distroless-container-images](docker-distroless-container-images/)

### Temeller

- [docker-containers-images](docker-containers-images/) = Container ve image kavramları, `docker run`, `docker ps`, `docker stop`
- [docker-volumes](docker-volumes/) = `docker volume create/ls/inspect`, Ghost örneği ile kalıcı veri
- [docker-exec-shell](docker-exec-shell/) = `docker exec`, interactive shell (`-it /bin/sh`)
- [docker-network](docker-network/) = `--network none`, custom bridge network, Caddy load balancer
- [docker-logs](docker-logs/) = `docker logs -f --tail`, `docker stats`, `docker top`, resource limits

### Build ve Publish

- [docker-build](docker-build/) = Dockerfile yazma, `docker build -t`, Go/Python server dockerize etme, `ENV PORT`
- [optimize-container-images-with-multi-stage-builds](optimize-container-images-with-multi-stage-builds/) = Multi-stage build ile Go (~800MB+ → 15-20MB) ve TypeScript image optimizasyonu, `COPY --from`
- [docker-distroless-container-images](docker-distroless-container-images/) = distroless image hiyerarşisi (`static` → `base` → `cc` → runtimes), `FROM scratch` karşılaştırması, `trivy` ile CVE taraması, `:nonroot`/`:debug` tag'leri, Chainguard/Chisel alternatifleri
- [docker-publish](docker-publish/) = Docker Hub'a `docker push/pull`, tag (`latest` vs semver), deployment pipeline

### Compose ve Operasyon

- [docker-compose-basic](docker-compose-basic/) = `docker compose up/down/ps/logs/build/exec`
- [var-olan-volume-uzerine-docker-compose](var-olan-volume-uzerine-docker-compose/) = var olan volume'ü `external: true` ile compose'a bağlama
- [docker-kurulum-hardening-debian](docker-kurulum-hardening-debian/) = Debian'a güvenli kurulum, `daemon.json` hardening, iptables/UFW notları
- [docker-kurulum-hardening-rocky](docker-kurulum-hardening-rocky/) = Rocky'ye güvenli kurulum, firewalld + SELinux notları, Podman rootless karşılaştırması, `daemon.json` hardening
- [manage-docker-as-non-root-user](manage-docker-as-non-root-user/) = `docker` grubu, rootless notları, systemd ile boot'ta başlatma
