# clipboard-image

A Claude Code plugin for pasting clipboard images and dragging image files into the CLI.

## Install

```bash
/install-plugin /path/to/clipboard-image
```

## Usage

### Paste from clipboard

1. Copy an image (screenshot, browser image, etc.)
2. In Claude Code, type:

```
/paste
```

To save to a specific directory:

```
/paste --save ./screenshots
```

### Drag and drop

Drag any image file (.png, .jpg, .gif, .webp, .bmp) into the terminal. The plugin automatically detects and reads it.

## Platform Support

- **macOS** — supported (uses osascript)
- **Linux** — not yet supported
- **Windows** — not yet supported

## Requirements

- macOS
- Claude Code CLI
- No external dependencies
