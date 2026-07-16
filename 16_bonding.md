---
layout: default
title: "Настройка Бонда"
permalink: /16_bonding/
---

# 🧭 Настройка и виды сетевых бондов (bonding) в Linux

---

## 📑 Содержание

- [Что такое Bonding](#chto-takoe-bonding)
- [Основные преимущества bonding](#osnovnye-preimushchestva-bonding)
- [Пример сети](#primer-seti)
- [Настройка bonding через /etc/network/interfaces](#interfaces)
  - [Базовая структура файла](#interfaces-struktura-fayla)
  - [Объяснение параметров](#interfaces-obyasnenie-parametrov)
- [Режимы bonding (0–6)](#rezhimy-bonding-0-6)
  - [mode=0 — balance-rr](#mode0-balance-rr)
  - [mode=1 — active-backup](#mode1-active-backup)
  - [mode=2 — balance-xor](#mode2-balance-xor)
  - [mode=3 — broadcast](#mode3-broadcast)
  - [mode=4 — 802.3ad (LACP)](#mode4-8023ad-lacp)
  - [mode=5 — balance-tlb](#mode5-balance-tlb)
  - [mode=6 — balance-alb](#mode6-balance-alb)
- [Активация bonding и проверка](#aktivatsiya-bonding-i-proverka)
- [Сводная таблица режимов bonding](#svodnaya-tablica-rezhimov-bonding)
- [Настройка bonding через nmcli (NetworkManager)](#nmcli-networkmanager)
- [Вывод](#vyvod)
- [Параметр bond-xmit-hash-policy](#bond-xmit-hash-policy)
  - [Общий синтаксис](#bond-xmit-hash-policy-sintaksis)
  - [Возможные значения (режимы хеширования)](#bond-xmit-hash-policy-znacheniya)
  - [Подробное объяснение режимов](#bond-xmit-hash-policy-rezhimy)
  - [Рекомендации по выбору политики](#bond-xmit-hash-policy-rekomendatsii)
  - [Проверка текущей политики](#bond-xmit-hash-policy-proverka)
  - [Итого](#bond-xmit-hash-policy-itogo)
- [Условия для работы бондов](#usloviya-dlya-raboty-bondov)
  - [Важные моменты](#vazhnye-momenty)
- [Вывод по пакетам и модулям](#vyvod-po-paketam-i-modulyam)

---

<a id="chto-takoe-bonding"></a>
## Что такое Bonding

**Bonding** (или Teaming) — это технология объединения нескольких сетевых интерфейсов в один логический интерфейс (`bond0`), чтобы увеличить пропускную способность, повысить отказоустойчивость или распределять нагрузку между интерфейсами.

Применяется на серверах, маршрутизаторах, в виртуализации и кластерах.

---

<a id="osnovnye-preimushchestva-bonding"></a>
## ⚙️ Основные преимущества bonding

* ✅ Увеличение скорости передачи (агрегация каналов)
* ✅ Отказоустойчивость (резервирование)
* ✅ Гибкость настройки (7 режимов работы)
* ✅ Поддержка большинства дистрибутивов Linux

---

<a id="primer-seti"></a>
## 🌐 Пример сети

| Параметр       | Значение          |
| -------------- | ----------------- |
| Подсеть        | `10.0.0.0/28`     |
| Маска          | `255.255.255.240` |
| IP сервера     | `10.0.0.38`       |
| Шлюз           | `10.0.0.1`        |
| Интерфейсы     | `eth0`, `eth1`    |
| Bond-интерфейс | `bond0`           |

---

<a id="interfaces"></a>
## 🧩 Настройка bonding через `/etc/network/interfaces`

Для систем без NetworkManager (например, Debian, Ubuntu Server, Astra Linux).

> [!TIP]
> Перед настройкой bond необходимо установить пакет `ifenslave`

---

<a id="interfaces-struktura-fayla"></a>
### 📄 Базовая структура файла `/etc/network/interfaces`

```bash
# Loopback
auto lo
iface lo inet loopback

# Интерфейсы, входящие в bond
auto eth0 eth1 bond0
iface eth0 inet manual
iface eth1 inet manual

# Основная конфигурация bond0
iface bond0 inet static
    address 10.0.0.38
    netmask 255.255.255.240
    network 10.0.0.0
    gateway 10.0.0.1
    bond-miimon 100
    bond-downdelay 200
    bond-updelay 200
    bond-slaves eth0 eth1
```

<a id="interfaces-obyasnenie-parametrov"></a>
### 🔍 Объяснение параметров

| Параметр                  | Значение                                                   |
| ------------------------- | ---------------------------------------------------------- |
| `bond-mode 802.3ad`       | режим агрегации каналов (LACP)                             |
| `bond-miimon 100`         | проверка состояния линка каждые 100 мс                     |
| `bond-downdelay 200`      | задержка перед деактивацией порта                          |
| `bond-updelay 200`        | задержка перед активацией порта                            |
| `bond-lacp-rate slow`     | частота обмена LACP-пакетами (slow = 30 сек, fast = 1 сек) |
| `bond-xmit-hash-policy layer3+4` | метод балансировки (по IP, MAC или портам) — значение строковое (`layer2`/`layer3+4`/...), не число; число в скобках вида `layer3+4 (2)` — это просто внутренний индекс, который показывает `/proc/net/bonding/bondX`, в конфиге он не используется |
| `bond-slaves eth0 eth1`   | интерфейсы, входящие в бонд                                |

> [!NOTE]
> Для работы LACP требуется, чтобы свич или маршрутизатор также поддерживал 802.3ad и был соответствующим образом настроен.
---
<a id="rezhimy-bonding-0-6"></a>
## 🧱 Режимы bonding (0–6)

Linux поддерживает 7 режимов работы bonding. Ниже — подробные описания и готовые конфиги для каждого.

---

<a id="mode0-balance-rr"></a>
### 🟢 mode=0 — **balance-rr (Round Robin)**

Пакеты отправляются последовательно по всем интерфейсам.

**Плюсы:** максимальная пропускная способность
**Минусы:** может вызывать нарушение порядка пакетов; коммутатор должен поддерживать статическую агрегацию портов (etherchannel/trunk) — это НЕ то же самое, что LACP: LACP — протокол *динамического* согласования, специфичный именно для mode=4 (802.3ad, см. ниже), для mode=0 на свиче настраивается статическая группа портов без LACP-переговоров

```bash
iface bond0 inet static
    address 10.0.0.38
    netmask 255.255.255.240
    network 10.0.0.0
    gateway 10.0.0.1
    bond-mode balance-rr
    bond-miimon 100
    bond-downdelay 200
    bond-updelay 200
    bond-slaves eth0 eth1
```

---

<a id="mode1-active-backup"></a>
### 🟡 mode=1 — **active-backup**

Один интерфейс активен, второй в режиме ожидания.
При сбое активного — резервный становится активным.

```bash
iface bond0 inet static
    address 10.0.0.38
    netmask 255.255.255.240
    network 10.0.0.0
    gateway 10.0.0.1
    bond-mode active-backup
    bond-primary eth0
    bond-miimon 100
    bond-downdelay 200
    bond-updelay 200
    bond-slaves eth0 eth1
```

✅ Не требует настройки коммутатора.
💡 Самый часто используемый режим в серверах и системах с одной сетью.

---

<a id="mode2-balance-xor"></a>
### 🔵 mode=2 — **balance-xor**

Балансировка по XOR-хешу (по MAC, IP или портам).

```bash
iface bond0 inet static
    address 10.0.0.38
    netmask 255.255.255.240
    network 10.0.0.0
    gateway 10.0.0.1
    bond-mode balance-xor
    bond-xmit-hash-policy layer3+4
    bond-miimon 100
    bond-slaves eth0 eth1
```

💡 Требует поддержки агрегирования каналов на коммутаторе.

---

<a id="mode3-broadcast"></a>
### ⚫ mode=3 — **broadcast**

Отправляет все пакеты по всем интерфейсам.
Используется для критически важных соединений, где важна отказоустойчивость.

```bash
iface bond0 inet static
    address 10.0.0.38
    netmask 255.255.255.240
    network 10.0.0.0
    gateway 10.0.0.1
    bond-mode broadcast
    bond-miimon 100
    bond-slaves eth0 eth1
```

---

<a id="mode4-8023ad-lacp"></a>
### 🟣 mode=4 — **802.3ad (LACP)**

Стандарт IEEE для динамической агрегации каналов.
Требует настройки LACP (Link Aggregation) на коммутаторе.

```bash
iface bond0 inet static
    address 10.0.0.38
    netmask 255.255.255.240
    network 10.0.0.0
    gateway 10.0.0.1
    bond-mode 802.3ad
    bond-lacp-rate fast
    bond-xmit-hash-policy layer3+4
    bond-miimon 100
    bond-downdelay 200
    bond-updelay 200
    bond-slaves eth0 eth1
```

---

<a id="mode5-balance-tlb"></a>
### 🟠 mode=5 — **balance-tlb (Transmit Load Balancing)**

Распределяет исходящий трафик между интерфейсами без участия свича.

```bash
iface bond0 inet static
    address 10.0.0.38
    netmask 255.255.255.240
    network 10.0.0.0
    gateway 10.0.0.1
    bond-mode balance-tlb
    bond-miimon 100
    bond-slaves eth0 eth1
```

---

<a id="mode6-balance-alb"></a>
### 🔴 mode=6 — **balance-alb (Adaptive Load Balancing)**

Сочетает TLB + входящую балансировку через ARP.
Работает без поддержки коммутатора.

```bash
iface bond0 inet static
    address 10.0.0.38
    netmask 255.255.255.240
    network 10.0.0.0
    gateway 10.0.0.1
    bond-mode balance-alb
    bond-miimon 100
    bond-slaves eth0 eth1
```
---

<a id="aktivatsiya-bonding-i-proverka"></a>
## 🔧 Активация bonding и проверка

1. Загрузить модуль:

   ```bash
   sudo modprobe bonding
   ```
2. Добавить в автозагрузку:

   ```bash
   echo "bonding" | sudo tee -a /etc/modules
   ```
3. Перезапустить сеть:

   ```bash
   sudo systemctl restart networking
   ```
   > [!NOTE]
   > На практике перезапуск службы целиком не всегда переподнимает уже настроенные интерфейсы (известная особенность init-скрипта `ifupdown`, см. также статью "Подробнее" о сети) — если `bond0` не поднялся, надёжнее явно `sudo ifdown bond0 && sudo ifup bond0` (или просто перезагрузить машину при первой настройке bonding — так безопаснее для сетевого доступа).
4. Проверить состояние:

   ```bash
   cat /proc/net/bonding/bond0
   ```
---

<a id="svodnaya-tablica-rezhimov-bonding"></a>
## 🧭 Сводная таблица режимов bonding

| Mode | Название       | Требует свич | Балансировка | Резерв | Описание                |
| ---- | -------------- | ------------ | ------------ | ------ | ----------------------- |
| 0    | balance-rr     | ✅ (статическая агрегация) | ✔            | ✔      | Перебор интерфейсов     |
| 1    | active-backup  | ❌            | ❌            | ✔      | Резервирование          |
| 2    | balance-xor    | ✅ (статическая агрегация) | ✔            | ✔      | Балансировка по MAC/IP  |
| 3    | broadcast      | ❌            | ❌            | ✔      | Дублирование трафика    |
| 4    | 802.3ad (LACP) | ✅ (именно LACP, динамически) | ✔            | ✔      | Агрегация IEEE 802.3ad  |
| 5    | balance-tlb    | ❌            | ✔ (Tx)       | ✔      | Без настройки свича     |
| 6    | balance-alb    | ❌            | ✔ (Tx/Rx)    | ✔      | Адаптивная балансировка |

> [!NOTE]
> «Требует свич» у mode 0/2 и mode 4 — это РАЗНЫЕ вещи: mode 0/2 нужна ручная статическая агрегация портов (etherchannel/trunk, без переговоров), mode 4 — именно протокол LACP (динамическое согласование). Настройка на стороне свича отличается, не взаимозаменяема.

---

<a id="nmcli-networkmanager"></a>
## 🧰 Настройка bonding через `nmcli` (NetworkManager)

Для современных систем (Ubuntu 22+, Debian 12, RHEL 8+, Astra Linux Common Edition 2.12+).

---

### 🔹 1. Создание bond-интерфейса

```bash
sudo nmcli connection add type bond ifname bond0 mode 802.3ad
```

---

### 🔹 2. Добавление интерфейсов (слейвов)

```bash
sudo nmcli connection add type ethernet ifname eth0 master bond0
sudo nmcli connection add type ethernet ifname eth1 master bond0
```

---

### 🔹 3. Настройка IP-параметров

```bash
sudo nmcli connection modify bond0 ipv4.addresses 10.0.0.38/28
sudo nmcli connection modify bond0 ipv4.gateway 10.0.0.1
sudo nmcli connection modify bond0 ipv4.method manual
```

---

### 🔹 4. Установка дополнительных параметров bonding

```bash
sudo nmcli connection modify bond0 bond.options "miimon=100,mode=802.3ad,lacp_rate=fast,xmit_hash_policy=layer3+4"
```

---

### 🔹 5. Активация соединения

```bash
sudo nmcli connection up bond0
```

Проверка:

```bash
cat /proc/net/bonding/bond0
```

---

<a id="vyvod"></a>
## ✅ Вывод

Bonding — мощный инструмент для серверов Linux, позволяющий:

* объединять интерфейсы для скорости и надёжности,
* реализовывать отказоустойчивые схемы,
* повышать производительность без дорогостоящего оборудования.

## Про параметр bond-xmit-hash-policy

Это как раз то, что часто упускают про bonding — параметр `bond-xmit-hash-policy` **радикально влияет на распределение нагрузки** и эффективность работы бонда.
Вот подробное объяснение **всех возможных значений** этого параметра (актуально для Linux bonding-драйвера `ifenslave` и `bonding.ko`).

---

<a id="bond-xmit-hash-policy"></a>
# ⚙️ `bond-xmit-hash-policy` — Политика хеширования исходящих пакетов

Этот параметр определяет, **как ядро Linux вычисляет хеш (ключ)** для выбора интерфейса, через который будет отправлен исходящий пакет в многоканальном бонде (`balance-xor`, `802.3ad`, `balance-tlb`, `balance-alb` и т.д.).

---

<a id="bond-xmit-hash-policy-sintaksis"></a>
## 📘 Общий синтаксис

```bash
bond-xmit-hash-policy <режим>
```

Добавляется в секцию интерфейса `/etc/network/interfaces` или задаётся командой:

```bash
echo <режим> > /sys/class/net/bond0/bonding/xmit_hash_policy
```

---

<a id="bond-xmit-hash-policy-znacheniya"></a>
## 📊 Возможные значения (режимы хеширования)

| Политика          | Поддержка        | Алгоритм                | Описание                                                | Где использовать                                        |
| ----------------- | ---------------- | ----------------------- | ------------------------------------------------------- | ------------------------------------------------------- |
| `layer2`          | ✅ (по умолчанию) | MAC-адреса              | Хешируется MAC-адрес источника и назначения             | Простая локальная сеть, без IP-балансировки             |
| `layer2+3`        | ✅                | MAC + IP                | Хешируется MAC + IP источника и назначения              | Более равномерное распределение, поддержка IP           |
| `layer3+4`        | ✅                | IP + порты              | Хешируется IP и порты TCP/UDP                           | Оптимально для LACP (802.3ad) и балансировки по сессиям |
| `encap2+3`        | ⚙️ (новый)       | IP под туннелем         | Аналог layer2+3, но учитывает инкапсуляцию (VXLAN, GRE) | Современные виртуализированные среды                    |
| `encap3+4`        | ⚙️ (новый)       | IP + порты под туннелем | Аналог layer3+4, но с поддержкой туннелей               | LACP + overlay сети (OpenStack, K8s)                    |
| `vlan+srcmac`     | ⚙️               | VLAN ID + MAC           | Использует VLAN и MAC для хеша                          | Multi-VLAN среда, например, на trunk-портах             |

> [!WARNING]
> Проверено вживую (создан тестовый bond, записаны значения в `/sys/class/net/bondX/bonding/xmit_hash_policy`): `layer2`, `layer2+3`, `layer3+4`, `encap2+3`, `encap3+4`, `vlan+srcmac` — реальные, ядро их принимает. А вот **`vlan+ip`, `vlan+ip+port` и `layer3+4+srcmac` — таких значений в драйвере bonding НЕ существует**, ядро отвечает `invalid value` (видно и в `dmesg`). В предыдущей версии этой статьи они были описаны как настоящие опции с "рекомендациями" использовать их для VLAN/Kubernetes — это ошибка, эти строки и вся связанная рекомендация ниже удалены.

---

<a id="bond-xmit-hash-policy-rezhimy"></a>
## 📖 Подробное объяснение режимов

---

### 🔹 `layer2` — Балансировка на канальном уровне (L2)

Хеширование только по MAC-адресам.

**Принцип:**

```
hash = MAC(src) XOR MAC(dst)
```

**Плюсы:**

* Простая, надёжная схема.
* Хорошая совместимость со свичами.

**Минусы:**

* Если все пакеты идут к одному MAC, весь трафик идёт через один интерфейс.
* Плохая балансировка при работе с одним шлюзом.

**Пример:**

```bash
bond-xmit-hash-policy layer2
```

---

### 🔹 `layer2+3` — MAC + IP

Добавляет IP-адреса в хеш, повышая разнообразие потоков.

**Принцип:**

```
hash = (MAC(src) ⊕ MAC(dst)) ⊕ (IP(src) ⊕ IP(dst))
```

**Плюсы:**

* Лучше распределяет трафик по интерфейсам.
* Совместима с VLAN и LACP.

**Минусы:**

* Немного увеличивает нагрузку на CPU.

**Пример:**

```bash
bond-xmit-hash-policy layer2+3
```

---

### 🔹 `layer3+4` — IP и порты TCP/UDP

Самый сбалансированный вариант для современных сетей.
Хеширует комбинацию IP и портов.

**Принцип:**

```
hash = (IP(src) ⊕ IP(dst)) ⊕ (PORT(src) ⊕ PORT(dst))
```

**Плюсы:**

* Отличная балансировка множества соединений.
* Подходит для высоконагруженных серверов и балансировщиков.
* Рекомендуемый режим для `bond-mode=802.3ad`.

**Минусы:**

* Коммутатор должен поддерживать LACP с таким же хешем.

**Пример:**

```bash
bond-xmit-hash-policy layer3+4
```

---

### 🔹 `encap2+3` и `encap3+4` — Для туннельных интерфейсов

Используются при работе через **GRE, VXLAN, GENEVE** и другие инкапсулированные протоколы.

**Принцип:**

* `encap2+3`: MAC + IP из полезной нагрузки (внутри туннеля)
* `encap3+4`: IP + порты TCP/UDP внутри туннеля

**Плюсы:**

* Корректная балансировка для overlay-сетей.
* Поддержка виртуализации и контейнеризации.

**Пример:**

```bash
bond-xmit-hash-policy encap3+4
```

---

### 🔹 VLAN-aware политики (`vlan+*`)

Актуальны для trunk-интерфейсов, когда на bond навешиваются VLAN-интерфейсы (`bond0.100`, `bond0.200` и т.д.)

**Преимущества:**

* Разделение трафика по VLAN ID.
* Оптимизация в multi-tenant сетях.

**Пример** (реально существующее значение — `vlan+srcmac`, не `vlan+ip`/`vlan+ip+port` — см. предупреждение выше):

```bash
bond-xmit-hash-policy vlan+srcmac
```

---

<a id="bond-xmit-hash-policy-rekomendatsii"></a>
## 🧭 Рекомендации по выбору политики

| Среда                     | Рекомендуемый режим          | Примечание                         |
| ------------------------- | ---------------------------- | ---------------------------------- |
| Простая локальная сеть    | `layer2`                     | Не требует LACP                    |
| Несколько IP-клиентов     | `layer2+3`                   | Оптимально для DHCP/разных IP      |
| LACP (802.3ad)            | `layer3+4`                   | Лучший выбор                       |
| GRE/VXLAN (виртуализация) | `encap3+4`                   | Для OpenStack/KVM/VMWare           |
| VLAN trunk                | `vlan+srcmac`                | Если bond содержит VLAN-интерфейсы |
| Контейнеры / k8s          | `layer3+4`                   | Для балансировки по сессиям/портам |

---

<a id="bond-xmit-hash-policy-proverka"></a>
## 🧩 Проверка текущей политики

```bash
cat /proc/net/bonding/bond0 | grep "Transmit Hash"
```

Пример вывода:

```
Transmit Hash Policy: layer3+4 (2)
```

---

<a id="bond-xmit-hash-policy-itogo"></a>
## 🧠 Итого

* `bond-xmit-hash-policy` определяет, **на основе чего выбирается интерфейс для исходящего трафика**.
* При использовании LACP (`mode=4`) **нужно согласовать хеш с коммутатором** (обычно `layer3+4`).
* Для автономных режимов (`mode=2,5,6`) можно использовать любые политики — ядро решает локально.

<a id="usloviya-dlya-raboty-bondov"></a>
## Условия для работы бондов

---

### 1. `bonding.ko`

* Это **ядро-модуль** Linux, который реализует функциональность bonding (объединения нескольких сетевых интерфейсов в один логический).
* **Нужен только один раз для всей системы.**
* Когда модуль загружен (`modprobe bonding`), ты можешь создавать сколько угодно bond-интерфейсов (`bond0`, `bond1` и т.д.).
* В пакетах дистрибутива он обычно идёт как часть `kernel-modules` или отдельно как `kernel-modules-extra` (название зависит от дистрибутива).

В большинстве современных дистрибутивов Linux **модуль `bonding.ko` действительно включён в ядро и доступен по умолчанию**, но есть нюансы:

---

### 1.1. Проверка наличия модуля

Можно проверить, доступен ли модуль в системе:

```bash
modinfo bonding
```

Если вывод есть — модуль доступен.

Также можно проверить, загружен ли он сейчас:

```bash
lsmod | grep bonding
```

---

### 1.2. Загрузка модуля

Если модуль есть, но ещё не загружен:

```bash
sudo modprobe bonding
```

После этого можно создавать bond-интерфейсы.
---

### 2. `ifenslave`

* Это **пользовательская утилита**, которая привязывает физические интерфейсы к bond-интерфейсу.
* Утилита тоже **одна на систему**, а не на каждый bond.
* С её помощью ты выполняешь команды типа:

```bash
ifenslave bond0 eth0 eth1
```

* Можно создать несколько bond-интерфейсов, и для всех них используется **один и тот же ifenslave**.

---

<a id="vazhnye-momenty"></a>
### 3. Важные моменты

1. **Не нужен отдельный пакет на каждый bond.**

   * Модуль ядра (`bonding.ko`) и утилита (`ifenslave`) один раз ставятся для всей системы.
2. **Конфигурация бондов** делается через конфиги сети (`/etc/network/interfaces`, `nmcli`, `systemd-networkd` или `netplan`) для каждого bond отдельно.
3. **Поддержка разных режимов bonding** (active-backup, LACP, balance-alb и т.д.) встроена в модуль ядра, нет необходимости ставить что-то дополнительно.

---

<a id="vyvod-po-paketam-i-modulyam"></a>
✅ **Вывод:**

* `bonding.ko` — один модуль для всей системы.
* `ifenslave` — одна утилита для всей системы.

