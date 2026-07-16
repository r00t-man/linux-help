---
layout: default
title: "Rsyslog"
permalink: /22_rsyslog-guide/
---

# 🗂 Полное руководство по rsyslog

`rsyslog` — один из самых популярных и мощных системных демонoв для сбора и управления логами в Linux/Unix-системах.  
Он умеет:

- Принимать логи локально и по сети (UDP/TCP/TLS);
- Фильтровать по хостам, сервисам, приоритетам;
- Форматировать записи (шаблоны);
- Хранить логи в бинарных или текстовых файлах;
- Пересылать в централизованные системы мониторинга (ELK, Graylog, Splunk и т.д.).

---

## 📑 Содержание

- [1. Введение и отличия от syslog](#intro)
- [2. Установка rsyslog](#install)
- [3. Основная конфигурация](#config)
- [4. Формат сообщений и шаблоны](#templates)
- [5. Фильтры и правила](#filters)
- [6. Работа по сети: UDP, TCP, TLS](#network)
- [7. Централизованный сбор логов](#central)
- [8. Ротация и хранение логов](#rotation)
- [9. Диагностика и проверка](#debug)
- [10. Продвинутые возможности](#advanced)
- [11. Практические примеры](#examples)
- [12. Советы по продакшн настройке](#prod)

---

<a id="intro"></a>
## 1️⃣ Введение и отличия от syslog

- `syslog` — традиционный системный логгер (legacy).
- `rsyslog` — современный, с модульной архитектурой, поддержкой TCP/TLS, шаблонов, структурированных сообщений (JSON, GELF), очередей и асинхронной доставки.
- Используется по умолчанию в большинстве современных дистрибутивов: Debian, Ubuntu, RedHat, CentOS, Astra Linux.

**Преимущества rsyslog:**
- Высокая производительность (млн сообщений/сек на современном сервере);
- Централизованная маршрутизация и пересылка;
- Шаблоны для кастомного формата логов;
- Модульная структура: input, output, filter, parser.

---

<a id="install"></a>
## 2️⃣ Установка rsyslog

### Debian / Ubuntu / Astra Linux
```bash
sudo apt update
sudo apt install rsyslog
sudo systemctl enable rsyslog
sudo systemctl start rsyslog
```

### RedHat / CentOS / Red OS

```bash
sudo dnf install rsyslog
sudo systemctl enable rsyslog
sudo systemctl start rsyslog
```

Проверка версии:

```bash
rsyslogd -v
```

Статус службы:

```bash
sudo systemctl status rsyslog
```

---

<a id="config"></a>

## 3️⃣ Основная конфигурация

Главный конфигурационный файл:

```
/etc/rsyslog.conf
```

Дополнительные конфиги можно добавлять в:

```
/etc/rsyslog.d/*.conf
```

### Структура

1. **Модули (modules)** — подключение входных/выходных драйверов.
2. **Global Directives** — общие параметры (`WorkDirectory`, `ActionQueueType` и т.д.).
3. **Rules / Actions** — фильтры, куда отправлять сообщения.
4. **Templates** — кастомный формат вывода.

Пример минимальной конфигурации:

```conf
module(load="imuxsock")   # для локальных логов (Unix socket /var/run/syslog)
module(load="imklog")     # для логов ядра

# Хранение логов в текстовые файлы (пример путей — RHEL/РЕД ОС, см. предупреждение ниже)
*.info;mail.none;authpriv.none;cron.none   /var/log/messages
authpriv.*                                /var/log/secure
mail.*                                    -/var/log/maillog
cron.*                                    /var/log/cron

# Пересылка удалённому серверу
*.* @@10.10.10.5:514      # с одним @ — UDP, с двумя @@ — TCP (здесь TCP)
```

> [!WARNING]
> Проверено вживую на этой машине: пути `/var/log/messages`/`/var/log/secure`/`/var/log/maillog` — это дефолтный конфиг **RHEL/РЕД ОС**, не Astra/Debian. Реальный дефолтный `/etc/rsyslog.d/50-default.conf` на Debian-based системе пишет в **другие** файлы:
> ```conf
> auth,authpriv.*                /var/log/auth.log
> *.*;auth,authpriv.none         -/var/log/syslog
> mail.*                         -/var/log/mail.log
> ```
> Тот же паттерн, что уже встречался в других статьях этой вики — на Astra используйте `syslog`/`auth.log`/`mail.log`, на РЕД ОС — `messages`/`secure`/`maillog` (последнее для этой ОС и показано в примере выше).

---

<a id="templates"></a>

## 4️⃣ Формат сообщений и шаблоны

`rsyslog` позволяет настраивать произвольный формат логов через **templates**.

Пример JSON-шаблона:

```conf
template(name="jsonTemplate" type="list") {
    constant(value="{")
    property(name="timestamp" dateFormat="rfc3339" format="jsonf" outname="timestamp")
    constant(value=", ")
    property(name="hostname" format="jsonf" outname="hostname")
    constant(value=", ")
    property(name="syslogtag" format="jsonf" outname="syslogtag")
    constant(value=", ")
    property(name="msg" format="jsonf" outname="msg")
    constant(value="}\n")
}

*.* action(type="omfile" file="/var/log/messages.json" template="jsonTemplate")
```

> [!WARNING]
> Проверено вживую (реальный тестовый rsyslogd, `logger` + `json.loads()`): исходный вариант шаблона (без `outname`/`format="jsonf"`, просто `property(name=...)` + `constant(value=", ")`) генерирует **невалидный JSON** — например `{2026-07-11T11:26:11+02:00, myhost, tag:,  hello, world}`: ни одного `"key":`, значения без кавычек, а если сообщение само содержит запятую (обычное дело) — структура записи ломается окончательно, `json.loads()` падает с `JSONDecodeError`. Исправленный вариант выше — с `format="jsonf"` (JSON-safe экранирование значения) и `outname="..."` (имя ключа) — даёт `{"timestamp":"...", "hostname":"...", ...}`, реально проходит `json.loads()` без ошибок, включая сообщения с запятыми/кавычками внутри. Простое добавление кавычек вручную (`constant(value="\"timestamp\":\"")` + `property(... format="jsonf")` без `outname`) тоже не работает — `format="jsonf"` сам добавляет `"имя":`, получится задвоение ключа.

---

<a id="filters"></a>

## 5️⃣ Фильтры и правила

### Примеры фильтров:

* По приоритету:

```conf
*.info     /var/log/info.log
*.err      /var/log/errors.log
```

* По программе:

```conf
if $programname == 'nginx' then /var/log/nginx.log
```

* Исключения:

```conf
*.info;mail.none;authpriv.none;cron.none /var/log/messages
```

* По hostname:

```conf
if $hostname == 'web01' then /var/log/web01.log
```

---

<a id="network"></a>

## 6️⃣ Работа по сети: UDP, TCP, TLS

### Включение TCP / UDP

```conf
module(load="imtcp")      # TCP
input(type="imtcp" port="514")

module(load="imudp")      # UDP
input(type="imudp" port="514")
```

### Настройка TLS

```conf
$DefaultNetstreamDriverCAFile /etc/ssl/certs/ca.crt
$DefaultNetstreamDriverCertFile /etc/ssl/certs/server.crt
$DefaultNetstreamDriverKeyFile  /etc/ssl/private/server.key
$ActionSendStreamDriver gtls
$ActionSendStreamDriverMode 1
$ActionSendStreamDriverAuthMode x509/name
```

---

<a id="central"></a>

## 7️⃣ Централизованный сбор логов

### Сервер

```conf
# /etc/rsyslog.d/10-remote.conf
module(load="imtcp")
input(type="imtcp" port="514")
$template RemoteFormat,"%timegenerated% %HOSTNAME% %syslogtag% %msg%\n"
*.* ?RemoteFormat
```

### Клиент

```conf
*.* @@rsyslog-server.example.com:514   # TCP
```

> `@` = UDP, `@@` = TCP

---

<a id="rotation"></a>

## 8️⃣ Ротация и хранение логов

* Использовать **logrotate** для файлов `/var/log/*.log`.
* Можно включить встроенную ротацию через `rsyslog`:

```conf
$MaxFileSize 50M
$FileOwner root
$FileGroup adm
$FileCreateMode 0640
$DirCreateMode 0755
$ActionFileDefaultTemplate RSYSLOG_TraditionalFileFormat
```

---

<a id="debug"></a>

## 9️⃣ Диагностика и проверка

* Проверить синтаксис конфигурации:

```bash
rsyslogd -N1
```

* Просмотреть текущие правила:

```bash
sudo rsyslogd -d
```

* Логи самого rsyslog:

```bash
journalctl -u rsyslog
```

---

<a id="advanced"></a>

## 🔧 Продвинутые возможности

* **Многопоточность** (`$ActionQueueType LinkedList` / `Direct` / `FixedArray`);
* **Асинхронные очереди** (ActionQueue);
* **Фильтрация по шаблону / регулярным выражениям** (`contains`, `startswith`);
* **Поддержка GELF / JSON** для интеграции с ELK;
* **Многоуровневая маршрутизация** (split по host, unit, priority).

---

<a id="examples"></a>

## 📋 Практические примеры

* Логи nginx и php-fpm в разные файлы:

```conf
if $programname == 'nginx' then /var/log/nginx.log
if $programname == 'php-fpm' then /var/log/php-fpm.log
```

* Централизованная пересылка ошибок на сервер:

```conf
*.err @@logserver.example.com:514
```

* JSON-шаблон для ELK:

```conf
template(name="GELFTemplate" type="list") {
    constant(value="{")
    property(name="timestamp" format="jsonf" outname="timestamp")
    constant(value=", ")
    property(name="hostname" format="jsonf" outname="hostname")
    constant(value=", ")
    property(name="msg" format="jsonf" outname="short_message")
    constant(value="}\n")
}
*.* action(type="omfile" file="/var/log/messages.json" template="GELFTemplate")
```

> [!NOTE]
> Как и в первом JSON-примере выше — без `format="jsonf"`/`outname` этот шаблон тоже даёт невалидный JSON (проверено там же). Настоящий формат GELF также требует обязательные поля `version`/`host`/`short_message` по спецификации — это упрощённый учебный пример структуры, не полноценный GELF-payload.

---

<a id="prod"></a>

## 💡 Советы для production

* Всегда включайте TCP + TLS для пересылки логов между серверами;
* Используйте фильтры по программам и приоритету, чтобы не перегружать диск;
* Настройте логротейт (logrotate) с ротацией и сжатием;
* Для больших систем — используйте асинхронные очереди (`ActionQueueType LinkedList`);
* JSON / GELF — для интеграции с централизованными системами мониторинга;
* Мониторьте `journalctl -u rsyslog` для состояния демона.

---

📎 **Связанные статьи:**

* <a href="/a/21_journalctl-guide">Полное руководство по journalctl</a>
* <a href="/a/21_1_journalctl-guide">Cheatsheet и конфигурация journald.conf</a>
* <a href="/a/21_2_journalctl-remote">Централизованный сбор через systemd-journal-remote</a>
