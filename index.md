# 📘 Основные Linux команды - заметки 🗃️

[![Platform](https://img.shields.io/badge/platform-Linux-lightgrey?style=flat-square&logo=linux)](https://kernel.org)
![Tested on](https://img.shields.io/badge/tested%20on-Red%20OS%207.3%20%7C%208.0%20%7C%20Astra%20SE%201.7.5%20%7C%201.8-orange?style=flat-square)
![Docs](https://img.shields.io/badge/docs-markdown-blueviolet?style=flat-square)

Решил собрать полезные инструкции и шпаргалки для работы в Linux.  
(Astra Linux, РЕД ОС и другие дистрибутивы).  
Все материалы собрал в формате отдельных статей для быстрого использования 📝

## 📌 Как использовать

Можно открывать статьи прямо на сайте.  
Либо клонировать себе мой репозиторий с GitHub:  
👉 [Перейти к репозиторию](https://github.com/r00t-man/linux-help)  

А также можно просто скачать статьи с GitHub и использовать на своё усмотрение. <br>
Если скачать статьи — удобно просматривать в [VSCode](https://code.visualstudio.com/), все статьи написаны в Markdown.

Как всегда делаю для себя — делюсь со всеми 💁

---

## 📑 Содержание

<div class="two-columns">

<h3>⏰ Время и синхронизация</h3>
<ol>
  <li><a href="01_ntp">Настройка NTP (chrony, ntpd, systemd-timesyncd)</a></li>
  <li><a href="07_file_timestamps">Временные метки файлов в Linux</a></li>
</ol>

<h3>📂 Работа с файлами и резервным копированием</h3>
<ol start="3">
  <li><a href="02_cp_cron_rsync">Копирование файлов: cp, cron, rsync</a></li>
  <li><a href="03_copy_advanced">Расширенные техники копирования</a></li>
  <li><a href="04_archives">Работа с архивами</a></li>
  <li><a href="18_archive">Работа с архивами расширено</a></li>
  <li><a href="13_file_operation">Операции с файлами</a></li>
  <li><a href="14_dir_operation">Операции с директориями</a></li>
  <li><a href="15_find_file">Поиск и просмотр файлов</a></li>
  <li><a href="rsync-ssh-auto-transfer">Автоматическая передача файлов между серверами через rsync и SSH</a></li>
</ol>

<h3>📊 Мониторинг и информация о системе</h3>
<ol start="11">
  <li><a href="05_monitoring">Мониторинг и отладка Linux (top, htop, dstat, journalctl)</a></li>
  <li><a href="beszel-quick-start">Мониторинг Beszel — быстрый старт</a></li>
  <li><a href="06_sysinfo">Инфо о системе</a></li>
  <li><a href="06_01_system-audit">Инфо о системе - подробно</a></li>
</ol>

<h3>👯 Пользователи и права</h3>
<ol start="13">
  <li><a href="08_users">Управление пользователями и группами</a></li>
  <li><a href="12_permissions">Управление правами доступа (chmod, chown, umask, ACL)</a></li>
  <li><a href="unlock_admin_astra">Разблокировка администратора Astra - ПРОВЕРЕННО </a></li>
  <li><a href="unlock_user_astra"> Доп. Разблокировка пользователя в Astra Linux</a></li>
  <li><a href="02_pam_unlock">Доп. Определение PAM-блокировок</a></li>
</ol>

<h3>🌐 Сети</h3>
<ol start="15">
  <li><a href="10_network_basics">Кратко про работу с сетями</a></li>
  <li><a href="11_network_details">Подробнее про сети (ip, nmcli, netplan, firewall)</a></li>
  <li><a href="16_Bonding">Настройка и виды сетевых бондов (bonding) в Linux</a></li>
  <li><a href="17_inet_static">Статичные интерфейсы</a></li>
  <li><a href="20_network-port-check">Проверка сетевой доступности</a></li>
</ol>

<h3>⚙️ История/логи и автоматизация</h3>
<ol start="20">
  <li><a href="09_shell_history">История команд (прокачиваем history)</a></li>
  <li><a href="23_log-rotation-native-only">Ротация логов встроенными средствами</a></li>
  <li><a href="23_1_journalctl-audit-check">Проверка аудита журнала и контроля состояния journald</a></li>
  <li><a href="cron-guide">Планировщик Cron</a></li>
</ol>

<h3>🔧 Управление службами</h3>
<ol start="23">
  <li><a href="19_systemctl-guide">Полный гайд по systemctl и unit-файлам (systemd)</a></li>
  <li><a href="21_journalctl-guide">Journalctl — исчерпывающий справочник</a></li>
  <li><a href="21_1_journalctl-guide">Journalctl — Часть 2: конфигурация systemd-journald</a></li>
  <li><a href="21_2_journalctl-remote">Journalctl — Часть 3: Централизованный сбор журналов</a></li>
  <li><a href="22_rsyslog-guide">Руководство по rsyslog</a></li>
</ol>


<h3>🛡️ Безопасность сервера (раздел info)</h3>
<ol start="28">
  <li><a href="12_permissions">Управление правами доступа (chmod, chown, umask, ACL)</a></li>
  <li><a href="23_1_journalctl-audit-check">Проверка аудита журнала и контроля состояния journald</a></li>
  <li><a href="unlock_admin_astra">Разблокировка администратора Astra - ПРОВЕРЕННО</a></li>
  <li><a href="unlock_user_astra">Разблокировка пользователя в Astra Linux</a></li>
  <li><a href="02_pam_unlock">Определение PAM-блокировок</a></li>
</ol>

<h3>📎 Доп - Инфо</h3>
<p>Раздел добавлен для переноса статей из <code>my-wiki</code> (кроме <code>Audit-history.md</code>).</p>

<h3>🖴 Управление дисками</h3>
<ol start="28">
  <li><a href="25_Disks_Partitioning">Работа с новыми дисками и создание разделов в Linux (fdisk, parted)</a></li>
  <li><a href="25_LVM">Настройка и расширение LVM в Linux</a></li>
</ol>
</div>
