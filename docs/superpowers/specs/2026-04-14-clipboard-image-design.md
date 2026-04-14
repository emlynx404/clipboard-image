# clipboard-image Plugin Design Spec

## Overview

A Claude Code plugin that enables pasting images from the clipboard and dragging image files into the CLI for screenshot debugging and code screenshot recognition.

## Problem

Claude Code CLI's native image paste (Ctrl+V / Alt+V) is unreliable — images copied from browsers/apps often fail. Multiple GitHub issues confirm this (#1361, #12644, #22353, #26679, #29776). No `.claude-plugin` format solution exists in any marketplace.

## Solution

A lightweight plugin with two interaction methods:

1. **`/paste` slash command** — extracts image from macOS clipboard via `osascript`, saves to file, Claude reads it
2. **`UserPromptSubmit` hook** — detects dragged image file paths, auto-triggers Claude to read them

## Plugin Structure

```
clipboard-image/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest
├── commands/
│   └── paste.md                 # /paste slash command
├── hooks/
│   └── hooks.json               # Drag-and-drop image detection
├── scripts/
│   └── grab-image.sh            # Clipboard image extraction (macOS)
└── README.md
```

## Components

### 1. Core Script: `scripts/grab-image.sh`

- **Input:** Optional save directory argument (default: `/tmp`)
- **Logic:**
  1. Check clipboard for image data via `osascript -e 'clipboard info'` (look for `PNGf` or `TIFF`)
  2. If no image, output error JSON and exit 1
  3. Export clipboard image as PNG via `osascript` AppleScript
  4. Output success JSON with file path
- **Output format:** `{"success": true, "path": "/tmp/claude-paste-20260414-120000.png"}` or `{"success": false, "error": "reason"}`
- **Filename pattern:** `claude-paste-YYYYMMDD-HHMMSS.png`

### 2. Slash Command: `commands/paste.md`

- **Usage:** `/paste` or `/paste --save <directory>`
- **Allowed tools:** `Bash`, `Read`
- **Behavior:**
  1. Parse `--save` argument if provided, otherwise use `/tmp`
  2. Execute `grab-image.sh` with save directory
  3. Parse JSON output
  4. On success: Read the image file with Read tool, confirm "已接收图片：<path>"
  5. On failure: Report error to user
  6. Wait for user's further instructions (no auto-analysis)

### 3. Hook: `hooks/hooks.json`

- **Event:** `UserPromptSubmit`
- **Matcher:** `*` (all prompts)
- **Type:** `prompt`
- **Logic:** Check if user input contains an image file path (endings: `.png`, `.jpg`, `.jpeg`, `.gif`, `.webp`, `.bmp`). If detected, instruct Claude to Read the image and confirm receipt. If not, approve without action.

### 4. Plugin Manifest: `.claude-plugin/plugin.json`

```json
{
  "name": "clipboard-image",
  "description": "Paste images from clipboard or drag image files into Claude Code CLI",
  "version": "0.1.0",
  "author": {
    "name": "emcow"
  },
  "keywords": ["clipboard", "image", "paste", "screenshot"]
}
```

## Interaction Patterns

| Method | Trigger | Save Location | Implementation |
|--------|---------|---------------|----------------|
| Slash command | `/paste` | `/tmp` (default) | commands/paste.md → grab-image.sh |
| Slash command | `/paste --save ./screenshots` | Specified directory | commands/paste.md → grab-image.sh |
| Drag & drop | Drag image file into terminal | N/A (file already exists) | UserPromptSubmit hook |

## Post-receive Behavior

After receiving an image (by any method), Claude:
1. Confirms: "已接收图片：<file path>"
2. Waits for user's next instruction
3. Does NOT auto-analyze

## Platform Support

- **Current:** macOS only (uses `osascript` / AppleScript)
- **Future:** Linux (`xclip` / `xsel`), Windows (`PowerShell Get-Clipboard`)

## Save Strategy

- Default: `/tmp/claude-paste-*.png` (ephemeral)
- Optional: `--save <directory>` to persist in project directory
- Directory auto-created if not exists (`mkdir -p`)

## Constraints

- macOS only for v0.1.0
- Requires no external dependencies (no `pngpaste`, no `brew` packages)
- Zero overhead when not used (hook only does text pattern matching)
