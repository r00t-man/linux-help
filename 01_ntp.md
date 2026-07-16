---
layout: default
title: "Настройка NTP"
permalink: /01_ntp/
---

# ⏱ Настройка NTP и часового пояса на Astra Linux и РЕД ОС

Инструкция по настройке синхронизации времени (**NTP**) и смены часового пояса для консолей **Astra Linux** и **РЕД ОС (7.3/8.0)**. Рассчитана на сети с собственным сервером времени и на работу **без интернета** (пакеты ставятся из заранее скачанного репозитория/файлов).

---

## 🔧 Общие понятия

- Собственный NTP-сервер в примерах: `192.168.10.100`
- Работаем **только в консоли**, без GUI
- NTP синхронизирует **UTC-время** внутри системы; часовой пояс — это только то, как это UTC-время **отображается** локально. Это две независимые настройки, менять их нужно по отдельности.
- На линуксе за синхронизацию времени отвечает **одна из трёх** служб (одновременно работает обычно только одна):
  - **`systemd-timesyncd`** — минимальный SNTP-клиент, идёт в комплекте с systemd. Только клиент (сам сервером времени быть не может), нет статистики дрейфа/джиттера.
  - **`chrony`** (юнит `chronyd`) — современный демон, хорошо держит время на нестабильных сетях и в виртуальных машинах, может работать и клиентом, и сервером.
  - **`ntpd`** (пакет `ntp`) — классический демон референсной реализации NTP.

> [!IMPORTANT]
> Название **юнита** для классического демона отличается не между версиями РЕД ОС (7.3/8.0 — одинаково), а **между семействами ОС**:
> - **Astra Linux** (Debian/apt) → пакет `ntp` → юнит **`ntp.service`**
> - **РЕД ОС** (RPM/RHEL-семейство, yum/dnf) → пакет `ntp` → юнит **`ntpd.service`**
>
> `chrony` в обеих семьях называется одинаково — юнит `chronyd.service`. Если запускаете `systemctl restart ntp` на РЕД ОС или `systemctl restart ntpd` на Astra — получите `Unit not found`, это самая частая ошибка при копировании команд между дистрибутивами.

---

## 📌 Шаг 1. Определить, какая служба времени сейчас работает

Не гадайте — проверьте на месте, что реально стоит и активно:

```bash
systemctl status systemd-timesyncd chronyd ntp ntpd 2>/dev/null | grep -E "^●|Active:|Loaded:"
# по отдельности (одна выведет "active", остальные "inactive"/"could not be found")
systemctl is-active systemd-timesyncd
systemctl is-active chronyd
systemctl is-active ntp      # Astra
systemctl is-active ntpd     # РЕД ОС
```

Также полезно посмотреть, что вообще установлено:

```bash
# Astra (Debian)
dpkg -l | grep -E "^ii\s+(ntp|chrony)"

# РЕД ОС (RPM)
rpm -qa | grep -E "^(ntp|chrony)"
```

---

## 🐧 Astra Linux (Debian, apt/dpkg)

### `systemd-timesyncd` (часто установлен по умолчанию)

```bash
sudo nano /etc/systemd/timesyncd.conf
```

```ini
[Time]
NTP=192.168.10.100
FallbackNTP=
```
`FallbackNTP` — заполняем, если есть второй (резервный) сервер времени.

```bash
sudo systemctl restart systemd-timesyncd
sudo systemctl enable systemd-timesyncd
timedatectl status
```

**Удаление** (переход на chrony/ntp — timesyncd нужно сначала выключить, иначе будет конфликтовать за порт 123):
```bash
sudo systemctl disable --now systemd-timesyncd
```
(пакет отдельно не удаляется — `systemd-timesyncd` идёт частью пакета `systemd`, просто отключаем юнит)

### `chrony` (рекомендуется — стабильнее на ВМ)

```bash
sudo apt update
sudo apt install chrony
sudo nano /etc/chrony/chrony.conf
```

```conf
server 192.168.10.100 iburst
# закомментируйте остальные строки server/pool из дефолтного конфига
```

```bash
sudo systemctl restart chrony
sudo systemctl enable chrony
chronyc sources -v
chronyc tracking
```

