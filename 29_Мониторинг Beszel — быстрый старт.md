---
layout: default
title: "Мониторинг Beszel — быстрый старт"
permalink: /beszel-quick-start/
---

# 📊 Мониторинг Beszel — быстрый старт

> Короткая памятка по развёртыванию Beszel для мониторинга Linux-хостов.

## 1) Установка через Docker Compose

```bash
mkdir -p /opt/beszel && cd /opt/beszel

cat > docker-compose.yml <<'YAML'
services:
  beszel:
    image: henrygd/beszel:latest
    container_name: beszel
    restart: unless-stopped
    ports:
      - "8090:8090"
    volumes:
      - ./data:/beszel_data
YAML

sudo docker compose up -d
```

## 2) Первый вход

1. Откройте `http://<IP_сервера>:8090`.
2. Создайте административного пользователя.
3. Добавьте хост и установите агент по инструкции из UI Beszel.

## 3) Что проверить после запуска

```bash
docker ps | grep beszel
docker logs --tail 100 beszel
ss -tulpn | grep 8090
```

## 4) Базовые рекомендации по безопасности

- Ограничьте доступ к порту `8090` через firewall.
- Размещайте Beszel за reverse-proxy с HTTPS.
- Делайте резервные копии каталога `./data`.

