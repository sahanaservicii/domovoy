# Domovoy — OpenCode Agents & Skills

Публичный репозиторий конфигурации **Домового**: набор агентов и навыков для [OpenCode](https://opencode.ai), ориентированный на офлайн-администрирование домашней инфраструктуры (LAN, роутер, устройства, документация) с упором на безопасность и воспроизводимость.

> Репозиторий содержит **только** определения агентов и навыков. Секреты, инвентарь, бэкапы, состояние и любые приватные данные сюда **не попадают** и не должны коммититься.

## Состав репозитория

| Путь | Назначение |
|---|---|
| `.config/opencode/agents/*.md` | Профили агентов: `domovoy` (primary), `network`, `home`, `android`, `security`, `backup`, `docs` |
| `.config/opencode/skills/**` | Навыки (`SKILL.md`, при необходимости вспомогательные скрипты) |
| `README.md` | Описание, установка, синхронизация |
| `LICENSE` | Лицензия (MIT) |
| `SECURITY.md` | Политика безопасности и работа с секретами |
| `CONTRIBUTING.md` | Правила изменений и вклада |
| `.gitignore` | Защита от случайного коммита секретов/состояния |

## Возможности (кратко)

- **Домовой (`domovoy`)** — оркестратор: сбор фактов, планирование, делегирование профильным подагентам, проверки после изменений.
- **Подагенты:** `network` (OpenWrt/сеть), `home` (Home Assistant), `android` (ADB/устройства), `security` (аудит LAN), `backup` (бэкапы/восстановление), `docs` (инвентарь/runbook/отчёты).
- **Навыки:** `homelab`, `openwrt-admin`, `home-assistant-local`, `android-local`, `android-device`, `ip-cameras`, `sysadmin`, `mikrotik-switch-engineer`, `pdf-reader`, `screenshot-reader`, `self-improve`, `android-apk-build`, `vpn-bypass`, `windows-exe-build`, `xray-openwrt` и др.

> Промпты агентов/навыков могут содержать параметры примера для домашней среды (например, подсеть `192.168.1.0/24`). Перед использованием адаптируйте их под свою инфраструктуру и модель угроз.

## Требования

- Установленный [OpenCode](https://opencode.ai) и доступ к его конфигурации (`~/.config/opencode/`).
- Для профильных задач — соответствующие локальные инструменты/доступы (SSH, ADB, API устройств и т.п.), настраиваемые **локально** вне этого репозитория.
- Git — для синхронизации и обновлений.

## Установка / восстановление

```bash
git clone https://github.com/sahanaservicii/Domovoy-.git
cd Domovoy-

mkdir -p ~/.config/opencode
cp -a .config/opencode/agents ~/.config/opencode/
cp -a .config/opencode/skills ~/.config/opencode/
```

Обновление существующей установки (осознанно перезапишет файлы агентов/навыков в целевых каталогах):

```bash
git pull
rsync -a --delete .config/opencode/agents/ ~/.config/opencode/agents/
rsync -a --delete .config/opencode/skills/ ~/.config/opencode/skills/
```

### Что настроить локально (не в git)

- `~/.config/opencode/opencode.json` — при необходимости: `default_agent`, плагины, MCP и прочие средо-зависимые параметры.
- `~/.config/opencode/secrets/*` (например, `homelab.env`) — токены, пути к ключам и другие секреты.
- Рабочие данные homelab (вне репозитория): `homelab.json`, `INVENTORY.md`, `state/`, `backups/`, `reports/` и т.д.

## Синхронизация изменений (workflow)

```bash
# проверка на секреты/приватные данные перед коммитом
git diff
git status

git add -A
git commit -m "feat: краткое описание изменения"
git push origin main
```

Рекомендации:
- Ветка по умолчанию: `main`.
- Осмысленные сообщения коммитов (например, Conventional Commits: `feat:`, `fix:`, `docs:`, `chore:`).
- Не коммитьте секреты, дампы конфигов устройств, бэкапы и персональные данные — см. [SECURITY.md](SECURITY.md).

## Безопасность

- Никогда не публикуйте токены, пароли, приватные ключи, `.env`, `service.json`, бэкапы и инвентарь с чувствительными данными.
- Перед push выполняйте проверку диффа и поиск по признакам секретов.
- Подробности: [SECURITY.md](SECURITY.md).

## Вклад

Предложения и улучшения приветствуются: исправления промптов, новые навыки, документация. См. [CONTRIBUTING.md](CONTRIBUTING.md).

## Лицензия

[MIT](LICENSE)
