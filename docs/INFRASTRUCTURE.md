# Infrastructure Documentation
## SmartCare / GoIPpro — Valera Kondrashov

**Last updated:** 2026-04-08
**Owner:** valera@gsm-travel.com

---

## Architecture Overview

```
Internet
    │
    ├── VPS .198 (81.17.140.198) — Main orchestrator
    │       ├── Shell MCP (port 3000)
    │       ├── Unified Voice Agent (port 8090)
    │       ├── FreeSWITCH (Docker)
    │       ├── WireGuard (10.10.0.1)
    │       └── Watchdogs (tunnel, site, goip)
    │
    ├── VPS .194 (81.17.140.194) — GoIPpro Backend
    │       ├── PostgreSQL (goippro_dev, goippro_prod)
    │       ├── ASTPP Billing
    │       └── GoIPpro API
    │
    └── Home Network (Porto, Portugal)
            ├── Mac (192.168.1.155 / WireGuard 10.10.0.2)
            │       ├── SSH tunnel → VPS .198 port 2222 (autossh)
            │       ├── Eigent AI Agent (port 5001)
            │       └── ADB Phone Farm
            │
            ├── NAS (192.168.1.185 / WireGuard 10.10.0.3)
            │       ├── OpenClaw
            │       ├── Backups storage (/home/ubnt/backups/)
            │       └── SSH key access from VPS
            │
            ├── MikroTik router (192.168.1.108)
            ├── GoIP8 (192.168.88.253) — 8 ports, GoIPpro Business
            └── GoIP4 (192.168.88.252) — 4 ports, Experiments + Debilnik
```

---

## Access Methods

| Device | Primary | Fallback |
|--------|---------|---------|
| Mac | `ssh -p 2222 -i ~/.ssh/vps_to_mac valera@localhost` | `ssh mac` (via NAS jump) |
| NAS | `ssh -i ~/.ssh/vps_to_nas ubnt@10.10.0.3` | Direct LAN |
| VPS .194 | `sshpass -p 'ezy7u2ffiFNR' ssh administrator@81.17.140.194` | — |

SSH config aliases on VPS .198: `~/.ssh/config`
- `ssh nas` → NAS via WireGuard
- `ssh mac` → Mac via NAS jump proxy

---

## Tunnel Architecture

Mac → VPS tunnel (autossh, port 2222):
- LaunchAgent: `~/Library/LaunchAgents/com.valera.vps-tunnel.plist`
- Binary: `/opt/homebrew/bin/autossh`
- Key: `/Users/valera/.ssh/valera_vps_tunnel`
- Options: ServerAliveInterval=30, ServerAliveCountMax=3, ExitOnForwardFailure=yes

NAS → VPS (WireGuard, always on):
- VPS key: `/home/administrator/.ssh/vps_to_nas`
- NAS IP on WireGuard: `10.10.0.3`

Watchdog (systemd service `tunnel-watchdog`):
- Checks every 2 min
- Auto-restores via NAS→Mac if tunnel drops
- Logs: `/home/administrator/tunnel_watchdog.log`

---

## Backup System

**Schedule:** Daily at 03:00 Lisbon time (systemd timer `backup-nas`)
**Retention:** 7 days
**Script:** `/home/administrator/backup_to_nas.sh`
**Logs:** `/home/administrator/logs/backup.log`

### What's backed up:

| Item | Source | Destination |
|------|--------|-------------|
| vault.env (VPS .198) | `/home/administrator/vault.env` | NAS `/home/ubnt/backups/vps198/DATE/` |
| vault.env (VPS .194) | `administrator@81.17.140.194:vault.env` | NAS same dir |
| PostgreSQL goippro_prod | VPS .194 pg_dump | NAS same dir |
| /shared/ MD files | `/home/administrator/shared/` | NAS same dir (tar.gz) |
| nginx configs | `/etc/nginx/` | NAS same dir (tar.gz) |
| PM2 process list | `~/.pm2/dump.pm2` | NAS same dir |
| FreeSWITCH dialplan | `~/newfreeswith/conf/` | NAS same dir (tar.gz) |

