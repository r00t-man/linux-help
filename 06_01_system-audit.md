---
layout: default
title: "Инфо о системе подробнее"
permalink: /06_01_system-audit/
---

# 🧩 Гайд по инвентаризации и аудиту Linux-систем

---

## 📑 Содержание

- [1️⃣ Просмотр всех служб в системе](#services)
- [2️⃣ Пользователи, группы и статусы учётных записей](#users)
- [3️⃣ Список установленных пакетов](#packages)
- [4️⃣ Информация о дистрибутиве и версии ОС](#os-info)
- [🧩 Дополнительные полезные команды](#extra)
- [💾 Автоматизация аудита](#automation)
- [📘 Итог](#summary)

---

## 📋 Быстрая таблица команд

| Раздел | Команда | Назначение | Полезные опции |
|--------|----------|-------------|----------------|
| 1️⃣ Службы | `systemctl list-unit-files` | Показать все юниты (службы, сокеты, таргеты и т.д.) | `--type=service` — только службы |
| 2️⃣ Пользователи | `getent passwd` + `passwd -S` + `id` | Вывести пользователей, их группы и статус УЗ | Цветной вывод — через `sed` |
| 3️⃣ Пакеты (Debian/Ubuntu/Astra) | `dpkg-query -l` | Список установленных пакетов | `dpkg-query -L <pkg>` — файлы пакета |
| 3️⃣ Пакеты (РЕД ОС) | `dnf list installed` или `rpm -qa` | Список установленных пакетов | `grep <pkg>` — фильтр по имени |
| 4️⃣ Информация о дистрибутиве | `lsb_release -a`, `/etc/astra/*`, `/etc/redos-release` | Узнать версию ОС | `cat /etc/os-release` — универсальный способ |

---

<a id="services"></a>
## ⚙️ 1. Просмотр всех служб в системе

Показать список всех юнитов (служб, сокетов, целей, таймеров и пр.):

```bash
systemctl list-unit-files
```

🧠 **Описание столбцов:**

* **UNIT FILE** — имя файла юнита (`.service`, `.socket`, `.target`, и др.)
* **STATE** — текущее состояние юнита:

  * `enabled` — включен, запускается при старте системы
  * `disabled` — выключен
  * `static` — не может быть включен вручную, используется другими юнитами
  * `masked` — полностью заблокирован (не может быть запущен)

📂 **Сохранение в файл (удобно при массовом аудите):**

```bash
HOSTNAME=$(hostname)
systemctl list-unit-files > /home/gis-audit/services_list_${HOSTNAME}
```

💡 **Совет:**
Для вывода только активных служб:

```bash
systemctl list-units --type=service --state=running
```

---

<a id="users"></a>

## 👥 2. Пользователи, группы и статусы учётных записей

Показать всех пользователей, их группы и статус пароля:

```bash
sudo -s   # если ещё не под root
getent passwd | cut -d: -f1 | xargs -n1 -I {} sh -c 'passwd -S {}; id {}'
```

> [!IMPORTANT]
> Проверено вживую: `passwd -S` для ЧУЖОГО пользователя требует root — обычным пользователем команда вернёт `passwd: You may not view or modify password information for <user>.` для всех, кроме себя самого. Без `sudo`/root этот аудит покажет не реальные статусы, а стену ошибок доступа — выполняйте от root.

🧠 **Что делает команда:**

* `getent passwd` — список пользователей
* `cut -d: -f1` — оставить только имена
* `passwd -S` — показывает состояние пароля
* `id` — выводит UID, GID и группы

🔒 **Статусы:**

* `L` — учётная запись заблокирована
* `P` — активна, пароль установлен
* `NP` — **пароль не задан вообще** (проверьте эти учётки в первую очередь при аудите безопасности — потенциально может означать passwordless-вход, если это разрешено PAM)

🌈 **Цветной вывод для удобства (на экране):**

```bash
getent passwd | cut -d: -f1 | xargs -n1 -I {} sh -c \
'passwd -S {} | sed "s/ L / \x1b[31mL\x1b[0m /; s/ P / \x1b[32mP\x1b[0m /"; id {}'
```

📂 **Сохранение в файл:**

```bash
HOSTNAME=$(hostname)
getent passwd | cut -d: -f1 | xargs -n1 -I {} sh -c 'passwd -S {}; id {}' \
> /home/gis-audit/user_pass_group_active_${HOSTNAME}
```

💡 **Совет:**
Вывести только активных пользователей:

```bash
getent passwd | cut -d: -f1 | xargs -I {} sh -c 'passwd -S {}' | grep " P "
```

---

<a id="packages"></a>

## 📦 3. Список установленных пакетов

### 🐧 Debian / Ubuntu / Astra Linux

```bash
dpkg-query -l
```

📂 **Сохранить в файл:**

```bash
dpkg-query -l > /home/gis-audit/dpkg_list_${HOSTNAME}
```

🔍 **Примеры:**

```bash
dpkg-query -l nano                 # Проверить пакет
dpkg-query -l | grep ^ii           # Только установленные
dpkg-query --status nano           # Статус и конфигурация
dpkg-query -L nano                 # Файлы пакета
```

---

### 🔴 РЕД ОС / RHEL-подобные

> [!NOTE]
> ALT Linux (Basealt) — отдельный, не связанный с РЕД ОС дистрибутив. Хоть пакеты там тоже RPM, менеджер пакетов у ALT — `apt-get`/`apt-cache` (их особенность — apt поверх rpm), а не `dnf`/`yum`, как у РЕД ОС/RHEL. Команды этого раздела рассчитаны на РЕД ОС и другие настоящие RHEL-подобные (RHEL/CentOS/Fedora), не на ALT.

```bash
dnf list installed
# или
yum list installed
# или
rpm -qa
```

🔍 **Примеры:**

```bash
rpm -qa | grep firefox
rpm -qi nano     # Информация о пакете
rpm -ql nano     # Список файлов пакета
```

📂 **Сохранить в файл:**

```bash
rpm -qa > /home/gis-audit/rpm_list_${HOSTNAME}
```

💡 **Совет:**
Форматированный вывод:

```bash
dnf list installed | awk '{print $1, $2}' | column -t
```

---

<a id="os-info"></a>

## 🧭 4. Информация о дистрибутиве и версии ОС

### 🟢 Astra Linux

```bash
lsb_release -a
cat /etc/astra/build_version
cat /etc/astra_license
cat /var/log/astra-history.log
```

💡 **Краткий вывод:**

```bash
lsb_release -d
```

---

### 🔴 Red OS 7.3 / 8.0

```bash
cat /etc/redos-release
cat /etc/os-release
hostnamectl
uname -a
cat /etc/issue
```

---

<a id="extra"></a>

## 🧩 Дополнительные полезные команды

| Цель                           | Команда                 | Комментарий               |
| ------------------------------ | ----------------------- | ------------------------- |
| Проверить занятое место        | `df -h`                 | Использование дисков      |
| Проверить использование памяти | `free -h`               | ОЗУ                       |
| Проверить загрузку CPU         | `top`, `htop`           | Мониторинг процессов      |
| Активные соединения            | `ss -tulpn`             | Порты и PID               |
| Время загрузки служб           | `systemd-analyze blame` | Что дольше всего стартует |
| Сетевые интерфейсы             | `ip a`                  | IP, MAC, статус           |

---

<a id="automation"></a>

## 💾 Автоматизация аудита

📦 Пример скрипта для массового аудита:

```bash
#!/bin/bash
# Требует root — passwd -S для чужих пользователей без root вернёт ошибки доступа
HOSTNAME=$(hostname)
AUDIT_DIR="/home/gis-audit"
mkdir -p "$AUDIT_DIR"

echo "[*] Сбор информации с $HOSTNAME..."

systemctl list-unit-files > "$AUDIT_DIR/services_list_${HOSTNAME}"
getent passwd | cut -d: -f1 | xargs -n1 -I {} sh -c 'passwd -S {}; id {}' > "$AUDIT_DIR/user_pass_group_active_${HOSTNAME}"
dpkg-query -l > "$AUDIT_DIR/dpkg_list_${HOSTNAME}" 2>/dev/null || rpm -qa > "$AUDIT_DIR/rpm_list_${HOSTNAME}"
lsb_release -a > "$AUDIT_DIR/os_info_${HOSTNAME}" 2>/dev/null || cat /etc/os-release > "$AUDIT_DIR/os_info_${HOSTNAME}"
df -h > "$AUDIT_DIR/disk_usage_${HOSTNAME}"
free -h > "$AUDIT_DIR/memory_usage_${HOSTNAME}"

echo "[+] Аудит завершён. Результаты сохранены в ${AUDIT_DIR}"
```

---

<a id="summary"></a>

## 📘 Итог

📗 Этот гайд подходит для:

* Быстрой проверки состояния системы
* Массового аудита серверов (через SSH)
* Сборки отчётов по безопасности и инвентаризации

💡 Совместим с:

* Astra Linux
* РЕД ОС
* Ubuntu / Debian
* CentOS / RHEL

> [!NOTE]
> ALT Linux сюда не входит, несмотря на RPM-пакеты — там свой пакетный менеджер (`apt-get` поверх RPM), команды раздела 3 для него не подойдут напрямую (см. предупреждение выше).

