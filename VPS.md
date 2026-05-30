# 🖥️ Полная инструкция по настройке VPN-ноды (Remnawave)

> **Поддерживаемые ОС:** Ubuntu 22.04/24.04, Debian 12 (bookworm), Debian 13 (trixie).
> Команды универсальны для всех перечисленных систем; отличия вынесены в отдельные заметки.
>
> **Что обновлено в этой версии:** актуальная ссылка на TLS-сканер, установка `systemd-resolved` на минимальных образах Debian, актуальные поля Xray (`target`/`raw` вместо `dest`/`tcp`), корректная проверка Xray для новых версий ноды (встроен в процесс `rw-core`), варианты DNS (AdGuard/Cloudflare/Google) и закрепление conntrack после перезагрузки.

---

## 📋 Содержание

1. [Подготовка сервера](#1-подготовка-сервера)
2. [Установка Docker](#2-установка-docker)
3. [Оптимизация системы](#3-оптимизация-системы)
4. [DNS over TLS](#4-dns-over-tls)
5. [Автоматические обновления ОС](#5-автоматические-обновления-ос)
6. [Firewall (UFW)](#6-firewall-ufw)
7. [Fail2ban](#7-fail2ban)
8. [TLS сканирование (выбор домена для маскировки)](#8-tls-сканирование-выбор-домена-для-маскировки)
9. [Установка Remnanode](#9-установка-remnanode)
10. [Настройка логирования](#10-настройка-логирования)
11. [Настройка в панели Remnawave](#11-настройка-в-панели-remnawave)
12. [Проверка работоспособности](#12-проверка-работоспособности)
13. [Диагностика проблем](#13-диагностика-проблем)
14. [Проверка и очистка Firewall](#14-проверка-и-очистка-firewall)

---

## 1. Подготовка сервера

### 1.1 Проверка базовой информации

```bash
# Информация об ОС
cat /etc/os-release | head -3

# IP адреса
ip a

# Геолокация IP (если curl ещё не установлен — см. шаг 1.2)
curl -s https://ipinfo.io/$(hostname -I | awk '{print $1}')

# Текущие лимиты
echo "=== Limits ==="
ulimit -n
cat /proc/sys/net/core/somaxconn
cat /proc/sys/net/netfilter/nf_conntrack_max 2>/dev/null || echo "conntrack not loaded"

# Проверка Docker
docker --version 2>/dev/null || echo "Docker not installed"

# Занятые порты
ss -tulpn | grep -E ':22|:80|:443'
```

> 💡 **Минимальные образы Debian.** На «голых» образах Debian 12/13 часто отсутствуют `curl`, `wget` и другие утилиты — это нормально, они ставятся на шаге 1.2. Если `curl` в блоке выше выдал `command not found` — просто продолжайте, после 1.2 команда заработает.

### 1.2 Обновление системы

```bash
apt update && apt upgrade -y
apt install -y mc htop btop iftop curl wget
```

### 1.3 Настройка часового пояса

```bash
timedatectl set-timezone Europe/Moscow
timedatectl
```

> 💡 Часовой пояс влияет только на время в логах и на время запуска cron-задачи автообновления (раздел 9.6, суббота 11:00). Если сервер в другом регионе и вам удобнее местное время — поставьте свой пояс, например `Europe/Helsinki`. На работу VPN это не влияет.

---

## 2. Установка Docker

Официальный скрипт установки Docker корректно определяет Ubuntu и Debian (включая 13/trixie):

```bash
curl -fsSL https://get.docker.com | sh
```

Проверка:

```bash
docker --version
docker compose version
```

Ожидаются две строки с версиями (Docker Engine и Docker Compose v2+). Если `docker compose version` выдаёт «not a docker command» — плагин compose не установился; переустановите Docker скриптом выше.

---

## 3. Оптимизация системы

### 3.1 Сетевые параметры ядра

```bash
cat >> /etc/sysctl.conf << 'EOF'

# VPN Optimization
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 65535
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_tw_reuse = 1
net.netfilter.nf_conntrack_max = 262144
EOF

sysctl -p
```

> ⚠️ **Если `sysctl -p` ругается на `nf_conntrack_max`** (`cannot stat .../nf_conntrack_max: No such file or directory`) — значит модуль `nf_conntrack` ещё не загружен. Это нормально: остальные параметры применятся, а conntrack подхватится после установки Docker и/или перезагрузки. См. также заметку 3.6 о закреплении значения.

### 3.2 Лимиты файловых дескрипторов

```bash
cat >> /etc/security/limits.conf << 'EOF'
* soft nofile 300000
* hard nofile 300000
root soft nofile 300000
root hard nofile 300000
EOF
```

### 3.3 Лимиты для systemd

```bash
mkdir -p /etc/systemd/system.conf.d/
cat > /etc/systemd/system.conf.d/limits.conf << 'EOF'
[Manager]
DefaultLimitNOFILE=300000
EOF
```

### 3.4 Автозагрузка модуля conntrack

```bash
echo "nf_conntrack" >> /etc/modules-load.d/conntrack.conf
```

### 3.5 Применение изменений

```bash
systemctl daemon-reload
```

> ⚠️ **Важно:** Для полного применения лимитов файловых дескрипторов требуется перезагрузка сервера.

### 3.6 Закрепление conntrack после перезагрузки (рекомендуется)

На некоторых системах значение `nf_conntrack_max` после перезагрузки сбрасывается на дефолтное (65536), потому что модуль `nf_conntrack` загружается позже, чем применяется `/etc/sysctl.conf`. Чтобы значение держалось стабильно, вынесите его в отдельный файл `/etc/sysctl.d/`, который применяется после загрузки модулей:

```bash
cat > /etc/sysctl.d/99-conntrack.conf << 'EOF'
net.netfilter.nf_conntrack_max = 262144
EOF
```

Проверить после перезагрузки:

```bash
sysctl net.netfilter.nf_conntrack_max   # Ожидается: 262144
```

> 💡 Это не критично для работы VPN (65536 одновременных соединений тоже достаточно для большинства нод), но рекомендуется для единообразия и высоконагруженных нод.

---

## 4. DNS over TLS

DNS over TLS (DoT) шифрует DNS-запросы сервера, чтобы провайдер не видел, какие домены резолвятся. Ниже — три варианта на выбор. **Рекомендуемый — AdGuard** (шифрование + блокировка рекламы и трекеров на уровне DNS). **Резервный — Cloudflare.** Также приведён Google.

### 4.0 Установка systemd-resolved (обязательно для Debian)

> ⚠️ **Ключевое отличие Debian от Ubuntu.** В Ubuntu `systemd-resolved` работает «из коробки». На Debian 12/13 (особенно минимальные образы) он **не установлен** — без него команды `resolvectl` и `systemctl restart systemd-resolved` выдадут ошибку `Unit systemd-resolved.service not found`.

Проверьте, установлен ли resolved:

```bash
systemctl status systemd-resolved --no-pager 2>/dev/null | head -3
```

Если служба не найдена — установите (на Ubuntu этот шаг можно пропустить):

```bash
apt install -y systemd-resolved
```

> 💡 При установке пакет сам сделает `/etc/resolv.conf` симлинком на stub-резолвер (`/run/systemd/resolve/stub-resolv.conf`). Если вы уже создавали `/etc/systemd/resolved.conf` до установки пакета — при установке появится вопрос о конфликте конфигов; выберите **N** (keep currently-installed version), чтобы сохранить свою настройку.

Если после установки `/etc/resolv.conf` всё ещё обычный файл (например, его генерирует панель хостинга вроде SolusVM), переключите его на stub вручную:

```bash
ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

### 4.1 Настройка — выберите ОДИН вариант

**Вариант A — AdGuard (рекомендуется): шифрование + блокировка рекламы**

```bash
cat > /etc/systemd/resolved.conf << 'EOF'
[Resolve]
DNS=94.140.14.14#dns.adguard-dns.com 94.140.15.15#dns.adguard-dns.com
FallbackDNS=1.1.1.1#cloudflare-dns.com 1.0.0.1#cloudflare-dns.com
DNSOverTLS=yes
DNSSEC=allow-downgrade
EOF

systemctl restart systemd-resolved
```

Здесь AdGuard — основной DNS, Cloudflare указан как резервный (`FallbackDNS`) на случай недоступности AdGuard.

**Вариант B — Cloudflare (резервный/нейтральный, без блокировки рекламы)**

```bash
cat > /etc/systemd/resolved.conf << 'EOF'
[Resolve]
DNS=1.1.1.1#cloudflare-dns.com 1.0.0.1#cloudflare-dns.com
DNSOverTLS=yes
DNSSEC=allow-downgrade
EOF

systemctl restart systemd-resolved
```

**Вариант C — Google**

```bash
cat > /etc/systemd/resolved.conf << 'EOF'
[Resolve]
DNS=8.8.8.8#dns.google 8.8.4.4#dns.google
DNSOverTLS=yes
DNSSEC=allow-downgrade
EOF

systemctl restart systemd-resolved
```

> 📌 **Справка по серверам DoT:**
> | Провайдер | IP | Hostname (для TLS) | Блокировка рекламы |
> |-----------|----|--------------------|--------------------|
> | AdGuard (Default) | `94.140.14.14`, `94.140.15.15` | `dns.adguard-dns.com` | ✅ Да |
> | AdGuard (Family) | `94.140.14.15`, `94.140.15.16` | `family.adguard-dns.com` | ✅ + взрослый контент |
> | Cloudflare | `1.1.1.1`, `1.0.0.1` | `cloudflare-dns.com` | ❌ Нет |
> | Google | `8.8.8.8`, `8.8.4.4` | `dns.google` | ❌ Нет |

### 4.2 Проверка DoT

**Тест 1: Проверка шифрования запроса**

```bash
resolvectl query google.com
```

Ожидаемый результат:
```
google.com: 142.250.184.238                    -- link: eth0

-- Information acquired via protocol DNS in 9.9ms.
-- Data is authenticated: no; Data was acquired via local or encrypted transport: yes
                                                                           ^^^
                                                                      СМОТРИ СЮДА
-- Data from: network
```

🔑 **Ключевой параметр: `encrypted transport: yes`**

| Значение | Что означает |
|----------|--------------|
| `yes` | ✅ DNS запросы **шифруются через TLS** — DoT работает |
| `no` | ❌ Запросы идут **открытым текстом** — DoT не работает |

> 💡 **Пояснение:** Формулировка `local or encrypted` описывает способ передачи. `local` — через локальный stub-резолвер (127.0.0.53), `encrypted` — шифрование TLS. При работающем DoT запрос идёт через локальный stub-резолвер, который затем шифрует его через TLS к DNS-серверу.

**Тест 2: Статус службы resolved**

```bash
resolvectl status | grep -E "DNSOverTLS|Current DNS"
```

Ожидаемый результат (пример для AdGuard):
```
         Protocols: +LLMNR -mDNS +DNSOverTLS DNSSEC=allow-downgrade/supported
Current DNS Server: 94.140.14.14#dns.adguard-dns.com
```

✅ **`+DNSOverTLS`** = протокол активен
✅ **`Current DNS Server`** = используется выбранный провайдер

**Тест 3 (только для AdGuard): проверка блокировки рекламы**

```bash
resolvectl query doubleclick.net
```

Если блокировка работает, запрос к рекламному домену будет отклонён — например:
```
doubleclick.net: resolve call failed: DNSSEC validation failed: no-signature (Filtered)
```
Ключевое слово — `Filtered`. Это значит, что AdGuard отфильтровал рекламный домен. (На Cloudflare/Google этот домen зарезолвится нормально — блокировки нет.)

### 4.3 Расшифровка результатов

| Параметр | Значение | Статус |
|----------|----------|--------|
| `encrypted transport: yes` | DNS запросы шифруются | ✅ DoT работает |
| `encrypted transport: no` | Запросы идут открытым текстом | ❌ Проверить конфиг |
| `+DNSOverTLS` | Протокол включён глобально | ✅ Настроено верно |
| `-DNSOverTLS` | Протокол выключен | ❌ Проверить resolved.conf |

---

## 5. Автоматические обновления ОС

```bash
apt install -y unattended-upgrades

cat > /etc/apt/apt.conf.d/50unattended-upgrades << 'EOF'
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}";
    "${distro_id}:${distro_codename}-security";
    "${distro_id}:${distro_codename}-updates";
};
Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
Unattended-Upgrade::Remove-New-Unused-Dependencies "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-WithUsers "true";
Unattended-Upgrade::Automatic-Reboot-Time "04:00";
EOF
```

> ⚠️ **`Automatic-Reboot "true"`** означает, что сервер сам перезагрузится в 04:00, если обновление потребует (например, новое ядро). Для VPN-ноды это короткий разрыв сервиса ночью. Если не хотите внезапных перезагрузок — поставьте `"false"` и перезагружайтесь вручную после обновлений.

> 💡 Пакет создаёт свой `50unattended-upgrades` при установке. Команда `cat >` выше выполняется **после** `apt install` и перезаписывает файл вашей версией — проверьте результат через `cat /etc/apt/apt.conf.d/50unattended-upgrades`.

---

## 6. Firewall (UFW)

### 6.1 Установка и базовые правила

```bash
apt install -y ufw

ufw allow 22/tcp comment 'SSH'
ufw allow 443/tcp comment 'VLESS Reality'
```

> ⚠️ **Не включайте UFW, пока не добавлено правило для SSH (22/tcp)** — иначе рискуете потерять доступ к серверу. В блоке выше оно добавлено первым, поэтому всё в порядке.

### 6.2 NODE_PORT — доступ для панели

NODE_PORT (по умолчанию `2222`, в этой инструкции используем `8443`) — это внутренний API для связи панели с нодой. **Рекомендуемый способ — открыть его только для IP панели Remnawave:**

```bash
ufw allow from <IP_REMNAWAVE_PANEL> to any port 8443 proto tcp comment 'Remnanode API'
```

> ⚠️ **Не открывайте NODE_PORT для всех (`ufw allow 8443/tcp`)!** Это внутренний API; доступ должен быть только с IP вашей панели.

**Альтернатива (если IP панели динамический или сервисов несколько):** некоторые администраторы открывают для IP панели сразу все порты:

```bash
ufw allow from <IP_REMNAWAVE_PANEL> comment 'Remnawave Panel - all ports'
```

Это шире по доступу и менее строго, но иногда удобнее (например, когда на ноде несколько внутренних сервисов, общающихся с панелью). Выбор за вами; для большинства случаев достаточно варианта с конкретным портом 8443.

### 6.3 Включение UFW

```bash
ufw --force enable
ufw status verbose
```

### 6.4 Если несколько IP на сервере

Для привязки порта к конкретному IP:

```bash
ufw allow in to <YOUR_VPN_IP> port 443 proto tcp comment 'VLESS Reality'
```

### 6.5 Правила комментирования

Все правила UFW должны иметь комментарии:

```bash
# Плохо — непонятно зачем порт
ufw allow 8443/tcp

# Хорошо — сразу понятно назначение
ufw allow 8443/tcp comment 'Remnanode API'
```

---

## 7. Fail2ban

```bash
apt install -y fail2ban
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
systemctl enable fail2ban
systemctl restart fail2ban
systemctl status fail2ban --no-pager | head -5
```

Проверьте, что jail для SSH реально активен:

```bash
fail2ban-client status          # должен быть jail "sshd"
fail2ban-client status sshd     # счётчики + строка Journal matches
```

> 💡 **Debian 12/13.** Логи SSH здесь обычно только в systemd-journal (нет классического `/var/log/auth.log`). Свежие версии fail2ban подхватывают journald автоматически (`Journal matches: _SYSTEMD_UNIT=ssh.service`). Если jail `sshd` **не** появился в списке — добавьте в `/etc/fail2ban/jail.local` в секцию `[sshd]` строку `backend = systemd` и перезапустите fail2ban.

---

## 8. TLS сканирование (выбор домена для маскировки)

### 8.1 Зачем это нужно

Reality маскирует VPN-трафик под обычный HTTPS-трафик к легитимному сайту. Для этого нужно выбрать **домен-донор** — сайт, под который будет маскироваться ваш VPN.

**Принцип работы:**
1. Клиент подключается к вашему VPN-серверу
2. DPI видит TLS-соединение с SNI (например `images.apple.com`)
3. VPN-сервер отвечает сертификатом донора — соединение выглядит легитимно
4. DPI пропускает трафик, думая что это обычный HTTPS

**Важно:** Лучше всего, если домен-донор находится на том же хостинге/IP-диапазоне, что и ваш сервер. Поэтому сканируем **соседние IP** — ищем сайты на той же площадке.

### 8.2 Скачивание сканера

> ⚠️ **Имя файла релиза менялось.** В старых версиях бинарь назывался `RealiTLScanner-linux-64`, в актуальных — `RealiTLScanner-linux-amd64`. Ссылка `latest/download/...` может вести на 404, если имя ассета не совпадает. Поэтому проверяйте актуальное имя файла на странице релизов: <https://github.com/XTLS/RealiTLScanner/releases>

Скачивание актуальной версии (на момент обновления — `v0.2.3`, архитектура linux/amd64):

```bash
wget https://github.com/XTLS/RealiTLScanner/releases/download/v0.2.3/RealiTLScanner-linux-amd64 -O /tmp/RealiTLScanner
chmod +x /tmp/RealiTLScanner

# Проверка, что скачался реальный бинарь, а не пустышка/HTML с ошибкой
ls -la /tmp/RealiTLScanner       # размер должен быть несколько МБ, не 0
/tmp/RealiTLScanner --help | head -20
```

> 💡 Если `wget` скачал файл размером `0` — значит ссылка отдала 404/редирект. Откройте страницу релизов в браузере, найдите точное имя файла для linux/amd64 и подставьте его в URL. Не используйте `wget -q` при отладке — без `-q` видна вся цепочка редиректов и HTTP-коды.

### 8.3 Сканирование соседних IP

```bash
/tmp/RealiTLScanner -addr <YOUR_IP> -thread 30 | head -50
```

> 💡 Предупреждение `Cannot open Country.mmdb` в начале вывода — это лишь отсутствие GeoIP-базы (поле `geo=N/A`). На поиск доменов не влияет, можно игнорировать.

**Пример вывода (актуальный формат v0.2.x):**
```
... ip=78.17.85.171 tls="TLS 1.3" alpn=h2 cert-domain=images.apple.com  cert-issuer="Apple Inc."
... ip=78.17.85.244 tls="TLS 1.3" alpn=h2 cert-domain=kaspersky.com     cert-issuer="DigiCert Inc"
... ip=78.17.87.15  tls="TLS 1.3" alpn=h2 cert-domain=www.cloudflare.com cert-issuer="Google Trust Services"  ← НЕ БРАТЬ
... ip=78.17.86.10  tls="TLS 1.3" alpn=h2 cert-domain=f.ivcapital.ru    cert-issuer="Let's Encrypt"           ← НЕ БРАТЬ
```

### 8.4 На что обращать внимание

| Параметр | Хорошо ✅ | Плохо ❌ |
|----------|-----------|----------|
| **TLS версия** | TLS 1.3 | TLS 1.2 и ниже |
| **ALPN** | h2 (HTTP/2) | http/1.1 |
| **Издатель сертификата** | GlobalSign, DigiCert, Apple, Microsoft, Amazon | Let's Encrypt, Cloudflare, Google Trust Services |
| **Тип сайта** | Крупные компании, CDN | Личные сайты, VPN-сервисы |

### 8.5 Критерии выбора домена

**✅ Подходят:**
- TLS 1.3 + ALPN h2
- Крупные компании (Apple, Microsoft, Kaspersky, Nvidia, Yahoo)
- Коммерческие сертификаты (GlobalSign, DigiCert, Apple, Microsoft)
- Без редиректа (HTTP 200)

**❌ НЕ подходят:**
- **Cloudflare, Discord** — популярны для VPN, легко детектятся
- **Let's Encrypt** — часто используется VPN-серверами, вызывает подозрения
- Домены с редиректом (301, 302)
- Малоизвестные/личные сайты, домены с «vpn», «proxy» в имени

### 8.6 Проверка домена на редирект

После выбора домена проверьте, что он отвечает без редиректа (нужен код 200):

```bash
curl -sI https://images.apple.com | head -3
```

**Хороший ответ:**
```
HTTP/2 200
```
или
```
HTTP/1.1 200 OK
```

**Плохой ответ (редирект — НЕ использовать):**
```
HTTP/2 301
HTTP/2 302
```

> 💡 Сервер может ответить на `curl -I` (HEAD-запрос) по HTTP/1.1, даже если в TLS-сканере ALPN был `h2`. Это не проблема — для Reality важен ALPN `h2` в TLS-рукопожатии (его показывает сканер), а код ответа `200` подтверждает отсутствие редиректа.

### 8.7 Рекомендуемые домены

| Домен | Издатель | Статус |
|-------|----------|--------|
| images.apple.com | Apple Inc. | ✅ |
| www.python.org | GlobalSign | ✅ |
| www.nvidia.com | DigiCert | ✅ |
| www.microsoft.com | Microsoft | ⚠️ Проверить |

> ⚠️ **Важно:** Всегда проверяйте актуальность домена перед использованием (TLS 1.3, ALPN h2, код 200). Сертификаты и настройки сайтов меняются. По возможности выбирайте домен из числа **соседей по вашему IP** (см. 8.3) — это надёжнее.

### 8.8 Что записать для дальнейших шагов

После выбора домена сохраните:
- **Домен** (например: `images.apple.com`) — понадобится для `target` и `serverNames` в конфиге inbound (раздел 11.4)
- Этот же домен пойдёт как **SNI** в настройках хоста (раздел 11.6)

---

## 9. Установка Remnanode

### 9.1 Создание директории

```bash
mkdir -p /opt/remnanode && cd /opt/remnanode
```

### 9.2 Создание ноды в панели Remnawave

1. Откройте панель Remnawave
2. Перейдите в **Ноды (Nodes)** → **Управление (Management)**
3. Нажмите кнопку **+** для добавления новой ноды
4. Заполните форму:
   - **Name:** название ноды (например: `FI-Helsinki-01`) — придерживайтесь единой схемы именования с другими нодами
   - **Address:** IP адрес сервера
   - **Node Port:** порт для внутреннего API (по умолчанию `2222`, в этой инструкции `8443`)
5. Нажмите **Copy docker-compose.yml** для копирования конфигурации

> 💡 Если у вас уже есть рабочие ноды — создавайте новую по их образцу (имя, профиль, сквад), чтобы парк оставался единообразным.

### 9.3 Создание docker-compose.yml

```bash
nano /opt/remnanode/docker-compose.yml
# или: mcedit /opt/remnanode/docker-compose.yml
```

Вставьте конфигурацию **из панели** (она содержит ваш уникальный `SECRET_KEY`). Пример структуры:

```yaml
services:
  remnanode:
    container_name: remnanode
    hostname: remnanode
    image: remnawave/node:latest
    network_mode: host
    restart: always
    ulimits:
      nofile:
        soft: 1048576
        hard: 1048576
    environment:
      - NODE_PORT=8443
      - SECRET_KEY="<YOUR_SECRET_KEY>"
    volumes:
      - /var/log/remnanode:/var/log/remnanode
```

> ⚠️ **`SECRET_KEY` уникален для вашей ноды** и связывает её с панелью. Берите его только из панели — не выдумывайте и не копируйте из примера.
> ⚠️ Блок `volumes` обязателен для работы логирования.
> Проверьте, что `NODE_PORT` совпадает с тем, что вы указали в панели.

### 9.4 Создание директории для логов

```bash
mkdir -p /var/log/remnanode
```

### 9.5 Запуск контейнера

```bash
cd /opt/remnanode
docker compose up -d && docker compose logs -f -t
```

Дождитесь в логах строки `Xray started`, затем нажмите `Ctrl+C` для выхода из просмотра (контейнер продолжит работать).

> 💡 На первом запуске Xray стартует только после того, как панель отправит ноде конфигурацию (привязанный профиль/inbound). Если в логах нода запущена, но `Xray started` ещё нет — завершите привязку профиля в панели (раздел 11.1).

### 9.6 Автообновление ноды (суббота 11:00 по времени сервера)

```bash
cat > /etc/cron.d/remnawave-update << 'EOF'
0 11 * * 6 root cd /opt/remnanode && echo "=== $(date) ===" >> /var/log/remnawave-update.log && docker compose pull >> /var/log/remnawave-update.log 2>&1 && docker compose down >> /var/log/remnawave-update.log 2>&1 && docker compose up -d >> /var/log/remnawave-update.log 2>&1 && docker image prune -f >> /var/log/remnawave-update.log 2>&1
EOF
chmod 644 /etc/cron.d/remnawave-update
```

**Что записывается в лог** (`cat /var/log/remnawave-update.log`):
- Дата и время запуска
- Результат `docker compose pull` — скачался ли новый образ
- Результат перезапуска контейнера
- Результат очистки старых образов (`docker image prune`)

> ⚠️ **Риск рассинхрона версий.** Если панель Remnawave обновится и начнёт требовать новую версию ноды между запусками крона — нода перестанет получать конфигурацию, Xray не запустится, VPN перестанет работать. Контейнер при этом будет выглядеть рабочим (`docker ps` покажет UP), но в логах не будет `Xray started`. Диагностика — раздел 13.2.

---

## 10. Настройка логирования

### 10.1 Установка logrotate

```bash
apt install -y logrotate
```

### 10.2 Конфигурация ротации логов

```bash
cat > /etc/logrotate.d/remnanode << 'EOF'
/var/log/remnanode/*.log {
    size 50M
    rotate 5
    compress
    missingok
    notifempty
    copytruncate
}
EOF
```

### 10.3 Проверка конфигурации

```bash
logrotate -vf /etc/logrotate.d/remnanode
```

> ⚠️ **Важно:** Обязательно настройте ротацию логов, иначе они заполнят диск.

---

## 11. Настройка в панели Remnawave

### 11.1 Завершение создания ноды

1. В карточке создания ноды нажмите **Next**
2. Выберите нужный **Config Profile** (для единообразия — тот же, что у других ваших нод)
3. Нажмите **Create**

### 11.2 Генерация ключей для Reality

Ключи можно сгенерировать в панели автоматически или вручную на сервере:

```bash
docker exec remnanode xray x25519
```

**Формат вывода зависит от версии Xray:**

Новый формат (Xray 26.x и новее):
```
PrivateKey: KOVlHfY5rJ-bofXdDP-By_OBo35cKf4vPRnQKUaQ5ns
Password (PublicKey): eN7EyPjQQeiZ4WUGsdOYqjEPLD89BNW1ZH3o6YFe0X4
Hash32: ACf2m8lmGTF4_hv3N9baoyNSbI7QYCZTHmd8XgNkrQM
```
- `PrivateKey` → в поле `privateKey` инбаунда
- `Password (PublicKey)` → это **PublicKey**, идёт в настройки хоста
- `Hash32` → не используется

Старый формат (на всякий случай):
```
Private key: ...
Public key:  ...
```

> ⚠️ **Каждая нода должна иметь свою пару ключей.** Не копируйте `privateKey`/`shortIds` с другой ноды — это небезопасно и приводит к конфликтам.

### 11.3 Генерация ShortIds

```bash
openssl rand -hex 8
openssl rand -hex 8
openssl rand -hex 8
openssl rand -hex 8
```

### 11.4 Конфиг Inbound

> ⚠️ **Актуальный синтаксис Xray.** В новых версиях ядра (Xray 24.x+/26.x) изменились два поля:
> - `dest` → **`target`**
> - `"network": "tcp"` → **`"network": "raw"`** (`raw` — новый синоним TCP)
>
> Ниже — актуальный вариант. Если вы работаете со старой версией ядра, где новые имена не поддерживаются, используйте `dest`/`tcp` (см. заметку после блока).

```json
{
  "log": {
    "error": "/var/log/remnanode/error.log",
    "access": "/var/log/remnanode/access.log",
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "tag": "VLESS_TCP_REALITY_<НАЗВАНИЕ>",
      "port": 443,
      "listen": "0.0.0.0",
      "protocol": "vless",
      "settings": {
        "clients": [],
        "decryption": "none"
      },
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls", "quic"]
      },
      "streamSettings": {
        "network": "raw",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "xver": 0,
          "target": "<ДОМЕН>:443",
          "serverNames": ["<ДОМЕН>"],
          "privateKey": "<PRIVATE_KEY>",
          "shortIds": ["<SHORT_ID_1>", "<SHORT_ID_2>", "<SHORT_ID_3>", "<SHORT_ID_4>"]
        }
      }
    }
  ],
  "outbounds": [
    {
      "tag": "DIRECT",
      "protocol": "freedom"
    },
    {
      "tag": "BLOCK",
      "protocol": "blackhole"
    }
  ],
  "routing": {
    "rules": [
      {
        "ip": ["geoip:private"],
        "type": "field",
        "outboundTag": "BLOCK"
      },
      {
        "domain": ["geosite:private"],
        "type": "field",
        "outboundTag": "BLOCK"
      },
      {
        "protocol": ["bittorrent"],
        "type": "field",
        "outboundTag": "BLOCK"
      }
    ]
  }
}
```

> 💡 **Совместимость со старым синтаксисом** (если ядро не понимает `target`/`raw`):
> ```json
> "network": "tcp",
> ...
> "dest": "<ДОМЕН>:443",
> ```
> Проверить версию ядра: `docker exec remnanode xray version`.

> 📌 **В Remnawave конфиг inbound задаётся в панели** (раздел Профили / Config Profile), а не редактированием файла на сервере. Нода получает конфиг от панели по API. Прямое редактирование `config.json` в контейнере бессмысленно — панель перезапишет его при синхронизации.

### 11.5 Если несколько IP на сервере

Измените `listen` на конкретный IP:

```json
"listen": "<YOUR_VPN_IP>",
```

### 11.6 Настройки хоста

| Параметр | Значение |
|----------|----------|
| Название | `🇫🇮 FI-Helsinki-01` (флаг-эмодзи в начале имени — простой способ задать флаг) |
| Адрес | `<IP_ИЛИ_ДОМЕН_СЕРВЕРА>` (например `fi1.example.com`) |
| Порт | `443` |
| SNI | `<ДОМЕН-ДОНОР>` (например `images.apple.com`) |
| PublicKey | `<PUBLIC_KEY>` |
| ShortId | `<ОДИН_ИЗ_SHORT_IDS>` |
| Fingerprint | `chrome` |
| ALPN | `h2` |

> ⚠️ **Не путайте два домена:**
> - **Адрес хоста** (`fi1.example.com`) — куда реально подключается клиент (ваш сервер). Если используете домен — у него должна быть A-запись на IP сервера (проверка: `dig +short fi1.example.com` должна вернуть IP сервера).
> - **SNI** (`images.apple.com`) — домен-маскировка для Reality. Это разные домены, так и должно быть.

> 💡 **Флаг страны.** В большинстве сборок панели флаг проставляется флаг-эмодзи прямо в названии (`🇫🇮 FI-Helsinki-01`) либо через поле кода страны (ISO: `FI` = Финляндия, `DE` = Германия и т.д.). Убедитесь, что выбрали правильный код — например `FI`, а не `IN` (Индия).

---

## 12. Проверка работоспособности

### 12.1 После перезагрузки

```bash
# Лимиты
ulimit -n                                    # Ожидается: 300000
cat /proc/sys/net/netfilter/nf_conntrack_max # Ожидается: 262144 (см. 3.6)

# Docker
docker ps

# Порты
ss -tlpn | grep -E ':443|:8443'

# DNS over TLS
resolvectl query google.com                  # Ожидается: "encrypted transport: yes"
resolvectl status | grep -E "DNSOverTLS|Current DNS"
# Ожидается: "+DNSOverTLS" и ваш выбранный DNS-сервер
```

### 12.2 Проверка Xray

> ⚠️ **Важное изменение.** В новых версиях ноды Remnawave Xray встроен в основной процесс **`rw-core`** — отдельного процесса `xray` в `docker exec remnanode ps aux` больше нет. Старая проверка `ps aux | grep xray` даст **ложный** «не запущен». Ориентируйтесь на listener порта 443 и логи.

```bash
# 1. Главный признак — Xray слушает порт 443
ss -tlnp | grep ':443'
# Ожидается: LISTEN на *:443 (процесс rw-core или xray, в зависимости от версии)

# 2. Логи — строка "Xray started" и извлечение пользователей
docker logs remnanode --tail 30
# Ожидается: "Xray started" и "VLESS_TCP_REALITY_... has N users"

# 3. Контейнер жив
docker ps --filter name=remnanode
# Ожидается: STATUS = Up

# 4. Панель подключена к ноде
ss -tn | grep 8443 | grep ESTAB
# Ожидается: ESTABLISHED соединение от IP панели
```

**Признаки, что всё работает:**
- `ss -tlnp | grep ':443'` показывает listener на 443
- В `docker logs` есть `Xray started` и `Starting user extraction from inbounds`
- `ss -tn | grep 8443` показывает ESTABLISHED от IP панели

**Признаки проблемы:**
- `ss -tlnp | grep ':443'` — **пусто** (Xray не слушает)
- В `docker logs` **нет** `Xray started`
- `ss -tn | grep 8443` — **пусто** (панель не подключена)

> Если видите признаки проблемы — переходите к [разделу 13](#13-диагностика-проблем).

### 12.3 Проверка логов

```bash
tail -f /var/log/remnanode/error.log
tail -f /var/log/remnanode/access.log
```

### 12.4 Тест подключения

Подключитесь через VPN-клиент и проверьте IP/геолокацию:
- https://2ip.ru
- https://whoer.net

> 💡 **Расхождение геолокации.** Разные сервисы (2ip, ip-api, ipinfo) могут показывать для свежего IP **разные страны** — это особенность их баз, которые обновляются с задержкой. Сверяйте по нескольким источникам и/или через RIPE. Если разные базы дают разные страны, а сам IP по RIPE/ipinfo принадлежит нужной стране — с сервером всё в порядке, базы со временем подтянутся. Главное при тесте — что IP клиента совпадает с IP вашей ноды (значит трафик идёт через неё).

---

## 13. Диагностика проблем

### 13.1 VPN подключается, но ничего не открывается

**Шаг 1: Проверить, слушает ли Xray порт 443**

```bash
ss -tlnp | grep ':443'
```

Если **пусто** — Xray не получил конфигурацию от панели или не стартовал.

> ⚠️ Не используйте `docker exec remnanode ps aux | grep xray` как индикатор — в новых версиях Xray встроен в `rw-core`, и эта проверка вводит в заблуждение.

**Шаг 2: Проверить связь панели с нодой**

```bash
ss -tn | grep 8443
```

Если **пусто** — панель не подключается. Причины:
- NODE_PORT (8443) заблокирован в UFW
- Изменился IP панели (если панель на динамическом IP)
- Панель не работает

**Шаг 3: Проверить логи на ошибки**

```bash
docker logs remnanode 2>&1 | grep -iE "error|warn|fail|inbound|Xray started" | tail -20
```

**Шаг 4: О пути к конфигу Xray**

> ⚠️ В новых версиях ноды файла `/usr/local/etc/xray/config.json` **больше нет** — конфиг передаётся в процесс иначе. Команда `docker exec remnanode cat /usr/local/etc/xray/config.json` выдаст `No such file or directory` — это **не** признак проблемы. Ориентируйтесь на listener 443 и логи (шаги 1–3), а не на наличие файла.

### 13.2 Нода отстала по версии от панели

**Симптомы:**
- Контейнер работает (`docker ps` — UP)
- В логах **нет** `Xray started` и `Starting user extraction from inbounds`
- Панель показывает ноду Offline или ошибку версии
- `ss -tn | grep 8443` — пусто

**Причина:** Панель обновилась и требует новую версию ноды.

**Решение:**

```bash
cd /opt/remnanode
docker compose pull
docker compose down
docker compose up -d
sleep 15
docker logs remnanode 2>&1 | grep -iE "inbound|Xray started|version"
```

**Проверить версию ноды:**

```bash
docker logs remnanode 2>&1 | grep "Remnawave Node"
```

### 13.3 NODE_PORT закрыт в UFW

**Симптом:** Нода работает, но панель не подключается (`ss -tn | grep 8443` пусто).

```bash
ufw status | grep 8443
```

Если правила нет — добавьте:

```bash
ufw allow from <IP_REMNAWAVE_PANEL> to any port 8443 proto tcp comment 'Remnanode API'
```

> ⚠️ **Не удаляйте это правило!** Без него панель потеряет связь с нодой.

### 13.4 Быстрая диагностика одной командой

```bash
echo "=== Версия ноды ==="
docker logs remnanode 2>&1 | grep "Remnawave Node" | tail -1

echo ""
echo "=== Xray слушает 443? ==="
ss -tlnp | grep ':443' && echo "✅ Xray слушает 443" || echo "❌ Xray НЕ слушает 443"

echo ""
echo "=== Xray started в логах? ==="
docker logs remnanode 2>&1 | grep -q "Xray started" && echo "✅ Xray стартовал" || echo "❌ нет Xray started"

echo ""
echo "=== Панель подключена? ==="
ss -tn | grep 8443 | grep ESTAB && echo "✅ Панель подключена" || echo "❌ Панель НЕ подключена"

echo ""
echo "=== Последние ошибки ==="
docker logs remnanode 2>&1 | grep -iE "error|fail" | tail -5

echo ""
echo "=== UFW — порт 8443 ==="
ufw status | grep 8443 || echo "❌ Порт 8443 не открыт в UFW!"
```

> 💡 Эта версия проверки Xray исправлена: вместо `ps aux | grep xray` (даёт ложный результат) используется listener порта 443 и наличие `Xray started` в логах.

### 13.5 Docker не пускает трафик

**Симптом:** Xray слушает порт, но клиенты не подключаются.

**Проверка FORWARD chain:**

```bash
iptables -L FORWARD -n -v --line-numbers | head -15
```

**Проверка изнутри контейнера:**

```bash
docker exec remnanode ping -c 2 1.1.1.1
docker exec remnanode nslookup google.com
```

Если пинг проходит и DNS резолвится — сеть контейнера в порядке.

---

## 14. Проверка и очистка Firewall

### 14.1 Сравнение портов

```bash
ss -tlpn       # что реально слушает сервер
ufw status     # что открыто в UFW
```

**Принцип:** Если порт открыт в UFW, но на нём ничего не слушает — правило лишнее.

### 14.2 Удаление лишних портов

```bash
ufw delete allow <PORT>
```

### 14.3 Добавление комментариев к правилам

```bash
ufw delete allow 443
ufw allow 443 comment 'Xray VPN'
```

### 14.4 Эталонный вид UFW (вариант с конкретным портом 8443)

```
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       Anywhere                   # SSH
443/tcp                    ALLOW       Anywhere                   # VLESS Reality
8443/tcp                   ALLOW       123.45.67.89               # Remnanode API
22/tcp (v6)                ALLOW       Anywhere (v6)              # SSH
443/tcp (v6)               ALLOW       Anywhere (v6)              # VLESS Reality
```

> 💡 Порт 8443 (Remnanode API) открыт только для IP панели, а не для всех.

---

## 🔧 Быстрая команда «всё в одном»

> ⚠️ Перед выполнением замените `<IP_REMNAWAVE_PANEL>` на IP вашей панели.
> Блок ставит DoT через **AdGuard** (с резервным Cloudflare). На Debian дополнительно ставится `systemd-resolved`.

```bash
apt update && apt upgrade -y && \
curl -fsSL https://get.docker.com | sh && \
apt install -y ufw fail2ban mc htop btop iftop unattended-upgrades curl wget logrotate systemd-resolved && \
timedatectl set-timezone Europe/Moscow && \
cat >> /etc/sysctl.conf << 'EOF'

# VPN Optimization
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 65535
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_tw_reuse = 1
net.netfilter.nf_conntrack_max = 262144
EOF
sysctl -p; \
cat > /etc/sysctl.d/99-conntrack.conf << 'EOF'
net.netfilter.nf_conntrack_max = 262144
EOF
cat >> /etc/security/limits.conf << 'EOF'
* soft nofile 300000
* hard nofile 300000
root soft nofile 300000
root hard nofile 300000
EOF
mkdir -p /etc/systemd/system.conf.d/ && \
cat > /etc/systemd/system.conf.d/limits.conf << 'EOF'
[Manager]
DefaultLimitNOFILE=300000
EOF
echo "nf_conntrack" >> /etc/modules-load.d/conntrack.conf && \
cat > /etc/systemd/resolved.conf << 'EOF'
[Resolve]
DNS=94.140.14.14#dns.adguard-dns.com 94.140.15.15#dns.adguard-dns.com
FallbackDNS=1.1.1.1#cloudflare-dns.com 1.0.0.1#cloudflare-dns.com
DNSOverTLS=yes
DNSSEC=allow-downgrade
EOF
systemctl restart systemd-resolved && \
cat > /etc/apt/apt.conf.d/50unattended-upgrades << 'EOF'
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}";
    "${distro_id}:${distro_codename}-security";
    "${distro_id}:${distro_codename}-updates";
};
Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
Unattended-Upgrade::Remove-New-Unused-Dependencies "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-WithUsers "true";
Unattended-Upgrade::Automatic-Reboot-Time "04:00";
EOF
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local && \
systemctl enable fail2ban && systemctl restart fail2ban && \
ufw allow 22/tcp comment 'SSH' && \
ufw allow 443/tcp comment 'VLESS Reality' && \
ufw allow from <IP_REMNAWAVE_PANEL> to any port 8443 proto tcp comment 'Remnanode API' && \
ufw --force enable && \
mkdir -p /opt/remnanode /var/log/remnanode && \
cat > /etc/logrotate.d/remnanode << 'EOF'
/var/log/remnanode/*.log {
    size 50M
    rotate 5
    compress
    missingok
    notifempty
    copytruncate
}
EOF
cat > /etc/cron.d/remnawave-update << 'EOF'
0 11 * * 6 root cd /opt/remnanode && echo "=== $(date) ===" >> /var/log/remnawave-update.log && docker compose pull >> /var/log/remnawave-update.log 2>&1 && docker compose down >> /var/log/remnawave-update.log 2>&1 && docker compose up -d >> /var/log/remnawave-update.log 2>&1 && docker image prune -f >> /var/log/remnawave-update.log 2>&1
EOF
chmod 644 /etc/cron.d/remnawave-update && \
systemctl daemon-reload && \
echo "=== Готово! ===" && \
echo "1. Перезагрузите сервер: reboot" && \
echo "2. После перезагрузки создайте ноду в панели Remnawave" && \
echo "3. Скопируйте docker-compose.yml из панели в /opt/remnanode/" && \
echo "4. Запустите: cd /opt/remnanode && docker compose up -d" && \
echo "5. Проверьте DoT: resolvectl query google.com (ожидается encrypted transport: yes)" && \
echo "6. Проверьте UFW: ufw status (порт 8443 — только для IP панели)"
```

---

## 📝 История изменений (этой редакции)

- **DNS (раздел 4):** добавлен AdGuard как основной вариант (шифрование + блокировка рекламы), Cloudflare — резервный, Google — альтернатива. Добавлен обязательный для Debian шаг установки `systemd-resolved` (раздел 4.0).
- **TLS-сканер (раздел 8.2):** исправлено имя файла релиза (`RealiTLScanner-linux-amd64`), добавлена проверка на «пустой» бинарь и совет сверять имя ассета на странице релизов.
- **Конфиг Xray (раздел 11.4):** `dest` → `target`, `network: tcp` → `network: raw`; добавлена заметка о совместимости со старым синтаксисом.
- **Проверка Xray (разделы 12.2, 13.1, 13.4):** убрана ложная проверка `ps aux | grep xray` (Xray встроен в `rw-core`); проверка теперь по listener порта 443 и логам.
- **Путь к config.json (раздел 13.1):** отмечено, что файла `/usr/local/etc/xray/config.json` в новых версиях нет.
- **conntrack (раздел 3.6):** добавлено закрепление через `/etc/sysctl.d/`.
- **Геолокация (раздел 12.4):** добавлена заметка о расхождении гео-баз для свежих IP.
- **ОС:** инструкция адаптирована под Ubuntu и Debian 12/13.

---

## 📚 Полезные ссылки

- [Официальная документация Remnawave Node](https://docs.rw/docs/install/remnawave-node)
- [Remnawave Telegram](https://t.me/remnawave)
- [Xray-core GitHub](https://github.com/XTLS/Xray-core)
- [RealiTLScanner — релизы](https://github.com/XTLS/RealiTLScanner/releases)
- [AdGuard DNS](https://adguard-dns.io/)

---

[← Назад к README](README.md) | [Zabbix →](ZABBIX.md) | [Prometheus →](PROMETHEUS.md)
