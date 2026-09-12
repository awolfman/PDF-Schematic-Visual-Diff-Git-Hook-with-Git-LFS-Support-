# PDF Schematic Visual Diff Git Hook (with Git LFS Support)

Автоматический Git-хук (`pre-commit`) для визуального контроля изменений в PDF-схемах и чертежах, разработанный специально для репозиториев электронной разработки (CAD/EDA) с поддержкой **Git LFS**.

> ⚡ **Когда отрабатывает хук:** Скрипт автоматически запускается **в момент выполнения команды `git commit`**. Он перехватывает проиндексированные (staged) PDF-файлы, на лету извлекает и разворачивает данные из хранилища Git LFS, выполняет попиксельный анализ страниц текущей версии против предыдущего коммита (`HEAD`), формирует наглядный графический отчет `[имя]_diff.pdf` и **автоматически добавляет его в этот же текущий коммит**.

---

## ✨ Логика цветовой маркировки чертежей

Алгоритм работает на низком уровне сравнения яркости пикселей (параметр `THRESHOLD="0.01"`), что позволяет безошибочно вычислять изменения геометрии CAD-схем:

- **Удаление компонента**: Все исчезнувшие УГО выделяются **синим цветом**.
- **Добавление компонента или Изменение RefDes**: Выделяются **красным цветом**.
- **Перемещение компонента**: Старое положение компонента на схеме окрашивается в **синий цвет**, новое положение — в **красный цвет**.

---

## 🚀 Особенности реализации

- **Полная интеграция с Git LFS**: Автоматически распознает текстовые указатели LFS (LFS pointers) в истории коммитов и безопасно выполняет процедуру `smudge` в изолированном пространстве.
- **Устойчивость к антиалиасингу**: Низкоуровневые вычисления `-fx` в связке с морфологическим расширением (`Dilate Disk:1`) улавливают микросдвиги векторных линий толщиной в 1 пиксель, предотвращая ложные пропуски изменений.
- **Высокая скорость**: Обработка страниц распараллелена на все доступные ядра CPU через `GNU Parallel`. Внутренний параллелизм утилит при этом ограничен для защиты от утечек памяти (RAM Spikes).
- **Кроссплатформенность**: Работает на OpenSUSE, Debian/Ubuntu/Mint, Fedora/RHEL, Arch Linux. Поддерживает как ImageMagick 7 (`magick`), так и ImageMagick 6 (`convert`) — см. раздел про совместимость ниже.
- **Полностью локально**: Никаких сетевых сервисов, облаков и подписок. Всё работает на вашей машине.

---

## 🛠 Зависимости и пакеты

Перед установкой хука убедитесь, что в вашей системе установлены следующие пакеты:

| Команда в скрипте | Назначение утилиты | OpenSUSE | Debian/Ubuntu/Mint | Fedora/RHEL |
| :--- | :--- | :--- | :--- | :--- |
| `git-lfs` | Извлечение тяжелых бинарных PDF из хранилища LFS | `git-lfs` | `git-lfs` | `git-lfs` |
| `magick` | Низкоуровневая попиксельная FX-математика | `ImageMagick` | `imagemagick` | `ImageMagick` |
| `pdftoppm` | Рендеринг страниц PDF в растровые PNG | `poppler-tools` | `poppler-utils` | `poppler-utils` |
| `parallel` | Распределение задач по ядрам процессора | `parallel` | `parallel` | `parallel` |
| `img2pdf` | Сборка финального PDF-отчета без пережатия | `python3-img2pdf` | `img2pdf` | `img2pdf` |

### Установка по дистрибутивам

**OpenSUSE Tumbleweed / Leap 15.4+:**
```bash
sudo zypper install git git-lfs ImageMagick poppler-tools parallel python3-img2pdf
```

**Debian 11+ / Ubuntu 22.04+ / Linux Mint 21+:**
```bash
sudo apt update
sudo apt install git git-lfs imagemagick poppler-utils parallel img2pdf
```

**Fedora 38+ / RHEL 9+ / AlmaLinux 9+:**
```bash
sudo dnf install git git-lfs ImageMagick poppler-utils parallel img2pdf
```

**Arch Linux / Manjaro:**
```bash
sudo pacman -S git git-lfs imagemagick poppler parallel img2pdf
```

### ✅ Проверка установки

```bash
for tool in git magick pdftoppm img2pdf parallel; do
    command -v "$tool" >/dev/null 2>&1 && echo "OK: $tool" || echo "MISSING: $tool"
done
```

Если какая-то утилита отсутствует — вернитесь к блоку установки и поставьте её.

---

## ⚠️ Важно: `magick` (IM 7) vs `convert` (IM 6)

