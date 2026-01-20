# 📊 Мониторинг: Zabbix Agent 2

Инструкция по установке и настройке Zabbix Agent 2 для мониторинга VPN-ноды.

## 📋 Содержание

1. [Требования](#требования)
2. [Установка Zabbix Agent 2](#установка-zabbix-agent-2)
3. [Настройка агента](#настройка-агента)
4. [Мониторинг Docker](#мониторинг-docker)
5. [Firewall](#firewall)
6. [Настройка на Zabbix Server](#настройка-на-zabbix-server)
7. [Проверка](#проверка)

---

## Требования

- Zabbix Server 7.0+
- Ubuntu 24.04 LTS на ноде
- Сетевой доступ от Zabbix Server к ноде (порт 10050)

---

## Установка Zabbix Agent 2

### Добавление репозитория

```bash
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu24.04_all.deb
dpkg -i zabbix-release_latest_7.0+ubuntu24.04_all.deb
apt update
```

### Установка агента и плагинов

```bash
apt install -y zabbix-agent2 zabbix-agent2-plugin-*
```

---

## Настройка агента

### Основной конфиг

```bash
cat > /etc/zabbix/zabbix_agent2.conf << 'EOF'
PidFile=/run/zabbix/zabbix_agent2.pid
LogFile=/var/log/zabbix/zabbix_agent2.log
LogFileSize=0
Server=<IP_ZABBIX_SERVER>
ServerActive=<IP_ZABBIX_SERVER>
Hostname=<HOSTNAME>
Include=/etc/zabbix/zabbix_agent2.d/*.conf
PluginTimeout=30
Timeout=30
EOF
```

**Замените:**
- `<IP_ZABBIX_SERVER>` — IP адрес вашего Zabbix Server
- `<HOSTNAME>` — имя хоста как он будет отображаться в Zabbix (например: `de1-vpn`)

### Пример

```bash
cat > /etc/zabbix/zabbix_agent2.conf << 'EOF'
PidFile=/run/zabbix/zabbix_agent2.pid
LogFile=/var/log/zabbix/zabbix_agent2.log
LogFileSize=0
Server=10.10.10.50
ServerActive=10.10.10.50
Hostname=de1-vpn
Include=/etc/zabbix/zabbix_agent2.d/*.conf
PluginTimeout=30
Timeout=30
EOF
```

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

### Разрешить подключения от Zabbix Server

```bash
ufw allow from <IP_ZABBIX_SERVER> to any port 10050 proto tcp comment 'Zabbix Agent'
```

### Пример

```bash
ufw allow from 10.10.10.50 to any port 10050 proto tcp comment 'Zabbix Agent'
```

---

## Запуск агента

```bash
systemctl enable zabbix-agent2
systemctl restart zabbix-agent2
systemctl status zabbix-agent2
```

---

## Настройка на Zabbix Server

### Создание хоста

1. **Configuration** → **Hosts** → **Create host**
2. Заполните:
   - **Host name:** `de1-vpn` (должен совпадать с Hostname в конфиге агента)
   - **Groups:** Выберите или создайте группу (например `VPN Nodes`)
   - **Interfaces:** Add → Agent
     - IP address: `<IP_НОДЫ>`
     - Port: `10050`

### Привязка шаблонов

На вкладке **Templates** добавьте:

| Шаблон | Описание |
|--------|----------|
| `Linux by Zabbix agent` | Базовый мониторинг Linux |
| `Docker by Zabbix agent 2` | Мониторинг Docker контейнеров |

### Сохранение

Нажмите **Add** для создания хоста.

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
# Тест подключения
zabbix_get -s <IP_НОДЫ> -k agent.ping
zabbix_get -s <IP_НОДЫ> -k system.uptime
zabbix_get -s <IP_НОДЫ> -k docker.info
```

**Ожидаемый результат `agent.ping`:**
```
1
```

### В веб-интерфейсе

1. **Monitoring** → **Hosts**
2. Найдите хост, проверьте:
   - **Availability:** ZBX должен быть зелёным
   - **Latest data:** должны появиться метрики

---

## 🔧 Быстрая установка

```bash
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu24.04_all.deb && \
dpkg -i zabbix-release_latest_7.0+ubuntu24.04_all.deb && \
apt update && \
apt install -y zabbix-agent2 zabbix-agent2-plugin-* && \
cat > /etc/zabbix/zabbix_agent2.conf << 'EOF'
PidFile=/run/zabbix/zabbix_agent2.pid
LogFile=/var/log/zabbix/zabbix_agent2.log
LogFileSize=0
Server=<IP_ZABBIX_SERVER>
ServerActive=<IP_ZABBIX_SERVER>
Hostname=<HOSTNAME>
Include=/etc/zabbix/zabbix_agent2.d/*.conf
PluginTimeout=30
Timeout=30
EOF
usermod -aG docker zabbix && \
cat > /etc/zabbix/zabbix_agent2.d/docker.conf << 'EOF'
Plugins.Docker.Endpoint=unix:///var/run/docker.sock
EOF
systemctl enable zabbix-agent2 && \
systemctl restart zabbix-agent2 && \
ufw allow from <IP_ZABBIX_SERVER> to any port 10050 proto tcp comment 'Zabbix Agent' && \
echo "=== Zabbix Agent 2 установлен ==="
```

> ⚠️ Замените `<IP_ZABBIX_SERVER>` и `<HOSTNAME>` перед выполнением!

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

1. Проверьте firewall на ноде
2. Проверьте firewall на сервере
3. Проверьте правильность IP в конфиге
4. Проверьте совпадение Hostname

### Docker метрики не собираются

```bash
# Проверьте группу
groups zabbix

# Перезапустите агент после добавления в группу
systemctl restart zabbix-agent2
```

---

[← VPS.md](VPS.md) | [README](README.md) | [Prometheus →](PROMETHEUS.md)
