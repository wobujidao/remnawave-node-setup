# 🖥️ Полная инструкция по настройке VPN-ноды

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
13. [Проверка и очистка Firewall](#13-проверка-и-очистка-firewall)

---

## 1. Подготовка сервера

### 1.1 Проверка базовой информации

```bash
# Информация об ОС
cat /etc/os-release | head -3

# IP адреса
ip a

# Геолокация IP
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

---

## 2. Установка Docker

Используем официальный скрипт установки Docker:

```bash
curl -fsSL https://get.docker.com | sh
```

Проверка:

```bash
docker --version
docker compose version
```

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

> ⚠️ **Важно:** Для полного применения лимитов требуется перезагрузка сервера.

---

## 4. DNS over TLS

### 4.1 Настройка

```bash
cat > /etc/systemd/resolved.conf << 'EOF'
[Resolve]
DNS=1.1.1.1#cloudflare-dns.com 1.0.0.1#cloudflare-dns.com 8.8.8.8#dns.google 8.8.4.4#dns.google
DNSOverTLS=yes
DNSSEC=allow-downgrade
EOF

systemctl restart systemd-resolved
```

### 4.2 Проверка DoT

**Тест 1: Проверка шифрования запроса**

```bash
resolvectl query google.com
```

Ожидаемый результат:
```
google.com: 142.250.184.238                    -- link: ens3

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

> 💡 **Пояснение:** Формулировка `local or encrypted` означает способ передачи. `local` — через локальный резолвер (127.0.0.53), `encrypted` — шифрование TLS. При работающем DoT запрос идёт через локальный stub-резолвер, который затем шифрует его через TLS к DNS-серверу (Cloudflare/Google).

**Тест 2: Статус службы resolved**

```bash
resolvectl status
```

Ожидаемый результат:
```
Global
         Protocols: -LLMNR -mDNS +DNSOverTLS DNSSEC=allow-downgrade/supported
                                 ^^^^^^^^^^^
                                 DoT ВКЛЮЧЁН
  resolv.conf mode: stub
Current DNS Server: 1.1.1.1#cloudflare-dns.com
       DNS Servers: 1.1.1.1#cloudflare-dns.com 1.0.0.1#cloudflare-dns.com ...
```

✅ **`+DNSOverTLS`** = протокол DNS-over-TLS активен  
✅ **`Current DNS Server: 1.1.1.1#cloudflare-dns.com`** = используется Cloudflare DoT

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

---

## 6. Firewall (UFW)

### 6.1 Установка и базовая настройка

```bash
apt install -y ufw

ufw allow 22/tcp comment 'SSH'
ufw allow 443/tcp comment 'VLESS Reality'
ufw --force enable
ufw status
```

### 6.2 NODE_PORT — только для IP панели!

NODE_PORT (по умолчанию 2222, в нашем примере используем 8443) должен быть открыт **только для IP панели Remnawave**:

```bash
ufw allow from <IP_REMNAWAVE_PANEL> to any port 8443 proto tcp comment 'Remnanode API'
```

> ⚠️ **Не открывайте NODE_PORT для всех!** Это внутренний API для связи панели с нодой. Доступ должен быть только с IP вашей панели Remnawave.

### 6.3 Если несколько IP на сервере

Для привязки порта к конкретному IP:

```bash
ufw allow in to <YOUR_VPN_IP> port 443 proto tcp comment 'VLESS Reality'
```

### 6.4 Правила комментирования

Все правила UFW должны иметь комментарии для понимания их назначения:

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
systemctl status fail2ban
```

---

## 8. TLS сканирование (выбор домена для маскировки)

### 8.1 Зачем это нужно

Reality маскирует VPN-трафик под обычный HTTPS-трафик к легитимному сайту. Для этого нужно выбрать **домен-донор** — сайт, под который будет маскироваться ваш VPN.

**Принцип работы:**
1. Клиент подключается к вашему VPN-серверу
2. DPI видит TLS-соединение с SNI `www.python.org` (или другой домен)
3. VPN-сервер отвечает сертификатом донора — соединение выглядит легитимно
4. DPI пропускает трафик, думая что это обычный HTTPS

**Важно:** Домен должен быть на том же хостинге/IP-диапазоне, что и ваш сервер. Поэтому сканируем **соседние IP** — ищем сайты на той же площадке.

### 8.2 Скачивание сканера

```bash
wget -q https://github.com/XTLS/RealiTLScanner/releases/latest/download/RealiTLScanner-linux-64 -O /tmp/RealiTLScanner
chmod +x /tmp/RealiTLScanner
```

### 8.3 Сканирование соседних IP

```bash
/tmp/RealiTLScanner -addr <YOUR_IP> -thread 30 | head -50
```

**Пример вывода:**
```
64.188.70.15:443  www.python.org         TLS 1.3  h2         GlobalSign
64.188.70.22:443  images.apple.com       TLS 1.3  h2         Apple Inc.
64.188.70.31:443  www.nvidia.com         TLS 1.3  h2         DigiCert
64.188.70.45:443  discord.com            TLS 1.3  h2         Cloudflare    ← НЕ БРАТЬ
64.188.70.52:443  some-site.com          TLS 1.2  http/1.1   Let's Encrypt ← НЕ БРАТЬ
```

### 8.4 На что обращать внимание

| Параметр | Хорошо ✅ | Плохо ❌ |
|----------|-----------|----------|
| **TLS версия** | TLS 1.3 | TLS 1.2 и ниже |
| **ALPN** | h2 (HTTP/2) | http/1.1 |
| **Издатель сертификата** | GlobalSign, DigiCert, Apple, Microsoft | Let's Encrypt, Cloudflare |
| **Тип сайта** | Крупные компании, CDN | Личные сайты, VPN-сервисы |

### 8.5 Критерии выбора домена

**✅ Подходят:**
- TLS 1.3 + ALPN h2
- Крупные компании (Apple, Microsoft, Python, IGN, Nvidia)
- Коммерческие сертификаты (GlobalSign, DigiCert)
- Без редиректа (HTTP 200)

**❌ НЕ подходят:**
- **Cloudflare, Discord** — популярны для VPN, легко детектятся
- **Let's Encrypt** — часто используется VPN-серверами, вызывает подозрения
- Домены с редиректом (301, 302)
- Малоизвестные сайты

### 8.6 Проверка домена на редирект

После выбора домена проверьте что он отвечает без редиректа:

```bash
curl -sI https://www.python.org | head -3
```

**Хороший ответ:**
```
HTTP/2 200
```

**Плохой ответ (редирект — НЕ использовать):**
```
HTTP/2 301
HTTP/2 302
```

### 8.7 Рекомендуемые домены

| Домен | Издатель | Статус |
|-------|----------|--------|
| www.python.org | GlobalSign | ✅ |
| images.apple.com | Apple Inc. | ✅ |
| www.nvidia.com | DigiCert | ✅ |
| ign.com | GlobalSign | ⚠️ Проверить |
| www.microsoft.com | Microsoft | ⚠️ Проверить |

> ⚠️ **Важно:** Всегда проверяйте актуальность домена перед использованием. Сертификаты и настройки сайтов могут меняться.

### 8.8 Что делать дальше

После выбора домена, запишите:
- **Домен** (например: `www.python.org`) — понадобится для `dest` и `serverNames` в конфиге inbound
- Этот же домен будет использоваться как **SNI** в настройках хоста

---

## 9. Установка Remnanode

### 9.1 Создание директории

```bash
mkdir -p /opt/remnanode && cd /opt/remnanode
```

### 9.2 Создание ноды в панели Remnawave

1. Откройте панель Remnawave
2. Перейдите в **Nodes** → **Management**
3. Нажмите кнопку **+** для добавления новой ноды
4. Заполните форму:
   - **Name:** Название ноды (например: `DE-Frankfurt-01`)
   - **Address:** IP адрес сервера
   - **Node Port:** Порт для внутреннего API (по умолчанию `2222`, в примере `8443`)
5. Нажмите **Copy docker-compose.yml** для копирования конфигурации

### 9.3 Создание docker-compose.yml

```bash
mcedit /opt/remnanode/docker-compose.yml
```

Вставьте скопированную конфигурацию из панели. Пример структуры:

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

> ⚠️ **Важно:** Параметр `volumes` обязателен для работы логирования!

> ⚠️ **Важно:** Используйте конфигурацию из панели! Она содержит правильный SECRET_KEY.

### 9.4 Создание директории для логов

```bash
mkdir -p /var/log/remnanode
```

### 9.5 Запуск контейнера

```bash
cd /opt/remnanode
docker compose up -d && docker compose logs -f -t
```

Нажмите `Ctrl+C` для выхода из просмотра логов.

### 9.6 Автообновление ноды (суббота 11:00 MSK)

```bash
cat > /etc/cron.d/remnawave-update << 'EOF'
0 11 * * 6 root cd /opt/remnanode && docker compose pull -q && docker compose down && docker compose up -d >> /var/log/remnawave-update.log 2>&1
EOF
chmod 644 /etc/cron.d/remnawave-update
```

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

> ⚠️ **Важно:** Обязательно настройте ротацию логов, иначе они заполнят диск!

---

## 11. Настройка в панели Remnawave

### 11.1 Завершение создания ноды

1. В карточке создания ноды нажмите **Next**
2. Выберите нужный **Config Profile**
3. Нажмите **Create**

### 11.2 Генерация ключей для Reality

В панели автоматически генерируются ключи. Если нужно сгенерировать вручную:

```bash
docker exec remnanode xray x25519
```

**Пример вывода:**
```
PrivateKey: aBcDeFgHiJkLmNoPqRsTuVwXyZ1234567890abcdefg
Password: XyZ1234567890abcdefgaBcDeFgHiJkLmNoPqRsTuVw   <-- Это PublicKey!
```

### 11.3 Генерация ShortIds

```bash
openssl rand -hex 8
openssl rand -hex 8
openssl rand -hex 8
openssl rand -hex 8
```

### 11.4 Конфиг Inbound

```json
{
  "log": {
    "error": "/var/log/remnanode/error.log",
    "access": "/var/log/remnanode/access.log",
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "tag": "VLESS_TCP_Reality_<НАЗВАНИЕ>",
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
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "dest": "<ДОМЕН>:443",
          "xver": 0,
          "serverNames": ["<ДОМЕН>"],
          "privateKey": "<PRIVATE_KEY>",
          "shortIds": ["<SHORT_ID_1>", "<SHORT_ID_2>", "<SHORT_ID_3>", "<SHORT_ID_4>"]
        }
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "tag": "DIRECT"
    },
    {
      "protocol": "blackhole",
      "tag": "BLOCK"
    }
  ],
  "routing": {
    "rules": [
      {
        "ip": ["geoip:private"],
        "outboundTag": "BLOCK",
        "type": "field"
      },
      {
        "protocol": ["bittorrent"],
        "outboundTag": "BLOCK",
        "type": "field"
      }
    ]
  }
}
```

### 11.5 Если несколько IP на сервере

Измените `listen` на конкретный IP:

```json
"listen": "<YOUR_VPN_IP>",
```

### 11.6 Настройки хоста

| Параметр | Значение |
|----------|----------|
| Название | 🇩🇪 DE-Frankfurt-01 |
| Адрес | `<IP_СЕРВЕРА>` |
| Порт | `443` |
| SNI | `<ДОМЕН>` |
| PublicKey | `<PUBLIC_KEY>` |
| ShortId | `<ОДИН_ИЗ_SHORT_IDS>` |
| Fingerprint | `chrome` |
| ALPN | `h2` |

---

## 12. Проверка работоспособности

### 12.1 После перезагрузки

```bash
# Лимиты
ulimit -n                                    # Ожидается: 300000
cat /proc/sys/net/netfilter/nf_conntrack_max # Ожидается: 262144