Скрипт использует команду **`magick`** — это интерфейс **ImageMagick 7**. В большинстве дистрибутивов через менеджер пакетов ставится **ImageMagick 6**, где та же функциональность доступна через команду **`convert`**.

### Как узнать версию

```bash
magick --version    # IM 7 — команда magick есть
convert --version   # IM 6 — команда magick может отсутствовать
```

### Если у вас ImageMagick 6

**Вариант A. Установить ImageMagick 7 из репозитория** (если доступно):

```bash
# Debian/Ubuntu — из PPA
sudo add-apt-repository ppa:imagemagick/ppa
sudo apt update
sudo apt install imagemagick
```

**Вариант B. Установить через snap** (если snap установлен):

```bash
sudo snap install imagemagick
```

**Вариант C. Подменить `magick` на `convert` в скрипте** (самый быстрый):

```bash
sed -i 's/\bmagick\b/convert/g' .git/hooks/pre-commit
grep -c "convert" .git/hooks/pre-commit   # проверка
```

Функционально команды почти идентичны — синтаксис опций совпадает, различия касаются редких случаев (SVG, работа с цветовыми профилями). Для нашей задачи — рендеринг PDF, работа с масками, композитинг — подмена безопасна.

---

## 📦 Установка `img2pdf` через Python

Если пакет `img2pdf` отсутствует в репозитории вашего дистрибутива (актуально для старых версий или нестандартных сборок), установите через `pip`:

### Способ 1. Простая установка в пользовательский каталог

```bash
pip install --user img2pdf
```

### Способ 2. Виртуальное окружение (рекомендуется)

```bash
python3 -m venv ~/.venv-img2pdf
~/.venv-img2pdf/bin/pip install img2pdf
```

Затем в скрипте `.git/hooks/pre-commit` замените вызов `img2pdf` на полный путь:

```bash
sed -i 's|^\(.*\)img2pdf |\1~/.venv-img2pdf/bin/img2pdf |g' .git/hooks/pre-commit
```

### Способ 3. `pipx` (изолированная установка)

```bash
pipx install img2pdf
```

---

## 💻 Настройка репозитория и установка хука

### Шаг 1. Инициализация Git LFS для PDF и исходников CAD

В зависимости от используемой среды проектирования (EDA), бинарные файлы электрических схем, а также выходные PDF-документы необходимо перевести под контроль Git LFS. Выполните в корне репозитория команды для вашей CAD-системы:

* **Для Cadence Allegro / OrCAD Capture:**
  ```bash
  git lfs install
  git lfs track "*.pdf" "*.dsn"
  git add .gitattributes
  ```

* **Для Mentor Graphics PADS / Expedition:**
  ```bash
  git lfs install
  git lfs track "*.pdf" "*.sch"
  git add .gitattributes
  ```

* **Для Altium Designer:**
  ```bash
  git lfs install
  git lfs track "*.pdf" "*.SchDoc"
  git add .gitattributes
  ```

* **Для KiCad:**
  ```bash
  git lfs install
  git lfs track "*.pdf" "*.kicad_sch"
  git add .gitattributes
  ```

> **Примечание:** хук автоматически игнорирует файлы с суффиксом `_diff.pdf`, чтобы не перегружать LFS-сервер дубликатами отчетов.

### Шаг 2. Установка pre-commit хука

1. Скопируйте код работающего скрипта в файл `.git/hooks/pre-commit` вашего локального репозитория.
2. Сделайте файл исполняемым:
   ```bash
   chmod +x .git/hooks/pre-commit
   ```

Теперь при каждом вызове команды `git commit` проект будет автоматически проверять PDF-файлы схемы и генерировать точные визуальные отчёты об изменениях.

### Шаг 3. Первый тестовый коммит

```bash
# Внесите небольшое изменение в схему, сохраните PDF
git add schematic.pdf
git commit -m "Test visual diff hook"
```

В выводе должны увидеть строки вида:

```
Генерация diff для schematic.pdf (страниц: 28, было: 28, стало: 28)...
```

После успешного коммита рядом с PDF появится файл `schematic_diff.pdf` с подсветкой изменений.

---

## 📁 Структура файлов после коммита

```
project/
├── schematic.pdf              # сама схема (в Git LFS)
├── schematic_diff.pdf         # визуальный diff (добавляется хуком)
└── .git/
    └── hooks/
        └── pre-commit         # сам хук
```

Если вы хотите складывать diff-файлы в отдельную папку (например, `diff/`), измените в скрипте строку:

```bash
DIFF_PDF="${PDF_FILE%.pdf}_diff.pdf"
```

на:

```bash
DIFF_PDF="$(dirname "$PDF_FILE")/diff/$(basename "${PDF_FILE%.pdf}")_diff.pdf"
mkdir -p "$(dirname "$DIFF_PDF")"
```

