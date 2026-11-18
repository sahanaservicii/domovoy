---
name: pdf-reader
description: "Чтение PDF в офлайн-среде: модель не принимает PDF-ввод через read, но доступны poppler (pdftotext/pdftoppm), tesseract (OCR) и Python-библиотеки (PyMuPDF/fitz, pypdf, pdfminer). Импорт, извлечение текста, рендер страниц и OCR сканов."
---

# Чтение PDF-файлов (оффлайн)

## Важное ограничение
- **Инструмент `read` не принимает PDF-ввод** для этой модели (`model does not support pdf input`). Не пытайся «открыть» PDF через `read` — используй локальные инструменты ниже.
- Среда офлайн: новые пакеты из интернета не ставятся без отдельного разрешения. Сейчас установлены: `poppler-utils` (`pdftotext`, `pdftoppm`, `pdfinfo`, `pdfimages`), `tesseract-ocr` (+ `eng`, `rus`, `osd`), Python: `fitz` (PyMuPDF), `pypdf`, `pdfminer.six` (`pdfminer`), `PIL`.

## 1) Получить PDF в workspace
```bash
A="$OPENCODE_CONFIG_HOME/skills/android-device/android_device.py"
python3 "$A" file-pick --target-dir inbox --mime "application/pdf"
python3 "$A" file-list --path inbox
```
Поиск уже имеющихся: `find /root/workspace -type f -iname '*.pdf' 2>/dev/null | head`

## 2) Текстовый PDF — извлечь текст (предпочтительно)
```bash
pdfinfo /path/file.pdf                         # метаданные, число страниц
pdftotext -layout /path/file.pdf -             # текст в stdout (сохраняет раскладку)
pdftotext /path/file.pdf /tmp/out.txt          # в файл

# PyMuPDF (хорошо для сложных/быстрых случаев)
python3 - <<'PY'
import fitz, sys
doc = fitz.open(sys.argv[1])
for i,p in enumerate(doc,1):
    t = p.get_text().strip()
    if t: print(f"\n--- page {i} ---\n{t}")
PY
 /path/file.pdf

# Альтернативы: pypdf / pdfminer / простой stdlib-фолбэк
python3 -c "from pypdf import PdfReader; r=PdfReader('/path/file.pdf'); print('\n'.join(p.extract_text() or '' for p in r.pages))"
python3 /root/workspace/homelab/pdf_text.py /path/file.pdf --meta
```
Выбирай `pdftotext` или `fitz` в первую очередь; `pdf_text.py` — только как лёгкий фолбэк.

## 3) Скан/изображения в PDF — рендер + OCR
```bash
mkdir -p /tmp/pdfpages
# Рендер страниц в PNG (200-300 dpi обычно достаточно)
pdftoppm -png -r 200 /path/file.pdf /tmp/pdfpages/page

# OCR одной/всех страниц (rus+eng при смешанном тексте)
tesseract /tmp/pdfpages/page-1.png stdout -l rus+eng
for f in /tmp/pdfpages/page-*.png; do echo "--- $f ---"; tesseract "$f" stdout -l rus+eng; done
```
Через PyMuPDF (рендер):
```bash
python3 - <<'PY'
import fitz, sys
doc = fitz.open(sys.argv[1])
for i,p in enumerate(doc):
    pix = p.get_pixmap(dpi=200)
    pix.save(f"/tmp/pdfpages/page-{i+1}.png")
print("saved", len(doc))
PY
 /path/file.pdf
```
Встроенные изображения: `pdfimages -list /path/file.pdf` / `pdfimages -png /path/file.pdf /tmp/img`.

## 4) Результат
- Сохраняй извлечённое в workspace (`inbox/<name>.txt`, markdown) для дальнейшего анализа.
- Для больших PDF обрабатывай нужные страницы/диапазоны (`pdftotext -f N -l M`, рендер конкретных страниц).
- OCR не идеален: проверяй критичные числа/имена по оригиналу (страницы-изображения).

## Конфиденциальность
PDF могут содержать личные данные — не публикуй содержимое без необходимости.
