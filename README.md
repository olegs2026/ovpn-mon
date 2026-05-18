# ovpn-mon
OpenVpn monitoring script
# ovpn-mon

> Универсальная утилита мониторинга и диагностики **OpenVPN** для Linux.
> Работает одинаково с **нативными** инстансами (systemd / standalone) и
> с контейнерами **Docker** (`kylemanna/openvpn`, `linuxserver/openvpn-as`,
> любой свой образ с `openvpn` внутри).

Один Bash-скрипт, без внешних зависимостей сверх стандартного userland
(`awk`, `grep`, `ss`/`netstat`, `openssl`, `docker` — по необходимости).

---

## ✨ Возможности

- 🔍 **Автодискавери** — находит запущенные OpenVPN-инстансы:
  - `systemd` units (`openvpn-server@*.service`, `openvpn@*.service`)
  - standalone `*.conf` в `/etc/openvpn/`
  - Docker-контейнеры (по образу и процессу)
- 👥 **Watch-режим** (`-w`) — список онлайн-клиентов с авто-обновлением:
  - CN, реальный адрес, виртуальный IP, RX/TX, длительность, idle
  - цветные индикаторы активности (●)
  - поддержка **status log v1 и v2/v3**
- 🕓 **История** (`-H`) — события подключений/отключений за N дней:
  - источники: лог-файл, `journalctl`, `docker logs`
  - распознаёт форматы syslog, journalctl short-iso, native OpenVPN
  - события: `CONNECT`, `DISCONNECT`, `AUTH-OK`, `AUTH-FAIL`, `TLS-FAIL`, `TIMEOUT`
- 🩺 **Диагностика** (`-d`):
  - `ip_forward`, tun-интерфейсы, listening порты
  - правила `nftables` / `iptables` NAT
  - срок действия сертификатов CA/server (inline и внешние)
  - sanity-check конфига (port/proto/dev/server/cipher)
  - проверка состояния docker-контейнера, tun и сокетов **внутри** него

---

## 📦 Установка

```bash
sudo curl -fsSL https://raw.githubusercontent.com/<YOUR_USER>/ovpn-mon/main/ovpn-mon \
  -o /usr/local/bin/ovpn-mon
sudo chmod +x /usr/local/bin/ovpn-mon
```

Или вручную:

```bash
git clone https://github.com/<YOUR_USER>/ovpn-mon.git
cd ovpn-mon
sudo install -m 755 ovpn-mon /usr/local/bin/ovpn-mon
```

---

## 🚀 Использование

```text
ovpn-mon [-w | -s | -H | -d | -a | -h]
```

| Команда         | Что делает                                                              |
|-----------------|-------------------------------------------------------------------------|
| `-w`, `--watch` | **(default)** Онлайн-клиенты с авто-обновлением каждые `WATCH_INTERVAL` |
| `-s`, `--snapshot` | Один снимок watch (без цикла) — удобно для cron/CI                   |
| `-H`, `--history` | События за последние `HISTORY_DAYS` дней (до `HISTORY_LIMIT` строк)   |
| `-d`, `--diag`  | Полная диагностика (система, порты, firewall, сертификаты, конфиг)      |
| `-a`, `--all`   | Snapshot + history + diag (одним проходом, для отчёта)                  |
| `-h`, `--help`  | Справка                                                                 |

### Примеры

```bash
sudo ovpn-mon                       # watch с интервалом 3с (по умолчанию)
WATCH_INTERVAL=1 sudo ovpn-mon -w   # обновление каждую секунду
sudo ovpn-mon -s                    # один снимок (для скриптов)
sudo ovpn-mon -H                    # история за 7 дней
HISTORY_DAYS=30 HISTORY_LIMIT=100 sudo ovpn-mon -H
sudo ovpn-mon -d                    # диагностика
sudo ovpn-mon -a > report.txt       # полный отчёт в файл
```

---

