---
name: homelab
description: "Офлайн-инструментарий homelab.py: сеть, Home Assistant, OpenWrt, Android, камеры, бэкапы. Используй для любых задач с домашней инфраструктурой."
---

# Навык: homelab

Единый локальный инструмент: `/root/workspace/homelab/homelab.py`.
Только Python stdlib, SSH и LAN. Интернет не нужен и не используется.

## Перед работой

```bash
cd /root/workspace/homelab
python3 homelab.py doctor          # доступность узлов
python3 homelab.py inventory       # известные устройства
```

Инвентарь: `homelab.json` (IP, роли, заметки, без секретов).
Секреты: `~/.config/opencode/secrets/homelab.env` (`HASS_TOKEN`,
`HOMELAB_SSH_KEY`). Никогда не выводи их содержимое.

## Команды

| Команда | Назначение |
|---|---|
| `doctor` | проверка шлюза, SSH роутера, HA, ADB ТВ, моста телефона |
| `scan [--hosts ...] [--ports ...] [--json]` | скан LAN, результат в `state/scan.json` |
| `inventory [--json]` | известные/незнакомые хосты и порты |
| `ha states\|state\|call\|template\|config\|errors\|services\|raw` | Home Assistant REST |
| `router status\|exec\|uci\|backup` | OpenWrt по SSH |
| `tv\|phone\|adb ...` | Android ADB по TCP |
| `cam <ip>` | HTTP/RTSP-проба камеры/NVR |
| `backup [--with-router] [--with-ha]` | локальные архивы + манифест sha256 |
| `report` | Markdown-отчёт в `/root/workspace/reports/` |
| `notify --message ...` | уведомление на этот телефон (локальный мост) |

## Примеры

```bash
python3 homelab.py scan --json
python3 homelab.py ha states --domain automation
python3 homelab.py ha call light turn_on --data '{"entity_id":"light.hall"}'
python3 homelab.py ha template "{{ states('sensor.temperature') }}"
python3 homelab.py router exec "ubus call system info"
python3 homelab.py tv key 3
python3 homelab.py tv shell "pm list packages -3"
python3 homelab.py backup --with-router --with-ha
```

## Железные правила

- Только `192.168.1.0/24`. Чужие сети не сканируем.
- Разрушительное (reboot, sysupgrade, firewall, удаление) — с подтверждением.
- Перед изменениями — бэкап; после — проверка.
- Не выводить секреты в чат/файлы/отчёты.
- Не выдумывать IP/сущности: проверять `inventory`, `scan`, `ha states`.
- Если инструмент вернул `ОШИБКА:` — прочитай её, не повторяй вслепую.
- Интернета нет: не предлагай `apt install`, `pip install`, `opkg update`,
  облачные API. Используй stdlib и уже установленное.

## Ограничения среды

Home Assistant сейчас **отложен** (`home_assistant.enabled=false` в
`homelab.json`): `doctor` его пропускает, настраивать не нужно до отдельной
команды пользователя. Сеть/WireGuard/роутер важнее.

Этот телефон (qroot/Ubuntu) может быть не в домашней сети. Если `doctor`
показывает недоступность — это не поломка инструмента, а отсутствие Wi-Fi/LAN
или выключенный WireGuard. Смотри строку «Локальный адрес для 192.168.1.1»:
`10.x` — это мобильная сеть/туннель, а не домашний LAN.
