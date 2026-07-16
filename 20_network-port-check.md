---
layout: default
title: "Проверка сетевой доступности"
permalink: /20_network-port-check/
parent: "🌐 Сети"
nav_order: 5
---

# 📡 Проверка сетевой доступности и портов

В этом гайде собраны основные CLI-команды для проверки сетевой доступности, маршрута, открытых портов и состояния сервисов в Linux и Windows.

---

## 🧭 Таблица основных утилит

| Команда | Назначение | Основные параметры | Пример |
|----------|-------------|--------------------|--------|
| `ping` | Проверяет доступность узла по ICMP | `-c <count>` — количество пакетов | `ping -c 4 10.50.0.1` |
| `traceroute` | Отслеживает маршрут к узлу | `-T` — TCP SYN, `-p` (строчная!) — порт | `traceroute -T -p 7880 10.50.0.1` |
| `mtr` | Комбинация ping + traceroute | `-T` — TCP, `-P` (ЗАГЛАВНАЯ!) — порт | `mtr -T -P 80 10.50.0.1` |
| `nc` (netcat) | Проверка портов и соединений | `-z` — без передачи данных, `-v` — подробный вывод, `-u` — UDP | `nc -zv 10.50.0.1 22` |
| `telnet` | Проверка TCP-порта вручную | `<host> <port>` | `telnet 10.50.0.1 443` |
| `/dev/tcp` | Встроенная проверка без nc | — | `bash -c "</dev/tcp/10.50.0.1/8080"` |
| `nmap` | Сканирование портов и сервисов | `-p` — порты, `-sS` — TCP SYN | `nmap -p 22,80,443 10.50.0.1` |
| `curl` | Проверка HTTP/HTTPS и API | `-I` — только заголовки, `-v` — отладка | `curl -I http://10.50.0.1:8080` |
| `ss` | Показ активных соединений | `-tulnp` — TCP/UDP, с PID | `ss -tulnp` |
| `lsof` | Просмотр процессов, занявших порты | `-i` — сеть, `-P` — не преобразовывать порты | `lsof -i -P` |
| `openssl s_client` | Проверка SSL/TLS-соединения | `-connect <host:port>` | `openssl s_client -connect 10.50.0.1:443` |
| `Test-NetConnection` | PowerShell: проверка TCP-порта | `-ComputerName`, `-Port` | `Test-NetConnection -ComputerName 10.50.0.1 -Port 14000` |

---

## 📑 Содержание