**Удаление:**
```bash
sudo systemctl disable --now chrony
sudo apt remove --purge chrony
```

### `ntp` (юнит `ntp.service`)

```bash
sudo apt install ntp
sudo nano /etc/ntp.conf
```

```conf
server 192.168.10.100 iburst
# закомментируйте штатные pool.ntp.org строки
```

```bash
sudo systemctl restart ntp
sudo systemctl enable ntp
ntpq -p
```

**Удаление:**
```bash
sudo systemctl disable --now ntp
sudo apt remove --purge ntp
```

---

## 🔴 РЕД ОС 7.3 / 8.0 (RPM, yum/dnf)

> [!NOTE]
> На РЕД ОС **нет** `apt`/`dpkg` — только `yum`/`dnf` и `.rpm`-пакеты. На 7.3 оба бинарника (`yum` и `dnf`) обычно присутствуют и работают (проверено на реальном разворачивании — см. `dnf install` и `yum install` в одной и той же инструкции по подготовке 7.3). На 8.0 ожидаемо основным является `dnf` (yum там обычно просто alias на dnf, как в RHEL8) — если сомневаетесь, проверьте `which yum dnf` на конкретной машине перед тем как писать команды в регламент.

### `chrony`

```bash
sudo yum install chrony       # или: sudo dnf install chrony
sudo nano /etc/chrony.conf    # иногда путь /etc/chrony/chrony.conf — проверьте, какой реально читается
```

```conf
server 192.168.10.100 iburst
```

```bash
sudo systemctl restart chronyd
sudo systemctl enable chronyd
chronyc sources -v
chronyc tracking
```

**Удаление:**
```bash
sudo systemctl disable --now chronyd
sudo yum remove chrony        # или dnf remove chrony
```

### `ntpd` (юнит **`ntpd.service`**, не `ntp`!)

```bash
sudo yum install ntp          # или: sudo dnf install ntp
sudo nano /etc/ntp.conf
```

```conf
server 192.168.10.100 iburst
```

```bash
sudo systemctl restart ntpd
sudo systemctl enable ntpd
ntpq -pn
```

**Удаление:**
```bash
sudo systemctl disable --now ntpd
sudo yum remove ntp            # или dnf remove ntp
```

> [!TIP]
> `systemd-timesyncd` на РЕД ОС можно не рассматривать вообще — на RHEL-семействе это не штатный путь настройки времени, используется `chrony`/`ntpd`.

---

## 📦 Установка без интернета (downloadonly) — офлайн-объекты

Рабочий сценарий: на виртуалке **с интернетом и такой же ОС** качаем пакет и все зависимости, переносим на объект флешкой/через контур.

### РЕД ОС (yum/dnf `--downloadonly`)

```bash
# ставим плагин, если downloadonly ещё недоступен (yum)
sudo yum install yum-plugin-downloadonly

# качаем пакет + зависимости, без установки, в свою папку
sudo yum install --downloadonly --downloaddir=/tmp/ntp-offline chrony
# или dnf (плагин download уже встроен через dnf-plugins-core/dnf download-)
sudo dnf install --downloadonly --downloaddir=/tmp/ntp-offline chrony
```

> [!WARNING]
> Если не указать `--downloaddir`, пакеты лягут в кэш (`/var/cache/dnf/.../packages` или `/var/cache/yum/...`) и **удалятся** после следующей операции с пакетным менеджером — сразу копируйте их в отдельную папку, см. [📥 Скачать пакет без установки](/a/gis-pkg_download_rpm_deb).

Перенос и установка на объекте без сети:
```bash
sudo yum localinstall /tmp/ntp-offline/*.rpm
# или
sudo dnf install /tmp/ntp-offline/*.rpm
```

### Astra Linux (apt `--download-only`)

```bash
sudo apt install --download-only chrony
# .deb-файлы окажутся в /var/cache/apt/archives/
sudo cp /var/cache/apt/archives/*.deb /tmp/ntp-offline/
```

Перенос и установка на объекте без сети:
```bash
sudo dpkg -i /tmp/ntp-offline/*.deb
sudo apt --fix-broken install -y   # если чего-то из зависимостей не хватило
```

