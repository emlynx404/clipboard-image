# Cross-Platform Clipboard Image Extraction — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add Linux and Windows support to the clipboard-image plugin so it works on all major platforms.

**Architecture:** Split `grab-image.sh` into a platform dispatcher that delegates to per-platform scripts (`grab-image-darwin.sh`, `grab-image-linux.sh`, `grab-image-win.ps1`). All scripts share the same JSON output contract.

**Tech Stack:** Bash, PowerShell, osascript, xclip, wl-paste, sips, ImageMagick

---

### File Structure

| File | Action | Responsibility |
|------|--------|----------------|
| `scripts/grab-image.sh` | Rewrite | Platform dispatcher — detect OS, delegate |
| `scripts/grab-image-darwin.sh` | Create | macOS: osascript + sips (extracted from current grab-image.sh) |
| `scripts/grab-image-linux.sh` | Create | Linux: xclip/wl-paste + optional ImageMagick |
| `scripts/grab-image-win.ps1` | Create | Windows: PowerShell + optional ImageMagick |
| `README.md` | Modify | Update Platform Support section |

---

### Task 1: Extract macOS logic to `grab-image-darwin.sh`

**Files:**
- Create: `scripts/grab-image-darwin.sh`

- [ ] **Step 1: Create `grab-image-darwin.sh`**

Copy the current `grab-image.sh` logic into a new file with English error messages:

```bash
#!/bin/bash
# macOS clipboard image extraction
# Usage: grab-image-darwin.sh [save_directory]
# Output: JSON to stdout

SAVE_DIR="${1:-/tmp}"
FILENAME="claude-paste-$(date +%Y%m%d-%H%M%S).png"
FILEPATH="${SAVE_DIR}/${FILENAME}"

# Check if clipboard contains image data
has_image=$(osascript -e 'clipboard info' 2>/dev/null | grep -c 'PNGf\|TIFF')

if [ "$has_image" -eq 0 ]; then
  echo '{"success":false,"error":"No image found in clipboard"}'
  exit 1
fi

mkdir -p "$SAVE_DIR"

# Export clipboard image as PNG
osascript -e "
  set imgData to the clipboard as «class PNGf»
  set filePath to POSIX file \"${FILEPATH}\"
  set fileRef to open for access filePath with write permission
  write imgData to fileRef
  close access fileRef
" 2>/dev/null

if [ -f "$FILEPATH" ]; then
  # Resize if wider than 1920px
  width=$(sips -g pixelWidth "$FILEPATH" 2>/dev/null | tail -1 | awk '{print $2}')
  if [ -n "$width" ] && [ "$width" -gt 1920 ]; then
    sips --resampleWidth 1920 "$FILEPATH" >/dev/null 2>&1
  fi
  echo "{\"success\":true,\"path\":\"${FILEPATH}\"}"
else
  echo '{"success":false,"error":"Failed to export clipboard image"}'
  exit 1
fi
```

- [ ] **Step 2: Make executable**

Run: `chmod +x scripts/grab-image-darwin.sh`

- [ ] **Step 3: Test on macOS**

Copy an image to clipboard, then run:
```bash
bash scripts/grab-image-darwin.sh /tmp
```
Expected: `{"success":true,"path":"/tmp/claude-paste-XXXXXXXX-XXXXXX.png"}`

- [ ] **Step 4: Commit**

```bash
git add scripts/grab-image-darwin.sh
git commit -m "feat: extract macOS clipboard logic to grab-image-darwin.sh"
```

---

### Task 2: Rewrite `grab-image.sh` as platform dispatcher

**Files:**
- Modify: `scripts/grab-image.sh`

- [ ] **Step 1: Rewrite `grab-image.sh`**

Replace the entire file with:

```bash
#!/bin/bash
# Platform dispatcher for clipboard image extraction.
# Detects OS and delegates to the appropriate platform script.
# Usage: grab-image.sh [save_directory]
# Output: JSON to stdout

SAVE_DIR="${1:-/tmp}"
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"

case "$(uname -s)" in
  Darwin)
    bash "$SCRIPT_DIR/grab-image-darwin.sh" "$SAVE_DIR"
    ;;
  Linux)
    bash "$SCRIPT_DIR/grab-image-linux.sh" "$SAVE_DIR"
    ;;
  MINGW*|MSYS*|CYGWIN*)
    powershell.exe -ExecutionPolicy Bypass -File "$SCRIPT_DIR/grab-image-win.ps1" "$SAVE_DIR"
    ;;
  *)
    echo "{\"success\":false,\"error\":\"Unsupported platform: $(uname -s)\"}"
    exit 1
    ;;
esac
```

- [ ] **Step 2: Test on macOS**

Copy an image to clipboard, then run:
```bash
bash scripts/grab-image.sh /tmp
```
Expected: `{"success":true,"path":"/tmp/claude-paste-XXXXXXXX-XXXXXX.png"}` (delegates to darwin script)

- [ ] **Step 3: Commit**

```bash
git add scripts/grab-image.sh
git commit -m "feat: rewrite grab-image.sh as platform dispatcher"
```

---

### Task 3: Create `grab-image-linux.sh`

**Files:**
- Create: `scripts/grab-image-linux.sh`

- [ ] **Step 1: Create `grab-image-linux.sh`**

