# 📊 Мониторинг: Zabbix Agent 2

Инструкция по установке и настройке Zabbix Agent 2 с PSK-шифрованием для мониторинга VPN-ноды.

## 📋 Содержание

1. [Требования](#требования)
2. [Установка Zabbix Agent 2](#установка-zabbix-agent-2)
3. [Настройка PSK-шифрования](#настройка-psk-шифрования)
4. [Настройка агента](#настройка-агента)
5. [Мониторинг Docker](#мониторинг-docker)
6. [Firewall](#firewall)
7. [Настройка на Zabbix Server](#настройка-на-zabbix-server)
8. [Проверка](#проверка)

---

## Требования

- Zabbix Server 7.4+
- Ubuntu 24.04 LTS на ноде
- Сетевой доступ от Zabbix Server к ноде (порт 10050)

---

## Установка Zabbix Agent 2

### Добавление репозитория

```bash
wget https://repo.zabbix.com/zabbix/7.4/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.4+ubuntu24.04_all.deb
dpkg -i zabbix-release_latest_7.4+ubuntu24.04_all.deb
apt update
```

### Установка агента и плагинов

```bash
apt install -y zabbix-agent2 zabbix-agent2-plugin-*
```

---

## Настройка PSK-шифрования

PSK (Pre-Shared Key) обеспечивает шифрование соединения между агентом и сервером Zabbix.

### Зачем нужно шифрование

- Защита данных мониторинга от перехвата
- Аутентификация агента на сервере
- Предотвращение MITM-атак

### Генерация PSK-ключа

```bash
openssl rand -hex 32 > /etc/zabbix/zabbix_agent2.psk
chmod 640 /etc/zabbix/zabbix_agent2.psk
chown zabbix:zabbix /etc/zabbix/zabbix_agent2.psk
```

### Просмотр ключа

```bash
cat /etc/zabbix/zabbix_agent2.psk
```

**Пример вывода:**
```
8bb2444ed62116f4085e4dd10aa75a04cd17647852ef1b6b10fcb65d2a694563
```

> ⚠️ **Важно:** Сохраните этот ключ — он понадобится при настройке хоста на Zabbix Server!

---

## Настройка агента

### Основной конфиг

```bash
cat > /etc/zabbix/zabbix_agent2.conf << 'EOF'
Server=<IP_ZABBIX_SERVER>
Hostname=<HOSTNAME>
ListenPort=10050

TLSConnect=psk
TLSAccept=psk
TLSPSKIdentity=<HOSTNAME>
TLSPSKFile=/etc/zabbix/zabbix_agent2.psk
EOF
```

**Замените:**
- `<IP_ZABBIX_SERVER>` — IP адрес вашего Zabbix Server
- `<HOSTNAME>` — имя хоста как он будет отображаться в Zabbix (например: `de1-vpn`)

### Пример конфига

```bash
cat > /etc/zabbix/zabbix_agent2.conf << 'EOF'
Server=10.10.10.50
Hostname=de1-vpn
ListenPort=10050

TLSConnect=psk
TLSAccept=psk
TLSPSKIdentity=de1-vpn
TLSPSKFile=/etc/zabbix/zabbix_agent2.psk
EOF
```

### Параметры шифрования

| Параметр | Значение | Описание |
|----------|----------|----------|
| `TLSConnect=psk` | psk | Тип шифрования исходящих соединений |
| `TLSAccept=psk` | psk | Тип шифрования входящих соединений |
| `TLSPSKIdentity` | hostname | Идентификатор PSK (обычно совпадает с Hostname) |
| `TLSPSKFile` | путь | Путь к файлу с PSK-ключом |

---

## Мониторинг Docker

### Добавление zabbix в группу docker

```bash
usermod -aG docker zabbix
```

### Конфиг плагина Docker

```bash
cat > /etc/zabbix/zabbix_agent2.d/docker.conf << 'EOF'
Plugins.Docker.Endpoint=unix:///var/run/docker.sock
EOF
```

---

## Firewall

### Разрешить подключения только от Zabbix Server

```bash
ufw allow from <IP_ZABBIX_SERVER> to any port 10050 proto tcp comment 'Zabbix Agent'
```

### Пример

```bash
ufw allow from 10.10.10.50 to any port 10050 proto tcp comment 'Zabbix Agent'
```

> ⚠️ **Не открывайте порт 10050 для всех!** Доступ должен быть только с IP вашего Zabbix Server.

---

## Запуск агента

```bash
systemctl enable zabbix-agent2
systemctl restart zabbix-agent2
systemctl status zabbix-agent2
```

**Ожидаемый вывод:**
```
● zabbix-agent2.service - Zabbix Agent 2
     Loaded: loaded (/usr/lib/systemd/system/zabbix-agent2.service; enabled; preset: enabled)
     Active: active (running) since ...
   Main PID: 4910 (zabbix_agent2)
...
Dec 08 06:33:32 server zabbix_agent2[4910]: Starting Zabbix Agent 2 (7.4.5)
Dec 08 06:33:32 server zabbix_agent2[4910]: Zabbix Agent2 hostname: [de1.dvpn.info]
```

---

## Настройка на Zabbix Server

### Создание хоста

1. **Сбор данных** → **Узлы сети** → **Создать узел сети**
2. Заполните:
   - **Имя узла сети:** `de1-vpn` (должен совпадать с Hostname в конфиге агента)
   - **Группы:** Выберите или создайте группу (например `VPN Nodes`)
   - **Интерфейсы:** Добавить → Агент
     - IP адрес: `<IP_НОДЫ>`
     - Порт: `10050`

### Настройка шифрования

На вкладке **Шифрование** установите:

| Параметр | Значение |
|----------|----------|
| Подключения к узлу сети | **PSK** |
| Подключения от узла сети | **PSK** |
| PSK идентификатор | `de1-vpn` (совпадает с TLSPSKIdentity) |
| PSK | Ключ из файла `/etc/zabbix/zabbix_agent2.psk` |

> ⚠️ **Важно:** PSK идентификатор на сервере должен **точно совпадать** с `TLSPSKIdentity` в конфиге агента!

### Привязка шаблонов

На вкладке **Шаблоны** добавьте:

| Шаблон | Описание |
|--------|----------|
| `Linux by Zabbix agent` | Базовый мониторинг Linux |
| `Docker by Zabbix agent 2` | Мониторинг Docker контейнеров |

### Сохранение

Нажмите **Добавить** для создания хоста.

---

## Проверка

### На ноде

```bash
# Статус агента
systemctl status zabbix-agent2

# Логи
tail -f /var/log/zabbix/zabbix_agent2.log

# Тест локально
zabbix_agent2 -t agent.ping
zabbix_agent2 -t system.uptime
zabbix_agent2 -t docker.info
```

### На Zabbix Server

```bash
# Тест подключения с PSK
zabbix_get -s <IP_НОДЫ> -k agent.ping \
  --tls-connect psk \
  --tls-psk-identity "<HOSTNAME>" \
  --tls-psk-file /path/to/psk/file
```

**Пример:**
```bash
zabbix_get -s 192.168.1.100 -k agent.ping \
  --tls-connect psk \
  --tls-psk-identity "de1-vpn" \
  --tls-psk-file /tmp/de1.psk
```

**Ожидаемый результат:**
```
1
```

### В веб-интерфейсе

1. **Мониторинг** → **Узлы сети**
2. Найдите хост, проверьте:
   - **Доступность:** ZBX должен быть зелёным
   - **Последние данные:** должны появиться метрики

### Ошибки шифрования

Если в логах видите:
```
failed to accept an incoming connection: from <IP>: connection of type "unencrypted" is not allowed
```

Это означает, что на Zabbix Server не настроено PSK шифрование для этого хоста. Проверьте вкладку **Шифрование** в настройках хоста.

---

## 🔧 Быстрая установка

> ⚠️ Замените `<IP_ZABBIX_SERVER>` и `<HOSTNAME>` перед выполнением!

```bash
wget https://repo.zabbix.com/zabbix/7.4/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.4+ubuntu24.04_all.deb && \
dpkg -i zabbix-release_latest_7.4+ubuntu24.04_all.deb && \
apt update && \
apt install -y zabbix-agent2 zabbix-agent2-plugin-* && \
openssl rand -hex 32 > /etc/zabbix/zabbix_agent2.psk && \
chmod 640 /etc/zabbix/zabbix_agent2.psk && \
chown zabbix:zabbix /etc/zabbix/zabbix_agent2.psk && \
cat > /etc/zabbix/zabbix_agent2.conf << 'EOF'
Server=<IP_ZABBIX_SERVER>
Hostname=<HOSTNAME>
ListenPort=10050

TLSConnect=psk
TLSAccept=psk
TLSPSKIdentity=<HOSTNAME>
TLSPSKFile=/etc/zabbix/zabbix_agent2.psk
EOF
usermod -aG docker zabbix && \
cat > /etc/zabbix/zabbix_agent2.d/docker.conf << 'EOF'
Plugins.Docker.Endpoint=unix:///var/run/docker.sock
EOF
systemctl enable zabbix-agent2 && \
systemctl restart zabbix-agent2 && \
ufw allow from <IP_ZABBIX_SERVER> to any port 10050 proto tcp comment 'Zabbix Agent' && \
echo "=== Zabbix Agent 2 установлен ===" && \
echo "PSK ключ для Zabbix Server:" && \
cat /etc/zabbix/zabbix_agent2.psk
```

---

## Данные для Zabbix Server

После установки агента вам понадобятся эти данные для создания хоста:

| Параметр | Где взять |
|----------|-----------|
| Hostname | Из конфига: `grep Hostname /etc/zabbix/zabbix_agent2.conf` |
| IP адрес | IP адрес VPS |
| Порт | 10050 (по умолчанию) |
| PSK Identity | Совпадает с Hostname |
| PSK ключ | `cat /etc/zabbix/zabbix_agent2.psk` |

---

## Полезные метрики

| Ключ | Описание |
|------|----------|
| `agent.ping` | Доступность агента |
| `system.uptime` | Время работы системы |
| `system.cpu.util` | Загрузка CPU |
| `vm.memory.utilization` | Использование RAM |
| `vfs.fs.size[/,pused]` | Использование диска |
| `net.if.in[eth0]` | Входящий трафик |
| `net.if.out[eth0]` | Исходящий трафик |
| `docker.info` | Информация о Docker |
| `docker.containers.running` | Количество запущенных контейнеров |

---

## Troubleshooting

### Агент не запускается

```bash
journalctl -u zabbix-agent2 -f
```

### Нет связи с сервером

1. Проверьте firewall на ноде: `ufw status`
2. Проверьте firewall на сервере
3. Проверьте правильность IP в конфиге
4. Проверьте совпадение Hostname

### Ошибка PSK

```
TLS handshake failed
```

1. Проверьте совпадение PSK Identity на агенте и сервере
2. Проверьте что PSK ключ одинаковый
3. Проверьте права на файл PSK: `ls -la /etc/zabbix/zabbix_agent2.psk`

### Docker метрики не собираются

```bash
# Проверьте группу
groups zabbix

# Если docker нет в списке
usermod -aG docker zabbix

# Перезапустите агент
systemctl restart zabbix-agent2
```

---

[← VPS.md](VPS.md) | [README](README.md) | [Prometheus →](PROMETHEUS.md)