> [!NOTE]
> Если конфигурационные пакеты (`chrony`/`ntp`) тянут зависимости, которых нет в оффлайн-репозитории объекта — на виртуалке с интернетом лучше явно раскомментировать/использовать те же репозитории, что будут доступны на целевой машине (локальный `file://`-репозиторий объекта), иначе версии зависимостей могут не совпасть. Подробный разбор обоих сценариев (Astra и Red OS) — в статье [📥 Скачать пакет без установки (для offline-установки)](/a/gis-pkg_download_rpm_deb).

---

## 🔥 Открытие порта (UDP 123)

Нужно **только если сама машина выступает NTP-сервером** для других хостов (принимает входящие запросы). Обычному клиенту, который сам ходит наружу к `192.168.10.100`, дополнительные правила обычно не нужны — исходящий запрос и ответ на него разрешены стандартной цепочкой `ESTABLISHED,RELATED`.

```bash
# iptables — открыть входящий NTP-порт (для роли сервера времени)
sudo iptables -A INPUT -p udp --dport 123 -j ACCEPT
```

> [!IMPORTANT]
> Для `nftables` или `firewalld` синтаксис будет другим (`nft add rule ...` / `firewall-cmd --add-service=ntp`) — команда выше только для классического `iptables`. На РЕД ОС в реальных развёртываниях этой вики firewall чаще держат через юнит `iptables.service` (пакет `iptables-services`), не `firewalld` — проверьте, что реально активно: `systemctl is-active firewalld iptables 2>/dev/null`.

---

## 🕒 Смена часового пояса на Екатеринбург

Часовой пояс: `Asia/Yekaterinburg`. Способ одинаковый и для Astra, и для РЕД ОС.

#### ✅ Через `timedatectl` (предпочтительно)

```bash
timedatectl                                    # проверить текущий
sudo timedatectl set-timezone Asia/Yekaterinburg
timedatectl                                    # проверить результат
date                                           # локальное время
```

#### ✅ Вручную через символическую ссылку

```bash
sudo rm -f /etc/localtime
sudo ln -s /usr/share/zoneinfo/Asia/Yekaterinburg /etc/localtime
```

> [!NOTE]
> `echo "Asia/Yekaterinburg" | sudo tee /etc/timezone` нужен **только на Astra/Debian** — этот файл читают только Debian-based системы. На РЕД ОС (RPM/RHEL-семейство) файла `/etc/timezone` в принципе нет и он ни на что не влияет — часовой пояс там определяется исключительно симлинком `/etc/localtime` (или `timedatectl`, что делает то же самое).

> [!TIP]
> После смены часового пояса перезапуск службы синхронизации времени не требуется — она как синхронизировала UTC, так и продолжает, меняется только отображение.

---

## 🧪 Проверка статуса — что означает вывод

| Служба | Команда | На что смотреть |
| --- | --- | --- |
| chrony | `chronyc tracking` | `Leap status: Normal` (не `Not synchronised`), `System time` — расхождение в секундах, малое = хорошо |
| chrony | `chronyc sources -v` | Строка с `^*` в начале — текущий выбранный источник времени; `^?` — источник недоступен |
| ntpd | `ntpq -p` / `ntpq -pn` | Строка с `*` слева — синхронизировались; колонка `reach` в восьмеричном виде, `377` = все последние 8 опросов дошли, `0` = сервер не отвечает (проверять сеть/firewall) |
| systemd-timesyncd | `timedatectl status` | Строка `System clock synchronized: yes` и `NTP service: active` |

Быстрая проверка, что сервер времени вообще доступен по сети (до настройки службы):
```bash
# однократный опрос без запуска демона
sudo chronyd -Q "server 192.168.10.100 iburst"
```

---

## 🐞 Диагностика и логи (по каждой службе)