### Restore procedure:
```bash
# 1. Получить список бэкапов
ssh -i ~/.ssh/vps_to_nas ubnt@10.10.0.3 "ls /home/ubnt/backups/vps198/"

# 2. Скачать нужный бэкап
scp -i ~/.ssh/vps_to_nas ubnt@10.10.0.3:/home/ubnt/backups/vps198/DATE/vault_198_DATE.env ./

# 3. Восстановить PostgreSQL
psql -U postgres goippro_prod < goippro_prod_DATE.sql

# 4. Восстановить конфиги
tar -xzf nginx_DATE.tar.gz -C /
tar -xzf shared_DATE.tar.gz -C /
```

---

## Key Services (VPS .198)

| Service | Port | PM2 name | Description |
|---------|------|----------|-------------|
| Unified Voice Agent | 8090 | voice-agent | alarm + support modes |
| Debilnik API | 8095 | debilnik-api | Android app backend |
| Debilnik Call Server | 8091 | debilnik-callserver | ESL call handler |
| Shell MCP | 3000/3007 | — | Claude tool access |
| FreeSWITCH | Docker | — | SIP/ESL |

---

## SSL Certificates

| Domain | Expires |
|--------|---------|
| smartcare.house | 2026-06-11 |
| goippro.com | 2026-06-01 |

Auto-renewal: Let's Encrypt (check with `certbot renew --dry-run`)

---

## WireGuard Network

| Device | WireGuard IP |
|--------|-------------|
| VPS .198 | 10.10.0.1 |
| Mac | 10.10.0.2 |
| NAS | 10.10.0.3 |
| MikroTik | — |

Config on VPS: `wg show wg0`

---

## Recovery Checklist (если всё упало)

1. **VPS .198 недоступен** → провайдер Aeza, панель управления
2. **Mac туннель упал** → автовосстановление через NAS за 30 сек
3. **NAS недоступен** → проверить WireGuard на VPS (`wg show`)
4. **Потеря данных** → бэкапы на NAS `/home/ubnt/backups/`
5. **Потеря всех серверов** → vault.env в бэкапах, код на GitHub goippro-main

---

## GitHub Repositories

Organization: [goippro-main](https://github.com/goippro-main)

| Repo | Purpose |
|------|---------|
| goippro-tma | GoIPpro Telegram Mini App |
| infrastructure | This doc + backup inventory |

---

*Документ автоматически обновляется. Хранить актуальную версию здесь и на VPS в `/home/administrator/shared/INFRASTRUCTURE.md`*

---

## FreeSWITCH Architecture

**Updated:** 2026-04-09

### PRIMARY — VPS .198 (81.17.140.198)
- Docker container: `freeswitch`
- SIP port: **5060**
- ESL port: **8021**, password: `ClueCon`
- Dialplan: `/home/administrator/newfreeswith/fs_config/default.xml`
- Extensions:
  - `goip8_business` → `voice_mode=support` → socket `127.0.0.1:8090`
  - `goip4_experiments` → `voice_mode=alarm` → socket `127.0.0.1:8090`

### BACKUP — VPS .194 (81.17.140.194)
- Docker container: `freeswitch-backup`
- SIP port: **5080**
- ESL port: **8022**, password: `ClueCon`
- Compose: `/home/administrator/freeswitch-backup/docker-compose.yml`
- Dialplan: `/home/administrator/freeswitch-backup/conf/default.xml`
- Activates automatically when GoIP loses connection to PRIMARY

### Failover Scheme
```
GoIP8 (192.168.88.253)
  ├── SIP Server 1: 81.17.140.198:5060  ← PRIMARY
  └── SIP Server 2: 81.17.140.194:5080  ← BACKUP (failover timeout: 30 sec)
```
GoIP8 настраивается в веб-интерфейсе (192.168.88.253) через Mac-туннель.

### Voice Agent (unified)
- Port: **8090**
- PM2: `voice-agent`
- Modes: `alarm` (Grok-3-fast) / `support` (Claude Haiku)
- Dispatch: через channel variable `voice_mode=alarm|support`

### Management
```bash
# PRIMARY (VPS .198) — через Shell MCP tool 14
docker exec freeswitch fs_cli -x "reloadxml"
docker exec freeswitch fs_cli -x "status"

# BACKUP (VPS .194)
sshpass -p 'ezy7u2ffiFNR' ssh administrator@81.17.140.194
sudo docker exec freeswitch-backup fs_cli -H 127.0.0.1 -P 8022 -p ClueCon -x "status"
sudo docker logs freeswitch-backup --tail 50
```
