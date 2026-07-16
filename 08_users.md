---
layout: default
title: "Пользователи и группы"
permalink: /08_users/
---

# 👤 Управление пользователями и группами в Linux

> [!TIP]  
> Эти команды универсальны для большинства дистрибутивов Linux. Различия могут быть только в путях конфигурационных файлов или менеджерах пакетов (Astra Linux / РЕД ОС).

---

## ⚡ 1. Основная информация о пользователях и группах

### Список пользователей
```bash
cat /etc/passwd
```

* Каждый пользователь описан строкой:

```
user:x:UID:GID:comment:home:shell
```

* `UID 0` — root
* `UID 1000+` — обычные пользователи

### Список групп

```bash
cat /etc/group
```

* Формат:

```
group_name:x:GID:user1,user2
```

### Проверить текущего пользователя

```bash
whoami
id
```

* `whoami` — имя пользователя
* `id` — UID, GID и группы пользователя

---

## 🛠 2. Создание пользователей

> [!WARNING]
> `adduser` — это НЕ universal shadow-utils команда, а отдельный интерактивный Perl-скрипт из пакета `adduser` (пришёл из Debian-мира). На **Astra Linux** (Debian) он ставит вопросы (пароль, ФИО, комнату и т.п.) сразу при вызове. На **РЕД ОС** (RHEL-семейство) отдельного пакета `adduser` обычно нет — если команда `adduser` вообще существует, это, как правило, просто симлинк на `useradd`, который работает **не интерактивно** и **не задаёт пароль автоматически** (учётка создастся заблокированной, без пароля, пока вы явно не выполните `passwd`). Проверяйте на месте: `type adduser` покажет, что это на самом деле — скрипт или симлинк на `useradd`.

### Простое создание пользователя

```bash
sudo adduser user1     # Astra: спросит пароль/данные интерактивно
sudo passwd user1      # РЕД ОС: adduser пароль не поставит — задать отдельно
```

* Astra: создаёт домашнюю директорию `/home/user1`, настраивает пароль, шелл, базовые группы через диалог
* РЕД ОС: то же самое достигается связкой `useradd` (или `adduser`-как-`useradd`) + отдельный `passwd`

### Создание пользователя без домашнего каталога

```bash
sudo adduser --no-create-home user2    # Astra (пакет adduser)
sudo useradd -M user2                  # РЕД ОС / универсально через shadow-utils
```

### Создание пользователя с конкретным UID и GID

```bash
sudo adduser --uid 1500 --gid 1001 user3   # Astra
sudo useradd -u 1500 -g 1001 user3         # РЕД ОС / универсально
```

### Добавление комментариев

```bash
sudo adduser --comment "Developer" devuser   # Astra
sudo useradd -c "Developer" devuser          # РЕД ОС / универсально
```

---

## 🔑 3. Установка и изменение пароля

```bash
sudo passwd user1         # установить пароль
sudo passwd -l user1      # заблокировать учетную запись
sudo passwd -u user1      # разблокировать
```

> [!IMPORTANT]
> Пароль root лучше оставить заблокированным для SSH-доступа и использовать sudo у обычных пользователей.

---

## 👑 4. Предоставление прав root через sudo

### Добавление пользователя в группу `sudo` или `wheel`

```bash
sudo usermod -aG sudo user1    # Debian / Astra
sudo usermod -aG wheel user1   # RHEL / RED OS
```

### Проверка прав sudo

```bash
sudo -l -U user1
```

> [!TIP]
> Группа `sudo` или `wheel` управляет доступом через `/etc/sudoers`. Для редактирования:

```bash
sudo visudo
```

---

## 🔐 5. Отключение встроенного root

```bash
sudo passwd -l root       # блокировка пароля root
sudo usermod -L root      # альтернативная блокировка (тот же эффект, что и -l выше)
```

* root всё ещё существует, но нельзя войти напрямую через пароль.
* Используем `sudo` у обычных пользователей для административных задач.

> [!WARNING]
> Блокировка пароля (`-l`/`-L`) закрывает только **парольный** вход. Она НЕ мешает войти под root по SSH-ключу, если в `sshd_config` разрешён `PermitRootLogin yes` (или `without-password`/`prohibit-password` — эти два прямо предполагают именно ключ, минуя пароль) и у root настроен `authorized_keys`. Чтобы реально закрыть прямой root-доступ по SSH — нужен ещё `PermitRootLogin no` в `/etc/ssh/sshd_config` (плюс `systemctl restart ssh`/`sshd`, см. раздел про SSH-доступ ниже). Блокировка пароля и запрет root в SSH — две независимые настройки, обе нужны для полного эффекта.

---

## 🧩 6. Создание групп и управление доступом

### Создание группы

```bash
sudo groupadd sshusers
sudo groupadd developers
```

### Добавление пользователя в группу

```bash
sudo usermod -aG sshusers user1
sudo usermod -aG developers devuser
```

### Проверка группы пользователя

```bash
groups user1
id user1
```

---

## 🖥 7. Ограничение SSH-доступа

### Разрешить вход только определённым пользователям/группам

1. Открыть конфигурацию SSH:

```bash
sudo nano /etc/ssh/sshd_config
```

2. Добавить:

```
AllowGroups sshusers
AllowUsers devuser user1
```

3. Перезапустить SSH:

```bash
sudo systemctl restart ssh     # Astra Linux — юнит называется "ssh"
sudo systemctl restart sshd    # РЕД ОС — юнит называется "sshd"
```

> [!IMPORTANT]
> Если указать `AllowGroups` или `AllowUsers`, убедитесь, что хотя бы один администратор имеет доступ, иначе вы заблокируете SSH.

---

## ⚙️ 8. Удаление пользователей и групп

> [!NOTE]
> `deluser` — та же история, что и `adduser` (см. предупреждение в разделе 2): пакет-специфичная Debian-обёртка. Универсальная shadow-utils команда — `userdel`, есть и на Astra, и на РЕД ОС.

### Удаление пользователя, оставив домашний каталог

```bash
sudo deluser user1    # Astra (пакет adduser)
sudo userdel user1    # РЕД ОС / универсально
```

### Удаление пользователя с домашним каталогом

```bash
sudo deluser --remove-home user1   # Astra
sudo userdel -r user1              # РЕД ОС / универсально
```

### Удаление группы

```bash
sudo groupdel sshusers
```

---

## 🔍 9. Просмотр сессий и активности пользователей

### Список текущих пользователей

```bash
who
w
```

### История входов

```bash
last
lastlog
```

### Проверка процессов пользователя

```bash
ps -u user1
```

---

## 🧾 10. Практические рекомендации

> [!TIP]
>
> * Используйте группы для управления доступом к ресурсам.
> * Не предоставляйте root-доступ напрямую через SSH.
> * Логи входа и sudo находятся в `/var/log/auth.log` (Debian/Astra) или `/var/log/secure` (RED OS).

> [!IMPORTANT]
> Перед массовым удалением пользователей или групп проверяйте активные процессы и домашние директории.

---

## 🔗 Справка

Внешние ссылки на объекте без интернета бесполезны — вся документация уже есть локально:

```bash
man adduser; man useradd; man usermod; man passwd
man deluser; man userdel; man groupadd; man visudo
man sshd_config
```
