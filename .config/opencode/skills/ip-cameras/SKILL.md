---
name: ip-cameras
description: "Локальные IP-камеры и NVR: XM/XMEye, ONVIF, RTSP, порты, изоляция, проверка рисков. Без облака, P2P и подбора паролей."
---

# IP-камеры и NVR локально

В сети: XM/XMEye-камеры (`.10`, `.93`, `.135`, `.206`), NVR Topsvision
(`.190`). Типовые порты: 80 (веб), 554 (RTSP), 34567 (XM-протокол), иногда
8000/8080.

## Проба

```bash
H=/root/workspace/homelab/homelab.py
python3 $H cam 192.168.1.10
```

Показывает открытые порты, HTTP-статус/Server/title и RTSP-ответ.
Дополнительно:

```bash
python3 - <<'PY'
import socket
for ip in ("192.168.1.10","192.168.1.93","192.168.1.135","192.168.1.190","192.168.1.206"):
    for p in (80,554,34567,8000,8080):
        s=socket.socket(); s.settimeout(0.7)
        print(ip,p,"open" if s.connect_ex((ip,p))==0 else "-"); s.close()
PY
```

## RTSP

Типовые пути (зависят от прошивки; проверять `OPTIONS`/`DESCRIBE`):

- `rtsp://user:pass@IP:554/`
- `rtsp://user:pass@IP:554/cam/realmonitor?channel=1&subtype=0` (Dahua/XM)
- `rtsp://user:pass@IP:554/user=admin_password=_channel=1_stream=0.sdp?real_stream`
- `rtsp://user:pass@IP:554/Streaming/Channels/101` (Hikvision)

Пароли не подбираем. Если неизвестны — пользователь вводит сам.

## ONVIF

Порт обычно 80 (иногда 8000/8080). Проверка — SOAP `GetDeviceInformation`
или `GetCapabilities` на `http://IP/onvif/device_service`. Без логина многие
камеры отдают только `GetSystemDateAndTime`. Не пытайся обойти авторизацию.

## Риски (по фактам)

- HTTP без TLS, дефолтные/старые прошивки XM — частая цель Mirai.
- P2P/облако камеры = исходящее соединение наружу; лучше выключить.
- UPnP на роутере может сам открыть порт камеры в интернет.
- Один пароль на камеры/NVR/роутер.
- Камеры в одной сети с management/HA без VLAN.

## Что рекомендовать

1. Отдельный VLAN/гостевая сеть для камер и NVR; доступ только с HA/NVR.
2. Блокировка выхода в интернет для камер (firewall), если не нужен удалённый
   просмотр. Удалённый доступ — через WireGuard, не проброс.
3. Смена дефолтных паролей, отключение P2P/облака/UPnP, обновление прошивки
   вручную из доверенного источника.
4. Учётные записи только для просмотра, где возможно.
5. Проверка после изменений: `cam`, `scan`, `fw4 print`.

Не меняй пароли и сети камер без запроса. Не выводи учётные данные в чат.
