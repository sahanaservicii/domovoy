---
name: home-assistant-local
description: "Локальный REST API Home Assistant: состояния, сервисы, шаблоны, автоматизации, проверка конфигурации, MQTT и Zigbee — без облака и Nabu Casa."
---

# Home Assistant локально

Адрес: `http://192.168.1.127:8123`, токен — `HASS_TOKEN` в
`~/.config/opencode/secrets/homelab.env`.

```bash
H=/root/workspace/homelab/homelab.py
python3 $H ha config
python3 $H ha states --domain light
python3 $H ha state light.hall
python3 $H ha call light turn_on --data '{"entity_id":"light.hall"}'
```

## REST API (все вызовы локальные)

| Метод | Endpoint | Назначение |
|---|---|---|
| GET | `/api/` | проверка и сообщение API |
| GET | `/api/config` | версия, location_name, time_zone |
| GET | `/api/states` | все сущности |
| GET | `/api/states/<entity_id>` | одна сущность |
| POST | `/api/services/<domain>/<service>` | вызов сервиса |
| POST | `/api/template` | Jinja2-шаблон |
| GET | `/api/error_log` | журнал ошибок |
| GET | `/api/services` | доступные сервисы |
| POST | `/api/config/core/check_config` | проверка конфигурации |
| GET/POST/DELETE | `/api/config/automation/config/<id>` | автоматизация |
| GET/POST/DELETE | `/api/config/script/config/<id>` | скрипт |

Произвольный запрос: `python3 $H ha raw GET /api/...`
(для POST — `--data '<json>'`).

## Автоматизации

1. Прочитать текущую:
   ```bash
   python3 $H ha raw GET /api/config/automation/config/<id>
   ```
   Сохранить копию в `/tmp/auto-<id>.json`.
2. Изменить JSON минимально (id, alias, trigger, condition, action, mode).
3. Записать:
   ```bash
   python3 $H ha raw POST /api/config/automation/config/<id> --data @new.json
   ```
   (CLI принимает JSON-строку; при длинном JSON используй Python-обёртку или
   передай строкой.)
4. Проверить: `python3 $H ha raw POST /api/config/core/check_config`.
5. Перезагрузить: `python3 $H ha call automation reload`.

Не пересоздавай `id` без нужды. Не удаляй чужие автоматизации. Перед массовой
правкой — бэкап (`backup --with-ha`) и копии JSON.

## Полезные шаблоны

```bash
python3 $H ha template "{{ states('sensor.temperature') }}"
python3 $H ha template "{{ states.light | selectattr('state','eq','on') | list | count }}"
python3 $H ha template "{{ state_attr('climate.home','current_temperature') }}"
python3 $H ha template "{{ states.automation | selectattr('state','eq','on') | list | count }}"
```

## Диагностика

- Недоступные: `states` в CLI уже показывает `unavailable/unknown`.
- Ошибки интеграций: `ha errors`, `ha raw GET /api/config`.
- MQTT: локальный брокер `192.168.1.127:1883`; проверять через
  `mosquitto_sub` только если установлен, иначе — состояние MQTT-сущностей.
- Zigbee/Z-Wave: смотреть координатор и `unavailable`-сущности, не пересоздавать
  сеть без нужды.
- История/логи: `ha raw GET /api/history/period/<ISO>?filter_entity_id=...`.

## Безопасность

- Только локальный URL. Не включай Nabu Casa и облачные вебхуки.
- Токены не печатать, не класть в отчёты.
- Полный restart HA — с подтверждением пользователя. Сначала `check_config`.
- Не устанавливай HACS/интеграции из интернета: среда офлайн.
