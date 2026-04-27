# FRIDAY Core

Floating circular AI assistant widget for macOS. Iron Man arc-reactor aesthetic, fits inside the chest circle of an Iron Man wallpaper. Local-first — runs on your Mac with Ollama, no cloud dependency required.

## What it does

- **Holographic widget** — radial filaments + sphere wireframe + pulsing core, drawn on Canvas 2D
- **Click → conversation** — input dialog → LLM → spoken reply
- **Voice output** — Lemonfox TTS (heart voice) with macOS `say` fallback
- **Multi-turn memory** — keeps last 12 turns in context
- **Tool calling** — LLM can:
  - read/write files, list directories
  - run shell commands
  - open macOS apps
  - search the web (DuckDuckGo)
  - get current time, battery, calendar events
  - control system volume
- **Smart click-through** — clicks outside the circle pass to the app below; only the circle itself is interactive
- **Position lock** (⌘⇧L), tray menu, persistent position

## Stack

- **Tauri v2** (Rust) — frameless transparent always-on-top window, ~9 MB binary
- **HTML/Canvas/JS** — frontend renderer + state machine
- **Ollama** (local) — `qwen2.5:7b` default; tool calling
- **Lemonfox** (optional) — STT + TTS
- **CoreGraphics** — global cursor polling for circle hit-testing

## Setup

### 1. Prerequisites

```bash
# Toolchain
brew install node
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
brew install sox        # only needed for voice input

# Local LLM
brew install ollama
ollama pull qwen2.5:7b
ollama serve            # leave running
```

### 2. API keys (optional — Ollama works without any)

Keys live in `~/.friday/`, never in the repo.

```bash
mkdir -p ~/.friday && chmod 700 ~/.friday

# Lemonfox (TTS heart voice + STT) — recommended
echo 'YOUR_LEMONFOX_KEY' > ~/.friday/lemonfox-key && chmod 600 ~/.friday/lemonfox-key

# Anthropic Claude direct (optional fallback)
echo 'sk-ant-...' > ~/.friday/anthropic-key && chmod 600 ~/.friday/anthropic-key

# AWS Bedrock long-term key (optional fallback)
echo 'BedrockAPIKey-...' > ~/.friday/bedrock-key && chmod 600 ~/.friday/bedrock-key
```

Order of preference for chat: Ollama (local) → Anthropic → Bedrock.

### 3. Build & run

```bash
git clone https://github.com/Srimi1/Friday-core.git
cd Friday-core
npm install
npm run tauri build
cp -R "src-tauri/target/release/bundle/macos/FRIDAY Core.app" /Applications/
xattr -dr com.apple.quarantine "/Applications/FRIDAY Core.app"
open -a "FRIDAY Core"
```

## Usage

- **Click the widget circle** → input dialog → speak/type → reply
- **Drag** to reposition (works anywhere outside the lock)
- **⌘⇧L** → toggle position lock
- **Tray icon** → Lock / Reset / Quit
- **Right-click widget** → state menu

Position is saved to `~/Library/Application Support/com.friday.core/position.json`.

## Tools available to the LLM

| Tool | Purpose |
|---|---|
| `get_time` | Current local date/time |
| `read_file`, `write_file`, `list_dir` | Filesystem |
| `run_shell` | Execute zsh commands |
| `open_app` | Launch a macOS app |
| `web_search` | DuckDuckGo search |
| `get_battery` | Mac battery + charging state |
| `calendar_today` | Today's Calendar.app events |
| `set_volume` | System output volume 0–100 |

The model decides when to call them. Try: *"what's on my calendar"*, *"open Safari"*, *"set volume to 30"*, *"search for the latest M4 Mac mini reviews"*.

## Architecture

```
┌──────────────────────────────────────────┐
│  Frameless transparent NSWindow (140×140)│
│  ┌────────────────────────────────────┐  │
│  │  HTML/Canvas — renderer.js         │  │
│  │   • filaments, sphere, particles   │  │
│  │   • state machine (5 states)       │  │
│  │   • interactions (click/drag)      │  │
│  └────────────────────────────────────┘  │
└─────────────┬────────────────────────────┘
              │ tauri invoke
┌─────────────▼────────────────────────────┐
│  Rust backend (main.rs)                  │
│   • passthrough watcher (CoreGraphics)   │
│   • chat: Ollama tool-loop with memory   │
│   • voice: sox → Lemonfox STT            │
│   • speak: Lemonfox TTS → afplay         │
│   • tools: file / shell / web / sys      │
│   • tray + global shortcut + lock        │
└──────────────────────────────────────────┘
```

## Configuration

Environment variables (optional):

| Var | Default | Purpose |
|---|---|---|
| `FRIDAY_LLM_MODEL` | `qwen2.5:7b` | Ollama model name |
| `FRIDAY_TTS_VOICE` | `heart` | Lemonfox voice |
| `FRIDAY_VOICE_MODE` | `0` | `1` = mic input on click, `0` = typed dialog |

## License

MIT
