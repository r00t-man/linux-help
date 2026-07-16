---
layout: default
title: "Systemd"
permalink: /19_systemctl-guide/
parent: "🔧 Службы"
nav_order: 1
---

# 🔧 Полный гайд по `systemctl` и unit-файлам (systemd)

---

## 📑 Содержание

- [Что такое systemd / systemctl](#what-is-systemd)
- [Основные команды systemctl](#basic-commands)
- [Где хранятся unit-файлы и их приоритеты](#unit-locations)
- [Структура unit-файла (service) — разбираем секции](#unit-structure)
- [Создание собственного .service — пошагово](#creating-service)
- [Шаблонные unit-файлы (instance units) — `@`-юниты](#template-units)
- [Drop-in, override и `systemctl edit`](#dropin-override)
- [Mask / Unmask — зачем маскировать службы](#masking)
- [Таймеры (`.timer`) и socket-активация (`.socket`)](#timers-sockets)
- [Жизненный цикл службы: start/stop/restart/reload и нюансы](#lifecycle)
- [Параметры перезапуска и поведение при сбоях](#restart-options)
- [Безопасность и ограничение привилегий unit-ов](#security)
- [Отладка и логирование (journalctl, systemctl status)](#debugging)
- [Полезные приёмы и чеклист для продакшн-служб](#tips)
- [Примеры: несколько готовых unit-файлов](#examples)
- [Ссылки / ресурсы (коротко)](#links)

---

<a id="what-is-systemd"></a>
## Что такое systemd / systemctl

`systemd` — это init-система и менеджер служб, которая управляет запуском системы, служб, сокетов, таймеров и т.д. `systemctl` — основной инструмент для взаимодействия с `systemd` (запуск/остановка/включение/отладка юнитов).

`systemd` использует *unit*-файлы (юниты) разного типа: `.service`, `.socket`, `.timer`, `.target`, `.mount`, `.path` и другие.

---

<a id="basic-commands"></a>
## Основные команды `systemctl`

Краткий справочник — самое часто используемое:

- `systemctl start <unit>` — запустить юнит сразу.
- `systemctl stop <unit>` — остановить юнит.
- `systemctl restart <unit>` — перезапустить (stop → start).
- `systemctl reload <unit>` — отправить службе сигнал для перечитывания конфигурации (если поддерживает).
- `systemctl enable <unit>` — включить автозапуск (при старте системы создаёт symlink).
- `systemctl disable <unit>` — отключить автозапуск.
- `systemctl enable --now <unit>` — включить и запустить сейчас.
- `systemctl is-enabled <unit>` — вернуть `enabled/disabled/static/masked`.
- `systemctl is-active <unit>` — вернуть `active/inactive/failed`.
- `systemctl status <unit>` — статус + последние логи.
- `systemctl list-units --type=service` — список текущих юнитов (запущенных/активных).
- `systemctl list-unit-files` — список всех доступных unit-файлов и их состояний (enabled/disabled/...).
- `systemctl daemon-reload` — перечитать unit-файлы после изменений на диске.
- `systemctl daemon-reexec` — перезапустить сам systemd (редко используется).
- `systemctl mask <unit>` — замаскировать (сделать невозможным запуск).
- `systemctl unmask <unit>` — снять маску.
- `systemctl reset-failed` — сбросить состояние failed (по одному или всем).
- `systemctl cat <unit>` — показать содержимое юнита с drop-in'ами.
- `systemctl edit <unit>` — открыть drop-in для переопределения (без редактирования основного файла).
- `systemctl link <path-to-unit>` — зарегистрировать юнит из произвольного пути.
- `systemctl kill <unit> --kill-who=main --signal=SIGTERM` — послать сигнал процессам юнита.

---

<a id="unit-locations"></a>
## Где хранятся unit-файлы и приоритет (важно)

Типичные пути (приоритет сверху вниз — чем выше, тем важнее):

1. `/etc/systemd/system/` — администратора: локальные изменения и overrides. (высокий приоритет)
2. `/run/systemd/system/` — runtime-юниты, генерируются в процессе работы.
3. `/lib/systemd/system/` или `/usr/lib/systemd/system/` — поставщик пакета (поставляются дистрибутивом). (низкий приоритет)

Также пользовательские юниты:
- `~/.config/systemd/user/` — юниты для `systemd --user`.
- `systemctl --user` — управляет юнитами пользователя.

**Важно:** правка файлов в `/lib/systemd/system/` не рекомендуется — при обновлении пакета изменения могут быть перезаписаны. Правильный способ — поместить override в `/etc/systemd/system/<unit>.d/` или использовать `systemctl edit`.

---

<a id="unit-structure"></a>
## Структура unit-файла (на примере `.service`)

Unit-файл — обычный INI-файл с секциями:

```ini
[Unit]
Description=My background service
Documentation=man:myservice(8)
After=network.target
Requires=network.target
Wants=postgresql.service

[Service]
Type=simple
User=svcuser
Group=svcgroup
WorkingDirectory=/opt/myapp
Environment=ENV=prod
EnvironmentFile=/etc/default/myapp
ExecStart=/usr/bin/myapp --config /etc/myapp/config.yml
ExecStartPre=/usr/bin/myapp-prep
ExecStop=/bin/kill -TERM $MAINPID
Restart=on-failure
RestartSec=5
LimitNOFILE=65536
PrivateTmp=true
ProtectSystem=full

[Install]
WantedBy=multi-user.target
```

Ключевые секции и опции:

* `[Unit]`

  * `Description=` — описание.
  * `Documentation=` — ссылки на документацию.
  * `After=` / `Before=` — порядок старта (логический, не зависит от зависимости).
  * `Requires=` — жёсткая зависимость: если Required не запустился — и этот unit не будет запущен.
  * `Wants=` — мягкая зависимость (не критична).
  * `Condition...` — условия запуска (например `ConditionPathExists=`).

* `[Service]`

  * `Type=` — `simple` (по умолчанию), `forking`, `oneshot`, `notify`, `dbus`.
  * `ExecStart=` — команда запуска.
  * `ExecStartPre=` / `ExecStartPost=` — команды до/после старта.
  * `ExecReload=` — команда для reload.
  * `ExecStop=` — команда для остановки.
  * `Restart=` — `no`, `on-success`, `on-failure`, `always`, `on-abnormal`, `on-watchdog`, `on-abort`.
  * `RestartSec=` — пауза перед рестартом.
  * `User=` / `Group=` — под каким пользователем запускать.
  * `Environment=` / `EnvironmentFile=` — переменные окружения.
  * Ограничения/безопасность: `PrivateTmp=`, `ProtectSystem=`, `ProtectHome=`, `NoNewPrivileges=`, `CapabilityBoundingSet=`, `ReadOnlyDirectories=`, и т.д.

* `[Install]`

  * `WantedBy=` — в какой таргет добавить symlink при enable (обычно `multi-user.target`).
  * `Alias=` — дополнительные имена.

---

<a id="creating-service"></a>

## Создание собственного `.service` — шаги и рекомендации

1. **Создаём unit** в `/etc/systemd/system/`:

   ```bash
   sudo tee /etc/systemd/system/myapp.service > /dev/null <<'EOF'
   [Unit]
   Description=MyApp background service
   After=network.target

   [Service]
   Type=simple
   User=myapp
   Group=myapp
   WorkingDirectory=/opt/myapp
   ExecStart=/usr/bin/myapp --config /etc/myapp/config.yml
   Restart=on-failure
   RestartSec=5
   LimitNOFILE=65536
   PrivateTmp=true

   [Install]
   WantedBy=multi-user.target
   EOF
   ```

2. **Перечитать конфигурацию systemd:**

   ```bash
   sudo systemctl daemon-reload
   ```

3. **Включить автозапуск и запустить:**

   ```bash
   sudo systemctl enable --now myapp.service
   ```

4. **Проверить статус и логи:**

   ```bash
   sudo systemctl status myapp.service
   sudo journalctl -u myapp.service -b --no-pager
   ```

**Советы:**

* Не храните рабочие каталоги и логи в `/root` — используйте отдельного пользователя.
* Для демонов, которые форкают процесс, ставьте `Type=forking`. Для программ, которые не форкают — `Type=simple`.
* Для сервисов, которые сообщают systemd о готовности, используйте `Type=notify` и библиотеку sd_notify.

---

<a id="template-units"></a>

## Шаблонные unit-файлы (instance units) — `@`-юниты

Шаблонный unit позволяет запускать несколько экземпляров одной службы с разными параметрами: `myservice@instance.service`.

**Пример `/etc/systemd/system/web@.service`:**

```ini
[Unit]
Description=Web instance %i
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/web --name %i --config /etc/web/%i.conf

[Install]
WantedBy=multi-user.target
```

Запуск экземпляра:

```bash
sudo systemctl start web@front.service
sudo systemctl enable --now web@front.service
```

В шаблоне `%i` — «instance name» (часть после `@`), `%f` — с escape, `%n` — полное имя юнита.

---

<a id="dropin-override"></a>

## Drop-in и `systemctl edit` — безопасная кастомизация

Вместо изменения оригинальных файлов пакета используйте drop-in или `systemctl edit`:

* `systemctl edit myapp.service` — откроет временный файл в `$EDITOR` и создаст drop-in в `/etc/systemd/system/myapp.service.d/override.conf` после сохранения.
* Вы можете перекрывать отдельные строки, например:

```ini
[Service]
Environment=NEW_VAR=1
Restart=always
```

* Чтобы посмотреть итоговый файл с учётом всех drop-in:

  ```bash
  systemctl cat myapp.service
  ```

---

<a id="masking"></a>

## Mask / Unmask — зачем маскировать службы

`mask` создаёт символическую ссылку `… -> /dev/null`, после чего `systemctl start` и другие команды не смогут запустить юнит (даже если попытаться зависимостью). Это надёжный способ **запретить** запуск сервиса (вручную или автоматически).

```bash
sudo systemctl mask foo.service
# Проверка:
systemctl is-enabled foo.service   # вернёт "masked"
# Снятие маски:
sudo systemctl unmask foo.service
```

**Когда использовать:**

* В проде, если служба конфликтует и её нельзя допустить к запуску.
* При миграциях, чтобы предотвратить случайный запуск.

**Не путать с `disable`**: `disable` лишь убирает автозапуск, `mask` — запрещает вообще.

---

<a id="timers-sockets"></a>

## Таймеры и socket-активация

### `.timer` — планировщик systemd (замена cron для юнитов)

Создаёт юнит, который вызывает `.service` по расписанию или через OnBootSec/OnActiveSec.

**Пример: `/etc/systemd/system/backup.timer` и `backup.service`**

`backup.timer`:

```ini
[Unit]
Description=Daily backup timer

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

`backup.service` — обычный сервис, исполняющий бэкап.

Запуск:

```bash
sudo systemctl enable --now backup.timer
systemctl list-timers
```

### `.socket` — socket-activated services

systemd слушает сокет и запускает службу по подключению. Это экономит ресурсы и полезно для on-demand сервисов.

`myapp.socket`:

```ini
[Socket]
ListenStream=12345

[Install]
WantedBy=sockets.target
```

`myapp@.service` — можно настроить `StandardInput=socket` или использовать шаблон.

---

<a id="lifecycle"></a>

## Жизненный цикл службы: нюансы

* `start` — запускает юнит.
* `stop` — останавливает.
* `restart` — stop → start; полезно, когда нельзя reload.
* `reload` — попросить службу перечитать конфигурацию (если `ExecReload=` задан и программа поддерживает).
* `try-restart` — перезапустить только если служба запущена.
* `condrestart` — как try-restart.
* `reload-or-restart` — reload если поддерживает, иначе restart.

**Про `daemon-reload`**: После любого изменения unit-файла или добавления — выполнить `systemctl daemon-reload`. Это *не* перезапускает службы — только перечитывает конфигурацию systemd.

---

<a id="restart-options"></a>

## Параметры перезапуска и поведение при падении

`Restart=` — ключевой для надежности:

* `no` — не перезапускать
* `on-success` — при успешном завершении
* `on-failure` — при не-нулевом коде, сигнале, таймауте
* `on-abnormal` — если завершение из-за сигнала/ошибки
* `on-abort`
* `on-watchdog` — если сработал watchdog-таймаут (пара к `WatchdogSec=` из чеклиста ниже — в исходной версии статьи это значение отсутствовало, хотя `WatchdogSec=` там же и рекомендуется)
* `always` — всегда перезапускать

Полезные опции:

* `RestartSec=` — время ожидания перед перезапуском.
* `StartLimitIntervalSec=` и `StartLimitBurst=` — лимиты попыток запуска (защищают от спама рестартов).

Если служба «падает» часто, systemd пометит её `failed`; `systemctl reset-failed` сбросит флаг.

---

<a id="security"></a>

## Безопасность: ограничиваем привилегии unit-ов

Systemd предоставляет много опций безопасности — используй их по мере возможности:

* Пользователь/группа: `User=`, `Group=`.
* `NoNewPrivileges=true` — запрет на повышение привилегий через execve.
* `PrivateTmp=true` — отдельный /tmp для сервиса.
* `ProtectSystem=` — три уровня, не два (проверено по `man systemd.exec`): `true`/`yes` — только `/usr` и загрузчик (`/boot`, `/efi`) на чтение; `full` — то же + дополнительно `/etc` тоже на чтение (это и есть отличие full от true, в исходной версии статьи `/etc` не упоминался вообще); `strict` — вся файловая система на чтение, кроме `/dev`, `/proc`, `/sys`.
* `ProtectHome=` — это НЕ один режим с двумя описаниями, а разные значения с разным эффектом: `true` — `/home`, `/root`, `/run/user` становятся **недоступны и пусты** (не просто read-only!); `read-only` — те же три каталога именно на чтение; `tmpfs` — временная пустая ФС поверх них.
* `ReadOnlyPaths=` / `ReadWritePaths=` — тонкая настройка (актуальные имена; `ReadOnlyDirectories=`/`ReadWriteDirectories=` из старых версий статьи — устаревшие имена этих же опций, в текущем `man systemd.exec` уже не упоминаются).
* `CapabilityBoundingSet=` — набор доступных Linux-capabilities.
* `RestrictAddressFamilies=` — ограничение семей сокетов.
* `SystemCallFilter=` — белый/чёрный список syscalls (при поддержке ядра).
* `SELinuxContext=` — задавать SELinux контекст (если используется).

**Рекомендация:** по умолчанию запускай сервисы от неправавного пользователя и используй `ProtectSystem`, `PrivateTmp` и `NoNewPrivileges` когда возможно.

---

<a id="debugging"></a>

## Отладка и логирование

* `systemctl status <unit>` — кратко статус + последние строки journal.
* `journalctl -u <unit>` — все логи юнита.

  * `journalctl -u <unit> -b` — с текущей загрузки.
  * `journalctl -u <unit> -f` — follow, как `tail -f`.
  * `journalctl -u <unit> --since "2025-10-01 10:00"` — по времени.
* `systemctl --failed` — показать все упавшие юниты.
* `systemctl show <unit>` — выводит свойства в формате key=value (удобно для скриптов).
* `systemd-analyze blame` и `systemd-analyze critical-chain` — узнать, что тормозит загрузку (полезно для `boot` оптимизаций).
* `strace`, `gdb` — при необходимости для самого процесса (не в systemd).

**Проверка переменных окружения**:

* `systemctl show-environment` — для systemd-environment.
* Логи часто содержат переменные среды, переданные сервису.

---

<a id="tips"></a>

## Полезные приёмы и чеклист для продакшн-служб

1. **Drop-in вместо редактирования пакета** — используем `systemctl edit`.
2. **Unit должен корректно реагировать на SIGTERM/SIGINT** — systemd шлёт SIGTERM при stop.
3. **Укажите `Restart=on-failure`** и корректные `RestartSec=`.
4. **Ограничте ресурсы**: `LimitNOFILE=`, `LimitNPROC=`, `CPUQuota=` при необходимости.
5. **Логи**: пишите в stdout/stderr — systemd capture через journal.
6. **Healthcheck**: используйте `WatchdogSec=` + `Type=notify` для автоматического обнаружения зависания.
7. **Тестирование**: проверяйте `systemctl daemon-reload` → `systemctl start` → `journalctl -u`.
8. **Документируйте** `Description=` и `Documentation=` в unit-файле.
9. **Тестовый режим**: `systemctl start --no-block` при асинхронных стартах, но чаще — избегать.
10. **CI/CD**: при доставке юнитов через пакетный менеджер используйте `systemctl preset` или `systemctl enable` в postinst.

---

<a id="examples"></a>

## Примеры unit-файлов

### 1) Простая web-служба (`/etc/systemd/system/myweb.service`)

```ini
[Unit]
Description=Simple HTTP server
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/srv/myweb
ExecStart=/usr/bin/python3 -m http.server 8080
Restart=on-failure
PrivateTmp=true
ProtectSystem=full

[Install]
WantedBy=multi-user.target
```

> [!NOTE]
> `www-data` — стандартный веб-пользователь Astra Linux/Debian. На РЕД ОС (RHEL-семейство) такого пользователя обычно нет — там типично `nginx`/`apache`, либо создайте свой: `sudo useradd -r -s /sbin/nologin myweb`.

### 2) Форкающийся демон (`Type=forking`)

```ini
[Unit]
Description=Legacy daemon

[Service]
Type=forking
ExecStart=/usr/sbin/legacy-daemon -D
PIDFile=/var/run/legacy.pid
Restart=on-failure
```

### 3) Таймер + service (`backup.timer` + `backup.service`)

`backup.timer`:

```ini
[Unit]
Description=Daily backup timer

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

`backup.service` — запускает скрипт бэкапа.

---

<a id="links"></a>

## Короткие ресурсы для дальнейшего чтения

* `man systemd.unit`, `man systemd.service`, `man systemctl`, `man systemd.exec`

---

## Заключение

Этот гайд даёт фундамент для:

* безопасного и корректного создания unit-файлов;
* надежного управления службами (enable/disable, mask/unmask);
* использования шаблонов, таймеров и socket-активации;
* настройки безопасности и отладки через `journalctl`.