```bash
#!/bin/bash
# Linux clipboard image extraction (X11 + Wayland)
# Usage: grab-image-linux.sh [save_directory]
# Output: JSON to stdout

SAVE_DIR="${1:-/tmp}"
FILENAME="claude-paste-$(date +%Y%m%d-%H%M%S).png"
FILEPATH="${SAVE_DIR}/${FILENAME}"

# Detect display server and choose tool
if [ -n "$WAYLAND_DISPLAY" ]; then
  CLIP_TOOL="wl-paste"
  CLIP_CMD="wl-paste --type image/png"
else
  CLIP_TOOL="xclip"
  CLIP_CMD="xclip -selection clipboard -t image/png -o"
fi

# Check tool is installed
if ! command -v "$CLIP_TOOL" &>/dev/null; then
  echo "{\"success\":false,\"error\":\"Required tool not found: $CLIP_TOOL\"}"
  exit 1
fi

mkdir -p "$SAVE_DIR"

# Extract clipboard image
$CLIP_CMD > "$FILEPATH" 2>/dev/null

if [ ! -f "$FILEPATH" ] || [ ! -s "$FILEPATH" ]; then
  rm -f "$FILEPATH"
  echo '{"success":false,"error":"No image found in clipboard"}'
  exit 1
fi

# Resize if wider than 1920px (requires ImageMagick)
if command -v convert &>/dev/null; then
  width=$(identify -format "%w" "$FILEPATH" 2>/dev/null)
  if [ -n "$width" ] && [ "$width" -gt 1920 ]; then
    convert "$FILEPATH" -resize 1920x "$FILEPATH" 2>/dev/null
  fi
fi

echo "{\"success\":true,\"path\":\"${FILEPATH}\"}"
```

- [ ] **Step 2: Make executable**

Run: `chmod +x scripts/grab-image-linux.sh`

- [ ] **Step 3: Commit**

```bash
git add scripts/grab-image-linux.sh
git commit -m "feat: add Linux clipboard image extraction"
```

---

### Task 4: Create `grab-image-win.ps1`

**Files:**
- Create: `scripts/grab-image-win.ps1`

- [ ] **Step 1: Create `grab-image-win.ps1`**

```powershell
# Windows clipboard image extraction
# Usage: grab-image-win.ps1 [save_directory]
# Output: JSON to stdout

param(
    [string]$SaveDir = $env:TEMP
)

Add-Type -AssemblyName System.Windows.Forms

$timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
$filename = "claude-paste-$timestamp.png"
$filepath = Join-Path $SaveDir $filename

# Check clipboard for image
$image = [System.Windows.Forms.Clipboard]::GetImage()

if ($null -eq $image) {
    Write-Output '{"success":false,"error":"No image found in clipboard"}'
    exit 1
}

# Ensure save directory exists
if (-not (Test-Path $SaveDir)) {
    New-Item -ItemType Directory -Path $SaveDir -Force | Out-Null
}

# Save as PNG
try {
    $image.Save($filepath, [System.Drawing.Imaging.ImageFormat]::Png)
} catch {
    Write-Output '{"success":false,"error":"Failed to export clipboard image"}'
    exit 1
}

# Resize if wider than 1920px (requires ImageMagick)
if ((Get-Command "convert" -ErrorAction SilentlyContinue) -and $image.Width -gt 1920) {
    & convert $filepath -resize 1920x $filepath 2>$null
}

$image.Dispose()

$filepath = $filepath -replace '\\', '/'
Write-Output "{`"success`":true,`"path`":`"$filepath`"}"
```

- [ ] **Step 2: Commit**

```bash
git add scripts/grab-image-win.ps1
git commit -m "feat: add Windows clipboard image extraction"
```

---

### Task 5: Update README.md

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Update Platform Support section**

Replace the Platform Support section with:

```markdown
## Platform Support

| Platform | Clipboard Tool | Resize Tool | Status |
|----------|---------------|-------------|--------|
| macOS | osascript (built-in) | sips (built-in) | Supported |
| Linux (X11) | xclip | ImageMagick (optional) | Supported |
| Linux (Wayland) | wl-paste | ImageMagick (optional) | Supported |
| Windows | PowerShell (built-in) | ImageMagick (optional) | Supported |

### Linux Prerequisites

Install the clipboard tool for your display server:

```bash
# X11
sudo apt install xclip

# Wayland
sudo apt install wl-clipboard
```

Optional — install ImageMagick for auto-resize of large screenshots:

```bash
sudo apt install imagemagick
```

### Windows Prerequisites

No additional tools required. Optional — install [ImageMagick](https://imagemagick.org/script/download.php#windows) for auto-resize of large screenshots.
```

- [ ] **Step 2: Update Requirements section**

Replace the Requirements section with:

```markdown
## Requirements

- Claude Code CLI
- Platform-specific clipboard tool (see Platform Support above)
```

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: update README with cross-platform support"
```

---

### Task 6: Final verification and push

- [ ] **Step 1: Verify file structure**

Run: `ls -la scripts/`

Expected:
```
grab-image.sh
grab-image-darwin.sh
grab-image-linux.sh
grab-image-win.ps1
```

- [ ] **Step 2: Test on macOS (end-to-end)**

Copy an image to clipboard, then run:
```bash
bash scripts/grab-image.sh /tmp
```
Expected: `{"success":true,"path":"/tmp/claude-paste-XXXXXXXX-XXXXXX.png"}`

- [ ] **Step 3: Push all commits**

```bash
git push origin main
```