# Docker
docker ps

# Порты
ss -tlpn | grep -E ':443|:8443'

# DNS over TLS
resolvectl query google.com
# Ожидается: "encrypted transport: yes"

resolvectl status | grep -E "DNSOverTLS|Current DNS"
# Ожидается: "+DNSOverTLS" и "Current DNS Server: 1.1.1.1#cloudflare-dns.com"
```

### 12.2 Проверка Xray

```bash
docker logs remnanode --tail 30
```

### 12.3 Проверка логов

```bash
tail -f /var/log/remnanode/error.log
tail -f /var/log/remnanode/access.log
```

### 12.4 Тест подключения

Подключитесь через VPN-клиент и проверьте IP:
- https://2ip.ru
- https://whoer.net

---

## 13. Проверка и очистка Firewall

После настройки рекомендуется проверить что в UFW нет лишних правил.

### 13.1 Сравнение портов

```bash
# Что реально слушает на сервере
ss -tlpn

# Что открыто в UFW
ufw status
```

**Принцип:** Если порт открыт в UFW, но ничего не слушает на этом порту — правило лишнее.

### 13.2 Удаление лишних портов

```bash
ufw delete allow <PORT>
```

**Пример:**
```bash
ufw delete allow 8080/tcp
ufw delete allow 80/tcp
```

### 13.3 Добавление комментариев к правилам

Если правило без комментария — пересоздайте с комментарием:

```bash
# Удаляем старое правило
ufw delete allow 443

