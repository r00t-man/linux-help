---
layout: default
title: "LVM в Linux"
permalink: /25_lvm/
parent: "🖴 Управление дисками"
nav_order: 2
---

# 🧱 Настройка и расширение LVM в Linux

---

## 📑 Содержание

- [Что такое LVM](#chto-takoe-lvm)
- [Структура и принципы работы](#struktura-lvm)
- [Создание раздела под LVM](#sozdanie-razdela)
- [Создание LVM (PV → VG → LV)](#sozdanie-lvm)
- [Создание файловой системы и монтирование](#sozdanie-fs)
- [Автомонтирование через /etc/fstab](#fstab)
- [Проверка состояния LVM](#proverka-lvm)
- [Расширение LVM и файловой системы](#rasshirenie-lvm)
- [Добавление нового диска в группу](#dobavlenie-diska)
- [Удаление LVM](#udalenie-lvm)
- [Под капотом LVM](#pod-kapotom)
- [ASCII-схема расширения](#ascii-schema)
- [Полезные команды и советы](#sovety)
- [Вывод](#vyvod)

---

<a id="chto-takoe-lvm"></a>
## 🔍 Что такое LVM

**LVM (Logical Volume Manager)** — подсистема Linux для гибкого управления дисковым пространством.  
Она позволяет:

- объединять несколько физических дисков в одно логическое хранилище;
- изменять размеры разделов без перезагрузки;
- создавать снапшоты для резервного копирования;
- добавлять или убирать диски из группы на ходу.

Используется в **RedOS**, **Astra Linux**, **Ubuntu**, **Debian** и других системах корпоративного уровня.

---

<a id="struktura-lvm"></a>
## 🧩 Структура и принципы работы LVM

| Уровень | Назначение | Пример |
| -------- | ----------- | ------- |
| **PV (Physical Volume)** | Физический том — реальный диск или раздел | `/dev/sda1`, `/dev/sdb1` |
| **VG (Volume Group)** | Группа физических томов | `vg_data` |
| **LV (Logical Volume)** | Логический том (виртуальный раздел) | `/dev/vg_data/lv_backup` |

📊 **Схема работы:**
```

[ /dev/sdb1 + /dev/sdc1 ] → VG (vg_data)
↓
LV (lv_backup)
↓
Файловая система (ext4, xfs, btrfs)

```

---

<a id="sozdanie-razdela"></a>
## 💽 Создание раздела под LVM с помощью parted

```bash
sudo parted /dev/sdb
```

**Пример последовательности:**

```
(parted) mklabel gpt                 ← создаём таблицу GPT
(parted) mkpart primary 0% 100%      ← создаём раздел
(parted) name 1 data_disk            ← даём имя
(parted) set 1 lvm on                ← включаем флаг LVM
(parted) print                       ← проверяем
(parted) quit                        ← выходим
```

> 💡 Альтернатива через `fdisk`:
>
> ```
> Command (m for help): g
> Command (m for help): n
> Partition number (1-128, default 1): 1
> First sector: <Enter>
> Last sector: <Enter>
> Command (m for help): w
> ```

После выхода появится раздел `/dev/sdb1`.

---

<a id="sozdanie-lvm"></a>

## 🏗️ Создание LVM (PV → VG → LV)

### 1️⃣ Инициализация физического тома

```bash
sudo pvcreate /dev/sdb1
sudo pvs
```

### 2️⃣ Создание группы томов

```bash
sudo vgcreate vg_data /dev/sdb1
sudo vgdisplay vg_data
```

### 3️⃣ Создание логического тома

```bash
sudo lvcreate -n lv_backup -L 100G vg_data
sudo lvs
```

---

<a id="sozdanie-fs"></a>

## 📁 Создание файловой системы и монтирование

```bash
sudo mkfs.ext4 /dev/vg_data/lv_backup
sudo mkdir -p /mnt/data
sudo mount /dev/vg_data/lv_backup /mnt/data
df -h /mnt/data
```

---

<a id="fstab"></a>

## ⚙️ Автомонтирование через `/etc/fstab`

1. Узнаём UUID:

   ```bash
   blkid /dev/vg_data/lv_backup
   ```
2. Добавляем строку:

   ```
   UUID=<указать_UUID> /mnt/data ext4 defaults 0 2
   ```
3. Проверяем:

   ```bash
   sudo mount -a
   ```

> [!NOTE]
> Ошибка в этой строке `/etc/fstab` может при следующей перезагрузке уронить загрузку в аварийный режим — на удалённом сервере без консоли это равносильно потере доступа. Подробный разбор риска и опции `nofail` — в статье <a href="/a/25_disks_partitioning">Работа с новыми дисками и создание разделов</a> (раздел "Автомонтирование при загрузке").

---

<a id="proverka-lvm"></a>

## 🔎 Проверка состояния LVM

```bash
sudo pvs
sudo vgs
sudo lvs
```

---

<a id="rasshirenie-lvm"></a>

## 📈 Расширение LVM и файловой системы

### 1️⃣ Добавляем новый PV

```bash
sudo pvcreate /dev/sdc1
sudo vgextend vg_data /dev/sdc1
```

### 2️⃣ Расширяем LV

```bash
sudo lvextend -l +100%FREE /dev/vg_data/lv_backup
```

> [!TIP]
> Оба шага (расширение LV + расширение ФС) можно сделать одной командой, добавив `-r`/`--resizefs` — `lvextend` сам вызовет нужный `resize2fs`/`xfs_growfs` после расширения:
> ```bash
> sudo lvextend -l +100%FREE -r /dev/vg_data/lv_backup
> ```

### 3️⃣ Расширяем файловую систему

**ext4:**

```bash
sudo resize2fs /dev/vg_data/lv_backup
```

**XFS:**

```bash
sudo xfs_growfs /mnt/data
```

> [!IMPORTANT]
> `xfs_growfs` берёт точку монтирования, а не устройство (в отличие от `resize2fs` выше) — это не опечатка. Важнее другое: XFS можно только **увеличивать**, уменьшить существующий раздел XFS штатными средствами нельзя вообще (в отличие от ext4, который `resize2fs` умеет уменьшать при отмонтированной ФС) — если нужна возможность уменьшения в будущем, ext4 гибче.

---

<a id="dobavlenie-diska"></a>

## 🧩 Добавление нового диска в существующую группу

Если добавлен диск `/dev/sdb`:

```bash
sudo parted /dev/sdb
(parted) mklabel gpt
(parted) mkpart primary 0% 100%
(parted) set 1 lvm on
(parted) quit
```

Затем:

```bash
sudo pvcreate /dev/sdb1
sudo vgextend vg_data /dev/sdb1
sudo lvextend -l +100%FREE /dev/vg_data/lv_backup
sudo resize2fs /dev/vg_data/lv_backup
```

---

<a id="udalenie-lvm"></a>

## 🧹 Удаление LVM

```bash
sudo umount /mnt/data
sudo lvremove /dev/vg_data/lv_backup
sudo vgremove vg_data
sudo pvremove /dev/sdb1
```

> [!NOTE]
> Если по ходу статьи в группу добавлялись ещё диски (`/dev/sdc1` из раздела "Расширение LVM", `/dev/sdb1` из раздела "Добавление нового диска") — `pvremove` нужно выполнить для **каждого** физического тома, который входил в `vg_data`, а не только для одного. `vgremove` уничтожает саму группу, но метки PV на остальных дисках без отдельного `pvremove` останутся — диск не будет частью активной VG, но старая LVM-сигнатура на нём сохранится (может путать `pvs`/будущее переиспользование этого диска).

<a id="pod-kapotom"></a>

## 🧠 Под капотом LVM

| Команда                            | Назначение                            |
| ---------------------------------- | ------------------------------------- |
| `pvcreate`                         | Инициализирует раздел под LVM         |
| `vgcreate`                         | Создаёт группу логических томов       |
| `vgextend`                         | Добавляет новый PV в существующую VG  |
| `lvcreate`                         | Создаёт логический том                |
| `lvextend`                         | Расширяет существующий логический том |
| `resize2fs` / `xfs_growfs`         | Расширяет файловую систему            |
| `lvremove`, `vgremove`, `pvremove` | Удаляют соответствующие объекты LVM   |

---

<a id="ascii-schema"></a>

## 📊 ASCII-схема: Расширение LVM (1 диск → 2 диска)

```
До:
┌────────────┐
│  /dev/sda1 │──┐
└────────────┘  │
                 │  → VG "vg_data"
                 ├──► LV "lv_backup" (100G)
                 │
После добавления /dev/sdb1:
┌────────────┐  ┌────────────┐
│  /dev/sda1 │  │  /dev/sdb1 │
└────────────┘  └────────────┘
      │               │
      └──────┬────────┘
             ▼
        VG "vg_data"
             │
             ▼
      LV "lv_backup" (расширен до 2TB)
             │
             ▼
       Файловая система (ext4 / xfs)
```

---

<a id="sovety"></a>

## 💡 Полезные советы

* Для серверов: **LVM + XFS** (высокая производительность)
* Для рабочих станций: **LVM + ext4** (гибкость и стабильность)
* Для SSD: добавляйте `discard` в `/etc/fstab`
* Проверяйте корректность перед перезагрузкой:

  ```bash
  sudo mount -a
  ```
* Для резервного копирования:

  ```bash
  sudo lvcreate -s -L 10G -n snapshot_backup /dev/vg_data/lv_backup
  ```

> [!WARNING]
> `-L 10G` здесь — размер выделенного места именно под **снапшот** (COW — copy-on-write), а не под сам бэкап. Если за время жизни снапшота в исходный том `lv_backup` (в примерах статьи — 100G+) будет записано/изменено больше 10G данных — место под снапшот кончится, и он станет **невалидным** (LVM пометит его `invalid`, снапшот больше нельзя использовать для восстановления), причём без явной ошибки в момент переполнения, если не следить самому. Проверяйте заполненность снапшота командой `sudo lvs -o+snap_percent` и берите размер с запасом (или используйте thin-provisioning LVM, где эта проблема решается иначе) — особенно если снапшот держится дольше нескольких минут/часов на активно изменяемом томе.

---

<a id="vyvod"></a>

## ✅ Вывод

**LVM** — мощная система управления томами, позволяющая:

* масштабировать хранилища без потерь данных;
* объединять диски в единое пространство;
* создавать снапшоты и резервные копии;
* работать без перезагрузки.

Использование **LVM** — стандарт для современных серверных и корпоративных систем Linux.
