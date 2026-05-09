# HandySpeechToText

A working setup guide for system-wide voice dictation on **Kubuntu (Wayland / KDE Plasma 6)** using [Handy](https://github.com/cjpais/Handy) — free, open-source, MIT-licensed, and 100% local. Toggle a hotkey, speak, toggle again, and the transcribed text appears at the cursor in any app or terminal. No subscriptions, no API keys, no cloud.

This repo is **documentation-only**. It captures the install steps, the configuration choices, and the reasoning behind them, so the same setup is reproducible on any similar Kubuntu/Wayland machine.

## Overview

The end result is a single keyboard shortcut you press to start recording, and press again to stop. When you stop, Handy transcribes locally (no network), copies the text to the clipboard, and pastes it into whatever window has focus — Kate, KWrite, Konsole, browsers, code editors, chat apps, etc.

**What this is a replacement for:** subscription-based dictation tools like Wispr Flow, which don't ship for Linux at all. Handy is the closest free, native, open-source equivalent.

## Architecture

```
  ┌──────────────────────────────────────────────────────────┐
  │  KDE Plasma 6 (Wayland)                                  │
  │                                                          │
  │   ┌───────────────────────┐   ┌───────────────────────┐  │
  │   │  Active text field    │   │  KDE Global Shortcut  │  │
  │   │  (any GUI / terminal) │   │  (System Settings)    │  │
  │   └──────────┬────────────┘   └──────────┬────────────┘  │
  │              │ Shift+Insert paste        │ user keypress │
  │              ▲                           ▼               │
  │              │             ┌─────────────────────────┐   │
  │              │             │ handy --toggle-         │   │
  │              │             │ transcription           │   │
  │              │             └──────────┬──────────────┘   │
  │              │                        │ IPC (single-     │
  │              │                        │ instance plugin) │
  │   ┌──────────┴──────────────────────┐ │                  │
  │   │  Handy app (Tauri / Rust)       │◄┘                  │
  │   │   ├─ Mic capture                │                    │
  │   │   ├─ Parakeet v3 (local CPU)    │                    │
  │   │   ├─ Clipboard write            │                    │
  │   │   └─ Sends Shift+Insert         │                    │
  │   └─────────────────────────────────┘                    │
  └──────────────────────────────────────────────────────────┘
```

The KDE Global Shortcut is the load-bearing piece. Wayland forbids applications from grabbing global hotkeys themselves, so KDE owns the keybinding and signals the running Handy instance over its CLI (`handy --toggle-transcription`). Inside Handy, the in-app keybinding setting is left unused.

## Key Components

| Component | Technology | Purpose |
|---|---|---|
| Application | [Handy](https://github.com/cjpais/Handy) (Tauri / Rust + React) | The dictation app — captures mic, runs the model, pastes the text |
| Speech model | NVIDIA Parakeet v3 | Local CPU inference; automatic language detection |
| Hotkey owner | KDE Global Shortcut | Wayland-safe global keybinding that drives Handy via CLI |
| Paste mechanism | Clipboard + `Shift+Insert` | The one paste shortcut that works in both GUI apps and Konsole |

## Prerequisites

- Kubuntu (or any KDE Plasma 6 distro) running a Wayland session
- A working microphone
- A CPU that can comfortably run a small ASR model (a 4C/8T x86_64 chip is plenty; Parakeet v3 is CPU-optimized)

GPU acceleration is not required. The setup documented here runs entirely on CPU.

## Setup

### 1. Install Handy from the latest .deb

```bash
cd /tmp
curl -s https://api.github.com/repos/cjpais/Handy/releases/latest \
  | grep "browser_download_url.*amd64.deb" \
  | cut -d '"' -f 4 \
  | xargs curl -L -o handy.deb
sudo apt install ./handy.deb
```

The package installs the binary to `/usr/bin/handy` and a desktop entry to `/usr/share/applications/handy.desktop`. Per-user config and downloaded models live in `~/.local/share/com.pais.handy/`.

### 2. Pick a speech model

Launch Handy and select **Parakeet v3** from the model dropdown. It will download on first use. Whisper variants (large-v3-turbo, medium, etc.) are also available if you want to compare.

### 3. Configure Handy → Advanced → Output

| Setting | Value | Why |
|---|---|---|
| Paste Method | **Clipboard (`Shift+Insert`)** | The only paste shortcut that works universally — Konsole uses `Ctrl+Shift+V` for paste, not `Ctrl+V`, so `Shift+Insert` is the consistent choice for both GUI apps and terminal |
| Clipboard Handling | **Copy to Clipboard** | Required for the clipboard-based paste method to have anything to paste |
| Typing Tool | **Auto (Recommended)** | Falls back to Handy's bundled `enigo` library when `ydotool`/`dotool` are not installed. Verified working on Plasma 6 Wayland |
| Auto Submit | **Off** | Transcription isn't perfect; auto-firing Enter would auto-execute terminal commands or send chat messages before review. The single keystroke saved isn't worth the risk |

### 4. Configure Handy → Hotkey

- Push-to-talk: **Off** (use toggle mode)
- In-app keybinding: leave unset — Wayland forbids apps from grabbing global hotkeys, so KDE owns the shortcut

### 5. Bind a KDE Global Shortcut

In **System Settings → Workspace → Shortcuts → Add Command**:

- **Command:** `handy --toggle-transcription`
- **Shortcut:** any key combination you like (assigned via the KDE shortcut UI)

This sends the toggle command to the already-running Handy instance via its single-instance IPC. Handy starts/stops recording on each press.

Other CLI flags worth knowing about:

| Flag | Effect |
|---|---|
| `--toggle-transcription` | Start/stop recording (the main one) |
| `--toggle-post-process` | Toggle Handy's text post-processing pass |
| `--cancel` | Cancel an in-progress recording without transcribing |

You can bind these to additional KDE shortcuts if useful.

## Usage

1. Click into any text field — GUI app, browser, Konsole, code editor, etc.
2. Press your KDE shortcut → Handy starts recording.
3. Speak.
4. Press the shortcut again → Handy stops, transcribes, copies to clipboard, sends `Shift+Insert` to the active window.
5. Text appears at the cursor.

## Verified Working

- Kate / KWrite
- Konsole (terminal sessions)
- Standard `Shift+Insert` also pastes manually in both contexts (regular `Ctrl+V` works in GUI apps; Konsole uses `Ctrl+Shift+V` — `Shift+Insert` is the one that's universal)

## Status

Working. Documentation-only repo describing a live, in-use setup.

## Notes

- **Wayland blocks app-level global hotkeys.** The KDE Global Shortcut + CLI flag pattern is required, not optional.
- **Konsole's paste shortcut is different from GUI apps** (`Ctrl+Shift+V` vs `Ctrl+V`). This is the single biggest reason to set Handy's paste method to `Shift+Insert` — it sidesteps the difference entirely.
- **Parakeet v3 is a sizeable first download.** Allow a few minutes the first time you select it.
- **Cost:** zero. No subscription, no API charges, no telemetry. All processing local.

## License

This documentation is released under [CC BY 4.0](LICENSE) — share, adapt, and reuse freely with attribution. Handy itself is MIT-licensed by its authors at https://github.com/cjpais/Handy.
