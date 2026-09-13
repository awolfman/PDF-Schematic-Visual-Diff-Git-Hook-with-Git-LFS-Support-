Language / Язык: [Russian](README.ru.md) | **English**

# pdf-schematic-visual-diff

An automated Git hook (`pre-commit`) for visual quality control of changes in PDF schematics and blueprints, designed specifically for hardware development repositories (CAD/EDA) with **Git LFS** support.

> ⚡ **When the hook triggers:** The script executes automatically **at the moment of running `git commit`**. It intercepts staged PDF files, extracts and processes data from the Git LFS storage on the fly, performs a pixel-by-pixel analysis of current page versions against the previous commit (`HEAD`), generates a clear visual report named `[name]_diff.pdf`, and **automatically appends it to the current commit**.

---

## ✨ Blueprint Color Coding Logic

The algorithm operates at a low level by comparing pixel brightness (using the parameter `THRESHOLD="0.01"`), which allows it to accurately detect structural changes in CAD schematics:

- **Component Removal**: All removed symbols/components are highlighted in **blue**.
- **Component Addition or RefDes Change**: Highlighted in **red**.
- **Component Movement**: The old position of the component on the schematic turns **blue**, while the new position turns **red**.

---

## 🚀 Implementation Features

- **Full Git LFS Integration**: Automatically detects LFS text pointers in the commit history and safely performs the `smudge` procedure in an isolated workspace.
- **Anti-Aliasing Resilience**: Low-level `-fx` mathematical calculations paired with morphological expansion (`Dilate Disk:1`) capture micro-shifts in 1-pixel thick vector lines, preventing false negatives.
- **High Performance**: Page processing is parallelized across all available CPU cores using `GNU Parallel`. At the same time, the internal parallelism of tools is limited to protect against memory leaks (RAM Spikes).

---

## 🛠 Dependencies and Packages

Before installing the hook, make sure the following packages are installed on your system:

| Script Command | Tool Purpose | OpenSUSE | Debian/Ubuntu/Mint | Fedora/RHEL |
| :--- | :--- | :--- | :--- | :--- |
| `git-lfs` | Extracts heavy binary PDFs from LFS storage | `git-lfs` | `git-lfs` | `git-lfs` |
| `magick` | Low-level pixel-by-pixel FX math | `ImageMagick` | `imagemagick` | `ImageMagick` |
| `pdftoppm` | Renders PDF pages into raster PNG images | `poppler-tools` | `poppler-utils` | `poppler-utils` |
| `parallel` | Distributes tasks across CPU cores | `parallel` | `parallel` | `parallel` |
| `img2pdf` | Assembles the final PDF report without re-compression | `python3-img2pdf` | `img2pdf` | `img2pdf` |

### Installation by Distribution

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

### ✅ Installation Verification

```bash
for tool in git magick pdftoppm img2pdf parallel; do
    command -v "$tool" >/dev/null 2>&1 && echo "OK: $tool" || echo "MISSING: $tool"
done
```

If any utility is missing, return to the installation block and install it.

---

## ⚠️ Important: `magick` (IM 7) vs `convert` (IM 6)

The script relies on the **`magick`** command, which is the native interface for **ImageMagick 7**. In most distributions, the package manager installs **ImageMagick 6**, where the same functionality is accessed via the **`convert`** command.

### How to Check the Version

```bash
magick --version    # IM 7 — the 'magick' command is available
convert --version   # IM 6 — the 'magick' command might be missing
```

### If You Have ImageMagick 6

**Option A. Install ImageMagick 7 from a repository** (if available):

```bash
# Debian/Ubuntu — via PPA
sudo add-apt-repository ppa:imagemagick/ppa
sudo apt update
sudo apt install imagemagick
```

**Option B. Install via snap** (if snap is installed):

```bash
sudo snap install imagemagick
```

**Option C. Replace `magick` with `convert` inside the script** (fastest solution):

```bash
sed -i 's/\bmagick\b/convert/g' .git/hooks/pre-commit
grep -c "convert" .git/hooks/pre-commit   # verification
```

Functionally, these commands are almost identical: the option syntax matches, and differences only occur in rare edge cases (SVG handling, color profiles). For our tasks—PDF rendering, mask operations, and compositing—the replacement is completely safe.

---

## 📦 Installing `img2pdf` via Python

If the `img2pdf` package is missing from your distribution's repositories (common in older releases or custom builds), install it via `pip`:

### Method 1. Simple installation into the user directory

```bash
pip install --user img2pdf
```

### Method 2. Virtual Environment (Recommended)

```bash
python3 -m venv ~/.venv-img2pdf
~/.venv-img2pdf/bin/pip install img2pdf
```

Then, update the `img2pdf` execution command in your `.git/hooks/pre-commit` script to use the absolute path:

```bash
sed -i 's|^.*img2pdf |\1~/.venv-img2pdf/bin/img2pdf |g' .git/hooks/pre-commit
```

### Method 3. `pipx` (Isolated Installation)

```bash
pipx install img2pdf
```

---

## 💻 Repository Configuration & Hook Setup

### Step 1. Initialize Git LFS for PDFs and CAD Sources

Depending on your Electronic Design Automation (EDA) environment, binary schematic files and output PDF documents must be placed under Git LFS tracking. Run the following commands in the root of your repository based on your CAD system:

* **For Cadence Allegro / OrCAD Capture:**
  ```bash
  git lfs install
  git lfs track "*.pdf" "*.dsn"
  git add .gitattributes
  ```

* **For Mentor Graphics PADS / Expedition:**
  ```bash
  git lfs install
  git lfs track "*.pdf" "*.sch"
  git add .gitattributes
  ```

