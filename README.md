# Veronica

Floating circular AI assistant widget for macOS. Built on **Friday Core** (the stable Tauri v2 runtime) with a **Project Friday phase system** that lets you extend Veronica's capabilities without ever touching the core engine.

Iron Man arc-reactor aesthetic. Local-first. Always on top. Always watching.

---

## What it does

- **Holographic widget** — radial filaments + sphere wireframe + pulsing core, drawn on Canvas 2D
- **Click → conversation** — input dialog → LLM → spoken reply (or mic if voice mode enabled)
- **Voice output** — Lemonfox TTS (heart voice) with macOS `say` fallback
- **Multi-turn memory** — keeps last 12 conversation turns in context
- **Tool calling** — AI can:
  - read/write files, list directories
  - run shell commands
  - open macOS apps
  - search the web (DuckDuckGo)
  - get current time, battery, calendar events
  - control system volume
- **Smart click-through** — clicks outside the circle pass to the app below; only the circle itself is interactive
- **Position lock** (⌘⇧L), tray menu, persistent position
- **Phase system** — Project Friday phases plug in at boot via `registerPhase()`, isolated from the core

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│  Veronica — Floating widget (140×140 transparent)   │
│                                                     │
│  ┌─────────────────────────────────────────────┐    │
│  │  Friday Core  (stable runtime, never broken)│    │
│  │   • renderer.js  — Canvas 2D hologram       │    │
│  │   • state-machine.js  — 5 visual states     │    │
│  │   • interactions.js  — click/drag/menu      │    │
│  │   • actions.js  — extension scaffolding     │    │
│  └───────────────────────┬─────────────────────┘    │
│                          │ VeronicaAPI              │
│  ┌───────────────────────▼─────────────────────┐    │
│  │  Veronica Layer  (src/veronica/)             │    │
│  │   • veronica-api.js  — stable phase contract│    │
│  │   • phase-registry.js  — phase lifecycle    │    │
│  └───────────────────────┬─────────────────────┘    │
│                          │ init(api)                │
│  ┌───────────────────────▼─────────────────────┐    │
│  │  Project Friday Phases  (pluggable)          │    │
│  │   phase-a, phase-b, phase-c …               │    │
│  │   Each isolated — one crash ≠ others crash  │    │
│  └─────────────────────────────────────────────┘    │
└───────────────────┬─────────────────────────────────┘
                    │ tauri invoke
┌───────────────────▼─────────────────────────────────┐
│  Rust backend  (src-tauri/src/main.rs)               │
│   • passthrough watcher (CoreGraphics)               │
│   • chat: Ollama tool-loop with memory               │
│   • voice: sox → Lemonfox STT                        │
│   • speak: Lemonfox TTS → afplay                     │
│   • tools: file / shell / web / sys                  │
│   • tray + global shortcut + lock                    │
└─────────────────────────────────────────────────────┘
```

**Key design rule:** Project Friday phases import from `src/veronica/veronica-api.js` only — never from Friday Core modules directly. This means:
- Friday Core internals can be refactored without breaking phases
- New phases in Project Friday don't impact or modify the core runtime
- A failing phase is caught and logged; it cannot crash other phases or the widget

---

## Stack

| Layer | Technology |
|---|---|
| Desktop shell | **Tauri v2** (Rust) — frameless transparent always-on-top window, ~9 MB binary |
| UI renderer | **HTML/Canvas/JS** — Friday Core visual engine + state machine |
| Phase system | **ES module plugins** via `VeronicaAPI` |
| LLM | **Ollama** (local) — `qwen2.5:7b` default; full tool calling |
| Voice input | **Lemonfox STT** + `sox` |
| Voice output | **Lemonfox TTS** (heart voice) with macOS `say` fallback |
| Click-through | **CoreGraphics** — global cursor polling for circle hit-testing |

---

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

LLM order of preference: Ollama (local) → Anthropic → Bedrock.

### 3. Build & run

```bash
git clone https://github.com/Srimi1/Veronica.git
cd Veronica
npm install
npm run tauri build
cp -R "src-tauri/target/release/bundle/macos/Veronica.app" /Applications/
xattr -dr com.apple.quarantine "/Applications/Veronica.app"
open -a "Veronica"
```

### 4. Dev mode

```bash
npm install
npm run tauri dev
```

---

## Usage

- **Click the widget circle** → input dialog → type your request → reply is spoken
- **Drag** to reposition (works anywhere when unlocked)
- **⌘⇧L** → toggle position lock
- **Tray icon** → Lock / Reset / Quit
- **Right-click widget** → state menu + phase debug
- **Keys 1–5** → force state (dev mode)

Position is saved to `~/Library/Application Support/com.veronica.app/position.json`.

---

## Tools available to the AI

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

Try: *"what's on my calendar"*, *"open Safari"*, *"set volume to 30"*, *"search latest M4 Mac mini reviews"*

---

## Writing a Project Friday Phase

A phase is a plain JS object with a `name`, `version`, `init(api)`, and optional `destroy()`.

```js
// src/phases/my-phase.js
import { registerPhase } from '../veronica/phase-registry.js';

