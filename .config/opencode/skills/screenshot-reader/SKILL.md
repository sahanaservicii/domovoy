---
name: screenshot-reader
description: "Чтение и анализ скриншотов/изображений в офлайн-среде: импорт файла, просмотр через read (если поддерживается моделью) и OCR текста через tesseract (rus/eng), без облака."
---

# Чтение скриншотов и изображений

## Возможности среды
- Офлайн. Доступно: `tesseract` (OCR, языки `eng`, `rus`, `osd`) и Python `PIL`/Pillow для работы с изображениями.
- Инструмент `read` поддерживает изображения (визуальный анализ моделью), если модель/среда поддерживает image-ввод. Для текста на скриншотах надёжный программный путь — OCR через `tesseract`.
- Программного OCR через tesseract/pytesseract-обвязки сверх установленного нет, но `tesseract` CLI достаточно.

## 1) Получить скриншот/изображение

### С этого телефона (локальный мост)
```bash
A="$OPENCODE_CONFIG_HOME/skills/android-device/android_device.py"
python3 "$A" file-pick --target-dir inbox --mime "image/*"
python3 "$A" file-list --path inbox
```
Файлы, расшаренные в OpenCode из других приложений, попадают в `inbox/`.

### С Android TV / устройства по ADB
```bash
H=/root/workspace/homelab/homelab.py
python3 $H tv screencap /root/workspace/inbox/tv.png
# другое устройство: python3 $H adb --target <IP> screencap /root/workspace/inbox/device.png
```

### Поиск в workspace
```bash
find /root/workspace -type f \( -iname '*.png' -o -iname '*.jpg' -o -iname '*.jpeg' -o -iname '*.webp' \) 2>/dev/null | head
```

## 2) Извлечь текст (OCR)
```bash
# русский + английский (частый случай интерфейсов)
tesseract /path/image.png stdout -l rus+eng
# только английский / только русский
tesseract /path/image.png stdout -l eng
tesseract /path/image.png stdout -l rus
# сохранить в файл (создаст out.txt)
tesseract /path/image.png /tmp/out -l rus+eng
```
Если качество низкое: попробуй более чёткий/крупный скриншот области; при необходимости предварительно обработай изображение через PIL (контраст/масштаб/обрезка).

## 3) Визуальный анализ моделью
1. Определи точный путь к файлу.
2. Вызови `read` с этим путём — модель увидит изображение (если image-ввод поддерживается).
3. Уточни задачу: выписать видимый текст, найти ошибку/настройку/элемент UI, сравнить экраны.

Используй OCR и визуальный анализ вместе: OCR даёт машинный текст, `read` помогает понять контекст/расположение.

## 4) Результат и приватность
- Сохраняй распознанное в `.txt/.md` в workspace при необходимости.
- Скриншоты могут содержать личные данные, токены, пароли — не публикуй и не копируй в отчёты без необходимости.
- Если `file-pick` недоступен — попроси пользователя положить файл в `inbox/` или указать путь.