- [Установка необходимых утилит](#install-tools)
- [Базовые проверки в Linux](#linux-checks)
- [PowerShell / Windows проверки](#windows-checks)
- [Дополнительные инструменты](#extra-tools)
- [Примеры практических проверок](#examples)
- [Полезные советы](#tips)
- [Минимальный набор утилит](#summary)

---

<a id="install-tools"></a>
## ⚙️ Установка необходимых утилит

Если некоторых команд нет в системе, установите их с помощью пакетного менеджера.

### 🟢 Astra / Debian / Ubuntu
```bash
sudo apt update
sudo apt install -y traceroute netcat nmap mtr-tiny telnet curl lsof iputils-ping
```

| Пакет            | Назначение                          |
| ---------------- | ----------------------------------- |
| `traceroute`     | Отслеживание маршрута TCP/ICMP      |
| `netcat` (`nc`)  | Проверка портов, TCP/UDP соединений |
| `nmap`           | Сканирование портов и сервисов      |
| `mtr-tiny`       | Комбинация traceroute + ping        |
| `telnet`         | Проверка TCP-портов вручную         |
| `curl`           | Проверка HTTP(S)/API                |
| `lsof`           | Просмотр открытых портов            |
| `iputils-ping`   | Добавляет ping, если отсутствует (обычно уже стоит из коробки) |

---

### 🔴 Red OS / ALT / RHEL / CentOS

```bash
sudo dnf install -y traceroute nmap nmap-ncat mtr telnet curl lsof iputils
```

или:

```bash
sudo yum install -y traceroute nmap nmap-ncat mtr telnet curl lsof iputils
```

| Пакет        | Назначение                                 |
| ------------ | ------------------------------------------ |
| `traceroute` | Анализ маршрута                            |
| `nmap-ncat`  | Утилита `nc`                               |
| `nmap`       | Расширенное сканирование                   |
| `mtr`        | Диагностика маршрута и потерь              |
| `telnet`     | Тест TCP-соединений                        |
| `curl`       | Проверка веб-сервисов                      |
| `lsof`       | Анализ портов                              |
| `iputils`    | Набор сетевых утилит (`ping`, `tracepath`) |

---

<a id="linux-checks"></a>

## 🐧 Базовые проверки в Linux

### 1. Проверка маршрута (TCP SYN)

```bash
traceroute -T -p 7880 10.50.0.1
```

> [!NOTE]
> Обратите внимание на регистр флага порта — у `traceroute` строчная `-p`, у `mtr` (ниже) заглавная `-P`. Частая опечатка при копировании команды между этими двумя похожими утилитами.

### 2. Проверка соединения через `nc`

```bash
nc -zv 10.50.0.1 7880
```

Диапазон портов:

```bash
nc -zv 10.50.0.1 20-25
```

UDP-проверка:

```bash
nc -zvu 10.50.0.1 53
```

### 3. Проверка через `telnet`

```bash
telnet 10.50.0.1 22
```

### 4. Проверка без netcat — `/dev/tcp`

```bash
timeout 3 bash -c "</dev/tcp/10.50.0.1/7880 && echo Port open || echo Port closed"
```

### 5. Проверка открытых портов локально

```bash
ss -tulnp
```

### 6. Проверка маршрута и задержек

```bash
mtr -T -P 7880 10.50.0.1
```

---

<a id="windows-checks"></a>

## 🪟 PowerShell / Windows проверки

### Проверка TCP-порта

```powershell
Test-NetConnection -ComputerName 10.50.0.1 -Port 14000
```

### Проверка маршрута

```powershell
tracert 10.50.0.1
```

### Проверка ICMP

```powershell
Test-Connection 10.50.0.1 -Count 4
```

### Проверка нескольких портов

```powershell
$ports = 22,80,443,14000
foreach ($p in $ports) {
    Test-NetConnection -ComputerName 10.50.0.1 -Port $p |
    Select-Object ComputerName, RemotePort, TcpTestSucceeded
}
```

---

<a id="extra-tools"></a>

## 🔍 Дополнительные инструменты

| Утилита                     | Платформа       | Назначение                              |
| --------------------------- | --------------- | --------------------------------------- |
| `nmap`                      | Linux / Windows | Глубокое сканирование портов и сервисов |
| `ss`                        | Linux           | Анализ активных TCP/UDP соединений      |
| `lsof -i`                   | Linux/macOS     | Показ процессов, занявших порты         |
| `curl`                      | Все             | Проверка HTTP/HTTPS                     |
| `openssl s_client -connect` | Все             | Проверка SSL/TLS-соединений             |
| `ping`, `mtr`, `tracert`    | Все             | Проверка маршрута и задержек            |

---

<a id="examples"></a>

## 🧩 Примеры практических проверок

| Цель                      | Команда                                                  | Комментарий           |
| ------------------------- | --------------------------------------------------------- | --------------------- |
| Проверить порт API (8080) | `nc -zv api.local 8080`                                    | TCP-проверка          |
| Проверить HTTP            | `curl -I http://10.50.0.1:8080`                            | Заголовки HTTP        |
| Проверить PostgreSQL      | `nc -zv 10.50.0.1 5432`                                    | TCP-порт              |
| Проверить Redis           | `timeout 3 bash -c "</dev/tcp/10.50.0.1/6379 && echo OK \|\| echo Fail"` | Без `nc` |
| Проверить RDP             | `Test-NetConnection 10.50.0.1 -Port 3389`                  | PowerShell            |
| Проверить SSH             | `nc -zv 10.50.0.1 22`                                      | Проверка доступности  |
| Проверить SSL             | `openssl s_client -connect 10.50.0.1:443`                  | Анализ сертификата    |

---

<a id="tips"></a>

## 💡 Полезные советы

* Если ICMP (ping) заблокирован — используйте `traceroute -T` или `nc -zv`.
* Нет `nc`? Используйте встроенный `/dev/tcp/host/port`.
* Для массовой проверки портов:

  ```bash
  nmap -p 22,80,443,14000 10.50.0.1
  ```
* Для аудита сохраняйте результаты:

  ```bash
  HOSTNAME=$(hostname)
  nc -zv 10.50.0.1 22-443 > /home/gis-audit/network_check_${HOSTNAME}.log
  ```

---

<a id="summary"></a>

## 🧰 Минимальный набор утилит

🔹 Рекомендуется установить:

```
traceroute netcat nmap mtr curl lsof telnet
```

🔹 Установка одной строкой:

* **Debian / Astra / Ubuntu:**

  ```bash
  sudo apt install -y traceroute netcat nmap mtr-tiny telnet curl lsof
  ```

* **RedOS / RHEL / ALT:**

  ```bash
  sudo dnf install -y traceroute nmap nmap-ncat mtr telnet curl lsof
  ```

---

📗 После установки этих инструментов вы сможете выполнять полный набор сетевых проверок — от ICMP и TCP до SSL и API-запросов на любой системе.
