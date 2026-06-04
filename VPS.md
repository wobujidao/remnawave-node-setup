# 🖥️ Полная инструкция по настройке VPN-ноды (Remnawave)

> **Поддерживаемые ОС:** Ubuntu 22.04/24.04, Debian 12 (bookworm), Debian 13 (trixie).
> Команды универсальны для всех перечисленных систем; отличия вынесены в отдельные заметки.
>
> **Что обновлено в этой версии:** **ДОБАВЛЕН (не заменён)** альтернативный метод маскировки — **self-steal через Caddy** (своя страница-заглушка на собственном домене ноды с реальным Let's Encrypt-сертификатом, раздел **8.8**). Метод заёма соседнего домена через TLS-сканер (разделы 8.1–8.7) **остаётся без изменений** и актуален для тех, у кого нет своего домена. Также добавлены: фикс DHCP-навязанного DNS (раздел 4.4), порт 80 под ACME (раздел 6), автообновление Caddy (раздел 5), нюанс отпечатка `firefox` именно для self-steal (раздел 11.6) и проверка маскировки (раздел 12.5).
>
> **Из прошлых версий:** актуальная ссылка на TLS-сканер, установка `systemd-resolved` на минимальных образах Debian, актуальные поля Xray (`target`/`raw` вместо `dest`/`tcp`), корректная проверка Xray для новых версий ноды (встроен в процесс `rw-core`), варианты DNS (AdGuard/Cloudflare/Google) и закрепление conntrack после перезагрузки.

---

## 📋 Содержание

1. [Подготовка сервера](#1-подготовка-сервера)
2. [Установка Docker](#2-установка-docker)
3. [Оптимизация системы](#3-оптимизация-системы)
4. [DNS over TLS](#4-dns-over-tls)
5. [Автоматические обновления ОС](#5-автоматические-обновления-ос)
6. [Firewall (UFW)](#6-firewall-ufw)
7. [Fail2ban](#7-fail2ban)
8. [TLS сканирование (выбор домена для маскировки) + self-steal через Caddy](#8-tls-сканирование-выбор-домена-для-маскировки)
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

> 💡 **Docker может быть уже установлен.** Некоторые провайдеры (например, HOSTKEY) отдают образ с уже предустановленным Docker. Если `docker --version` вернул версию — раздел 2 (установка) можно пропустить, достаточно проверить `docker compose version`.

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

> 💡 Сначала проверьте `docker --version` — на ряде хостингов Docker уже установлен (см. заметку 1.1). Если установлен — переходите к разделу 3.

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
Ключевое слово — `Filtered`. Это значит, что AdGuard отфильтровал рекламный домен. (На Cloudflare/Google этот домен зарезолвится нормально — блокировки нет.)

> 💡 Формулировка `DNSSEC validation failed: no-signature (Filtered)` — это **не ошибка настройки**, а ожидаемое поведение: AdGuard возвращает на заблокированный домен синтетический ответ без DNSSEC-подписи, и при `DNSSEC=allow-downgrade` resolved помечает его так. Итог для рекламного домена один — он недоступен. Легитимные подписанные домены при этом валидируются нормально.

### 4.3 Расшифровка результатов

| Параметр | Значение | Статус |
|----------|----------|--------|
| `encrypted transport: yes` | DNS запросы шифруются | ✅ DoT работает |
| `encrypted transport: no` | Запросы идут открытым текстом | ❌ Проверить конфиг |
| `+DNSOverTLS` | Протокол включён глобально | ✅ Настроено верно |
| `-DNSOverTLS` | Протокол выключен | ❌ Проверить resolved.conf |

### 4.4 Если провайдер навязывает свои DNS через DHCP (важно для HOSTKEY и подобных)

На ряде хостингов (например, **HOSTKEY**) сеть настроена через DHCP, и DHCP-сервер присылает свои DNS (часто `8.8.8.8`), которые systemd-resolved вешает **на сетевой линк** (`ens1`/`eth0`). Per-link DNS имеют приоритет над глобальным DoT — в итоге часть запросов идёт в обход AdGuard открытым текстом, хотя глобально `+DNSOverTLS` включён.

**Проверка** (смотрите блок именно по линку, а не Global):

```bash
resolvectl status ens1     # подставьте своё имя интерфейса (ip a)
```

Если в блоке линка в `Current Scopes`/`DNS Servers` висит `8.8.8.8` (или иной адрес от провайдера) — DHCP перебивает ваш DoT.

**Лечение — запретить dhclient подсовывать DNS через exit-hook:**

```bash
mkdir -p /etc/dhcp/dhclient-exit-hooks.d
cat > /etc/dhcp/dhclient-exit-hooks.d/00-no-dhcp-dns << 'EOF'
unset new_domain_name_servers old_domain_name_servers
unset new_dhcp6_name_servers old_dhcp6_name_servers
unset new_domain_search old_domain_search
unset new_dhcp6_domain_search old_dhcp6_domain_search
EOF
chmod +x /etc/dhcp/dhclient-exit-hooks.d/00-no-dhcp-dns

# сбросить уже навязанные с линка DNS и перезапустить resolved
resolvectl revert ens1          # имя своего интерфейса
systemctl restart systemd-resolved
```

После этого на линке не должно остаться чужих DNS (в `resolvectl status ens1` — `Current Scopes: LLMNR ...` без блока `DNS Servers`), а `resolvectl query doubleclick.net` снова фильтруется AdGuard-ом. **Хук переживает перезагрузку и повторную выдачу аренды DHCP** — это и есть его смысл (после ребута проверьте, что `8.8.8.8` не вернулся).

> 💡 Применимо к системам на `ifupdown` + `isc-dhcp-client` (классический `dhclient`; проверить — `which dhclient`). Если сеть на `systemd-networkd` — отключайте навязанный DNS через `[Network] UseDNS=no` в `.network`-файле; на `NetworkManager` — `dns=none` / соответствующая настройка соединения.

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

> 💡 **Если используете self-steal через Caddy (раздел 8.8)** и хотите, чтобы Caddy тоже обновлялся автоматически из cloudsmith — добавьте его origin отдельным drop-in файлом (он дополняет список, не затирая OS-origins):
> ```bash
> cat > /etc/apt/apt.conf.d/51custom-unattended << 'EOF'
> Unattended-Upgrade::Origins-Pattern:: "site=dl.cloudsmith.io";
> EOF
> ```
> Проверить, какие origin'ы разрешены: `unattended-upgrade --dry-run --debug 2>&1 | grep -i origin`.

---

## 6. Firewall (UFW)

### 6.1 Установка и базовые правила

```bash
apt install -y ufw

ufw allow 22/tcp comment 'SSH'
ufw allow 443/tcp comment 'VLESS Reality'
```

> ⚠️ **Не включайте UFW, пока не добавлено правило для SSH (22/tcp)** — иначе рискуете потерять доступ к серверу. В блоке выше оно добавлено первым, поэтому всё в порядке.

> ⚠️ **Только если используете self-steal через Caddy (раздел 8.8)** — дополнительно откройте порт 80, он нужен Caddy для выпуска и продления сертификата Let's Encrypt по ACME HTTP-01. Для метода-сканера (8.1–8.7) порт 80 не требуется.
> ```bash
> ufw allow 80/tcp comment 'Caddy ACME HTTP-01'
> ```

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

> 💡 **В этом разделе два метода маскировки — выбирайте по ситуации:**
> - **8.1–8.7 — заём чужого «соседнего» домена через TLS-сканер.** Подходит, если **своего домена для ноды нет**. Метод рабочий и описан полностью ниже.
> - **8.8 — self-steal через Caddy.** Если **у вас есть свой домен** для ноды — это надёжнее: своя заглушка на своём домене и IP, никакой зависимости от соседей. См. подраздел 8.8.

### 8.1 Зачем это нужно

Reality маскирует VPN-трафик под обычный HTTPS-трафик к легитимному сайту. Для этого нужно выбрать **домен-донор** — сайт, под который будет маскироваться ваш VPN.

**Принцип работы:**
1. Клиент подключается к вашему VPN-серверу с указанным SNI (именем домена-донора)
2. DPI видит TLS-соединение с этим SNI
3. VPN-сервер проксирует рукопожатие на реальный домен-донор и отвечает его настоящим сертификатом — соединение выглядит как обычный заход на этот сайт
4. DPI пропускает трафик, думая что это обычный HTTPS

**⚠️ Главное правило (изменилось): домен-донор должен резолвиться в IP из вашей же подсети.**

Современный DPI (в т.ч. ТСПУ) уже не просто смотрит на SNI — он **сопоставляет SNI с IP назначения**. Если в SNI стоит домен, который публично резолвится в адрес где-то в другой сети/стране, а пакет летит на IP вашего VPS, — это рассинхрон, и соединение могут сбросить (логика «белых списков», переход к которой массово заметен с конца 2025 — начала 2026).

Отсюда следствия:

- **Популярные мировые домены — худший выбор.** `apple.com`, `microsoft.com`, `cloudflare.com`, `google.com`, `python.org` резолвятся в свои глобальные CDN-адреса, никак не связанные с IP вашего сервера. Корреляцию SNI↔IP они НЕ проходят, а вдобавок «засвечены»: их используют тысячи нод, и под них в первую очередь пишут сигнатуры. (На практике именно ноды на `images.apple.com` / `www.python.org` отваливаются первыми после обновлений ТСПУ.)
- **Правильный донор — реальный сайт, физически живущий в вашем датацентре**, чей публичный DNS указывает на адрес из вашей `/24`. Тогда для DPI картина «клиент → ваш IP с SNI=этот-домен» выглядит как обычное обращение к сайту, который там и хостится.

Поэтому сканируем **соседние IP своей подсети** и из найденного отбираем домен, проходящий проверку `dig` (резолвится внутрь вашей /24).

> 📌 Точную гранулярность сверки (точный IP / подсеть /24 / ASN) РКН не публикует. Самый надёжный критерий из доступных — домен из вашей собственной `/24`. Если такого не нашлось — берите хотя бы домен, хостящийся в той же стране у того же провайдера (тот же ASN).

### 8.2 Скачивание сканера

> ⚠️ **Имя файла релиза менялось.** В старых версиях бинарь назывался `RealiTLScanner-linux-64`, в актуальных — `RealiTLScanner-linux-amd64`. Ссылка `latest/download/...` со старым именем отдаёт 404.

**Надёжный способ — не угадывать имя ассета, а взять ссылку из GitHub API:**

```bash
URL=$(curl -s https://api.github.com/repos/XTLS/RealiTLScanner/releases/latest \
  | grep -o '"browser_download_url": *"[^"]*"' | cut -d'"' -f4 \
  | grep -iE 'linux.*(64|amd64)' | head -1)
echo "Качаю: $URL"
curl -fL -o /tmp/RealiTLScanner "$URL" && chmod +x /tmp/RealiTLScanner

# Проверка, что скачался реальный бинарь, а не пустышка/HTML с ошибкой
ls -la /tmp/RealiTLScanner       # размер должен быть несколько МБ, не 0
file /tmp/RealiTLScanner         # ожидается: ELF 64-bit ... executable
/tmp/RealiTLScanner -h | head -20
```

Прямая ссылка (если точно знаете версию — на момент обновления это `v0.2.3`):

```bash
wget https://github.com/XTLS/RealiTLScanner/releases/download/v0.2.3/RealiTLScanner-linux-amd64 -O /tmp/RealiTLScanner
chmod +x /tmp/RealiTLScanner
```

> 💡 **Фолбэк — сборка из исходников** (если бинарь не качается или не запускается из-за архитектуры/glibc). Docker уже установлен на шаге 2, Go ставить на хост не нужно:
> ```bash
> docker run --rm -v /tmp:/out golang:latest \
>   sh -c 'go install github.com/XTLS/RealiTLScanner@latest && cp $(go env GOPATH)/bin/RealiTLScanner /out/'
> chmod +x /tmp/RealiTLScanner && /tmp/RealiTLScanner -h
> ```

> 💡 Не используйте `wget -q` при отладке — без `-q` видна вся цепочка редиректов и HTTP-коды. С `-q` ошибка 404 молча сохраняется как файл размером `0`. Если получили файл `0` байт — сверьте точное имя ассета на странице релизов: <https://github.com/XTLS/RealiTLScanner/releases>

### 8.3 Сканирование своей подсети

Сканируем **всю /24 своей подсети** (а не один IP) и складываем результат в CSV:

```bash
/tmp/RealiTLScanner -addr <YOUR_IP>/24 -thread 30 -out /root/scan.csv
```

> 💡 При `-addr` с одиночным IP включается «infinity mode» и сканер уходит вширь; для нашей задачи нужна именно своя `/24`. Предупреждение `Cannot open Country.mmdb` в начале — это лишь отсутствие GeoIP-базы (поле `geo=N/A`), на поиск доменов не влияет.

**⚠️ Важно понимать, что показывает сканер.** Он коннектится к каждому IP и читает предъявленный сертификат. У обычного сайта это его реальный cert, **а у чужой Reality-ноды — cert её донора**. Поэтому в дешёвых VPS-подсетях вы увидите кучу «доменов» вроде `apple.com`, `python.org`, `cloudflare.com`, `github.com`, `vk.com`, `yandex.*` на случайных адресах — **это не реальные сайты, а другие VPN-ноды**, маскирующиеся под те же бренды. В доноры их брать нельзя.

**Пример вывода (формат v0.2.x):**
```
... ip=64.188.70.184 tls="TLS 1.3" alpn=h2 cert-domain=images.apple.com   cert-issuer="Apple Inc."             ← чужая нода, НЕ брать
... ip=64.188.70.75  tls="TLS 1.3" alpn=h2 cert-domain=www.cloudflare.com cert-issuer="Google Trust Services"  ← чужая нода, НЕ брать
... ip=64.188.70.154 tls="TLS 1.3" alpn=h2 cert-domain=aitake.org         cert-issuer="Let's Encrypt"          ← кандидат, проверить dig
... ip=64.188.70.99  tls="TLS 1.3" alpn=h2 cert-domain=blog.bovi85.com    cert-issuer="Let's Encrypt"          ← кандидат, проверить dig
```

### 8.4 Критерии отбора

Из выдачи оставляем строки с **`TLS 1.3`** и **`alpn=h2`** (минимум для Reality), выкидываем явные famous/RU-бренды (это чужие ноды) и проверяем оставшиеся:

| Критерий | Как проверить | Что нужно |
|----------|---------------|-----------|
| TLS 1.3 + ALPN h2 | вывод сканера | обязательно |
| Резолвится в вашу /24 | `dig +short A <домен>` | **A-запись = адрес из вашей подсети** |
| Без редиректа | `curl -sI https://<домен>/` | код `200`, без `Location:` на чужой хост |
| Не «засвеченный» бренд | здравый смысл | НЕ apple/microsoft/cloudflare/google/discord и т.п. |

> ⚠️ **Издатель сертификата (CA) — НЕ критерий.** Старые гайды советовали избегать Let's Encrypt и брать «коммерческие» CA (DigiCert/GlobalSign). На практике это не работает: реальные локальные сайты в вашей подсети чаще всего как раз на Let's Encrypt, а под famous-бренды с «коммерческими» CA маскируются чужие ноды. Смотрите не на издателя, а на dig-резолв в вашу /24 и чистый `200`.

### 8.5 Проверка кандидата: резолв в подсеть + редирект

Главная проверка — что домен **публично резолвится в адрес из вашей /24**. Это и даёт совпадение SNI↔IP, и заодно отсеивает чужие ноды-маскировщики: у настоящего локального сайта DNS указывает ровно на тот IP, где сканер его нашёл, а у «фейкера» под apple.com DNS ведёт на серверы Apple.

```bash
for d in <домен1> <домен2> <домен3>; do
  printf '%-30s -> %s\n' "$d" "$(dig +short A "$d" | tr '\n' ' ')"
done
```

Оставляем тех, у кого A-запись попадает в вашу `/24`. Затем проверяем отсутствие редиректа (нужен код `200`):

```bash
curl -sI https://<домен>/ | head -3
```

**Хороший ответ:** `HTTP/2 200` или `HTTP/1.1 200 OK`.
**Плохой (НЕ использовать):** `301` / `302` / `307` с `Location:` на сторонний домен.

> 💡 Сервер может ответить на `curl -I` (HEAD) по HTTP/1.1, даже если в сканере ALPN был `h2`. Это не проблема — для Reality важен ALPN `h2` в TLS-рукопожатии (его показывает сканер), а код `200` подтверждает отсутствие редиректа.

### 8.6 Что предпочесть из подходящих

Среди прошедших проверку:
- **Нейтральный TLD и зарегистрированный домен** (`.org`, `.com`, `.cc`) надёжнее, чем динамический DNS (`*.freemyip.com`, `*.strangled.net`) — у последних IP может смениться в любой момент.
- `.ru`-домен на зарубежном IP технически валиден (если его DNS указывает в вашу подсеть — рассинхрона нет), но как лёгкую перестраховку лучше предпочесть нейтральный TLD.
- Сайт, отдающий реальный контент (`200` с нормальной страницей), предпочтительнее «пустого» хоста (`404`).

> 💡 **Минус этого подхода:** вы одалживаете чужой сервер-сосед под свою маскировку — он может лечь или сменить IP, и фолбэк сломается. Для большинства нод это приемлемо (при поломке достаточно пересканировать и сменить домен).
>
> **Надёжнее всего (permanent):** поднять на самом VPS свою сайт-заглушку на своём домене (A-запись на IP ноды, реальный Let's Encrypt-серт) и указать `target` на неё. Тогда SNI↔IP совпадают точно и вы ни от кого не зависите. Схема — Reality на 443 + fallback на локальный web на внутреннем порту. Подробная пошаговая реализация — **раздел 8.8**.

### 8.7 Что записать для дальнейших шагов

После выбора домена сохраните:
- **Домен** (например, отобранный из вашей подсети `aitake.org`) — понадобится для `target` и `serverNames` в конфиге inbound (раздел 11.4)
- Этот же домен пойдёт как **SNI** в настройках хоста (раздел 11.6)

> ⚠️ Для **каждой** ноды домен выбирается заново — из **её собственной** подсети. Нельзя поставить домен франкфуртской ноды на хельсинкскую: он резолвится во франкфуртский IP, и для хельсинкской ноды это снова рассинхрон SNI↔IP.

### 8.8 Self-steal через Caddy (рекомендуется, если есть свой домен)

Если у вас есть домен для ноды — это надёжнее заёма соседа: вместо чужого сайта на самом VPS поднимается локальный веб-сервер (Caddy) с реальной страницей-заглушкой и настоящим сертификатом Let's Encrypt на **вашем собственном домене**. Reality на :443, увидев «чужого» (не-Reality) клиента или DPI-пробера, проксирует рукопожатие на этот локальный сайт. Пробер видит обычный сайт с валидным сертом, физически живущий на этом же IP — SNI и IP совпадают точно, и вы ни от кого не зависите.

Схема: **Reality на :443** → fallback на **локальный Caddy `127.0.0.1:9443`** (HTTPS со своим LE-сертом). Порт `9443` выбран, чтобы не конфликтовать с NODE_PORT (`8443`).

**8.8.1 DNS: A-запись домена ноды на IP сервера**

Заведите/проверьте, что домен ноды (напр. `nl3.example.com`) резолвится в IP **этого** сервера:

```bash
resolvectl query <ВАШ_ДОМЕН>      # должно вернуть IP этого VPS
# или: dig +short A <ВАШ_ДОМЕН>
```

> ⚠️ Без корректной A-записи Caddy **не выпустит сертификат** (ACME HTTP-01 ходит на домен). Если переносите ноду на новый сервер — не забудьте обновить A-запись на новый IP **до** настройки Caddy.

**8.8.2 Установка Caddy из репозитория cloudsmith (НЕ из Debian!)**

> ⚠️ **Критично.** Ставьте Caddy из официального cloudsmith-репозитория (актуальная ветка 2.10+/2.11.x), а **не** `apt install caddy` из Debian (там старый 2.6.2). Старый Caddy не умеет постквантовый обмен ключами **X25519MLKEM768**, и современный TLS-отпечаток (uTLS) при работе с Reality на нём ломается — клиенты с fingerprint `chrome` могут не подключаться.

```bash
apt install -y debian-keyring debian-archive-keyring apt-transport-https curl gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | tee /etc/apt/sources.list.d/caddy-stable.list
apt update && apt install -y caddy

caddy version      # ожидается 2.10.x/2.11.x, НЕ 2.6.2
```

**8.8.3 Страница-заглушка**

Положите в `/var/www/html/index.html` любую правдоподобную страницу (форма входа, «техработы», лендинг — что угодно, что выглядит как обычный сайт; не используйте чужие бренды/логотипы). Минимальный пример — нейтральная страница входа:

```bash
mkdir -p /var/www/html
cat > /var/www/html/index.html << 'EOF'
<!doctype html>
<html lang="en"><head><meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Sign in</title></head>
<body style="font-family:system-ui,sans-serif;background:#0f172a;color:#e2e8f0;display:flex;min-height:100vh;align-items:center;justify-content:center;margin:0">
<main style="max-width:360px;width:100%;padding:2rem;background:#1e293b;border-radius:12px">
<h1 style="font-size:1.4rem;text-align:center">Welcome back</h1>
<p style="color:#94a3b8;text-align:center">Sign in to your account to continue</p>
<p><label>Email<br><input type="email" style="width:100%;padding:.6rem;border-radius:8px;border:1px solid #334155;background:#0f172a;color:#e2e8f0"></label></p>
<p><label>Password<br><input type="password" style="width:100%;padding:.6rem;border-radius:8px;border:1px solid #334155;background:#0f172a;color:#e2e8f0"></label></p>
<button style="width:100%;padding:.7rem;border:none;border-radius:8px;background:#3b82f6;color:#fff;font-weight:600">Sign in</button>
</main></body></html>
EOF
```

> 💡 **Если страница сложная.** При вставке многострочного HTML через буфер он иногда бьётся (особенно символы `<`, кавычки, `EOF` внутри). Надёжнее залить страницу в base64 одной строкой и сверить контрольную сумму:
> ```bash
> echo '<BASE64_СТРОКА>' | base64 -d > /var/www/html/index.html
> sha256sum /var/www/html/index.html   # сверьте с эталоном
> ```

**8.8.4 Caddyfile**

Caddy слушает `:80` (для ACME-челленджа и отдачи страницы) и `127.0.0.1:9443` (локальный HTTPS с сертом). Подставьте свой домен:

```bash
cat > /etc/caddy/Caddyfile << 'EOF'
{
    auto_https disable_redirects
}

:80 {
    root * /var/www/html
    file_server
}

<ВАШ_ДОМЕН>:9443 {
    bind 127.0.0.1
    tls {
        issuer acme {
            disable_tlsalpn_challenge
        }
    }
    root * /var/www/html
    file_server
}
EOF

caddy validate --config /etc/caddy/Caddyfile
systemctl restart caddy
```

> 💡 `disable_tlsalpn_challenge` заставляет Caddy выпускать серт только по HTTP-01 (через :80), чтобы он не пытался занять :443 — этот порт нужен Xray/Reality. `bind 127.0.0.1` на :9443 означает, что стаб слушает только локально (наружу торчит лишь Reality на :443).
> ⚠️ Для выпуска серта нужен **открытый порт 80** (раздел 6) и корректная A-запись (8.8.1).

**8.8.5 Проверка выпуска сертификата**

```bash
sleep 10
journalctl -u caddy --no-pager | grep -iE "certificate obtained|error" | tail
# страница отдаётся по локальному HTTPS:
curl -s --resolve <ВАШ_ДОМЕН>:9443:127.0.0.1 https://<ВАШ_ДОМЕН>:9443 | head
# сертификат именно Let's Encrypt на вашем домене:
echo | openssl s_client -connect 127.0.0.1:9443 -servername <ВАШ_ДОМЕН> 2>/dev/null | openssl x509 -noout -subject -issuer -dates
```

Ожидается `certificate obtained successfully` (без `error`), страница отдаётся, `subject=CN=<ВАШ_ДОМЕН>`, `issuer=...Let's Encrypt...`.

**8.8.6 Что записать для конфига inbound и хоста**

- **target** = `127.0.0.1:9443` (локальный Caddy-стаб) — раздел 11.4
- **serverNames** = `["<ВАШ_ДОМЕН>"]` (ваш домен) — раздел 11.4
- **SNI** в настройках хоста = `<ВАШ_ДОМЕН>` (тот же домен) — раздел 11.6
- **xver** = `0`, **Fingerprint** = `firefox` (см. заметку в 11.6)

**8.8.7 Автообновление Caddy**

См. заметку в разделе 5 — добавление origin `dl.cloudsmith.io` в unattended-upgrades, чтобы Caddy обновлялся вместе с ОС.

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
> 💡 Сразу после `docker compose up -d` Xray может секунд 30–50 ещё не слушать `:443` — нода подключается к панели и забирает конфиг. Если проверяете `ss -tlnp | grep ':443'` и пусто, а контейнер только что стартовал — просто подождите и проверьте снова.

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

> 💡 Если вы **пересоздаёте ранее удалённую ноду**, имейте в виду: Config Profile с инбаундом и Host для подписок обычно **не удаляются** вместе с нодой. В этом случае достаточно заново создать ноду и привязать существующий профиль — Xray поднимется на старом конфиге. Сменить нужно лишь IP в Host/A-записи (если IP сервера новый) и, по желанию, ключи Reality.

### 11.2 Генерация ключей для Reality

Ключи можно сгенерировать в панели автоматически (если в редакторе инбаунда есть кнопка генерации — это предпочтительнее, приватный ключ не покидает панель) или вручную на сервере:

```bash
docker exec remnanode xray x25519
```

> 🔒 **Не вставляйте сгенерированные ключи в публичные места / переписку / коммиты.** Приватный ключ Reality — секрет. Копируйте значения сразу в поля панели. Если ключ где-то засветился — сгенерируйте новый.

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
> 💡 **Ротация ключей** меняет publicKey в подписках, поэтому при смене ключей **меняйте `privateKey` в инбаунде и `PublicKey` + `ShortId` в хосте одновременно**, а клиенты должны обновить подписку (у кого не обновилось автоматически — refresh вручную).

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

> 💡 **Для self-steal через Caddy (раздел 8.8)** значения `realitySettings` другие — `target` указывает на локальный Caddy-стаб, а `serverNames` — ваш собственный домен:
> ```json
> "xver": 0,
> "target": "127.0.0.1:9443",
> "serverNames": ["<ВАШ_ДОМЕН>"],
> ```

> 💡 **Совместимость со старым синтаксисом** (если ядро не понимает `target`/`raw`):
> ```json
> "network": "tcp",
> ...
> "dest": "<ДОМЕН>:443",
> ```
> Проверить версию ядра: `docker exec remnanode xray version`.

> 📌 **В Remnawave конфиг inbound задаётся в панели** (раздел Профили / Config Profile), а не редактированием файла на сервере. Нода получает конфиг от панели по API. Прямое редактирование `config.json` в контейнере бессмысленно — панель перезапишет его при синхронизации.

> ⚠️ **`target` и `serverNames` меняются вместе.** Для метода-сканера (8.1–8.7) это домен-донор из подсети ноды; для self-steal (8.8) — `127.0.0.1:9443` и ваш домен. `target` — куда нода реально ходит за рукопожатием, `serverNames` — что она при этом предъявляет. Поменять одно без другого = cert не совпадёт с SNI, нода не заработает.

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
| SNI | `<ДОМЕН-ДОНОР из подсети ноды>` (например `aitake.org` — см. раздел 8); **для self-steal (8.8) — ваш собственный домен** |
| PublicKey | `<PUBLIC_KEY>` |
| ShortId | `<ОДИН_ИЗ_SHORT_IDS>` |
| Fingerprint | `chrome` (но для self-steal — см. заметку ниже) |
| ALPN | `h2` |

> ⚠️ **Не путайте два домена (актуально для метода-сканера, 8.1–8.7):**
> - **Адрес хоста** (`fi1.example.com`) — куда реально подключается клиент (ваш сервер). Если используете домен — у него должна быть A-запись на IP сервера (проверка: `dig +short fi1.example.com` должна вернуть IP сервера).
> - **SNI** (`aitake.org`) — домен-маскировка для Reality, выбранный из подсети ноды (раздел 8). Это разные домены, так и должно быть.
>
> **Для self-steal (8.8)** адрес хоста и SNI — это один и тот же ваш домен (`nl3.example.com`).

> 💡 **Fingerprint.** `chrome` — безопасный дефолт: имитирует TLS-отпечаток самого распространённого браузера. Избегайте экзотических отпечатков (например `qq` — браузер QQ), которые для трафика из РФ выглядят аномально и могут сами стать триггером, если DPI бьёт по фингерпринту. Параметр клиентский — в Remnawave он уезжает клиентам в подписке.
> ⚠️ **Нюанс self-steal (8.8).** При маскировке на свой Caddy с постквантовым обменом ключей (`X25519MLKEM768`) отпечаток `chrome` на практике может **не подключаться** из-за клиентского бага uTLS. Для таких нод ставьте **`firefox`** (тоже мейнстрим-отпечаток, работает стабильно). Для метода-сканера (8.1–8.7) `chrome` остаётся нормальным дефолтом.

> 💡 **Флаг страны.** В большинстве сборок панели флаг проставляется флаг-эмодзи прямо в названии (`🇫🇮 FI-Helsinki-01`) либо через поле кода страны (ISO: `FI` = Финляндия, `DE` = Германия и т.д.). Убедитесь, что выбрали правильный код — например `FI`, а не `IN` (Индия).

> 💡 **Видимость хоста.** Убедитесь, что у хоста включён тумблер видимости — иначе он не попадёт в подписки и клиент его не получит, даже если нода работает.

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

# DHCP не вернул свой DNS на линк (см. 4.4)
resolvectl status <iface> | grep -E "Current Scopes|DNS Servers"
# на линке НЕ должно быть 8.8.8.8 от провайдера
```

### 12.2 Проверка Xray

> ⚠️ **Важное изменение.** В новых версиях ноды Remnawave Xray встроен в основной процесс **`rw-core`** — отдельного процесса `xray` в `docker exec remnanode ps aux` больше нет. Старая проверка `ps aux | grep xray` даст **ложный** «не запущен». Ориентируйтесь на listener порта 443 и логи.

```bash
# 1. Главный признак — Xray слушает порт 443
ss -tlnp | grep ':443'
# Ожидается: LISTEN на *:443 (процесс rw-core или xray, в зависимости от версии)

# 2. Логи — строка "Xray started" и извлечение пользователей
docker logs remnanode --tail 30
# Ожидается: "Xray started" и "VLESS_..._REALITY_... has N users"

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

### 12.5 Проверка маскировки self-steal (раздел 8.8)

Только для нод с self-steal через Caddy. Главная проверка — что DPI-пробер, ткнувшийся в `:443` с вашим SNI, видит обычный сайт с валидным сертом (Reality проксирует не-Reality клиента на локальный Caddy):

```bash
echo | openssl s_client -connect <IP_СЕРВЕРА>:443 -servername <ВАШ_ДОМЕН> 2>/dev/null \
  | openssl x509 -noout -subject -issuer
# Ожидается: subject=CN=<ВАШ_ДОМЕН>, issuer=...Let's Encrypt...
```

Если openssl отдаёт ваш домен с сертом Let's Encrypt — маскировка работает. Дополнительно постквантовый обмен ключами можно подтвердить (ищите в выводе строку про `X25519MLKEM768` / Post-Quantum):

```bash
docker exec remnanode xray tls ping <ВАШ_ДОМЕН>
```

> 💡 Постквантовый ключ требует свежего Caddy из cloudsmith (раздел 8.8.2). Если его нет — `chrome`-отпечаток может не подключаться; переключите хост на `firefox` (раздел 11.6).

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

### 13.4 Нода/домен отвалились после обновления ТСПУ

**Симптом:** нода технически жива (443 слушает, панель подключена, `docker ps` — UP), но клиенты из РФ перестали подключаться — особенно если отвалилось несколько нод разом.

**Причина (вероятная):** сигнатурное правило DPI по перегруженному SNI и/или корреляция SNI↔IP. Чаще всего бьёт по нодам на популярных доменах (`images.apple.com`, `www.python.org` и т.п.).

**Диагностическая лесенка:**

1. **IP жив?** С устройства из РФ: `curl -m 5 -sv https://<IP_НОДЫ> 2>&1 | head -20`. Если TCP/TLS до IP вообще не доходит — IP в блоке, нужен новый адрес/сервер. Если хендшейк проходит — IP жив, копаем дальше.
2. **Сменить домен-донор** на тихий из **своей /24** (раздел 8.3–8.6) — меняем `target`+`serverNames` на ноде и `SNI` в панели. Это лечит случай «перегруженный SNI / корреляция». Ещё надёжнее — перейти на **self-steal (раздел 8.8)**: свой домен на своём IP убирает рассинхрон полностью.
3. **Проверить fingerprint** — должен быть `chrome` (или `firefox` для self-steal), а не экзотика (раздел 11.6).
4. **Тест живым клиентом из РФ.** Стабильно работает → была SNI/корреляция. Коннект-и-смерть через 1–2 минуты → бьют по фингерпринту самого Reality → переходить на транспорт **XHTTP**. Совсем не коннектится → см. шаг 1 (IP).

> 💡 Меняя домен, помните: для **каждой** ноды он берётся из её собственной подсети (раздел 8.7).

### 13.5 Быстрая диагностика одной командой

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

### 13.6 Docker не пускает трафик

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

### 13.7 Caddy не выпустил сертификат (только для self-steal, 8.8)

**Симптом:** `journalctl -u caddy` показывает ошибки ACME, серт не выпускается.

**Проверки по порядку:**
1. **A-запись.** `resolvectl query <ВАШ_ДОМЕН>` должна вернуть IP **этого** сервера (раздел 8.8.1). Частая причина при переносе ноды — забыли обновить A-запись на новый IP.
2. **Порт 80 открыт.** `ufw status | grep 80` — ACME HTTP-01 ходит на :80 (раздел 6.1).
3. **Версия Caddy.** `caddy version` — должна быть 2.10+/2.11.x из cloudsmith, не 2.6.2 (раздел 8.8.2).
4. ACME сам повторяет попытки; после исправления причины подождите минуту или `systemctl restart caddy`.

---

## 14. Проверка и очистка Firewall

### 14.1 Сравнение портов

```bash
ss -tlpn       # что реально слушает сервер
ufw status     # что открыто в UFW
```

**Принцип:** Если порт открыт в UFW, но на нём ничего не слушает — правило лишнее. (Исключение: порт 80 при self-steal держим открытым ради продления серта Caddy, даже если на нём временно ничего не висит.)

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
> 💡 **Если используете self-steal через Caddy (8.8)** — в этом списке добавится ещё строка `80/tcp ALLOW Anywhere # Caddy ACME HTTP-01` (порт нужен для выпуска/продления серта).

---

## 🔧 Быстрая команда «всё в одном» (базовая подготовка)

> ⚠️ Перед выполнением замените `<IP_REMNAWAVE_PANEL>` на IP вашей панели и `<iface>` на имя интерфейса (`ip a`, напр. `ens1`).
> Блок ставит DoT через **AdGuard** (с резервным Cloudflare) и DHCP-хук против навязанного DNS. На Debian дополнительно ставится `systemd-resolved`.
> Self-steal через Caddy (раздел 8.8) — отдельный шаг после этого блока, т.к. требует вашего домена; там же открывается порт 80.

```bash
apt update && apt upgrade -y && \
( docker --version >/dev/null 2>&1 || curl -fsSL https://get.docker.com | sh ) && \
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
mkdir -p /etc/dhcp/dhclient-exit-hooks.d && \
cat > /etc/dhcp/dhclient-exit-hooks.d/00-no-dhcp-dns << 'EOF'
unset new_domain_name_servers old_domain_name_servers
unset new_dhcp6_name_servers old_dhcp6_name_servers
unset new_domain_search old_domain_search
unset new_dhcp6_domain_search old_dhcp6_domain_search
EOF
chmod +x /etc/dhcp/dhclient-exit-hooks.d/00-no-dhcp-dns && \
systemctl restart systemd-resolved && resolvectl revert <iface>; \
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
echo "=== Базовая подготовка готова! ===" && \
echo "1. (если есть свой домен) Настройте self-steal по разделу 8.8: A-запись -> Caddy из cloudsmith -> Caddyfile -> заглушка -> ufw allow 80/tcp" && \
echo "2. (если домена нет) Выберите домен-донор сканером по разделам 8.1-8.7" && \
echo "3. Перезагрузите сервер: reboot" && \
echo "4. После перезагрузки создайте ноду в панели Remnawave и вставьте docker-compose.yml в /opt/remnanode/" && \
echo "5. Запустите: cd /opt/remnanode && docker compose up -d" && \
echo "6. Проверьте DoT: resolvectl query google.com (encrypted transport: yes) и линк без 8.8.8.8 (resolvectl status <iface>)" && \
echo "7. Проверьте UFW: ufw status (8443 — только для IP панели)"
```

---

## 📝 История изменений (этой редакции)

- **Маскировка (раздел 8):** **ДОБАВЛЕН (не заменяя сканер)** метод **self-steal через Caddy** — раздел **8.8**: своя страница-заглушка на собственном домене ноды с реальным Let's Encrypt-сертификатом, Reality на :443 проксирует «чужих» на локальный Caddy (`127.0.0.1:9443`), SNI↔IP совпадают идеально, нет зависимости от соседей. **Метод заёма соседнего домена через TLS-сканер (разделы 8.1–8.7) сохранён без изменений** и актуален для тех, у кого своего домена нет.
- **Caddy из cloudsmith (8.8.2):** обязательно ставить актуальный Caddy (2.10+/2.11.x) из cloudsmith, а не Debian-овский 2.6.2 — старый не умеет постквантовый `X25519MLKEM768`, и uTLS-отпечаток с Reality ломается. Caddyfile (`:80` ACME + `127.0.0.1:9443` HTTPS-стаб, `disable_tlsalpn_challenge`), заглушка, проверка серта.
- **DNS через DHCP (раздел 4.4):** добавлен фикс для хостеров (HOSTKEY и т.п.), где DHCP навязывает свой DNS (`8.8.8.8`) на линк поверх DoT — exit-hook для `dhclient` + `resolvectl revert`. Переживает перезагрузку.
- **Fingerprint (11.6):** дефолт `chrome` сохранён; добавлен нюанс — при self-steal с постквантовым Caddy отпечаток `chrome` может не подключаться (клиентский баг uTLS), для таких нод ставьте `firefox`.
- **UFW (раздел 6, 14.4):** добавлена заметка, что для self-steal нужен открытый порт 80 (ACME HTTP-01 для Caddy); для метода-сканера он не нужен.
- **Автообновление Caddy (раздел 5):** добавление origin `dl.cloudsmith.io` в unattended-upgrades.
- **Проверка (12.5) и диагностика (13.7):** проверка маскировки self-steal через `openssl s_client` (виден ваш домен + Let's Encrypt) и постквантового обмена через `xray tls ping`; новый раздел диагностики выпуска серта Caddy.
- **Inbound (11.4):** добавлен вариант realitySettings для self-steal (`target: 127.0.0.1:9443`, `serverNames: ваш домен`).
- **Мелочи:** заметка про предустановленный Docker (1.1/2), про сохранение Config Profile/Host при пересоздании ноды (11.1), про видимость хоста (11.6), про задержку старта Xray после `up -d` (9.5), про безопасное обращение с ключами Reality (11.2). В «всё в одном» добавлены DHCP-хук и проверка наличия Docker.

### Предыдущая редакция

- **Выбор домена (раздел 8):** раздел переписан под корреляцию SNI↔IP. Главный критерий теперь — домен-донор должен резолвиться в **вашу /24** (проверка `dig +short A`), а не быть «крупным мировым брендом». Убрана ошибочная рекомендация famous-доменов (apple/python/microsoft/nvidia) и критерий по CA-издателю (Let's Encrypt — это норма, а не «подозрительно»). Добавлено пояснение, что сканер в дешёвых подсетях показывает чужие VPN-ноды под famous-доменами, и как их отсеивать через `dig`. Скан теперь по всей `/24` в CSV. Добавлена заметка про permanent-вариант (своя сайт-заглушка на VPS).
- **Сканер (раздел 8.2):** добавлен надёжный способ скачивания через GitHub API (не зависит от меняющегося имени ассета) и фолбэк-сборка из исходников через Docker; уточнена проверка на «пустой» бинарь.
- **Конфиг Inbound (раздел 11.4):** добавлена заметка, что `target` и `serverNames` меняются вместе (домен-донор из подсети ноды).
- **Fingerprint (раздел 11.6):** добавлено пояснение, почему `chrome` — дефолт, и предупреждение про экзотические отпечатки (`qq` и т.п.).
- **Диагностика (раздел 13.4):** добавлен сценарий «нода отвалилась после обновления ТСПУ» с лесенкой действий (проверка IP → смена домена из своей /24 → fingerprint → XHTTP).

### Ещё раньше

- **DNS (раздел 4):** добавлен AdGuard как основной вариант (шифрование + блокировка рекламы), Cloudflare — резервный, Google — альтернатива. Добавлен обязательный для Debian шаг установки `systemd-resolved` (раздел 4.0).
- **TLS-сканер (раздел 8.2):** исправлено имя файла релиза (`RealiTLScanner-linux-amd64`), добавлена проверка на «пустой» бинарь и совет сверять имя ассета на странице релизов.
- **Конфиг Xray (раздел 11.4):** `dest` → `target`, `network: tcp` → `network: raw`; добавлена заметка о совместимости со старым синтаксисом.
- **Проверка Xray (разделы 12.2, 13.1, 13.5):** убрана ложная проверка `ps aux | grep xray` (Xray встроен в `rw-core`); проверка теперь по listener порта 443 и логам.
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
- [Caddy — установка из репозитория (cloudsmith)](https://caddyserver.com/docs/install#debian-ubuntu-raspbian)

---

[← Назад к README](README.md) | [Zabbix →](ZABBIX.md) | [Prometheus →](PROMETHEUS.md)