## 🔧 Переменные окружения

| Переменная        | По умолчанию | Описание                                       |
|-------------------|--------------|------------------------------------------------|
| `WATCH_INTERVAL`  | `3`          | Интервал обновления в `-w` (секунды)           |
| `HISTORY_DAYS`    | `7`          | За сколько дней показывать историю             |
| `HISTORY_LIMIT`   | `25`         | Максимум событий в выводе `-H`                 |
| `DEBUG`           | `0`          | Отладочные сообщения discovery (в stderr)      |

---

## 🖼 Пример вывода

```text
╔════════════════════════════════════════════════════════════════════╗
║   OpenVPN — Universal Status   (mode: watch)                       ║
╚════════════════════════════════════════════════════════════════════╝
 Host   : vpn-01    Time: 2026-05-18 13:05:12 UTC
 Native : -
 Version: not installed

━━━ Docker instances (1) ━━━

 ┌─ container openvpn (0e65465e3bdf)  image=kylemanna/openvpn
 └─  status: /etc/openvpn/status/openvpn-status.log

  status: docker:0e65465e3bdf:/etc/openvpn/status/openvpn-status.log
  updated: Mon May 18 13:05:10 2026

   CN                   REAL ADDRESS            VIRTUAL IP      RECV         SENT         DURATION        LAST REF
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────
 ● configuration         193.39.160.96:2758       192.168.255.6   3.45 KB      3.60 KB      9m 28s          Mon May 18 13:05:10 2026
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   Итого: 1 клиент(ов) онлайн  •  RECV: 3.45 KB  SENT: 3.60 KB  •  format: v1

Легенда: ● активен (<5m)   ● idle 5-30m   ● idle >30m
```

---

## 🧩 Поддерживаемые источники

### Status log (для `-w` / `-s`)
- **v1** — классический `OpenVPN CLIENT LIST` / `ROUTING TABLE` (по умолчанию в большинстве сборок и `kylemanna/openvpn`)
- **v2 / v3** — формат с префиксами `CLIENT_LIST,...`, `ROUTING_TABLE,...`
  (включается директивой `status-version 2|3`)

### Логи (для `-H`)
- Native OpenVPN: `Mon May 18 14:25:43 2026 ...`
- syslog: `May 18 13:25:11 host openvpn[pid]: ...`
- journalctl short-iso: `2026-05-18T13:25:11+0300 host openvpn[pid]: ...`
- `docker logs` (stdout/stderr контейнера)

---

## 🛠 Требования

- Bash 4+
- Linux с `awk`, `grep`, `sed`, `date`, `find` (coreutils)
- Опционально: `openssl` (проверка срока сертификатов), `ss`/`netstat`,
  `iptables`/`nft`, `systemctl`, `journalctl`, `docker`

Скрипт сам определяет, что доступно, и отключает соответствующие проверки,
если инструмент отсутствует.

---

## ⚠️ Особенности Docker

При работе OpenVPN в контейнере хост не видит ни `tun*`, ни UDP-сокет —
это **нормально**. `ovpn-mon` корректно проверяет эти параметры **внутри**
контейнера через `docker exec`.

Образ должен содержать `ip`, `ss` или `netstat` для полной диагностики
(в `kylemanna/openvpn` есть `ip`, нет `ss`/`netstat`).

---

## 🐞 Отладка

```bash
DEBUG=1 sudo ovpn-mon -d 2>&1 | less
```

Если что-то не парсится — соберите 5-10 строк сырого лога и откройте Issue.

---

## 📄 Лицензия

[MIT](./LICENSE)

---

## 🤝 Вклад

PR и issues приветствуются. Особенно полезно прислать:
- Примеры status-log/логов нестандартных сборок OpenVPN (pfSense, OPNsense, Mikrotik export и т. п.)
- Кейсы с несколькими инстансами одновременно
- Поддержку IPv6 в выводе клиентов
