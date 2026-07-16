---
layout: default
title: "Journalctl - part 3"
permalink: /21_2_journalctl-remote/
---

# 🌐 `journalctl` — Часть 3: Централизованный сбор журналов (`systemd-journal-remote`)

`systemd-journal-remote` позволяет собирать и сохранять логи с других хостов, где используется `systemd`.  
Это штатный инструмент systemd, без необходимости в rsyslog, syslog-ng или ELK.  
Он прост, надёжен и идеально подходит для инфраструктур Astra Linux, Red OS и других дистрибутивов с systemd.

---

## 📑 Содержание

- [Что такое systemd-journal-remote](#about)
- [Компоненты и схема работы](#schema)
- [Установка необходимых пакетов](#install)
- [Настройка сервера сбора логов (Receiver)](#receiver)
- [Настройка клиента (Sender)](#sender)
- [Проверка работы и примеры](#check)
- [Фильтрация, просмотр и ротация журналов](#view)
- [Безопасность и шифрование (HTTPS)](#security)
- [Практические советы и рекомендации](#tips)

---

<a id="about"></a>
## 🧠 Что такое `systemd-journal-remote`

`systemd-journal-remote` и его спутник `systemd-journal-upload` — это встроенные утилиты systemd для пересылки бинарных журналов (`journal`) между хостами.

**Основные возможности:**
- Приём логов с других серверов по HTTP(S);
- Хранение их в стандартных бинарных `.journal` файлах;
- Просмотр через обычный `journalctl`;
- Полная поддержка фильтрации и поиска по полям (`_HOSTNAME`, `_SYSTEMD_UNIT`, и т.п.);
- Без сторонних демонов и парсеров.

---

<a id="schema"></a>
## 🧩 Компоненты и схема работы

```

┌────────────────────────────────┐         HTTPS/HTTP       ┌────────────────────────────────┐
│    Клиент                      │ ───────────────────────▶ │  Сервер сбора                  │
│ (journal-upload)               │                          │ (journal-remote)               │
│ systemd-journal-upload.service │                          │ systemd-journal-remote.service │
└────────────────────────────────┘                          └────────────────────────────────┘

```

1. **Клиент** (`systemd-journal-upload`) — отправляет логи на удалённый сервер.
2. **Сервер** (`systemd-journal-remote`) — принимает и сохраняет журналы.
3. **Просмотрщик** (`journalctl --directory=/var/log/journal/remote/`) — анализирует полученные логи.

---

<a id="install"></a>
## ⚙️ Установка пакетов

### Для Debian / Astra Linux
```bash
sudo apt install systemd-journal-remote
```

### Для Red OS / RHEL / CentOS / ALT

```bash
sudo dnf install systemd-journal-remote
# или
sudo yum install systemd-journal-remote
```

> После установки появятся службы:
>
> * `systemd-journal-remote.service`
> * `systemd-journal-upload.service`
> * `systemd-journal-gatewayd.service` (опционально — web-доступ)

---

<a id="receiver"></a>

## 🖥 Настройка сервера сбора логов (Receiver)

1️⃣ **Включаем постоянное хранение журналов**

```bash
sudo mkdir -p /var/log/journal/remote
sudo chown systemd-journal-remote:systemd-journal-remote /var/log/journal/remote
sudo chmod 2755 /var/log/journal/remote
```

2️⃣ **Включаем и запускаем службу**

```bash
sudo systemctl enable systemd-journal-remote
sudo systemctl start systemd-journal-remote
```

> [!WARNING]
> Проверено вживую (установлен и запущен реальный пакет): штатный unit-файл `systemd-journal-remote.service` жёстко прописывает `--listen-https` (не HTTP!) и по умолчанию ждёт сертификат в `/etc/ssl/private/journal-remote.pem`. Без готового сертификата служба **не стартует вообще** — падает в цикл рестартов с ошибкой `Failed to read key from file '/etc/ssl/private/journal-remote.pem': Permission denied` (или "No such file"). То есть шаги 1️⃣–4️⃣ этого раздела сами по себе **не поднимут рабочий сервер** — нужен ОДИН из двух путей:
> - Настроить HTTPS-сертификаты **сейчас**, а не потом — см. раздел "Безопасность и HTTPS" ниже, он на самом деле не опциональное усиление, а обязательное условие запуска со штатным юнитом;
> - Либо явно переключиться на простой HTTP через override (если шифрование не требуется, например в закрытом внутреннем контуре):
>   ```bash
>   sudo systemctl edit systemd-journal-remote.service
>   ```
>   и в открывшемся drop-in указать:
>   ```ini
>   [Service]
>   ExecStart=
>   ExecStart=/usr/lib/systemd/systemd-journal-remote --listen-http=-3 --output=/var/log/journal/remote/
>   ```
>   (пустая строка `ExecStart=` перед новым значением обязательна — иначе новая команда добавится ВТОРЫМ `ExecStart=`, а не заменит исходный).

3️⃣ **Проверяем статус**

```bash
sudo systemctl status systemd-journal-remote
```

По умолчанию (без переключения на `--listen-http` выше) служба ожидает HTTPS:

```
https://0.0.0.0:19532/
```

4️⃣ **Проверяем порт**

```bash
sudo ss -tulnp | grep 19532
```

5️⃣ **Каталог хранения логов по умолчанию:**

```
/var/log/journal/remote/
```

---

<a id="sender"></a>

## 🛰 Настройка клиента (Sender)

1️⃣ **Создаём конфиг**
Файл: `/etc/systemd/journal-upload.conf`

```ini
[Upload]
# URL сервера сбора логов
URL=http://10.10.10.5:19532

# Отправлять только подтверждённые логи (без дубликатов)
ServerKeyFile=/etc/ssl/private/client.key
ServerCertificateFile=/etc/ssl/certs/client.crt
TrustedCertificateFile=/etc/ssl/certs/ca.crt
```

*(если HTTPS не используется — оставляем только URL)*

2️⃣ **Активируем и запускаем**

```bash
sudo systemctl enable systemd-journal-upload
sudo systemctl start systemd-journal-upload
```

3️⃣ **Проверяем отправку**

```bash
sudo systemctl status systemd-journal-upload
```

---

<a id="check"></a>

## 🔍 Проверка работы

На сервере приёмнике:

```bash
sudo journalctl --directory=/var/log/journal/remote/ | head
```

Проверяем, что приходят новые записи с разных хостов:

```
_HOSTNAME=web01
_SYSTEMD_UNIT=nginx.service
MESSAGE=nginx started successfully
```

Можно фильтровать по хосту:

```bash
sudo journalctl --directory=/var/log/journal/remote/ _HOSTNAME=web01
```

---

<a id="view"></a>

## 🧾 Фильтрация, просмотр и ротация

Фильтрация:

```bash
sudo journalctl --directory=/var/log/journal/remote/ _SYSTEMD_UNIT=sshd.service -p err
```

Просмотр логов конкретного клиента:

```bash
sudo journalctl --directory=/var/log/journal/remote/ _HOSTNAME=db01
```

Очистка и сжатие старых журналов:

```bash
sudo journalctl --directory=/var/log/journal/remote/ --vacuum-time=30days
```

---

<a id="security"></a>

## 🔐 Безопасность и HTTPS

По умолчанию `systemd-journal-remote` слушает HTTP (порт 19532).
Для production рекомендуется включить HTTPS и аутентификацию.

### 1️⃣ Генерация сертификатов (self-signed пример)

```bash
sudo mkdir -p /etc/systemd/ssl
cd /etc/systemd/ssl
sudo openssl req -x509 -newkey rsa:2048 -keyout server.key -out server.crt -days 365 -nodes -subj "/CN=logserver.local"
```

### 2️⃣ Настраиваем `/etc/systemd/journal-remote.conf`

```ini
[Remote]
Seal=yes
ListenStream=19532
ServerKeyFile=/etc/systemd/ssl/server.key
ServerCertificateFile=/etc/systemd/ssl/server.crt
```

### 3️⃣ Перезапускаем службу

```bash
sudo systemctl restart systemd-journal-remote
```

Теперь сервер слушает `https://0.0.0.0:19532`.

### 4️⃣ На клиенте (`journal-upload.conf`)

```ini
[Upload]
URL=https://logserver.local:19532
TrustedCertificateFile=/etc/systemd/ssl/server.crt
```

---

<a id="tips"></a>

## 💡 Практические советы и рекомендации

* 🚀 Разворачивайте **отдельный сервер логов**, где хранятся только журналы (`journal-remote` + `Storage=persistent`);
* 🔒 Используйте **HTTPS + сертификаты**, чтобы защитить логи при передаче;
* 🕓 Регулярно выполняйте очистку (`--vacuum-time=30days`);
* 📁 Для разных хостов можно хранить логи в отдельных директориях (`_HOSTNAME`);
* 📡 Для визуализации можно подключить `journal-gatewayd` (веб-интерфейс);
* 📊 Экспортируйте логи в JSON для анализа:

  ```bash
  sudo journalctl --directory=/var/log/journal/remote/ -o json-pretty > /tmp/logs.json
  ```
* ⚙️ Не забывайте включить `Storage=persistent` на клиентах, чтобы не терять записи при перезагрузке.

---

## ✅ Итог

`systemd-journal-remote` и `journal-upload` — это встроенная, надёжная альтернатива rsyslog для систем, использующих `systemd`.
Они позволяют легко и безопасно собирать логи с десятков серверов в централизованное хранилище без дополнительного софта.
В сочетании с `journalctl` и `journald.conf` это полноценная система управления журналами на уровне ОС.

---
