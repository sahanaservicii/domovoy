---
name: android-local
description: "Android TV и телефон без облака: ADB по TCP, пульт, скриншоты, приложения, настройки, локальный мост устройства (уведомления, буфер, файлы)."
---

# Android локально

## Телефон (это устройство) — мост

```bash
A="$OPENCODE_CONFIG_HOME/skills/android-device/android_device.py"
python3 "$A" health
python3 "$A" device-info
python3 "$A" notify --title "Домовой" --message "Готово"
python3 "$A" alarm --delay-seconds 600 --title "Напоминание" --message "Бэкап"
python3 "$A" toast --message "Работаю"
python3 "$A" clipboard-set --text "текст"
python3 "$A" clipboard-get
python3 "$A" open-url --url "http://192.168.1.127:8123"
python3 "$A" file-list --path . --recursive
python3 "$A" file-save --path report.md --destination downloads
```

Мост — локальный loopback, интернет не нужен. Не открывает порты в LAN.

## Android TV — ADB по TCP

В этой среде нет бинарника `adb`, но `homelab.py` реализует протокол ADB на
Python и умеет: подключение, shell, keyevent, ввод текста, скриншот, reboot.

```bash
H=/root/workspace/homelab/homelab.py
python3 $H tv devices
python3 $H tv shell "getprop ro.product.model"
python3 $H tv shell "pm list packages -3"
python3 $H tv key 3
python3 $H tv text "hello"
python3 $H tv screencap /root/workspace/tv.png
```

Другое устройство: `python3 $H adb --target 192.168.1.50 shell "..."`.

### Ключи (keyevent)

| Код | Кнопка | Код | Кнопка |
|---|---|---|---|
| 3 | HOME | 4 | BACK |
| 19/20 | UP/DOWN | 21/22 | LEFT/RIGHT |
| 23 | OK/центр | 66 | ENTER |
| 24/25 | VOL+ / VOL− | 26 | POWER |
| 85 | PLAY/PAUSE | 224 | WAKEUP |
| 126 | MEDIA_PLAY | 127 | MEDIA_PAUSE |

### Полезные shell-команды

```bash
getprop ro.build.version.release
wm size; wm density
dumpsys power | grep -E 'mWakefulness|Display Power'
dumpsys battery
df -h /data
ip -br a
settings get global wifi_on
cmd media_session volume --show
am start -n <package>/<activity>
monkey -p <package> 1
```

## Подключение и авторизация

- На ТВ: «Для разработчиков» → «Отладка по USB»/«Отладка по сети».
- Первое подключение может потребовать подтвердить RSA-ключ на экране ТВ.
  Если `homelab.py` пишет «требуется авторизация» — подтверди ключ на ТВ или
  переподключи отладку по сети. Обходить авторизацию нельзя.
- Порт по умолчанию `5555`. Если менялся — поправь `homelab.json` → `android`.

## Безопасность

- После работ выключить сетевой ADB:
  ```bash
  python3 $H tv shell "setprop persist.adb.tcp.port -1; setprop service.adb.tcp.port -1; setprop ctl.restart adbd"
  ```
- Не пробрасывать 5555 на WAN, не включать ADB в гостевых/чужих сетях.
- Удаление приложений (`pm uninstall`), сброс (`factory`), reboot — только с
  подтверждением.
- APK — только локальные и доверенные; никаких «оптимизаторов» из интернета.
- Скриншоты могут содержать личные данные: сохраняй осознанно, не публикуй.

## Ограничения

- Не все прошивки разрешают `setprop`/`reboot` без root.
- `screencap -p` на старых прошивках может отдавать PNG с CRLF — инструмент
  это исправляет.
- Если ADB-порт закрыт, а физического доступа нет — управление невозможно;
  не обещай чудес.
