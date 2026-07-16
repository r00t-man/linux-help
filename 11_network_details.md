---
layout: default
title: "Подробнее"
permalink: /11_network_details/
---

# 🌐 Ручная конфигурация сети в Linux

В этой статье мы рассмотрим, как **правильно вручную настраивать сетевые интерфейсы** в Linux, управлять сетевыми службами, использовать альтернативные утилиты и учитывать нюансы разных дистрибутивов.

> [!IMPORTANT]
> **Основные инструменты и команды:** `networking`, `NetworkManager`, `netplan`, `nmtui`, `nmcli`, `ip addr`, `ip route`

---

## ⚡ 1. Конфигурация через `/etc/network/interfaces` (Debian / Astra Linux / Ubuntu)

Файл `/etc/network/interfaces` используется для статической конфигурации интерфейсов.  

### Пример шаблона

```ini
# Loopback
auto lo
iface lo inet loopback

# LAN интерфейс с DHCP
auto eth0
iface eth0 inet dhcp

# LAN интерфейс со статическим IP
auto eth1
iface eth1 inet static
    address 192.168.1.10
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 8.8.4.4
```

* `auto <interface>` — интерфейс поднимается автоматически при старте.
* `iface <interface> inet <method>` — метод настройки (`dhcp` или `static`).
* `address`, `netmask`, `gateway` — статические настройки IP.
* `dns-nameservers` — указание DNS-серверов.

---

### 🔧 Управление службой `networking`

```bash
sudo systemctl restart networking       # Перезапуск сети
sudo systemctl stop networking          # Остановить
sudo systemctl start networking         # Запустить
sudo systemctl status networking        # Проверить статус
```

> [!NOTE]
> `systemctl restart networking` перезапускает службу целиком, но на практике не всегда переподнимает уже настроенные интерфейсы (известная особенность init-скрипта `ifupdown`). Надёжнее для одного интерфейса — точечно:
> ```bash
> sudo ifdown eth0 && sudo ifup eth0
> ```

> [!IMPORTANT]
> Если вы используете **графическую оболочку (Gnome/KDE)**, то её NetworkManager может конфликтовать с ручной настройкой.
> В этом случае рекомендуется **замаскировать NM**:

```bash
sudo systemctl stop NetworkManager
sudo systemctl disable NetworkManager
sudo systemctl mask NetworkManager
```

---

### 📝 Нюансы

* Astra Linux (Debian-подобная) использует `/etc/network/interfaces`.
* В RED OS чаще используется `ifcfg-*` файлы, но старый метод через `/etc/network/interfaces` возможен через пакет `network-scripts`.
* После редактирования конфигурации интерфейсов необходимо перезапустить службу сети.

---

## 🌐 2. Альтернатива: Netplan (Ubuntu 18.04+ / Astra SE 1.8+)

Netplan — современный способ конфигурации сети через YAML.

> [!NOTE]
> Netplan — изначально Ubuntu-инструмент; на Debian (и, соответственно, на Astra) он НЕ обязательно стоит по умолчанию, даже на 1.8 — проверьте `dpkg -l netplan.io`, при отсутствии — `sudo apt install netplan.io`.

### Пример конфигурации (`/etc/netplan/01-netcfg.yaml`)

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: true
    eth1:
      addresses:
        - 192.168.1.10/24
      gateway4: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

* `renderer: networkd` — системный демонт управления сетью (можно использовать NetworkManager).
* Для применения изменений:

```bash
sudo netplan apply
```

* Для проверки:

```bash
sudo netplan try        # применяет временно, откатит сам, если не подтвердить за 120с — безопасно для удалённой машины
sudo netplan generate   # только собирает конфиги backend'а (systemd-networkd/NM) из YAML, ничего не применяет — для отладки
```

> [!TIP]
> Netplan особенно удобен для серверов и виртуальных машин, позволяет легко управлять VLAN, bridges, bonds.

---

## 🖥 3. Управление через NetworkManager / nmtui

`nmtui` — консольный текстовый интерфейс для настройки сети.

```bash
sudo nmtui
```

* Возможности:

  * Настройка интерфейсов DHCP / статического IP.
  * Настройка WiFi и VPN.
  * Управление соединениями и приоритетами.

> [!TIP]
> Для графических рабочих станций `nmtui` удобнее, чем ручное редактирование конфигурационных файлов.

---

### 🔧 Команды `nmcli` для продвинутых пользователей

```bash
nmcli connection show              # Список всех соединений
nmcli connection up <name>         # Поднять соединение
nmcli connection down <name>       # Опустить соединение
```

Создание нового соединения — конкретные рабочие примеры:

```bash
# DHCP на eth0
sudo nmcli connection add type ethernet con-name eth0-dhcp ifname eth0

# Статический IP на eth1
sudo nmcli connection add type ethernet con-name eth1-static ifname eth1 \
  ipv4.method manual ipv4.addresses 192.168.1.10/24 ipv4.gateway 192.168.1.1 ipv4.dns 8.8.8.8
```

* Полезно для автоматизации и скриптов.

---

## 📌 4. Особенности и рекомендации по дистрибутивам

| ОС                 | Рекомендуемый метод                 | Примечания                                                           |
| ------------------ | ----------------------------------- | -------------------------------------------------------------------- |
| Astra SE 1.7 / 1.8 | `/etc/network/interfaces` / Netplan | NetworkManager optional                                              |
| RED OS 7.3 / 8.0   | `ifcfg-*` файлы + `nmtui`           | Можно использовать `/etc/network/interfaces` через `network-scripts` |
| Ubuntu 18.04+      | Netplan                             | YAML конфигурации, поддержка systemd-networkd или NetworkManager     |

---

## 🔍 5. Полезные команды для проверки сети

```bash
ip addr show               # Проверка IP-адресов
ip route show              # Таблица маршрутизации
ping 8.8.8.8               # Проверка доступности внешнего узла
ping google.com             # Проверка DNS
ethtool eth0                # Статистика интерфейса
nmcli device status         # Статус сетевых устройств
```

> [!TIP]
> Всегда проверяйте работу сети после изменения конфигурации. На серверах рекомендуется сначала проверить через `ifdown/ifup` или `netplan try` перед перезапуском службы.

---

## 🧩 6. Дополнительные советы

* Для серверов **статический IP предпочтительнее**, особенно для сетевых служб и контейнеров.
* При использовании WiFi на сервере лучше настроить `wpa_supplicant` через `/etc/wpa_supplicant/wpa_supplicant.conf`.
* Всегда делайте резервную копию конфигурационных файлов перед изменением.

---

Эта инструкция даёт **полное руководство по ручной конфигурации сети** для Astra Linux, RED OS и Ubuntu, включая работу с `/etc/network/interfaces`, Netplan и `nmtui`, с учётом нюансов GUI и серверных установок.

```
