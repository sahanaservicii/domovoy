---
name: xray-openwrt
description: "Xray (VLESS/Reality и др.) на OpenWrt как прозрачный прокси для YouTube/Facebook/Messenger: диагностика, проверка апстримов, безопасное переключение endpoint-ов, health-check и откат."
---

# Xray на OpenWrt (прозрачный доступ к YouTube/Facebook/Messenger)

Используй, когда на роутере уже развёрнут Xray и нужно понять состояние, починить доступ или переключить рабочий канал. Среда офлайн/ограниченная: новые пакеты не ставятся, свежие подписки сами не находятся (нужен URL/конфиг от пользователя или доступный источник).

## 1. Факты перед действиями

```bash
H=/root/workspace/homelab/homelab.py
python3 $H router exec "ps w | grep '[x]ray'; echo ---; ls -la /etc/xray; echo ---; ubus call network.interface.wan status | head -20"
```

Проверить логи и health-check:
```bash
python3 $H router exec "tail -20 /tmp/xray-hc.log; echo ---; tail -30 /tmp/xray-hc.state 2>/dev/null; echo ---; logread -e xray | tail -20"
```

Важно: конфиги (`/etc/xray/config.json`, `test.json` и др.) содержат UUID/ключи/адреса — **не печатать и не копировать в чат/отчёты**. Для анализа использовать локальные проверки на роутере.

## 2. Как устроена типовая схема на этом роутере

- Xray: `/etc/xray/xray run -c /etc/xray/config.json` (обычно под procd/init).
- Inbounds: прозрачный `dokodemo-door` (redir, напр. `:12345`) + локальный `socks` (напр. `127.0.0.1:10808`) для проб.
- Routing: домены/IP YouTube и Facebook/Messenger направляются в прокси-outbound, остальное — `direct`.
- Health-check (`/etc/xray/health.sh`): периодически проверяет доступ через локальный SOCKS до контрольного URL (напр. facebook.com). При успехе включает nft-таблицу/правила редиректа LAN tcp 80/443 на Xray; при серии неудач — снимает правила, чтобы не ломать интернет ("fail-open").
- Пул альтернативных endpoint-ов может лежать в `/etc/xray/test.json` (каждому outbound сопоставлен локальный SOCKS-порт для теста).

## 3. Диагностика (read-only)

Прямой доступ с роутера (ожидаемо может таймаутить при блокировке):
```bash
python3 $H router exec "curl -sS -m 8 -o /dev/null -w '%{http_code}' https://www.facebook.com/; echo; curl -sS -m 8 -o /dev/null -w '%{http_code}' https://www.youtube.com/; echo"
```

Проба через текущий локальный SOCKS Xray:
```bash
python3 $H router exec "curl -sS -m 8 --socks5-hostname 127.0.0.1:10808 -o /dev/null -w '%{http_code}' https://www.facebook.com/"
```

Проверка, включены ли правила прозрачного редиректа:
```bash
python3 $H router exec "nft list table inet xrayredir 2>/dev/null || echo 'rules OFF'"
```

### Тест пула из `test.json` (если есть)

Запускать временный инстанс Xray на тестовом конфиге и проверять его локальные SOCKS-порты небольшими батчами (чтобы не упираться в таймаут SSH). Пример идеи:
```bash
# на роутере: старт тестового xray, проверка диапазона портов, остановка
/etc/xray/xray run -c /etc/xray/test.json >/tmp/xray_test_run.log 2>&1 & sleep 3;
for p in 11001 11002 ...; do
  code=$(curl -sS -m 5 --socks5-hostname 127.0.0.1:$p -o /dev/null -w '%{http_code}' https://www.facebook.com/);
  echo "$p $code";
done; pkill -f 'xray run -c /etc/xray/test.json'
```
Рабочий код (200/301/302/307/308 и т.п.) означает, что соответствующий outbound пригоден для пробы. Тестировать батчами по 8–12 портов.

## 4. Безопасное переключение канала (только с явным подтверждением пользователя)

1. **Бэкап:** `python3 $H backup --with-router` и/или копия `/etc/xray/config.json` с меткой времени в `/root/` на роутере.
2. **Правка:** заменить параметры активного outbound (или маршрутизацию) на проверенно рабочий endpoint. Не менять лишнее. Валидировать JSON (`jq` может отсутствовать — аккуратная правка + проверка запуском).
3. **Применение:** перезапуск сервиса Xray точечно (init/procd), не перезагружая роутер.
4. **Проверка:** `curl` через `127.0.0.1:10808` до facebook.com/youtube.com; убедиться, что health-check снова включает `nft`-правила (`nft list table inet xrayredir`) и доступ есть с клиента LAN.
5. **Откат:** вернуть сохранённый `config.json`, перезапустить Xray, убедиться что `rules OFF`/интернет в норме.

## 5. Если рабочих endpoint-ов нет

- Все имеющиеся каналы могут «умереть» одновременно — это нормально для бесплатных/публичных VLESS.
- Нужен новый рабочий конфиг/ссылка (VLESS/Reality, WireGuard и т.д.) из доверенного источника. Роутер может скачивать по URL, если хост достижим напрямую (например, github.com/raw.githubusercontent.com иногда доступны) — **только по явной просьбе пользователя и с конкретным URL**; публичные списки несут риск MITM/компрометации.
- Альтернатива: VPN на телефоне/ПК и раздача в LAN, или подключение отдельного исходящего WG/OpenVPN клиента при наличии конфига.

## 6. Ограничения

- Не раскрывать UUID, ключи, shortId, адреса серверов.
- Не устанавливать пакеты/бинарники из интернета без отдельного подтверждения и понимания источника.
- Не ломать fail-open логику health-check: при нерабочем апстриме интернет для остального трафика должен оставаться прямым.
