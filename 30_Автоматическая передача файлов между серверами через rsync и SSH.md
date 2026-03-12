---
layout: default
title: "Автоматическая передача файлов между серверами через rsync и SSH"
permalink: /rsync-ssh-auto-transfer/
---

# 🔁 Автоматическая передача файлов между серверами через `rsync` и SSH

[![Platform](https://img.shields.io/badge/platform-Linux-lightgrey?style=flat-square&logo=linux)]()
[![Category](https://img.shields.io/badge/category-Files%20%26%20Backup-blue?style=flat-square)]()
[![Tools](https://img.shields.io/badge/tools-rsync%20%7C%20ssh%20%7C%20cron-orange?style=flat-square)]()

> Полная инструкция: безопасно настроить синхронизацию между серверами, автоматизировать её, логировать и контролировать ошибки.

---

## Когда применять этот подход

Подходит, если нужно:

- регулярно передавать файлы/бэкапы между Linux-серверами;
- передавать **только изменения** (экономия трафика);
- работать через защищённый канал SSH;
- запускать по расписанию (`cron` или `systemd timer`).

---

## 1) Подготовка серверов

Предположим:

- источник: `src-host`;
- приёмник: `backup-host`;
- пользователь на приёмнике: `backupuser`;
- исходные данные: `/data/source/`;
- целевая директория: `/data/backup/source/`.

### Проверка, что `rsync` установлен

На **обоих** серверах:

```bash
rsync --version
ssh -V
```

Если не установлен:

```bash
# Debian/Ubuntu
sudo apt update && sudo apt install -y rsync openssh-client

# RHEL/Rocky/Alma
sudo dnf install -y rsync openssh-clients
```

---

## 2) Создание отдельного SSH-ключа для синхронизации

На `src-host`:

```bash
ssh-keygen -t ed25519 -a 100 -f ~/.ssh/rsync_sync -C "rsync-sync@src-host"
chmod 600 ~/.ssh/rsync_sync
chmod 644 ~/.ssh/rsync_sync.pub
```

Передайте публичный ключ на `backup-host`:

```bash
ssh-copy-id -i ~/.ssh/rsync_sync.pub backupuser@backup-host
```

Проверьте вход:

```bash
ssh -i ~/.ssh/rsync_sync backupuser@backup-host 'hostname && whoami'
```

> [!TIP]
> Для production лучше использовать отдельного пользователя под бэкапы, без shell-доступа к критичным командам.

---

## 3) Первичная ручная синхронизация

```bash
rsync -avz --delete \
  -e "ssh -i ~/.ssh/rsync_sync" \
  /data/source/ backupuser@backup-host:/data/backup/source/
```

Разбор ключей:

- `-a` — архивный режим (права, время, рекурсия и т.д.);
- `-v` — подробный вывод;
- `-z` — сжатие по сети;
- `--delete` — удалять на приёмнике то, чего уже нет на источнике.

> [!IMPORTANT]
> `--delete` мощный и опасный ключ: сначала всегда тестируйте с `--dry-run`.

Тест без изменений:

```bash
rsync -avz --delete --dry-run \
  -e "ssh -i ~/.ssh/rsync_sync" \
  /data/source/ backupuser@backup-host:/data/backup/source/
```

---

## 4) Надёжный production-скрипт

Создайте файл `/usr/local/bin/rsync-sync.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

SRC_DIR="/data/source/"
DST_HOST="backupuser@backup-host"
DST_DIR="/data/backup/source/"
SSH_KEY="$HOME/.ssh/rsync_sync"
LOG_FILE="/var/log/rsync-sync.log"
LOCK_FILE="/tmp/rsync-sync.lock"

# Не запускать второй экземпляр одновременно
exec 9>"$LOCK_FILE"
if ! flock -n 9; then
  echo "[$(date '+%F %T')] another sync is running" >> "$LOG_FILE"
  exit 1
fi

# Проверка доступности приёмника
if ! ssh -i "$SSH_KEY" -o BatchMode=yes -o ConnectTimeout=10 "$DST_HOST" "echo ok" >/dev/null 2>&1; then
  echo "[$(date '+%F %T')] destination is unreachable" >> "$LOG_FILE"
  exit 2
fi

# Синхронизация
rsync -aHAX --delete --numeric-ids --stats \
  --partial --partial-dir=.rsync-partial \
  --human-readable \
  -e "ssh -i ${SSH_KEY} -o StrictHostKeyChecking=accept-new" \
  "$SRC_DIR" "${DST_HOST}:${DST_DIR}" >> "$LOG_FILE" 2>&1

echo "[$(date '+%F %T')] sync finished successfully" >> "$LOG_FILE"
```

Сделайте исполняемым:

```bash
sudo chmod +x /usr/local/bin/rsync-sync.sh
```

Создайте лог-файл:

```bash
sudo touch /var/log/rsync-sync.log
sudo chmod 640 /var/log/rsync-sync.log
```

---

## 5) Автозапуск через cron

Откройте crontab:

```bash
crontab -e
```

Пример запуска каждые 15 минут:

```cron
*/15 * * * * /usr/local/bin/rsync-sync.sh
```

Пример запуска каждый день в 02:30:

```cron
30 2 * * * /usr/local/bin/rsync-sync.sh
```

Проверка:

```bash
crontab -l
tail -n 50 /var/log/rsync-sync.log
```

---

## 6) Альтернатива: systemd service + timer (надёжнее cron)

### `rsync-sync.service`

`/etc/systemd/system/rsync-sync.service`:

```ini
[Unit]
Description=Rsync backup sync job
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
User=YOUR_USER
ExecStart=/usr/local/bin/rsync-sync.sh
```

### `rsync-sync.timer`

`/etc/systemd/system/rsync-sync.timer`:

```ini
[Unit]
Description=Run rsync sync every 15 minutes

[Timer]
OnBootSec=5min
OnUnitActiveSec=15min
Persistent=true

[Install]
WantedBy=timers.target
```

Активация:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now rsync-sync.timer
systemctl list-timers | grep rsync-sync
```

---

## 7) Безопасность и hardening

### Ограничение ключа в `authorized_keys`

На приёмнике в `~backupuser/.ssh/authorized_keys` можно ограничить ключ:

```text
from="10.10.10.5",no-agent-forwarding,no-port-forwarding,no-pty,no-user-rc ssh-ed25519 AAAA... rsync-sync@src-host
```

### Chroot/ограниченный пользователь

Для критичных сред:

- выделяйте отдельный аккаунт только для бэкапов;
- ограничивайте доступ к директориям;
- блокируйте интерактивный shell.

### Firewall

Разрешайте SSH только с конкретного IP источника.

---

## 8) Полезные сценарии rsync

### Исключить мусор и кеши

Файл исключений `/etc/rsync-excludes.txt`:

```text
*.tmp
*.cache
node_modules/
.git/
```

Использование:

```bash
rsync -a --delete --exclude-from=/etc/rsync-excludes.txt \
  -e "ssh -i ~/.ssh/rsync_sync" \
  /data/source/ backupuser@backup-host:/data/backup/source/
```

### Ограничить скорость передачи

```bash
rsync -a --bwlimit=10m -e "ssh -i ~/.ssh/rsync_sync" \
  /data/source/ backupuser@backup-host:/data/backup/source/
```

### Синхронизировать только конкретный тип файлов

```bash
rsync -a --include='*/' --include='*.sql.gz' --exclude='*' \
  -e "ssh -i ~/.ssh/rsync_sync" \
  /backups/ backupuser@backup-host:/data/backup/sql/
```

---

## 9) Диагностика проблем

### Ошибка `Permission denied (publickey)`

Проверьте:

```bash
ls -l ~/.ssh/rsync_sync*
ssh -vvv -i ~/.ssh/rsync_sync backupuser@backup-host
```

Частые причины:

- неверные права на ключи;
- ключ не добавлен в `authorized_keys`;
- `sshd_config` запрещает авторизацию ключами.

### Ошибка `connection timed out`

```bash
ping -c 3 backup-host
nc -zv backup-host 22
```

Проверьте маршрутизацию, firewall и доступность sshd.

### Ошибка удаления из-за `--delete`

Сначала всегда запускайте:

```bash
rsync -av --delete --dry-run -e "ssh -i ~/.ssh/rsync_sync" /data/source/ backupuser@backup-host:/data/backup/source/
```

---

## 10) Проверка восстановления (обязательно)

Бэкап без теста восстановления — риск.

Периодически делайте проверку:

1. Выберите тестовый каталог.
2. Восстановите на отдельный путь.
3. Сверьте контрольные суммы и права.

Пример:

```bash
mkdir -p /tmp/restore-test
rsync -a -e "ssh -i ~/.ssh/rsync_sync" \
  backupuser@backup-host:/data/backup/source/projectA/ /tmp/restore-test/projectA/
```

---

## 11) Краткий чек-лист

- [ ] Используется отдельный SSH-ключ для синхронизации.
- [ ] Запуск автоматизирован (cron/systemd timer).
- [ ] Включены логи и защита от параллельного запуска.
- [ ] Перед `--delete` используется `--dry-run`.
- [ ] Периодически выполняется тест восстановления.

Готово: теперь синхронизация файлов между серверами работает регулярно, безопасно и предсказуемо ✅
