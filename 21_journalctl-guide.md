---
layout: default
title: "Journalctl - part 1"
permalink: /21_journalctl-guide/
---

# 🗂 `journalctl` — исчерпывающий справочник

`journalctl` — основной инструмент для доступа к журналам `systemd-journald`. В этом материале — всё, что нужно знать для эффективной диагностики: фильтры, форматы вывода, работа с таймингом, очистка, экспорт и практические примеры.

---

## 📑 Содержание

- [Важные предварительные замечания](#prelim)
- [Быстрый старт — основные команды](#quick)
- [Фильтрация по юнитам, PID, полям и приоритету](#filter)
- [Фильтрация по времени и по загрузкам (boots)](#time-boot)
- [Форматы вывода и парсинг (json, cat, verbose)](#output)
- [Реальное время / follow / tail](#follow)
- [Работа с файлом журнала / экспорт / импорт](#file-export)
- [Обслуживание журнала (disk usage, vacuum, rotate)](#maintenance)
- [Отладка и расширенные приёмы](#debug)
- [Полезные готовые примеры (cheatsheet)](#examples)
- [Короткие советы и рекомендации](#tips)

---

<a id="prelim"></a>
## 🔰 Важные предварительные замечания

- Для большинства действий с системными журналами требуются права root. Используйте `sudo journalctl ...`.  
  Некоторые дистрибутивы позволяют чтение членам группы `systemd-journal`, но чаще проще запускать с `sudo`.

- Журнал может быть **volatile** (в памяти: `/run/log/journal`) или **persistent** (на диске: `/var/log/journal`).  
  Чтобы включить хранение на диске:  
  ```bash
  # опция 1: создать директорию для persistent storage и перезапустить journald
  sudo mkdir -p /var/log/journal
  sudo systemd-tmpfiles --create --prefix /var/log/journal   # выставляет верные root:systemd-journal + setgid
  sudo systemctl restart systemd-journald

  # опция 2: в /etc/systemd/journald.conf установить:
  # Storage=persistent
  # и затем:
  sudo systemctl restart systemd-journald
  ```

После этого логи будут храниться между перезагрузками.

---

<a id="quick"></a>

## 🚀 Быстрый старт — основные команды

Показать все логи (начиная с самого древнего):

```bash
sudo journalctl
```

Показать последние N строк:

```bash
sudo journalctl -n 100        # последние 100 строк
sudo journalctl -e           # перейти в конец (как tail -n +?)
```

Следить в реальном времени (аналог tail -f):

```bash
sudo journalctl -f
```

Показать логи конкретной службы (unit):

```bash
sudo journalctl -u nginx.service
```

Показать логи с текущей загрузки:

```bash
sudo journalctl -b
```

Показать логи ядра (dmesg):

```bash
sudo journalctl -k
```

---

<a id="filter"></a>

## 🔎 Фильтрация — по юниту, PID, полям, приоритету и тексту

### По unit (службе)

```bash
sudo journalctl -u ssh.service
sudo journalctl -u nginx.service -u php-fpm.service   # несколько юнитов
```

### По процессу / PID / User

```bash
sudo journalctl _PID=12345
sudo journalctl _UID=1000
sudo journalctl _EXE=/usr/sbin/sshd
```

> Поля (такие как `_PID`, `_UID`, `_EXE`, `_COMM`, `_SYSTEMD_UNIT`, `SYSLOG_IDENTIFIER`) можно использовать как фильтры `key=value`.

### По приоритету (severity)

Уровни: `emerg(0)`, `alert(1)`, `crit(2)`, `err(3)`, `warning(4)`, `notice(5)`, `info(6)`, `debug(7)`.

Показывать ошибки и выше (err, crit, alert, emerg):

```bash
sudo journalctl -p err
```

Цифровой вариант:

```bash
sudo journalctl -p 3
```

### По тексту (grep)

`journalctl` сам по себе умеет фильтровать по полям, но для поиска по тексту удобно pipe + grep:

```bash
sudo journalctl -u nginx.service | grep "timeout"
```

Или лучше (включает форматирование, не теряя цветов) — используйте `-o cat`:

```bash
sudo journalctl -u nginx.service -o cat | grep "timeout"
```

---

<a id="time-boot"></a>

## ⏱ Фильтрация по времени и по загрузкам (boot)

### `--since` / `--until`

Поддерживаются абсолютные (YYYY-MM-DD HH:MM:SS) и относительные форматы (`"today"`, `"yesterday"`, `"2 hours ago"`):

```bash
sudo journalctl --since "2025-10-01 08:00:00" --until "2025-10-01 12:00:00"
sudo journalctl --since "2 hours ago"
sudo journalctl --since today
```

### По загрузкам (boots)

Показать логи текущей загрузки:

```bash
sudo journalctl -b
```

Показать логи предыдущей загрузки:

```bash
sudo journalctl -b -1
```

Список доступных загрузок:

```bash
sudo journalctl --list-boots
```

Выведет что-то вроде:

```
-1  9f5b… Thu 2025-10-02 10:00:00 CEST — Thu 2025-10-02 12:00:00 CEST
 0  a3c4… Thu 2025-10-02 12:00:01 CEST — now
```

Можно выбрать boot по его индексу (`-b -1`) или по ID (`-b a3c4…`).

---

<a id="output"></a>

## 🖨 Форматы вывода и примеры (опция `-o` / `--output`)

`journalctl` поддерживает множество форматов. Вот самые полезные:

* `-o short` (по умолчанию) — компактный вывод с датой.
* `-o short-iso` — ISO timestamps: удобно в скриптах.
* `-o short-precise` / `short-iso-precise` — timestamps с микросекундами.
* `-o verbose` — полная информация: все поля (очень много).
* `-o export` — сериализованный текстовый формат (не настоящий бинарник, хотя и не для чтения глазами напрямую) для передачи между инстансами journald (`systemd-journal-remote`/`journal-upload`) — **не читается обратно через `journalctl --file=...`**, см. предупреждение ниже.
* `-o json` / `-o json-pretty` / `-o json-sse` — для парсинга (jq).
* `-o cat` — печатает только значение `MESSAGE` (без заголовков/метаданных).

**Примеры:**

```bash
sudo journalctl -u nginx -n 50 -o short-iso
sudo journalctl -u nginx -n 50 -o json-pretty | jq '.[]._SYSTEMD_UNIT, .[].MESSAGE'
sudo journalctl -u nginx -o cat | tail -n 20
```

---

<a id="follow"></a>

## 🔁 Follow / tail — смотреть логи в реальном времени

Аналоги `tail -f`:

```bash
sudo journalctl -f                    # все новые логи
sudo journalctl -u nginx -f           # только nginx, в реальном времени
sudo journalctl -f -o cat             # только сообщения (удобно для парсинга)
sudo journalctl -u nginx -n 200 -f    # последние 200 строк, затем follow
```

---

<a id="file-export"></a>

## 💾 Экспорт и чтение файлов журнала

### Экспорт текущих записей в переносимый формат (`-o export`)

```bash
sudo journalctl -u nginx --since "2025-10-07" -o export > /tmp/nginx.export
```

> [!WARNING]
> Проверено вживую: `.journal`-расширение в имени файла не делает файл настоящим journal-файлом. `journalctl --file=/tmp/nginx.export` на файле, полученном через `-o export`, реально падает с ошибкой **`Failed to open files: Bad message`** — формат `-o export` НЕ совместим с `--file=`. `--file=` понимает только настоящие бинарные `.journal`-файлы из `/var/log/journal/` (или скопированные оттуда один в один), не сериализованный вывод `-o export`. Формат `-o export` предназначен для передачи в `systemd-journal-remote`/`journal-upload`, либо для собственного парсинга скриптом (это по сути текст, `FIELD=value` построчно) — не для обратного чтения через `journalctl --file=`.
>
> Если нужно именно "выгрузить кусок журнала и потом читать его через `journalctl --file=`" — копируйте настоящий `.journal`-файл из `/var/log/journal/<machine-id>/` напрямую (`cp`), а не пересобирайте через `-o export`.

### Экспорт в текст / JSON

```bash
sudo journalctl -u nginx --since today -o json-pretty > /tmp/nginx.json
```

**Передача/анализ**: JSON удобно передавать в аналитические инструменты или парсить `jq`.

---

<a id="maintenance"></a>

## 🧹 Обслуживание журналов: disk-usage, vacuum, rotate

Проверить занимаемое место журналом:

```bash
sudo journalctl --disk-usage
# Output: "Archived and active journals take up 1.2G in the file system."
```

Освободить место (несколько вариантов):

* Удалить старые файлы, пока общее занятое место не станет <= 500M:

```bash
sudo journalctl --vacuum-size=500M
```

* Удалить файлы старше заданного времени:

```bash
sudo journalctl --vacuum-time=2weeks
```

* Оставить не более N файлов журнала:

```bash
sudo journalctl --vacuum-files=5
```

Принудительно выполнить ротацию журналов:

```bash
sudo journalctl --rotate
```

> [!NOTE]
> По `man systemd-journald.service`: `journalctl --rotate` под капотом — это то же самое, что послать journald сигнал **`SIGUSR2`** (`sudo systemctl kill -s SIGUSR2 systemd-journald`). `SIGHUP` для journald нигде не задокументирован (в man-странице сигналов для него нет вообще) — это не рабочий способ ни для ротации, ни для перечитывания конфига. Если реально нужно применить изменения `journald.conf` — единственный надёжный способ — `sudo systemctl restart systemd-journald` (полный перезапуск, не сигнал).

Проверить/верифицировать целостность файлов:

```bash
sudo journalctl --verify
```

---

<a id="debug"></a>

## 🛠 Отладка и расширенные приёмы

### Показать все поля для одной записи

```bash
sudo journalctl -u nginx -n 1 -o verbose
```

Вы увидите полный набор полей `_PID`, `_UID`, `_SYSTEMD_UNIT`, `_COMM`, `_EXE`, `_CAP_EFFECTIVE`, `MESSAGE` и др.

### Показать сообщения в обратном порядке (новые первыми)

```bash
sudo journalctl -r -n 200
```

### Показывать расширенные подсказки для сообщений (catalog)

```bash
sudo journalctl -u ssh -p err -x
```

Опция `-x` пытается добавить объяснения из базы `systemd` (если есть).

### Использовать `--no-pager` / `--no-full`

* `--no-pager` — вывод не через `less`, полезно в скриптах.
* `--no-full` — сокращает длинные поля.

### Смотреть только сообщения kernel (dmesg-like)

```bash
sudo journalctl -k -b             # kernel messages for current boot
sudo journalctl -k -b -1          # kernel messages for previous boot
```

### Комбинирование большого количества фильтров

Пример: показать ошибки nginx и php-fpm за последние 24 часа, в формате ISO:

```bash
sudo journalctl -u nginx.service -u php-fpm.service -p err --since "24 hours ago" -o short-iso
```

### Парсинг JSON + jq

Получить только сообщения и метку времени:

```bash
sudo journalctl -u nginx -o json-pretty | jq -r '.MESSAGE, .__REALTIME_TIMESTAMP'
```

---

<a id="examples"></a>

## 📋 Cheatsheet — готовые полезные примеры

1. Последние 200 строк для nginx:

```bash
sudo journalctl -u nginx -n 200
```

2. Следить nginx и php-fpm в реальном времени (в одном окне):

```bash
sudo journalctl -u nginx -u php-fpm -f -o cat
```

3. Ошибки системного уровня за сегодня:

```bash
sudo journalctl -p err --since today
```

4. Показать логи предыдущей загрузки (и сразу перейти в конец):

```bash
sudo journalctl -b -1 -e
```

5. Экспорт логов nginx за вчера и сжать (для передачи/архива — не для повторного чтения через `journalctl --file=`, см. предупреждение выше):

```bash
sudo journalctl -u nginx --since "yesterday" --until "today" -o export > /tmp/nginx.export
gzip /tmp/nginx.export
```

6. Сохранить только messages (без метаданных):

```bash
sudo journalctl -u nginx -o cat > /tmp/nginx.log
```

7. Найти все записи, где PROCESS_NAME=cron и ошибка:

```bash
sudo journalctl _COMM=cron -p err
```

8. Показать все логи с конкретного исполняемого файла:

```bash
sudo journalctl _EXE=/usr/sbin/sshd
```

9. Посмотреть, сколько места занимает журнал и автоматически очистить старое, если >1G:

```bash
sudo journalctl --disk-usage
sudo journalctl --vacuum-size=1G
```

10. Быстрый просмотр последних сообщений (без pager) — удобно для скрипта:

```bash
sudo journalctl -u ssh -n 20 --no-pager
```

---

<a id="tips"></a>

## 💡 Советы и рекомендации (best practices)

* **Запуск от root**: большинство операций требуют `sudo`. Добавлять пользователя в группу `systemd-journal` можно, но проще — `sudo`.
* **Хранение на диске**: включайте `Storage=persistent` на production-серверах — иначе после перезагрузки потеряете логи.
* **Ограничение размера**: в `/etc/systemd/journald.conf` задавайте `SystemMaxUse`, `SystemKeepFree`, `SystemMaxFileSize` и т.п., чтобы журналы не заняли весь диск.
* **Пишите в stdout/stderr**: сервисы должны писать логи в stdout/stderr — systemd-journald автоматически их поймает.
* **Используйте `-u` и `--since` для точной выборки** — это быстрее и читабельнее, чем grep по всем журналам.
* **JSON + jq** — удобный путь для интеграции в пайплайны / мониторинг.
* **Архивируйте важные журналы**: экспортируйте `-o export` и храните/передавайте бинарный файл, если нужно перенести или сохранить целостность.

---

## ✅ Итог

`journalctl` — гибкий и мощный инструмент: он удобен и для быстрого интерактивного просмотра логов, и для автоматизированного экспорта/анализа. Сочетание фильтров по юнитам, времени, приоритету и полям даёт точные и быстрые выборки — используйте `-u`, `--since/--until`, `-p`, `-o json` и `-f` как базовый набор при отладке.

---
