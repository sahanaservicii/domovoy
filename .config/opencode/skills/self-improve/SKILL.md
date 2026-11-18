---
name: self-improve
description: "Цикл самоулучшения Домового в офлайн-среде: одна маленькая безопасная улучшалка за запуск, проверка, журнал и бэкап после успеха."
---

# Самоулучшение (Домовой)

Используй этот навык для периодических или ручных сеансов улучшения в `/root/workspace`.

## Источники
- Runbook: `/root/workspace/homelab/self_improve/RUNBOOK.md`
- Идеи: `/root/workspace/homelab/self_improve/IDEAS.md`
- Журнал: `/root/workspace/homelab/self_improve/CHANGELOG.md`
- Состояние: `/root/workspace/homelab/self_improve/state.json`
- Помощник: `python3 /root/workspace/homelab/self_improve/cycle.py status|resource-check|log|backup-homelab`

## Процедура (один запуск = одно улучшение)
1. **Факты:** `python3 /root/workspace/homelab/homelab.py doctor` (при необходимости `inventory`). Проверить ресурсы: `python3 /root/workspace/homelab/self_improve/cycle.py resource-check` — при `ok=false` завершить со статусом skipped.
2. **Выбор:** одна идея из `IDEAS.md` в разрешённой области (`homelab/`, `agentos/`, навыки, docs/reports). Никаких изменений инфраструктуры (роутер/HA/firewall/устройства).
3. **Изменение:** минимальное, обратимое.
4. **Проверка:** `python3 -m py_compile <изменённые.py>`, `python3 -m json.tool <файл.json>`, dry-run/selftest затронутого компонента (если применимо). Неуспех → откат изменения, `cycle.py log --status failure ...`, завершить.
5. **Успех:** обновить `CHANGELOG.md` и `state.json` (или `cycle.py log --status success --summary "..."`), затем бэкап:
   - `python3 /root/workspace/homelab/homelab.py backup`
   - по возможности `python3 "$OPENCODE_CONFIG_HOME/skills/android-device/android_device.py" file-backup`
6. **Уведомление:** `python3 /root/workspace/homelab/homelab.py notify --message "Самоулучшение: <кратко результат>"` (если доступно).

## Ограничения
- Офлайн: без `apt/pip/opkg install`, без облаков.
- Без секретов в логах/чатах. Без разрушительных действий и без правок вне разрешённой области.
- Интервал щадящий (по умолчанию раз в 4 часа), не «каждую минуту»; тяжёлые сканы не повторять без нужды.
