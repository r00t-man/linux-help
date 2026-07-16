---
layout: default
title: "Ротация логов"
permalink: /23_log-rotation-native-only/
---

# ♻️ Ротация логов встроенными средствами (без logrotate)

В Astra Linux и Red OS можно управлять логами **полностью штатными средствами**,  
без использования `logrotate`.  
Основой является системный компонент **`systemd-journald`**,  
который автоматически собирает, хранит, сжимает и очищает логи системы.

---

## 📑 Содержание

- [1. Что такое systemd-journald](#what)
- [2. Где хранятся логи](#where)
- [3. Управление ротацией и хранением](#rotation)
- [4. Настройка journald.conf (практика)](#config)
- [5. Очистка и ручное управление логами](#cleanup)
- [6. Проверка текущего состояния журнала](#check)
- [7. Настройки для production-серверов](#prod)
- [8. Дополнительные команды для анализа](#commands)
- [9. Советы и рекомендации](#tips)

---

<a id="what"></a>
## 🧩 1️⃣ Что такое systemd-journald

`systemd-journald` — это системный демон, который:

- собирает все системные и служебные сообщения (ядро, systemd, rsyslog, приложения);
- хранит логи в бинарном виде (вместо текстовых файлов);
- управляет объёмом, сроком и сжатием логов;
- имеет встроенную ротацию (без logrotate);
- умеет писать логи в оперативную память (RAM) или на диск.

---

<a id="where"></a>
## 📂 2️⃣ Где хранятся логи

| Тип хранилища | Путь | Описание |
|----------------|------|----------|
| В памяти (по умолчанию) | `/run/log/journal/` | очищается при перезагрузке |
| На диске (persistent) | `/var/log/journal/` | сохраняется между перезагрузками |

Постоянное хранение включается параметром:
```bash
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal
sudo systemctl restart systemd-journald
```

> [!NOTE]
> Шаг `systemd-tmpfiles --create` не косметика — он выставляет правильного владельца и группу (`root:systemd-journal`) и setgid-бит на каталоге по правилам из `tmpfiles.d`. Без него каталог, созданный голым `mkdir`, останется с дефолтными правами — journald при этом обычно всё равно продолжит работать от root, но пользователи из группы `systemd-journal` не получат штатный доступ на чтение логов.

---

<a id="rotation"></a>

## ♻️ 3️⃣ Управление ротацией и хранением

Ротация и очистка логов в journald осуществляется **автоматически**,
на основе параметров из файла `/etc/systemd/journald.conf`.

### Основные механизмы:

* **Ограничение по размеру (`SystemMaxUse`)**
* **Ограничение по времени (`MaxRetentionSec`)**
* **Резервирование свободного места (`SystemKeepFree`)**
* **Сжатие старых логов (`Compress=yes`)**
* **Автоматическая ротация при достижении лимита**

---

<a id="config"></a>

## ⚙️ 4️⃣ Настройка journald.conf (практика)

Открой файл конфигурации:

```bash
sudo nano /etc/systemd/journald.conf
```

### Пример конфигурации для постоянного хранения логов:

```conf
[Journal]
Storage=persistent         # хранить логи на диске
Compress=yes               # сжимать старые записи
Seal=yes                   # защита от подмены (см. предупреждение ниже — нужен доп. шаг)
SplitMode=uid              # разделять логи по пользователям
SystemMaxUse=2G            # общий лимит для логов
SystemKeepFree=1G          # оставлять свободное место
SystemMaxFileSize=200M     # максимальный размер одного файла
RuntimeMaxUse=256M         # ограничение для логов в памяти
MaxRetentionSec=3month     # хранить 3 месяца
ForwardToSyslog=yes        # дублировать в rsyslog (если включен)
```

> [!WARNING]
> `Seal=yes` сам по себе НЕ включает реальную защиту (Forward Secure Sealing) — по `man journald.conf`: sealing активируется только если **есть ключ**, а ключ создаётся отдельной командой `journalctl --setup-keys`. `Seal=yes` — это и так уже значение по умолчанию, без ключа оно ничего не меняет. Чтобы реально защититься от подмены логов:
> ```bash
> sudo journalctl --setup-keys
> ```

> [!NOTE]
> `ForwardToSyslog=yes` дублирует записи в системный syslog-сокет, а КУДА именно они попадут дальше — зависит от конфига самого rsyslog, не journald. На **Astra Linux** (Debian) это обычно `/var/log/syslog`, на **РЕД ОС** (RHEL-семейство) — `/var/log/messages` (тот же путь, что уже отмечался в других статьях этой вики).

Применить изменения:

```bash
sudo systemctl restart systemd-journald
```

Проверить состояние службы:

```bash
systemctl status systemd-journald
```

---

<a id="cleanup"></a>

## 🧹 5️⃣ Очистка и ручное управление логами

### Проверить объём занимаемого места:

```bash
sudo journalctl --disk-usage
```

### Удалить логи старше 14 дней:

```bash
sudo journalctl --vacuum-time=14d
```

### Удалить логи, если общий объём превышает 500 МБ:

```bash
sudo journalctl --vacuum-size=500M
```

### Принудительно выполнить ротацию журнала:

```bash
sudo journalctl --rotate
```

### Полностью очистить журнал (осторожно!):

```bash
sudo journalctl --rotate
sudo journalctl --vacuum-time=1s
```

---

<a id="check"></a>

## 🔍 6️⃣ Проверка текущего состояния журнала

### Сколько логов сохранено:

```bash
sudo journalctl --disk-usage
```

### Последние записи:

```bash
sudo journalctl -n 20
```

### По конкретной службе:

```bash
sudo journalctl -u ssh.service    # Astra Linux — юнит "ssh"
sudo journalctl -u sshd.service   # РЕД ОС — юнит "sshd"
```

### По времени:

```bash
sudo journalctl --since "2025-10-01" --until "2025-10-07"
```

### Только ошибки:

```bash
sudo journalctl -p err -b
```

---

<a id="prod"></a>

## 🏢 7️⃣ Настройки для production-серверов

Рекомендуемая конфигурация для серверов Astra / Red OS:

```conf
[Journal]
Storage=persistent
Compress=yes
SystemMaxUse=2G
SystemKeepFree=1G
SystemMaxFileSize=200M
RuntimeMaxUse=256M
MaxRetentionSec=1month
ForwardToSyslog=yes
```

* `persistent` — сохранять логи после перезагрузки
* `SystemMaxUse` — не допускать переполнения диска
* `MaxRetentionSec` — хранить не более 1 месяца
* `Compress=yes` — включить сжатие
* `ForwardToSyslog=yes` — дублировать логи в rsyslog, если он включён (Astra → `/var/log/syslog`, РЕД ОС → `/var/log/messages`)

---

<a id="commands"></a>

## 🧠 8️⃣ Дополнительные команды для анализа

| Задача                     | Команда                     |
| -------------------------- | --------------------------- |
| Просмотреть логи ядра      | `sudo journalctl -k`        |
| Только ошибки              | `sudo journalctl -p err`    |
| Сессия текущего запуска    | `sudo journalctl -b`        |
| Предыдущий запуск          | `sudo journalctl -b -1`     |
| По пользователю            | `sudo journalctl _UID=1000` |
| По process ID              | `sudo journalctl _PID=1234` |
| По unit (службе)           | `sudo journalctl -u nginx`  |
| Вывести в реальном времени | `sudo journalctl -f`        |

---

<a id="tips"></a>

## 💡 9️⃣ Советы и лучшие практики

✅ **Проверяй размер журналов регулярно**

```bash
du -sh /var/log/journal
journalctl --disk-usage
```

✅ **Используй `SystemKeepFree`** — оставляй запас свободного места на системном разделе.
Это защищает систему от переполнения `/var`.

✅ **Настраивай `MaxRetentionSec`** по требованиям безопасности (обычно 30–90 дней).

✅ **Для систем без rsyslog** достаточно journald —
он уже покрывает все функции системного журналирования и ротации.

✅ **Не удаляй файлы вручную из `/var/log/journal/`**,
используй только `journalctl --vacuum-*` — иначе база journald может повредиться.

✅ **В Astra Linux SE** каталог `/var/log/journal` может быть отключён по политике —
если логи не сохраняются, проверь параметр `Storage=` в конфиге.

---

## 🧩 Итог

🔹 В Astra Linux и Red OS встроенная система `systemd-journald` полностью заменяет внешние утилиты для ротации логов.
🔹 Все операции (сжатие, очистка, ограничение по размеру и времени) выполняются автоматически.
🔹 Настраивается всё в одном файле `/etc/systemd/journald.conf`.
🔹 Не требует установки `logrotate` или сторонних скриптов.

💡 Это безопасный и надёжный способ управления логами, рекомендованный для защищённых систем.

