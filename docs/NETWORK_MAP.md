# NETWORK MAP — Карта сети и маршрутов
## Валера Кондрашов | Обновлено: 09.04.2026

---

## ПРИНЦИП ЧТЕНИЯ ЭТОГО ДОКУМЕНТА

Здесь описано КАК устройства соединены между собой.
Тип соединения влияет на то, что можно делать и как быстро.

Типы соединений:
- **ПРЯМОЙ** — устройства видят друг друга напрямую
- **WireGuard** — зашифрованный VPN туннель, работает как прямой
- **SSH туннель** — обратный туннель через SSH (медленнее, зависит от Mac)
- **SIP** — голосовой протокол (GoIP → FreeSWITCH)
- **ADB WiFi** — управление Android через WiFi

---

## КАРТА СЕТЕЙ

```
ИНТЕРНЕТ (внешняя сеть)
    │
    ├─── VPS .198 (81.17.140.198)  ← ГЛАВНЫЙ УЗЕЛ
    │         │
    │         ├─── VPS .194 (81.17.140.194)   [ПРЯМОЙ, одна подсеть Aeza]
    │         ├─── VPS .193 (81.17.140.193)   [ПРЯМОЙ, одна подсеть Aeza]
    │         ├─── VPS .195 (81.17.140.195)   [ПРЯМОЙ, одна подсеть Aeza]
    │         │
    │         ├─── WireGuard сервер (10.10.0.1/24)
    │         │         ├─── Mac        (10.10.0.2)  [WireGuard]
    │         │         └─── NAS        (10.10.0.3)  [WireGuard]
    │         │
    │         └─── SSH туннель ←── Mac (порт 2222)
    │                   └── autossh, восстанавливается автоматически
    │
    └─── Домашняя сеть Порту (192.168.1.0/24 + 192.168.88.0/24)
              │
              ├─── MikroTik (192.168.1.108)  ← РОУТЕР домашней сети
              │         ├── Mac  (192.168.1.155)
              │         ├── NAS  (192.168.1.185)
              │         └── GoIP устройства через другой порт
              │
              ├─── Mac (192.168.1.155 / WireGuard 10.10.0.2)
              │         └── ADB WiFi → Samsung (192.168.1.168:5555)
              │
              ├─── NAS (192.168.1.185 / WireGuard 10.10.0.3)
              │
              ├─── GoIP8 (192.168.88.253)   ← отдельная подсеть 88.x
              │         └── SIP → VPS .198 :5060 (FreeSWITCH)
              │
              └─── GoIP4 (192.168.88.252)   ← отдельная подсеть 88.x
                        └── SIP → VPS .198 :8090 (unified voice agent)
```

---

## ТАБЛИЦА ВСЕХ МАРШРУТОВ

| Откуда | Куда | Тип | Команда / Протокол | Примечание |
|--------|------|-----|-------------------|------------|
| VPS .198 | VPS .194 | ПРЯМОЙ | `ssh administrator@81.17.140.194` | Пароль в vault.env |
| VPS .198 | VPS .193 | ПРЯМОЙ | `ssh administrator@81.17.140.193` | Резерв |
| VPS .198 | Mac | SSH туннель | `ssh -p 2222 -i ~/.ssh/vps_to_mac valera@localhost` | autossh на Mac |
| VPS .198 | NAS | WireGuard | `ssh nas` или `ssh -i ~/.ssh/vps_to_nas ubnt@10.10.0.3` | Всегда активен |
| VPS .198 | Mac (резерв) | WireGuard+NAS | `ssh mac` (ProxyJump через NAS) | Если туннель упал |
| NAS | Mac | ПРЯМОЙ (LAN) | `ssh valera@192.168.1.155` + ключ vps_to_mac | Через локальную сеть |
| Mac | NAS | ПРЯМОЙ (LAN) | `ssh ubnt@192.168.1.185` | |
| Mac | GoIP8 | ПРЯМОЙ (LAN) | `http://192.168.88.253` | Другая подсеть, нужен маршрут |
| Mac | GoIP4 | ПРЯМОЙ (LAN) | `http://192.168.88.252` | Другая подсеть, нужен маршрут |
| Mac | ADB телефон | ADB WiFi | `adb -s 192.168.1.168:5555` | Всегда подключён |
| GoIP8 | FreeSWITCH | SIP | UDP 5060 → VPS .198 | Регистрация SIP |
| GoIP4 | Voice Agent | SIP/ESL | UDP → VPS .198 :8090 | Дебильник |
| Любой | Shell MCP | HTTPS | `https://shell.smartcare.house` | FastMCP 3.1.1, инструмент 14 |

---

