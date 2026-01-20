# 🚀 Remnawave Node Setup Guide

> Полная документация по настройке VPN-нод на базе Remnawave/Xray с VLESS + TCP + Reality

## 📋 Содержание документации

| Файл | Описание |
|------|----------|
| [VPS.md](VPS.md) | Полная инструкция по настройке VPN-ноды |
| [ZABBIX.md](ZABBIX.md) | Настройка мониторинга через Zabbix Agent 2 |
| [PROMETHEUS.md](PROMETHEUS.md) | Настройка мониторинга через Prometheus + Grafana |

## 🎯 Быстрый старт

### Требования

- Ubuntu 24.04 LTS
- Минимум 1 vCPU, 1 GB RAM
- Чистый IP (не в спам-листах)
- Доступ к панели Remnawave

### Порядок настройки

1. **Подготовка VPS** → [VPS.md](VPS.md)
2. **Мониторинг (опционально)**:
   - Zabbix → [ZABBIX.md](ZABBIX.md)
   - Prometheus → [PROMETHEUS.md](PROMETHEUS.md)

## 📝 Чеклист новой ноды

```
[ ] Обновление системы
[ ] Установка Docker
[ ] Оптимизация сети (sysctl, limits)
[ ] DNS over TLS
[ ] Unattended Upgrades
[ ] UFW + Fail2ban
[ ] TLS сканирование
[ ] Remnanode (docker-compose)
[ ] Автообновление ноды (cron)
[ ] Inbound + Host в панели Remnawave
[ ] Перезагрузка и проверка
[ ] Мониторинг (опционально)
```

## 🔧 Быстрая установка

Полная настройка VPS одной командой (без Remnanode):

```bash
curl -sSL https://raw.githubusercontent.com/<USER>/remnawave-node-setup/main/scripts/setup.sh | bash
```

## 📚 Полезные ссылки

- [Remnawave Docs](https://docs.rw)
- [Remnawave Telegram](https://t.me/remnawave)
- [Xray-core GitHub](https://github.com/XTLS/Xray-core)
- [RealiTLScanner](https://github.com/XTLS/RealiTLScanner)

## 📄 Лицензия

MIT License

---

**Версия:** 1.0  
**Обновлено:** Январь 2026
