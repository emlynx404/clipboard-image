# clipboard-image

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin that lets you paste clipboard images directly into conversations. Take a screenshot, ask a question about it — all in one step.

## Install

Clone the repository:

```bash
git clone https://github.com/Emlynx404/clipboard-image.git
```

Then start Claude Code with the plugin:

```bash
claude --plugin-dir /path/to/clipboard-image
```

## Usage

Copy an image to your clipboard (e.g. `Cmd+Shift+4` for a region screenshot), then in Claude Code:

**Paste with a prompt (recommended):**

```
/clipboard-image:paste What bugs do you see in this screenshot?
```

The image and your question are sent together, so Claude can respond immediately.

**Paste without a prompt:**

```
/clipboard-image:paste
```

Claude will confirm the image was received. You can then ask questions about it in your next message.

**Save to a specific directory:**

```
/clipboard-image:paste --save ./screenshots Analyze this architecture diagram
```

The image will be saved to the specified directory instead of `/tmp`.

## Features

- Extract images directly from macOS clipboard
- Ask questions about images in a single interaction
- Auto-resize large screenshots (>1920px) for faster processing
- Save images to a custom directory with `--save`

## Platform Support

- **macOS** — supported (uses osascript + sips)
- **Linux** — not yet supported
- **Windows** — not yet supported

## Requirements

- macOS
- Claude Code CLI