## ОГРАНИЧЕНИЯ МАРШРУТОВ

| Проблема | Причина | Решение |
|---------|---------|---------|
| VPS не видит Mac напрямую | Mac за NAT (домашний роутер) | Использовать SSH туннель (порт 2222) или WireGuard |
| VPS не видит GoIP напрямую | GoIP в подсети 192.168.88.x за MikroTik | Только через Mac туннель или WireGuard |
| VPS не видит NAS напрямую по LAN | NAS в домашней сети | WireGuard 10.10.0.3 — всегда работает |
| Mac не видит GoIP (иногда) | MikroTik не маршрутизирует 192.168.1.x ↔ 192.168.88.x | Нерешённая проблема MikroTik routing |

---

## WATCHDOG И ВОССТАНОВЛЕНИЕ

| Соединение | Watchdog | Что делает при падении |
|-----------|---------|----------------------|
| Mac SSH туннель (2222) | `tunnel-watchdog.service` (каждые 2 мин) | Убивает CLOSE-WAIT, перезапускает LaunchAgent через NAS |
| NAS WireGuard | Нет (очень стабилен) | Поднимается сам |
| GoIP SIP | FreeSWITCH registration check | GoIP сам переподключается |

---

## SSH CONFIG (алиасы на VPS .198)

Файл: `/home/administrator/.ssh/config`

```
Host nas        → ubnt@10.10.0.3 (WireGuard, ключ vps_to_nas)
Host mac        → valera@192.168.1.155 (ProxyJump через nas, ключ vps_to_mac)
Host mac-direct → valera@localhost:2222 (прямой туннель, ключ vps_to_mac)
```

---

## SSH КЛЮЧИ

| Ключ | Где лежит | Для чего |
|------|----------|---------|
| vps_to_mac | `/home/administrator/.ssh/vps_to_mac` | VPS → Mac (через туннель) |
| vps_to_nas | `/home/administrator/.ssh/vps_to_nas` | VPS → NAS (WireGuard) |
| nas_tunnel | `/home/administrator/.ssh/nas_tunnel` | NAS → VPS (резерв) |
| valera_vps_tunnel | `/Users/valera/.ssh/valera_vps_tunnel` (Mac) | Mac → VPS (autossh туннель) |

---

## WIREGUARD СЕТЬ (10.10.0.0/24)

| Устройство | WireGuard IP | Интерфейс | Статус |
|-----------|-------------|-----------|--------|
| VPS .198 | 10.10.0.1 | wg0 | ✅ Сервер, всегда активен |
| Mac | 10.10.0.2 | wg0 | ✅ Клиент |
| NAS | 10.10.0.3 | wg0 | ✅ Клиент, всегда активен |

---

## ПОРТЫ И СЕРВИСЫ (VPS .198)

| Порт | Протокол | Сервис | Доступен откуда |
|------|---------|--------|----------------|
| 22 | TCP | SSH | Везде |
| 2222 | TCP | SSH туннель ← Mac | Только localhost (от autossh) |
| 3000 | TCP | Shell MCP (FastMCP) | Везде (через cloudflared) |
| 3007 | TCP | Exec bridge | Localhost |
| 5060 | UDP | FreeSWITCH SIP | GoIP8 |
| 8021 | TCP | FreeSWITCH ESL | Localhost |
| 8090 | TCP | Unified Voice Agent | Localhost (FreeSWITCH) |
| 8095 | TCP | Debilnik API | Везде |
| 8096 | TCP | Vaultwarden | Localhost (nginx proxy) |
| 51820 | UDP | WireGuard | Везде |

---

## БЫСТРЫЙ ЧЕКЛИСТ — ЧТО ИСПОЛЬЗОВАТЬ

**Нужен VPS .194?**
→ `sshpass -p 'ezy7u2ffiFNR' ssh administrator@81.17.140.194`

**Нужен Mac?**
→ Сначала: `ssh -p 2222 -i ~/.ssh/vps_to_mac valera@localhost`
→ Если не работает: `ssh mac` (через NAS)

**Нужен NAS?**
→ `ssh nas`

**Нужен GoIP веб-интерфейс?**
→ Только через Mac: открыть браузер на Mac → http://192.168.88.253

**Нужен ADB телефон?**
→ Через Mac: `adb -s 192.168.1.168:5555 <команда>`

**Нужен Shell MCP?**
→ Инструмент 14 в Claude — он уже подключён

---

*Этот файл — эталон. При изменениях в сети — обновлять первым.*
*Копия: /home/administrator/shared/NETWORK_MAP.md*
*GitHub: goippro-main/goippro-tma/docs/NETWORK_MAP.md*
