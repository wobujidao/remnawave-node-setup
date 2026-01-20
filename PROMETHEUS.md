# 📈 Мониторинг: Prometheus + Grafana

Инструкция по настройке мониторинга VPN-нод через Prometheus и визуализации в Grafana.

## 📋 Содержание

1. [Архитектура](#архитектура)
2. [Node Exporter на ноде](#node-exporter-на-ноде)
3. [Firewall на ноде](#firewall-на-ноде)
4. [Настройка Prometheus Server](#настройка-prometheus-server)
5. [Настройка Grafana](#настройка-grafana)
6. [Дашборды](#дашборды)
7. [Алерты](#алерты)

---

## Архитектура

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  VPN Node 1 │     │  VPN Node 2 │     │  VPN Node 3 │
│  :9100      │     │  :9100      │     │  :9100      │
│  node-exp   │     │  node-exp   │     │  node-exp   │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
                    ┌──────▼──────┐
                    │  Prometheus │
                    │  :9090      │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │   Grafana   │
                    │  :3000      │
                    └─────────────┘
```

---

## Node Exporter на ноде

### Создание директории

```bash
mkdir -p /opt/monitoring && cd /opt/monitoring
```

### Docker Compose

```bash
cat > /opt/monitoring/docker-compose.yml << 'EOF'
services:
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: always
    network_mode: host
    pid: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
      - '--web.listen-address=:9100'
EOF
```

### Запуск

```bash
cd /opt/monitoring
docker compose up -d
```

### Проверка

```bash
curl http://localhost:9100/metrics | head -20
```

---

## Firewall на ноде

Разрешите доступ только с IP Prometheus Server:

```bash
ufw allow from <IP_PROMETHEUS_SERVER> to any port 9100 proto tcp comment 'Node Exporter'
```

### Пример

```bash
ufw allow from 10.10.10.60 to any port 9100 proto tcp comment 'Node Exporter'
```

---

## Настройка Prometheus Server

### Docker Compose для Prometheus + Grafana

```bash
mkdir -p /opt/prometheus && cd /opt/prometheus

cat > docker-compose.yml << 'EOF'
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: always
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: always
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=<SECURE_PASSWORD>
      - GF_USERS_ALLOW_SIGN_UP=false

volumes:
  prometheus_data:
  grafana_data:
EOF
```

### Конфиг Prometheus

```bash
cat > /opt/prometheus/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: []

rule_files: []

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'vpn-nodes'
    static_configs:
      - targets:
        - '<NODE_1_IP>:9100'
        - '<NODE_2_IP>:9100'
        - '<NODE_3_IP>:9100'
    relabel_configs:
      - source_labels: [__address__]
        regex: '(.*):\d+'
        target_label: instance
        replacement: '${1}'
EOF
```

> ⚠️ Замените `<NODE_X_IP>` на реальные IP адреса ваших нод.

### Запуск

```bash
cd /opt/prometheus
docker compose up -d
```

### Проверка

- Prometheus: http://<IP_SERVER>:9090
- Grafana: http://<IP_SERVER>:3000

---

## Настройка Grafana

### Первый вход

1. Откройте http://<IP_SERVER>:3000
2. Логин: `admin`
3. Пароль: указанный в `GF_SECURITY_ADMIN_PASSWORD`

### Добавление Data Source

1. **Configuration** → **Data Sources** → **Add data source**
2. Выберите **Prometheus**
3. URL: `http://prometheus:9090`
4. Нажмите **Save & Test**

---

## Дашборды

### Node Exporter Full (рекомендуется)

1. **Dashboards** → **Import**
2. ID: `1860`
3. Нажмите **Load**
4. Выберите Data Source: Prometheus
5. Нажмите **Import**

### Другие полезные дашборды

| ID | Название | Описание |
|----|----------|----------|
| 1860 | Node Exporter Full | Полный мониторинг Linux |
| 11074 | Node Exporter for Prometheus | Компактный вариант |
| 13659 | Blackbox Exporter | Мониторинг доступности |

---

## Алерты

### Пример правил алертов

Создайте файл `/opt/prometheus/alerts.yml`:

```yaml
groups:
  - name: vpn-nodes
    rules:
      - alert: NodeDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Node {{ $labels.instance }} is down"
          description: "Node has been down for more than 1 minute"

      - alert: HighCPU
        expr: 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is above 80% for 5 minutes"

      - alert: HighMemory
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is above 85%"

      - alert: DiskSpaceLow
        expr: (1 - (node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes{fstype!="tmpfs"})) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk space low on {{ $labels.instance }}"
          description: "Disk usage is above 85%"
```

### Обновление prometheus.yml

```yaml
rule_files:
  - 'alerts.yml'
```

### Перезагрузка конфига

```bash
curl -X POST http://localhost:9090/-/reload
```

---

## 🔧 Быстрая установка Node Exporter на ноде

```bash
mkdir -p /opt/monitoring && cd /opt/monitoring && \
cat > docker-compose.yml << 'EOF'
services:
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: always
    network_mode: host
    pid: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
      - '--web.listen-address=:9100'
EOF
docker compose up -d && \
ufw allow from <IP_PROMETHEUS_SERVER> to any port 9100 proto tcp comment 'Node Exporter' && \
echo "=== Node Exporter установлен ==="
```

> ⚠️ Замените `<IP_PROMETHEUS_SERVER>` перед выполнением!

---

## Полезные метрики

| Метрика | Описание |
|---------|----------|
| `up` | Доступность ноды (1 = up, 0 = down) |
| `node_cpu_seconds_total` | Время CPU по режимам |
| `node_memory_MemAvailable_bytes` | Доступная память |
| `node_filesystem_avail_bytes` | Свободное место на диске |
| `node_network_receive_bytes_total` | Входящий трафик |
| `node_network_transmit_bytes_total` | Исходящий трафик |
| `node_load1` | Load average 1 минута |

---

## Полезные PromQL запросы

### CPU Usage %

```promql
100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

### Memory Usage %

```promql
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

### Disk Usage %

```promql
(1 - (node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes{fstype!="tmpfs"})) * 100
```

### Network Traffic (bytes/sec)

```promql
irate(node_network_receive_bytes_total{device="eth0"}[5m])
irate(node_network_transmit_bytes_total{device="eth0"}[5m])
```

---

## Troubleshooting

### Node Exporter не отвечает

```bash
# Проверка контейнера
docker ps | grep node-exporter
docker logs node-exporter

# Проверка порта
ss -tlpn | grep 9100

# Тест локально
curl http://localhost:9100/metrics
```

### Prometheus не видит ноду

1. Проверьте firewall на ноде
2. Проверьте URL в prometheus.yml
3. Проверьте статус в Prometheus UI → Status → Targets

### Grafana не показывает данные

1. Проверьте Data Source
2. Проверьте временной диапазон
3. Проверьте запрос в панели

---

[← Zabbix](ZABBIX.md) | [README](README.md) | [VPS.md →](VPS.md)