* **For Altium Designer:**
  ```bash
  git lfs install
  git lfs track "*.pdf" "*.SchDoc"
  git add .gitattributes
  ```

* **For KiCad:**
  ```bash
  git lfs install
  git lfs track "*.pdf" "*.kicad_sch"
  git add .gitattributes
  ```

> **Note:** The hook automatically ignores files with the `_diff.pdf` suffix to prevent bloating the LFS server with redundant report copies.

### Step 2. Install the pre-commit Hook

1. Copy the script code into the `.git/hooks/pre-commit` file of your local repository.
2. Make the file executable:
   ```bash
   chmod +x .git/hooks/pre-commit
   ```

Now, every time `git commit` is executed, the project will automatically scan the schematic PDF files and generate accurate visual diff reports.

### Step 3. First Test Commit

```bash
# Make a small change to the schematic, then save the PDF
git add schematic.pdf
git commit -m "Test visual diff hook"
```

You should see output lines similar to this:

```
Generating diff for schematic.pdf (pages: 28, original: 28, current: 28)...
```

Following a successful commit, a `schematic_diff.pdf` file with highlighted modifications will appear next to your original PDF.

---

## 📁 Post-Commit File Structure

```
project/
├── schematic.pdf              # The schematic itself (tracked in Git LFS)
├── schematic_diff.pdf         # Visual diff report (automatically appended by the hook)
└── .git/
    └── hooks/
        └── pre-commit         # The Git hook script
```

If you prefer to store diff reports in a separate directory (e.g., `diff/`), modify this line in the script:

```bash
DIFF_PDF="${PDF_FILE%.pdf}_diff.pdf"
```

Change it to:

```bash
DIFF_PDF="$(dirname "$PDF_FILE")/diff/$(basename "${PDF_FILE%.pdf}")_diff.pdf"
mkdir -p "$(dirname "$DIFF_PDF")"
```

---

## 🔧 Troubleshooting

### `magick: command not found`

You are using ImageMagick 6. Refer to the "Important: `magick` vs `convert`" section above—either upgrade to IM 7 or replace `magick` with `convert` inside the script.

### `LFS pointer detected` or Empty PNGs After Rendering

The PDF is stored as an LFS pointer in the repository, but LFS has not been fetched locally. Verify your setup:

```bash
git lfs install
git lfs pull
```

Ensure that your repository root contains a `.gitattributes` file with the following rule: `*.pdf filter=lfs diff=lfs merge=lfs -text`.

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

### The Hook Does Not Run on `git commit`

Verify the following:
1. The file is strictly named `.git/hooks/pre-commit` (not `.git/hooks/pre-commit.sh`).
2. The file is executable (`ls -la .git/hooks/pre-commit` should display `-rwxr-xr-x`).
3. You are not using the `--no-verify` flag during your commit.
4. The hook path has not been overridden via `git config core.hooksPath`.

### All Pages Marked as "Modified" Though the Schematic Hasn't Changed

This is caused by **differing poppler rendering behavior** between PDF versions or a change in DPI. Check your files:

```bash
pdfinfo old.pdf | grep -E "Pages|Page size"
pdfinfo new.pdf | grep -E "Pages|Page size"
```

If the page boundaries differ (e.g., `1684 x 2384` vs `2384 x 1684`), the PDFs were exported in different orientations. You must normalize the PDFs before committing: either consistently print using the same layout format (e.g., A1 landscape), or add a pre-rotation step to the hook using `pdftk` or `qpdf`.

If the page dimensions match but full-page false positives persist, try lowering the sensitivity by increasing the `THRESHOLD` value:

```bash
THRESHOLD="0.02"   # default was 0.01
```

Avoid raising it past `0.05`, as it will begin to skip actual schematic changes.

### Diagnostics: How to Inspect Intermediate Masks

Set the following environment variable before committing:

```bash
MSK_DEBUG=1 git commit -m "..."
```

Upon completion, the script will output the path to a temporary directory containing intermediate PNG assets (masks, layers, base files) for inspection.

### Commit Takes Too Long to Process

By default, the script renders at `DPI=300`. For a 28-page A1 layout, this can take 30–60 seconds per page. Lower the resolution to 150 DPI:

```bash
DPI=150
```

This resolution remains fully sufficient for tracking components and Reference Designators (RefDes).

---

## 🐳 CI/Docker Integration

If you run this hook inside a CI/CD environment, you can utilize a pre-built Docker image configuration:

```dockerfile
FROM python:3.12-slim

RUN apt-get update && apt-get install -y \
    git git-lfs imagemagick poppler-utils parallel \
    && rm -rf /var/lib/apt/lists/*

RUN pip install img2pdf

# Substitute magick with convert for ImageMagick 6 environments
RUN sed -i 's/\bmagick\b/convert/g' /usr/local/bin/pre-commit-hook
```

Example GitLab CI workflow configuration:

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

## 🔒 Security and Privacy

- **Zero Telemetry.** The script never transmits data to external servers.
- **NDA Compliant.** Schematics and their corresponding diff reports reside strictly within your repository ecosystem.

## 👥 Authors & AI Contributors

* **awolfman** — *Project Concept, Hook Logic, Bash Implementation, and Hardware CAD/EDA Integration Testing*

* **DeepSeek** — *Optimization of Low-Level FX Math, Linux Package Diagnostics, and Memory Leak (RAM Spikes) Protections*
* **Claude** — *Parallelization Strategy (GNU Parallel Infrastructure) and Anti-Aliasing Resilience Operations*
* **ChatGPT** — *CI/CD Integration Architecture, Docker Environment Deployment, and Troubleshooting Resolution Logic*
* **Gemini (Google AI)** — *Technical Documentation Refinement, English Localization, and Bilingual Layout Structuring*
