# clipboard-image

A Claude Code plugin for pasting clipboard images and dragging image files into the CLI.

## Install

```bash
/install-plugin /path/to/clipboard-image
```

## Usage

1. Copy an image (screenshot, browser image, etc.)
2. In Claude Code, type:

```
/paste 这张图里的代码有什么bug？
```

Just paste without a prompt:

```
/paste
```

Save to a specific directory:

```
/paste --save ./screenshots 帮我分析这个架构图
```

## Platform Support

- **macOS** — supported (uses osascript)
- **Linux** — not yet supported
- **Windows** — not yet supported

## Requirements

- macOS
- Claude Code CLI
- No external dependencies
