---
layout: default
title: "Journalctl - part 2"
permalink: /21_1_journalctl-guide/
---

# 🧾 `journalctl` — Часть 2: Cheatsheet и конфигурация systemd-journald

В этом дополнении собраны:
- 📘 20 самых полезных команд `journalctl`, собранных в одном месте;
- ⚙️ Практические рекомендации по настройке `/etc/systemd/journald.conf` для production-систем.

---

## 📑 Содержание

- [Часть 1 — Быстрый Cheatsheet (20 команд)](#cheatsheet)
- [Часть 2 — Настройка journald.conf](#config)
- [Часть 3 — Проверка и обслуживание journald](#maintenance)
- [Часть 4 — Рекомендации для production](#prod-tips)

---

<a id="cheatsheet"></a>
## 🧭 Часть 1 — Быстрый Cheatsheet (20 команд `journalctl`)

| № | Команда | Описание |
|---|----------|-----------|
| 1 | `sudo journalctl` | Показать все записи в журнале |
| 2 | `sudo journalctl -n 50` | Последние 50 записей |
| 3 | `sudo journalctl -e` | Перейти в конец журнала |
| 4 | `sudo journalctl -f` | Следить за логами в реальном времени |
| 5 | `sudo journalctl -u nginx.service` | Логи конкретной службы |
| 6 | `sudo journalctl -b` | Логи текущей загрузки |
| 7 | `sudo journalctl -b -1` | Логи предыдущей загрузки |
| 8 | `sudo journalctl --list-boots` | Список всех загрузок (boots) |
| 9 | `sudo journalctl -p err` | Только ошибки и выше |
| 10 | `sudo journalctl -u nginx -p 3 --since today` | Ошибки nginx за сегодня |
| 11 | `sudo journalctl --since "2 hours ago"` | Логи за последние 2 часа |
| 12 | `sudo journalctl -k` | Только сообщения ядра (dmesg) |
| 13 | `sudo journalctl -r` | Показать в обратном порядке |
| 14 | `sudo journalctl -u ssh -n 100 --no-pager` | Удобный вывод без `less` |
| 15 | `sudo journalctl -u nginx -o cat` | Только содержимое сообщений |
| 16 | `sudo journalctl -u nginx --since yesterday -o json-pretty` | Экспорт JSON для анализа |
| 17 | `sudo journalctl --disk-usage` | Посмотреть, сколько места занимают логи |
| 18 | `sudo journalctl --vacuum-size=500M` | Очистить старые журналы до 500 МБ |
| 19 | `sudo journalctl --vacuum-time=1month` | Удалить журналы старше месяца |
| 20 | `sudo journalctl -u nginx -n 100 -f -o short-iso` | Последние 100 строк nginx + follow с ISO временем |

💡 **Совет:**  
Для быстрого анализа:  
- `-f` = follow (живой поток),  
- `-u` = unit (служба),  
- `-p` = приоритет,  
- `--since` / `--until` = диапазон времени,  
- `-o` = формат вывода (например, `cat` или `json`).

---

<a id="config"></a>
## ⚙️ Часть 2 — Настройка `/etc/systemd/journald.conf`

Файл конфигурации `journald.conf` определяет:
- где и как хранятся журналы,
- какой объём памяти/диска они занимают,
- и как происходит ротация логов.

Обычно путь:
```

/etc/systemd/journald.conf

```

После правок обязательно перезапустите службу:
```bash
sudo systemctl restart systemd-journald
```

---

### 🔹 Основные параметры

| Параметр                | Описание                                                    | Рекомендация                                             |
| ----------------------- | ----------------------------------------------------------- | -------------------------------------------------------- |
| `Storage=`              | Где хранить логи: `volatile`, `persistent`, `auto`, `none`  | `persistent` — чтобы сохранять логи между перезагрузками |
| `SystemMaxUse=`         | Максимальный общий объём логов                              | `1G`–`5G` в зависимости от сервера                       |
| `SystemKeepFree=`       | Минимум свободного места, которое journald должен оставлять | `500M`                                                   |
| `SystemMaxFileSize=`    | Максимальный размер одного файла журнала                    | `100M`                                                   |
| `RuntimeMaxUse=`        | Лимит для временных (RAM) журналов                          | `200M`                                                   |
| `MaxRetentionSec=`      | Максимальный срок хранения логов                            | `1month` (или `2weeks`)                                  |
| `RateLimitIntervalSec=` | Интервал подсчёта для ограничения частоты записей           | `30s`                                                    |
| `RateLimitBurst=`       | Максимальное количество сообщений за интервал               | `1000`                                                   |
| `Compress=`             | Сжимать старые журналы                                      | `yes`                                                    |
| `Seal=`                 | Подписывать архивные журналы для защиты от подделки         | `yes`                                                    |
| `SplitMode=`            | Разделение журналов: `uid`, `none`                          | `uid` (по пользователям)                                 |

---

### 💡 Пример оптимизированного journald.conf (Production)

```ini
[Journal]
Storage=persistent
Compress=yes
Seal=yes
SplitMode=uid

SystemMaxUse=2G
SystemKeepFree=500M
SystemMaxFileSize=200M
RuntimeMaxUse=200M

MaxRetentionSec=1month

RateLimitIntervalSec=30s
RateLimitBurst=1000

ForwardToSyslog=no
ForwardToConsole=no
ForwardToWall=no
```

> ⚠️ После изменений:
>
> ```bash
> sudo systemctl restart systemd-journald
> sudo journalctl --verify
> ```

---

<a id="maintenance"></a>

## 🧹 Часть 3 — Проверка и обслуживание журналов

### Проверить состояние и объём логов:

```bash
sudo journalctl --disk-usage
```

### Очистить логи старше 2 недель:

```bash
sudo journalctl --vacuum-time=2weeks
```

### Уменьшить объём логов до 1 ГБ:

```bash
sudo journalctl --vacuum-size=1G
```

### Принудительно выполнить ротацию:

```bash
sudo journalctl --rotate
```

---

<a id="prod-tips"></a>

## 🚀 Часть 4 — Рекомендации для production-сред

* Всегда **включайте persistent storage** (`Storage=persistent`).
* Настраивайте **ограничение размера**, чтобы логи не заполнили диск (`SystemMaxUse`, `SystemKeepFree`).
* Не пересылайте логи в syslog, если используете `journald` как основное хранилище (`ForwardToSyslog=no`).
* Используйте `Compress=yes` — экономит до 70% места.
* Настраивайте `RateLimitBurst` и `RateLimitIntervalSec`, чтобы избежать "флуда".
* В больших инфраструктурах — пересылайте журналы через `systemd-journal-remote` / `journal-gatewayd` (отдельный пакет `systemd-journal-remote`, не входит в базовую установку — `sudo apt install systemd-journal-remote` / `sudo yum install systemd-journal-remote`).
* Периодически выполняйте `journalctl --verify` и `--vacuum-*` для профилактики.
* Для автоматизации добавьте cron-задачу очистки старых логов:

  ```bash
  0 3 * * 0 root /usr/bin/journalctl --vacuum-time=30days
  ```
  > [!NOTE]
  > Обратите внимание на поле `root` перед командой — это формат **системного** `/etc/cron.d/`/`/etc/crontab` (5 полей времени + пользователь + команда). Если вставить эту строку как есть в личный `crontab -e` (см. <a href="/a/cron-guide">Автоматизация задач с помощью cron</a>) — cron попытается выполнить несуществующую команду `root`, задание не сработает. В `crontab -e` поле пользователя убирается: `0 3 * * 0 /usr/bin/journalctl --vacuum-time=30days`.

---

✅ **Итог:**

* Файл `/etc/systemd/journald.conf` — ключ к стабильной работе логов;
* Команды из Cheatsheet покрывают 99% практических случаев;
* Системное обслуживание (`vacuum`, `rotate`, `verify`) важно выполнять регулярно.

---
