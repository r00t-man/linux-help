---
layout: default
title: "Проверка аудита"
permalink: /23_1_journalctl-audit-check/
parent: "⚙️ Логи/история — автоматизация"
nav_order: 3
---

# 🧾 Проверка аудита журнала и контроля состояния journald

Эта инструкция предназначена для системных администраторов и аудиторов, чтобы проверить корректность работы `systemd-journald`:  
сохранение логов, ротацию, защиту от переполнения и целостность журнала.

---

## 📑 Содержание

- [Проверка состояния службы journald](#check-journald-status)
- [Проверка статистики и использования места](#check-disk-usage)
- [Проверка применённых настроек journald](#check-config)
- [Проверка механизма ротации](#check-rotation)
- [Контроль защиты от переполнения](#check-overflow)
- [Проверка целостности логов](#check-integrity)
- [Проверка прав и доступа](#check-permissions)
- [Проверка активности записи логов](#check-live-logging)
- [Анализ производительности journald](#check-performance)
- [Контрольный чек-лист аудитора](#checklist)

---

<a id="check-journald-status"></a>
## ⚙️ Проверка состояния службы journald

Проверим, запущена ли служба и нет ли ошибок:

```bash
systemctl status systemd-journald
```

**Что проверить:**

* `Active: active (running)` — служба работает;
* нет ошибок `Failed to start`, `corrupted` или `no space left on device`;
* время аптайма journald (обычно совпадает с аптаймом системы).

---

<a id="check-disk-usage"></a>

## 💾 Проверка статистики и использования места

```bash
journalctl --disk-usage
```

Пример вывода:

```
Archived and active journals take up 512.0M on disk.
```

📊 Это реальный объём журналов на диске.
Сравни с лимитом `SystemMaxUse` в `/etc/systemd/journald.conf`.

---

<a id="check-config"></a>

## 🧩 Проверка применённых настроек journald

```bash
systemd-analyze cat-config systemd/journald.conf
```

Показывает **все текущие параметры**, включая дефолтные и переопределённые.
Полезно убедиться, что:

* `Storage=persistent` — логи сохраняются на диск;
* `SystemMaxUse` и `SystemKeepFree` заданы;
* `MaxRetentionSec` определяет срок хранения.

> 💡 Если `Storage=volatile`, создайте постоянный каталог:
>
> ```bash
> sudo mkdir -p /var/log/journal
> sudo systemd-tmpfiles --create --prefix /var/log/journal
> sudo systemctl restart systemd-journald
> ```
> Второй шаг (`systemd-tmpfiles --create`) выставляет правильные владельца/группу (`root:systemd-journal`) и setgid — без него каталог от голого `mkdir` останется с дефолтными правами.

---

<a id="check-rotation"></a>

## 🔁 Проверка механизма ротации

Можно проверить работу встроенного механизма очистки вручную:

```bash
sudo journalctl --vacuum-time=7d
```

Удалит все записи старше 7 дней.
Также можно ограничить размер:

```bash
sudo journalctl --vacuum-size=500M
```

Повтори `journalctl --disk-usage` — размер должен уменьшиться.

---

<a id="check-overflow"></a>

## 🚫 Контроль защиты от переполнения

Проверь, установлен ли лимит:

```bash
sudo systemd-analyze cat-config systemd/journald.conf | grep SystemMaxUse
```

Если лимит достигнут — journald автоматически удаляет старые записи.
Проверить факт ротации можно по логам самой службы:

```bash
journalctl -u systemd-journald | grep -i "rotat" -A3
```

> [!WARNING]
> Проверено вживую: реальное сообщение journald при ротации — **`Received client request to rotate journal, rotating.`** Слова "rotation" (с окончанием `-ion`) там нет — `grep rotation` (как было в исходном варианте статьи) не находит вообще ничего, даже сразу после подтверждённой ротации (`journalctl --rotate`). Используйте `grep -i "rotat"` (общий корень слова), чтобы поймать и "rotate", и "rotating".

---

<a id="check-integrity"></a>

## 🔐 Проверка целостности логов

Проверим, нет ли повреждённых файлов:

```bash
sudo journalctl --verify
```

Пример:

```
PASS: /var/log/journal/123abc/system.journal
```

Если есть ошибки — возможно, была некорректная перезагрузка или сбой диска.

---

<a id="check-permissions"></a>

## 🧱 Проверка прав и доступа

```bash
ls -ld /var/log/journal
ls -l /var/log/journal/*/
```

Убедись, что:

* владелец — `root`;
* группа — `systemd-journal`;
* права: `drwxr-sr-x`.

Если обычный пользователь должен читать логи:

```bash
sudo usermod -aG systemd-journal <username>
```

---

<a id="check-live-logging"></a>

## 📡 Проверка активности записи логов

```bash
sudo journalctl -f
```

(аналог `tail -f`)

В другом терминале вызови, например:

```bash
sudo systemctl restart ssh    # Astra Linux — юнит "ssh"
sudo systemctl restart sshd   # РЕД ОС — юнит "sshd"
```

Если появляются новые строки — journald работает корректно.

---

<a id="check-performance"></a>

## 🕓 Анализ производительности journald

Проверим, насколько быстро journald стартует и корректен ли юнит:

```bash
systemd-analyze blame | grep journal
systemd-analyze verify systemd-journald.service
```

**Вывод покажет:**

* время запуска journald;
* есть ли ошибки в юните;
* не блокирует ли journald загрузку системы.

---

<a id="checklist"></a>

## ✅ Контрольный чек-лист аудитора

| Проверка                 | Команда                                           | Ожидаемый результат                 |
| ------------------------ | ------------------------------------------------- | ----------------------------------- |
| Служба активна           | `systemctl status systemd-journald`               | `active (running)`                  |
| Логи сохраняются на диск | `journalctl --disk-usage`                         | > 0 и не переполнено                |
| Настройки применены      | `systemd-analyze cat-config`                      | `Storage=persistent`, лимиты заданы |
| Ротация работает         | `journalctl --vacuum-time=7d`                     | Старые логи удалены                 |
| Целостность проверена    | `journalctl --verify`                             | `PASS`                              |
| Права доступа корректны  | `ls -ld /var/log/journal`                         | root:systemd-journal                |
| Логи пишутся             | `journalctl -f`                                   | Новые события видны                 |
| Конфигурация корректна   | `systemd-analyze verify systemd-journald.service` | Без ошибок                          |

---

💡 **Совет:**
Для автоматического аудита можно создать еженедельный скрипт, который выполняет эти проверки и записывает результат в файл `/home/gis-audit/journal_audit_$(hostname)_$(date +%F).log`.