registerPhase({
  name: 'my-phase',
  version: '1.0.0',

  async init(api) {
    // api is a VeronicaAPI instance — the ONLY thing phases touch.
    // Never import renderer.js, state-machine.js, etc. directly.

    // React to state changes
    const unsub = api.onStateChange((state) => {
      console.log('[my-phase] state changed to', state);
    });

    // Drive state transitions
    api.setState('LISTENING');

    // Call Rust backend commands
    const locked = await api.invoke('is_locked');

    // Inter-phase events
    api.emit('my-phase:ready', { version: '1.0.0' });
    await api.listen('other-phase:data', ({ payload }) => {
      console.log('[my-phase] received', payload);
    });

    // Store cleanup ref
    this._unsub = unsub;
  },

  destroy() {
    this._unsub?.();
  }
});
```

Then import your phase file before `main.js` runs (or dynamically via `window.VERONICA.registerPhase()`).

### Phase contract

| Property | Required | Description |
|---|---|---|
| `name` | ✅ | Unique string identifier |
| `version` | recommended | Semver string |
| `init(api)` | recommended | Called once at boot with `VeronicaAPI` |
| `destroy()` | optional | Called on unload or explicit disable |

### VeronicaAPI surface

```js
api.setState(state)           // transition: IDLE/LISTENING/THINKING/SPEAKING/ALERT
api.getState()                // → current state string
api.onStateChange(fn)         // subscribe → returns unsubscribe fn
api.setAlert(message)         // trigger ALERT with message

api.invoke(cmd, args)         // call a Tauri Rust command
api.listen(event, handler)    // subscribe to Tauri or inter-phase events
api.emit(event, payload)      // broadcast inter-phase event

api.rendererState             // read-only renderer snapshot
api.version                   // Veronica API version string
```

---

## Phase isolation guarantee

```
Phase A fails init()
  └── error caught, logged as PhaseStatus.ERROR
      └── Phase B, Phase C, Friday Core — all unaffected ✓
```

Every phase's `init()` and `destroy()` runs inside a try/catch. A broken phase is quarantined — it never propagates exceptions to other phases or to the widget runtime.

---

## Configuration

Environment variables (optional):

| Var | Default | Purpose |
|---|---|---|
| `FRIDAY_LLM_MODEL` | `qwen2.5:7b` | Ollama model name |
| `FRIDAY_TTS_VOICE` | `heart` | Lemonfox voice |
| `FRIDAY_VOICE_MODE` | `0` | `1` = mic input on click, `0` = typed dialog |

---

## Debug console

Open DevTools in Tauri dev mode:

```js
// Check phases
window.VERONICA.phases()
// → [{ name, version, status, error }]

// Drive state from console
window.VERONICA.api.setState('ALERT')

// Register a phase at runtime
window.VERONICA.registerPhase({ name: 'test', init(api) { console.log(api.version) } })

// Friday Core internals (backwards compat)
window.FRIDAY.stateMachine.state
window.FRIDAY.renderer
```

---

## Project structure

```
Veronica/
├── src/
│   ├── main.js              # Entry point — boots Friday Core + Veronica layer
│   ├── renderer.js          # Canvas 2D hologram (filaments, sphere, particles)
│   ├── state-machine.js     # 5-state lifecycle
│   ├── interactions.js      # Click, drag, context menu
│   ├── actions.js           # Extension scaffolding
│   ├── styles.css           # Transparent widget CSS
│   ├── index.html           # Widget shell
│   └── veronica/
│       ├── veronica-api.js  # Stable API contract for phases
│       └── phase-registry.js # Phase lifecycle manager
├── src-tauri/
│   ├── src/main.rs          # Rust backend (chat, tools, tray, position)
│   ├── Cargo.toml
│   ├── tauri.conf.json
│   └── capabilities/
│       └── default.json
├── package.json
└── vite.config.js
```

---

## License

MIT
