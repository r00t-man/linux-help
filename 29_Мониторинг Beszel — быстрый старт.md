---
layout: default
title: "Мониторинг Beszel — быстрый старт"
permalink: /beszel-quick-start/
---

# 📊 Мониторинг Beszel — быстрый старт

[![Platform](https://img.shields.io/badge/platform-Linux-lightgrey?style=flat-square&logo=linux)]()
[![Category](https://img.shields.io/badge/category-Monitoring-blue?style=flat-square)]()
[![Stack](https://img.shields.io/badge/stack-Docker%20Compose-informational?style=flat-square)]()

> Ниже — полноценный практический сценарий: как развернуть Beszel, подключить узлы, настроить безопасный доступ и не потерять данные.

---

## Что такое Beszel и когда он удобен

**Beszel** — лёгкая self-hosted система мониторинга серверов и контейнеров.
Подходит, когда нужен:

- простой web-интерфейс без «тяжёлого» стека Prometheus/Grafana;
- быстрый старт с минимумом инфраструктуры;
- мониторинг нескольких Linux-хостов и Docker-окружения;
- базовые алерты и централизованный обзор состояния.

---

## 1) Подготовка сервера

### Минимальные требования

- Linux-сервер (Ubuntu/Debian/РЕД ОС/Astra и т.д.);
- установленный Docker Engine + Docker Compose plugin;
- открытый порт для web-доступа (в примере `8090/tcp`);
- корректное время на сервере (NTP).

### Проверка окружения

```bash
docker --version
docker compose version
uname -a
free -h
df -h
```

Если Docker не установлен:

```bash
# пример для Debian/Ubuntu
sudo apt update
sudo apt install -y docker.io docker-compose-plugin
sudo systemctl enable --now docker
```

> [!TIP]
> Добавьте своего пользователя в группу `docker`, чтобы запускать команды без `sudo`:
>
> ```bash
> sudo usermod -aG docker "$USER"
> newgrp docker
> ```

---

## 2) Развёртывание Beszel через Docker Compose

Создайте рабочую директорию:

```bash
sudo mkdir -p /opt/beszel
sudo chown -R "$USER":"$USER" /opt/beszel
cd /opt/beszel
```

Создайте `docker-compose.yml`:

```yaml
services:
  beszel:
    image: henrygd/beszel:latest
    container_name: beszel
    restart: unless-stopped
    ports:
      - "8090:8090"
    volumes:
      - ./data:/beszel_data
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://127.0.0.1:8090/"]
      interval: 30s
      timeout: 5s
      retries: 5
```

Запустите сервис:

```bash
docker compose up -d
docker compose ps
```

Проверьте логи первого запуска:

```bash
docker compose logs -f --tail 100
```

---

## 3) Первый вход и базовая настройка

1. Откройте `http://<IP_сервера>:8090`.
2. Создайте администратора (логин/пароль).
3. Сразу смените слабый пароль на сложный.
4. Проверьте часовой пояс, название инстанса и базовые параметры в UI.

Рекомендуется сразу зафиксировать:

- кто владелец инстанса;
- где хранится каталог `data`;
- как часто делается backup;
- кто получает алерты.

---

## 4) Подключение хостов и агентов

Обычно в UI Beszel для добавления узла генерируется команда установки/регистрации агента.
Общий подход:

1. На центральном сервере создайте новый node.
2. Скопируйте команду для Linux-агента.
3. Выполните её на целевом сервере.
4. Проверьте, что хост появился в списке и отдаёт метрики.

Проверки на хосте с агентом:

```bash
systemctl status beszel-agent --no-pager
journalctl -u beszel-agent -n 100 --no-pager
ss -tulpn
```

> [!IMPORTANT]
> Если серверы находятся в разных сетях/VPN, заранее проверьте маршрутизацию и firewall.

---

## 5) Безопасность (обязательно)

### 5.1 Ограничьте доступ к порту

Если доступ только из админской сети:

```bash
sudo ufw allow from 10.10.0.0/16 to any port 8090 proto tcp
sudo ufw deny 8090/tcp
sudo ufw status
```

### 5.2 Поставьте reverse proxy + HTTPS

Продакшн-вариант: Nginx/Caddy/Traefik перед Beszel.

Минимальный пример Nginx:

```nginx
server {
  listen 443 ssl http2;
  server_name monitor.example.org;

  ssl_certificate     /etc/letsencrypt/live/monitor.example.org/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/monitor.example.org/privkey.pem;

  location / {
    proxy_pass http://127.0.0.1:8090;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
  }
}
```

### 5.3 Резервное копирование данных Beszel

Данные хранятся в `/opt/beszel/data`.

Пример nightly-backup:

```bash
#!/usr/bin/env bash
set -euo pipefail

SRC="/opt/beszel/data"
DST="/opt/backups/beszel"
DATE="$(date +%F_%H-%M)"

mkdir -p "$DST"
tar -czf "$DST/beszel_${DATE}.tar.gz" -C /opt/beszel data
find "$DST" -type f -name 'beszel_*.tar.gz' -mtime +14 -delete
```

---

## 6) Полезные команды эксплуатации

```bash
cd /opt/beszel

# состояние
Docker_ps(){ docker compose ps; }
Docker_logs(){ docker compose logs --tail 200; }

# обновление контейнера
Docker_update(){ docker compose pull && docker compose up -d; }

# перезапуск
Docker_restart(){ docker compose restart; }

# проверка порта
ss -tulpn | grep 8090
```

Можно использовать и без функций:

```bash
docker compose ps
docker compose pull
docker compose up -d
docker compose logs -f --tail 100
```

---

## 7) Диагностика типовых проблем

### Проблема: интерфейс не открывается

Проверьте:

```bash
docker compose ps
ss -tulpn | grep 8090
curl -I http://127.0.0.1:8090
```

Причины:

- контейнер не стартовал;
- порт занят другим сервисом;
- firewall режет входящий доступ;
- неверный IP/FQDN.

### Проблема: агент не регистрируется

Проверьте:

```bash
journalctl -u beszel-agent -n 200 --no-pager
ping -c 3 <ip_beszel_server>
curl -vk https://<fqdn_beszel>
```

Причины:

- нет сетевого доступа между хостами;
- неверный токен/команда регистрации;
- ошибка TLS/сертификата на прокси;
- сильно расходится системное время.

---

## 8) Рекомендуемый рабочий регламент

- Ежедневно: смотреть алерты и unavailable-хосты.
- Еженедельно: проверять утилизацию CPU/RAM/Disk и тренды.
- Ежемесячно: обновлять контейнер Beszel, делать тест восстановления backup.
- После изменений в сети: тестировать доступность агентов.

---

## 9) Быстрый чек-лист «в прод»

- [ ] Beszel доступен только по HTTPS.
- [ ] Доступ к UI ограничен trusted-сетями.
- [ ] Настроены backup и хранение минимум 14 дней.
- [ ] Определён ответственный за алерты.
- [ ] Есть инструкция восстановления после сбоя.

---

## 10) Короткая памятка команд

```bash
# запуск / остановка
cd /opt/beszel && docker compose up -d
cd /opt/beszel && docker compose down

# обновление
cd /opt/beszel && docker compose pull && docker compose up -d

# логи
cd /opt/beszel && docker compose logs -f --tail 200

# backup
sudo tar -czf /opt/backups/beszel_$(date +%F).tar.gz -C /opt/beszel data
```

Удачного мониторинга 👨‍💻