```bash
# chrony
journalctl -u chronyd -f              # живой лог
journalctl -u chronyd --since "1 hour ago"
chronyc activity                      # сколько источников online/offline/burst
chronyc sourcestats                   # стабильность каждого источника (частота дрейфа)

# ntpd — юнит "ntp" на Astra, "ntpd" на РЕД ОС
journalctl -u ntp -f                  # Astra
journalctl -u ntpd -f                 # РЕД ОС
ntpq -c rv                            # подробное состояние демона (stratum, offset, precision)

# systemd-timesyncd
journalctl -u systemd-timesyncd -f
timedatectl show-timesync --all       # (systemd 245+) детальный статус клиента
```

**Частые сообщения и что они значат:**
- `chronyd`: `Server dropped: no data` / `Not synchronised` — сервер `192.168.10.100` недоступен по сети или закрыт порт 123 на его стороне.
- `ntpq -p` показывает пустую таблицу или `reach = 0` у всех строк — тот же диагноз: сеть/firewall, либо неверный IP/имя сервера в конфиге.
- Служба стартует, но время не двигается сразу после первого запуска — у `ntpd` есть защитный порог (не скачет одним махом больше ~1000 секунд без флага `-g`); если время на машине изначально сильно "уехало", сначала синхронизируйте разово: `sudo ntpd -gq`, либо для chrony добавьте/используйте `makestep 1.0 3` в `chrony.conf` (разрешает резкий скачок первые 3 попытки).

---

## ⚠️ Частые проблемы (подводные камни)

- **РЕД ОС — это RPM/yum-dnf, не ALT/apt.** Если в регламенте случайно оказалась команда `apt-get` для РЕД ОС — это ошибка, скопированная с Astra-раздела.
- **Юнит называется по-разному не из-за версии РЕД ОС, а из-за семейства ОС**: `ntp.service` — Astra, `ntpd.service` — РЕД ОС (обе версии, 7.3 и 8.0).
- **Две службы времени одновременно работать не должны** — если переходите с timesyncd на chrony (или наоборот), сначала `disable --now` у старой, иначе обе будут спорить за UDP 123 и синхронизация будет "дёргаться".
- **При работе без интернета** — офлайн-репозиторий/скачанные `.rpm`/`.deb` должны содержать не только сам пакет, но и все его зависимости (при `--downloadonly`/`--download-only` без `--downloaddir` — не забыть скопировать из кэша, он чистится).
- **На РЕД ОС проверяйте, что реально используется — `firewalld` или классический `iptables.service`** — команды управления firewall для них разные, а в этой инфраструктуре по факту чаще встречается `iptables.service`.
- **`/etc/timezone` — только для Astra/Debian.** На РЕД ОС этот файл не читается, часовой пояс там только через `/etc/localtime`/`timedatectl`.
- **Правило на UDP 123 нужно только серверу времени**, не рядовому клиенту — не открывайте порт без необходимости.

---

## ⚡ Пример полной настройки

### Astra Linux + chrony + Екатеринбург
```bash
sudo timedatectl set-timezone Asia/Yekaterinburg
echo "server 192.168.10.100 iburst" | sudo tee /etc/chrony/chrony.conf
sudo systemctl restart chrony
sudo systemctl enable chrony
timedatectl
chronyc tracking
date
```

### РЕД ОС 7.3/8.0 + chrony + Екатеринбург
```bash
sudo timedatectl set-timezone Asia/Yekaterinburg
echo "server 192.168.10.100 iburst" | sudo tee /etc/chrony.conf
sudo systemctl restart chronyd
sudo systemctl enable chronyd
timedatectl
chronyc tracking
date
```

---

## 🔖 Рекомендации

* **chrony** — самая точная и быстрая синхронизация, лучше всего ведёт себя на виртуальных машинах и нестабильных сетях. Рекомендуется по умолчанию для обоих семейств ОС.
* **ntpd** — классика, есть почти везде "из коробки" в старых регламентах, но менее гибок при большом начальном рассинхроне.
* **systemd-timesyncd** — простой клиент только для Astra/Debian, для серьёзных объектов лучше сразу ставить chrony.

> [!NOTE]
> Применимо для:
>
> * Astra SE 1.7.5 / 1.8
> * РЕД ОС 7.3 / 8.0
> * Названия юнитов/пакетов зависят от семейства ОС (Debian vs RHEL), а не от конкретной версии внутри семейства — см. таблицу и предупреждение в начале статьи.