---

## 🔧 Troubleshooting

### `magick: command not found`

У вас ImageMagick 6. См. раздел «Важно: `magick` vs `convert`» выше — либо установите IM 7, либо замените `magick` на `convert` в скрипте.

### `LFS pointer detected` или пустые PNG после рендеринга

PDF в репозитории хранится как LFS-указатель, но LFS не развёрнут. Убедитесь:

```bash
git lfs install
git lfs pull
```

В корне репозитория должен быть файл `.gitattributes` со строкой `*.pdf filter=lfs diff=lfs merge=lfs -text`.

### `parallel: command not found`

```bash
# OpenSUSE
sudo zypper install parallel
# Debian/Ubuntu/Mint
sudo apt install parallel
# Fedora
sudo dnf install parallel
```

### `Permission denied: .git/hooks/pre-commit`

```bash
chmod +x .git/hooks/pre-commit
```

### Хук не запускается при `git commit`

Проверьте:
1. Файл лежит именно в `.git/hooks/pre-commit` (не в `.git/hooks/pre-commit.sh`).
2. Файл исполняемый (`ls -la .git/hooks/pre-commit` — должно быть `-rwxr-xr-x`).
3. Нет флага `--no-verify` при коммите.
4. Хук не отключён через `git config core.hooksPath`.

### Все страницы помечены как «изменённые», хотя схема не менялась

Причина — **разный рендеринг poppler** между версиями PDF или смена DPI. Проверьте:

```bash
pdfinfo old.pdf | grep -E "Pages|Page size"
pdfinfo new.pdf | grep -E "Pages|Page size"
```

Если размеры страниц отличаются (`1684 x 2384` и `2384 x 1684`) — PDF отрендерены в разных ориентациях. В этом случае нужно нормализовать PDF перед коммитом: либо всегда печатать в одном формате (A1 landscape), либо добавить в хук предварительный поворот через `pdftk`/`qpdf`.

Если размеры совпадают, но различия всё равно «по всей странице» — попробуйте понизить чувствительность, увеличив `THRESHOLD`:

```bash
THRESHOLD="0.02"   # было 0.01
```

Не поднимайте выше `0.05` — начнёте пропускать реальные изменения.

### Диагностика: как посмотреть промежуточные маски

Установите переменную окружения перед коммитом:

```bash
MSK_DEBUG=1 git commit -m "..."
```

В конце работы скрипт выведет путь к временной папке с промежуточными PNG (маски, слои, base). Их можно открыть и посмотреть, что не так.

### Слишком долгий рендеринг при коммите

По умолчанию `DPI=300`. На 28 страницах A1 это может давать 30–60 секунд на страницу. Понизьте DPI до 150:

```bash
DPI=150
```

Для подсветки RefDes и компонентов этого достаточно.

---

## 🐳 Использование в CI/Docker

Если хук запускается в CI-среде, можно использовать готовый Docker-образ:

```dockerfile
FROM python:3.12-slim

RUN apt-get update && apt-get install -y \
    git git-lfs imagemagick poppler-utils parallel \
    && rm -rf /var/lib/apt/lists/*

RUN pip install img2pdf

# Заменяем magick на convert для IM 6
RUN sed -i 's/\bmagick\b/convert/g' /usr/local/bin/pre-commit-hook
```

Пример GitLab CI:

```yaml
stages:
  - validate

visual-diff:
  stage: validate
  image: your-registry/pdf-diff-runner:latest
  script:
    - git lfs install
    - git lfs pull
    - .git/hooks/pre-commit
  only:
    - merge_requests
```

---

## 🔒 Безопасность и приватность

- **Никаких сетевых запросов.** Все утилиты работают локально.
- **Никакой телеметрии.** Скрипт не отправляет данные ни на какие серверы.
- **Совместимо с NDA.** Схемы и их diff-отчёты остаются в вашем репозитории и никуда не утекают.
- **Альтернатива платным облачным сервисам.** Не требует подписок, лицензий и передачи данных третьим лицам.

---

## 📋 Совместимость

| ОС | Версия | Статус |
| :--- | :--- | :--- |
| OpenSUSE Tumbleweed | 2025+ | ✅ Проверено |
| OpenSUSE Leap | 15.5+ | ✅ Работает |
| Debian | 12 (bookworm) | ✅ Работает (с `convert`) |
| Ubuntu | 22.04 / 24.04 LTS | ✅ Работает (с `convert`) |
| Linux Mint | 21+ | ✅ Работает (с `convert`) |
| Fedora | 38+ | ✅ Работает |
| RHEL / AlmaLinux | 9+ | ✅ Работает |
| Arch Linux / Manjaro | rolling | ✅ Работает |
