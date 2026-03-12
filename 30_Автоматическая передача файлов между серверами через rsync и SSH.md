---
layout: default
title: "Автоматическая передача файлов между серверами через rsync и SSH"
permalink: /rsync-ssh-auto-transfer/
---

# 🔁 Автоматическая передача файлов между серверами через `rsync` и SSH

## 1) Подготовка SSH-ключа

```bash
ssh-keygen -t ed25519 -C "rsync-sync" -f ~/.ssh/rsync_sync
ssh-copy-id -i ~/.ssh/rsync_sync.pub user@remote-host
```

## 2) Тест синхронизации

```bash
rsync -avz --delete \
  -e "ssh -i ~/.ssh/rsync_sync" \
  /data/source/ user@remote-host:/data/backup/
```

## 3) Скрипт автоматизации

```bash
#!/usr/bin/env bash
set -euo pipefail

SRC_DIR="/data/source/"
DST_HOST="user@remote-host"
DST_DIR="/data/backup/"
SSH_KEY="$HOME/.ssh/rsync_sync"

rsync -az --delete --stats \
  -e "ssh -i ${SSH_KEY} -o StrictHostKeyChecking=accept-new" \
  "${SRC_DIR}" "${DST_HOST}:${DST_DIR}"
```

Сохраните как `/usr/local/bin/rsync-sync.sh` и сделайте исполняемым:

```bash
sudo chmod +x /usr/local/bin/rsync-sync.sh
```

## 4) Запуск по расписанию (cron)

```bash
crontab -e
```

```cron
*/15 * * * * /usr/local/bin/rsync-sync.sh >> /var/log/rsync-sync.log 2>&1
```

