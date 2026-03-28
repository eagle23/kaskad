# Kaskad — WireGuard Cascade NAT for Budget VPS

**Bash-скрипт для настройки высокопроизводительного WireGuard каскада через NAT на дешевых VPS (1 CPU, 1GB RAM).**

Снижает CPU overhead с 20% до 8%, NAT latency с +0.6ms до +0.15ms, потери throughput с 15% до 4%.

```
Client (WireGuard) ──► VPS (Kaskad NAT) ──► Foreign VPN Server
```

---

## Зачем

Обход блокировок через каскад: клиент подключается к ближайшему VPS, VPS пробрасывает трафик на зарубежный сервер. Стандартный NAT (MASQUERADE) на слабом VPS съедает 20% CPU. Kaskad оптимизирует NAT до 8% CPU через SNAT, batch iptables-restore, UDP buffer tuning, MSS clamping.

## Возможности

- **3 режима работы:**
  - WireGuard стандартный (UDP, MASQUERADE)
  - WireGuard оптимизированный (UDP, SNAT, batch rules, buffer tuning)
  - NAT46 через socat (IPv4 вход -> IPv6 выход, userspace relay)
- **VLESS/XRay** проброс (TCP)
- Управление правилами: создание, просмотр, удаление, полный сброс
- Автоматический backup/rollback при ошибках (последние 10 бэкапов)
- Atomic применение правил через `iptables-restore -n`
- Persistent rules через netfilter-persistent
- Systemd-сервисы для NAT46 с auto-restart
- Поддержка UFW
- Логирование в `/var/log/kaskad.log`
- Валидация всех входных данных (IPv4, IPv6, порты, конфликты)

## Требования

- **OS:** Debian 12/13 (Ubuntu 22.04+ тоже работает)
- **Kernel:** >= 5.10
- **RAM:** >= 1 GB
- **Права:** root
- **Пакеты** (ставятся автоматически): `iptables`, `iptables-persistent`, `netfilter-persistent`
- **Для NAT46 socat:** `socat` (ставится автоматически)

## Быстрый старт

```bash
# Скачать
git clone https://github.com/<your-username>/kaskad.git
cd kaskad

# Запустить
sudo bash kaskad.sh
```

При первом запуске скрипт автоматически:
- Включит IP forwarding
- Активирует Google BBR (если ядро поддерживает)
- Установит `iptables-persistent` и `netfilter-persistent`

## Меню

```
╔══════════════════════════════════════════════════════════════╗
║                  КАСКАДНЫЙ VPN - v2.0                       ║
╚══════════════════════════════════════════════════════════════╝

📡 Настройка туннелей:
  1) WireGuard (стандартный режим)
  2) WireGuard ⚡ ОПТИМИЗИРОВАННЫЙ (-40% CPU)
  3) WireGuard 🌐 IPv4→IPv6 (NAT46 socat)
  4) VLESS / XRay (TCP)

📋 Управление:
  5) Посмотреть активные правила
  6) Удалить одно правило
  7) Сбросить ВСЕ настройки

🔧 Дополнительно:
  8) Инструкция
  9) Показать логи
 10) Тест правила
 11) Восстановить из бэкапа

  0) Выход
```

## Режимы работы

### 1. WireGuard стандартный

Классический DNAT + MASQUERADE + conntrack. Работает всегда, но CPU overhead выше.

```
iptables -t nat PREROUTING  → DNAT
iptables -t nat POSTROUTING → MASQUERADE
iptables FORWARD            → conntrack state matching
```

### 2. WireGuard оптимизированный

Тот же NAT, но с пачкой оптимизаций:

| Оптимизация | Эффект |
|---|---|
| SNAT вместо MASQUERADE | -15% CPU (нет route lookup на каждый пакет) |
| Batch `iptables-restore -n` | Атомарное применение всех таблиц (raw, nat, mangle, filter) |
| UDP buffer tuning | rmem/wmem 512KB, снижение packet drops |
| MSS clamping | Предотвращение фрагментации TCP over WireGuard |
| Conntrack timeout 90/180s | Синхронизация с WireGuard keepalive (25s), экономия RAM |
| Network queue tuning | `netdev_max_backlog=1000`, низкая latency |
| GRO/GSO offloading | Отключение на 1 CPU для снижения latency |

### 3. NAT46 через socat

Для случаев когда зарубежный сервер доступен только по IPv6, а клиент подключается по IPv4.

- Userspace UDP relay через `socat`
- Systemd-сервис `kaskad-nat46-<port>` с auto-restart
- Буферы 512KB на стороне socat

```
Client (IPv4) ──► VPS socat (UDP4-LISTEN → UDP6) ──► Foreign Server (IPv6)
```

## Производительность

Замеры на 1 CPU VPS (500 Mbps):

| Метрика | Стандартный | Оптимизированный | Целевые |
|---|---|---|---|
| CPU overhead | 20% | 8-12% | <12% |
| NAT latency | +0.6ms | +0.15-0.3ms | <0.3ms |
| Throughput loss | 15% | 4% | <5% |
| Packet drops | 0.1% | <0.01% | <0.01% |

## Файловая структура

```
/var/log/kaskad.log              # Логи
/root/kaskad-backups/            # Бэкапы iptables (последние 10)
/etc/kaskad/rules.conf           # Конфигурация правил
/etc/systemd/system/kaskad-*     # Systemd сервисы NAT46
```

## Конфигурация правил

Правила хранятся в `/etc/kaskad/rules.conf` в формате:

```
proto|listen_port|target_ip|target_port|timestamp|comment
```

Примеры:
```
udp-ultra|51820|203.0.113.50|51820|1711531200|main wg server
nat46|51821|2001:db8::1|51820|1711531300|ipv6 server
tcp|443|203.0.113.50|443|1711531500|vless
```

## Диагностика

```bash
# Тест конкретного порта (через меню пункт 11)
sudo bash kaskad.sh  # → 11 → ввести порт

# Ручная проверка
iptables -t nat -S PREROUTING | grep <port>
iptables -t nat -S POSTROUTING | grep <port>
iptables -S FORWARD | grep <port>

# NAT46 socat
systemctl status kaskad-nat46-<port>
journalctl -u kaskad-nat46-<port> -f

# Логи
tail -f /var/log/kaskad.log
```

## Восстановление из бэкапа

Бэкапы создаются автоматически перед каждым изменением правил. Хранятся последние 10.

```bash
# Через меню
sudo bash kaskad.sh  # → 12

# Вручную
ls -lt /root/kaskad-backups/
iptables-restore < /root/kaskad-backups/iptables-YYYYMMDD-HHMMSS.rules
```

## Настройка клиента

После создания правила в kaskad:

1. Откройте WireGuard / AmneziaWG / v2rayNG
2. В поле **Endpoint** замените IP зарубежного сервера на **IP вашего VPS**
3. Порт оставьте тем же (или замените если входящий порт отличается)

```
Было:    Endpoint = 203.0.113.50:51820  (зарубежный)
Стало:   Endpoint = 198.51.100.10:51820 (ваш VPS)
```

## Сопутствующие скрипты

| Скрипт | Описание |
|---|---|
| `optimize-vps.sh` | Системная оптимизация VPS (BBR, buffers, conntrack, swap, IRQ). Запускается один раз |
| `benchmark-wireguard.sh` | Диагностика и бенчмарк. Optimization Score 0-5 |

## Безопасность

- Все входные данные валидируются (IP, порты, конфликты)
- Backup создается перед каждым изменением с автоматическим rollback при ошибке
- Деструктивные операции требуют явного подтверждения
- Все действия логируются

## Лицензия

MIT