# Создаём с комментарием
ufw allow 443 comment 'Xray VPN'
```

### 13.4 Эталонный вид UFW для VPN-ноды

```
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       Anywhere                   # SSH
443                        ALLOW       Anywhere                   # Xray VPN
8443/tcp                   ALLOW       123.45.67.89               # Remnanode API
22/tcp (v6)                ALLOW       Anywhere (v6)              # SSH
443 (v6)                   ALLOW       Anywhere (v6)              # Xray VPN
```

> 💡 **Обратите внимание:** Порт 8443 (Remnanode API) открыт только для IP панели Remnawave, а не для всех!

---

## 🔧 Быстрая команда "всё в одном"

> ⚠️ **Перед выполнением замените `<IP_REMNAWAVE_PANEL>` на IP вашей панели Remnawave!**

```bash
apt update && apt upgrade -y && \
curl -fsSL https://get.docker.com | sh && \
apt install -y ufw fail2ban mc htop btop iftop unattended-upgrades curl wget logrotate && \
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
sysctl -p && \
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
DNS=1.1.1.1#cloudflare-dns.com 1.0.0.1#cloudflare-dns.com 8.8.8.8#dns.google 8.8.4.4#dns.google
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
0 11 * * 6 root cd /opt/remnanode && docker compose pull -q && docker compose down && docker compose up -d >> /var/log/remnawave-update.log 2>&1
EOF
chmod 644 /etc/cron.d/remnawave-update && \
systemctl daemon-reload && \
echo "=== Готово! ===" && \
echo "1. Перезагрузите сервер: reboot" && \
echo "2. После перезагрузки создайте ноду в панели Remnawave" && \
echo "3. Скопируйте docker-compose.yml из панели в /opt/remnanode/" && \
echo "4. Запустите: cd /opt/remnanode && docker compose up -d" && \
echo "5. Проверьте UFW: ufw status (порт 8443 должен быть открыт только для IP панели)"
```

---

## 📚 Полезные ссылки

- [Официальная документация Remnawave Node](https://docs.rw/docs/install/remnawave-node)
- [Remnawave Telegram](https://t.me/remnawave)
- [Xray-core GitHub](https://github.com/XTLS/Xray-core)

---

[← Назад к README](README.md) | [Zabbix →](ZABBIX.md) | [Prometheus →](PROMETHEUS.md)
