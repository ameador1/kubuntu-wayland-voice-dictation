# HandySpeechToText

A working setup guide for system-wide voice dictation on **Kubuntu (Wayland / KDE Plasma 6)** using [Handy](https://github.com/cjpais/Handy) — free, open-source, MIT-licensed, and 100% local. Press a hotkey, speak, press it again, and Handy transcribes locally and copies the text to your clipboard. Then you paste it into whatever window has focus.

This repo is **documentation-only**. It captures the install steps, the configuration choices, and the reasoning behind them, so the same setup is reproducible on any similar Kubuntu/Wayland machine.

## Overview

The end result is one keyboard shortcut you press to start recording and press again to stop. When you stop, Handy transcribes locally (no network, no cloud) and copies the result to the system clipboard. You then paste it in your active window with `Shift+Insert` (or `Ctrl+V` / `Ctrl+Shift+V`).

**Honest disclosure up front:** Handy's settings imply it will press the paste shortcut for you automatically. On Wayland, that does **not** happen — the clipboard half works, the keystroke-injection half does not. You paste manually. See [How paste actually works](#how-paste-actually-works) below for details and the workaround if you ever want true auto-paste.

**What this is a replacement for:** subscription-based dictation tools like Wispr Flow, which don't ship for Linux at all. Handy is the closest free, native, open-source equivalent.

## Architecture

```
  ┌──────────────────────────────────────────────────────────┐
  │  KDE Plasma 6 (Wayland)                                  │
  │                                                          │
  │   You press the global shortcut (e.g. Super+Space)       │
  │                  │                                       │
  │                  ▼                                       │
  │   ┌─────────────────────────────────────────┐            │
  │   │ KDE Global Shortcut runs                │            │
  │   │ `handy --toggle-transcription`          │            │
  │   └────────────────┬────────────────────────┘            │
  │                    │ IPC to running Handy instance       │
  │                    ▼                                     │
  │   ┌─────────────────────────────────────────┐            │
  │   │ Handy (Tauri / Rust)                    │            │
  │   │   1. Records mic audio                  │            │
  │   │   2. Transcribes via Parakeet v2/v3     │            │
  │   │      (local CPU, no network)            │            │
  │   │   3. Copies the result to the clipboard │            │
  │   └─────────────────────────────────────────┘            │
  │                                                          │
  │   You press your paste shortcut yourself:                │
  │     Shift+Insert  (universal — works everywhere)         │
  │     Ctrl+V        (GUI apps)                             │
  │     Ctrl+Shift+V  (Konsole / many terminals)             │
  │                  │                                       │
  │                  ▼                                       │
  │   ┌─────────────────────────────┐                        │
  │   │ Active text field           │ ← text appears here    │
  │   └─────────────────────────────┘                        │
  └──────────────────────────────────────────────────────────┘
```

The KDE Global Shortcut is the load-bearing piece. Wayland forbids applications from grabbing global hotkeys themselves, so KDE owns the keybinding and signals the running Handy instance over its CLI (`handy --toggle-transcription`). Handy's *in-app* keybinding setting is left at its default (unused).

## Key Components

| Component | Technology | Purpose |
|---|---|---|
| Application | [Handy](https://github.com/cjpais/Handy) (Tauri / Rust + React) | The dictation app — captures mic, runs the model, writes to clipboard |
| Speech model | NVIDIA Parakeet v2 or v3 | Local CPU inference; automatic language detection |
| Hotkey owner | KDE Global Shortcut | Wayland-safe global keybinding that drives Handy via CLI |
| Paste mechanism | System clipboard + manual paste | You press `Shift+Insert` / `Ctrl+V` / `Ctrl+Shift+V` after recording stops |

## Prerequisites

- Kubuntu (or any KDE Plasma 6 distro) running a Wayland session
- A working microphone
- A CPU that can comfortably run a small ASR model (a 4C/8T x86_64 chip is plenty; Parakeet is CPU-optimized)

GPU acceleration is not required. The setup documented here runs entirely on CPU.

---

## Setup

The configuration walk-through follows Handy's actual settings UI. Open Handy and use the left-hand menu: **General**, **Models**, **Advanced**, History, About. Walk through each in order; the tables below mirror the on-screen sections one-for-one.

### Step 1 — Install Handy from the latest .deb

```bash
cd /tmp
curl -s https://api.github.com/repos/cjpais/Handy/releases/latest \
  | grep "browser_download_url.*amd64.deb" \
  | cut -d '"' -f 4 \
  | xargs curl -L -o handy.deb
sudo apt install ./handy.deb
```

The package installs the binary to `/usr/bin/handy` and a desktop entry to `/usr/share/applications/handy.desktop`. Per-user config and downloaded models live in `~/.local/share/com.pais.handy/`.

Launch Handy from the application launcher; it should appear in the system tray (assuming Show Tray Icon is on, which we'll set in Step 4).

### Step 2 — Configure **General** menu

#### General → General section

| Setting | Value | Why |
|---|---|---|
| Transcribe Shortcut | leave at default (`Ctrl+Space`) | This is Handy's *in-app* keybinding. **It does nothing in this setup** because Wayland prevents apps from grabbing global hotkeys. The actual hotkey is bound at the KDE level in Step 5. The default is fine — it just sits unused. |
| Push to Talk | **Off** | Use toggle mode (press once to start, again to stop). Push-to-talk would require Handy to own the global hotkey, which Wayland doesn't allow anyway. |

#### General → Sound section

| Setting | Value | Why |
|---|---|---|
| Microphone | **Pick your specific device** from the list | If you have multiple audio inputs, pick the actual mic you want to use — don't rely on Default. |
| Mute While Recording | **Off** | Leave other audio flowing during recording. |
| Audio Feedback | **Off** | No audio cues on record start/stop. Turn on if you want explicit beeps. |
| Output Device | greyed out (Default) | Only relevant if Audio Feedback is on. |
| Volume | greyed out (100%) | Only relevant if Audio Feedback is on. |

### Step 3 — Configure **Models** menu

This is where you download speech models and pick which one is active.

- **Parakeet v2** — **English-only**, higher accuracy. The recommended pick if you only dictate in English. Active in this setup.
- **Parakeet v3** — multilingual. Slightly lower accuracy than v2 (Handy's own model listing notes the trade-off). Pick this if you need a language other than English.
- **Whisper variants** (large-v3-turbo, medium, etc.) — alternative model family if Parakeet underperforms on your speech or domain.

**Recommendation:** download **Parakeet v2** if you only dictate in English — it's noticeably more accurate in side-by-side testing. Download v3 only if multilingual support matters; otherwise the multilingual capability isn't free, you trade accuracy for it.

You can have multiple models downloaded simultaneously; Handy uses whichever you've marked active. Switching is instant from the Models menu — no config file editing.

The first model download pulls a few hundred megabytes. Allow a minute or two.

### Step 4 — Configure **Advanced** menu

#### Advanced → App section

| Setting | Value | Why |
|---|---|---|
| Start Hidden | personal preference (recommended **On**) | Handy starts in the tray instead of opening its window — reduces clutter at login. |
| Launch on Startup | personal preference (recommended **On**) | Auto-starts Handy at login so the global shortcut works without you having to launch the app first. |
| Show Tray Icon | **On — important, treat as required** | The tray icon is the **only** on-screen indicator of recording state: a hand glyph when idle, an ear glyph when actively recording. Without it you have no way to tell whether the next shortcut press will start or stop a recording. Turn it on. |
| Overlay Position | **None** | Handy itself recommends "None" on Linux. The overlay tends to misbehave on Wayland. |
| Unload Model | **After 5 Minutes** (default) | Frees memory when idle. Default is fine. |
| Experimental Features | **Off** | Stick with stable behavior. |

#### Advanced → Output section

| Setting | Value | Why |
|---|---|---|
| Paste Method | **Clipboard (`Shift+Insert`)** | The transcript is copied to the system clipboard. (See [How paste actually works](#how-paste-actually-works) below — this Wayland setup pastes manually; the synthetic keystroke half of this option does not actually fire.) |
| Clipboard Handling | **Copy to Clipboard** | Required for any clipboard-based paste to work. The alternative ("Don't Modify Clipboard") leaves you with nothing to paste. |
| Auto Submit | **Off** | Don't auto-fire `Enter` (or `Ctrl+Enter` / `Super+Enter`) after transcription. Auto-submitting would auto-execute terminal commands or send chat messages before you can review what was transcribed. The single keystroke saved isn't worth the blast radius. |

#### Advanced → Transcription section

| Setting | Value | Why |
|---|---|---|
| Custom Words | (optional) | Add domain-specific terms (proper nouns, jargon, names) to bias the model toward correct spellings. Useful for technical dictation. |
| Append Trailing Space | personal preference (default **Off**) | If on, Handy adds a space at the end of every transcription. Convenient for chaining dictations; mildly annoying if you're inserting at end-of-line. |

### Step 5 — Bind a KDE Global Shortcut to start/stop recording

Open **System Settings → Workspace → Shortcuts → Add Command**:

- **Command:** `handy --toggle-transcription`
- **Shortcut:** any key combination you like — assigned via the KDE shortcut UI

This sends the toggle command to the already-running Handy instance via its single-instance IPC. Handy starts/stops recording on each press.

**About shortcut conflicts:** KDE global shortcuts are intercepted by the compositor (KGlobalAccel) before any application sees them, so individual apps like VS Code, IntelliJ, Konsole, etc. **cannot conflict with them** — your in-app shortcuts continue to work. The only real conflict surface is *other KDE-level shortcuts* (input method switching, keyboard layout switching, custom shortcuts you've already set). KDE's shortcut UI warns when you pick a combo that's already taken; pick something else if it does.

A common choice for this is `Super+Space` (Windows key + Space). If you have multiple keyboard layouts configured or fcitx/ibus running for input methods, `Super+Space` may already be taken — in that case KDE will warn you, and you can either reassign the existing shortcut or pick a different combo (e.g. `Meta+Alt+V`, `Ctrl+Alt+D`, etc.).

Other Handy CLI flags that can be bound to additional shortcuts if useful:

| Flag | Effect |
|---|---|
| `--toggle-transcription` | Start/stop recording (the main one) |
| `--toggle-post-process` | Toggle Handy's text post-processing pass |
| `--cancel` | Cancel an in-progress recording without transcribing |

---

## Usage

1. Click into any text field — GUI app, browser, Konsole, code editor, etc.
2. Press your KDE shortcut. The tray icon switches to an ear glyph; Handy is recording.
3. Speak.
4. Press the shortcut again. The tray icon switches back to a hand; Handy has stopped, transcribed, and copied the result to the clipboard.
5. **Paste it yourself** with `Shift+Insert` (universal), `Ctrl+V` (GUI apps), or `Ctrl+Shift+V` (Konsole). Text appears at the cursor.

If you didn't see the tray icon switch glyphs, either the tray icon is off (re-enable it in Advanced → App → Show Tray Icon) or Handy isn't running.

## How paste actually works

Despite Handy's "Paste Method = Clipboard (Shift+Insert)" setting suggesting it will press the paste shortcut for you, **on Wayland it does not actually inject the keystroke** into the active window. Wayland's security model restricts applications from synthesizing input events for other windows, and Handy's bundled `enigo` library does not bypass that restriction.

What you actually get:

- ✅ The transcript reliably lands on the system clipboard.
- ❌ The follow-up paste keystroke is *not* delivered to your active window — you press it yourself.

This may not match what older notes (including a previous version of this README) claimed. It's documented honestly here.

**If you want true auto-paste**, the workaround uses one of two third-party Linux utilities:

- **`ydotool`** and **`dotool`** are small, separate command-line programs (not part of Handy) that inject keyboard and mouse input at the **kernel** level — through `/dev/uinput` — instead of going through the Wayland protocol. Because the kernel sees them as just another input device, Wayland's cross-window restriction doesn't apply. They exist precisely as a workaround for Wayland's input-injection lockdown. Pick either; they do the same job.

The setup steps are roughly:

1. Install one of them: `sudo apt install ydotool` (or `dotool` from its own packaging).
2. Start its background daemon and make sure your user can access `/dev/uinput` — this usually means joining the `input` group and logging out/in. The package documentation will spell it out.
3. If your version of Handy exposes a "Typing Tool" setting, point it at `ydotool` or `dotool` explicitly. Otherwise Handy auto-detects whichever is on `$PATH`.

This setup is **not** configured here — manual paste with `Shift+Insert` is fine for the current dictation volume, and the daemon + uinput permission setup is more system surface than is worth maintaining for one keystroke saved per dictation. If you find yourself dictating heavily enough that the manual paste is real friction, the `ydotool`/`dotool` route is the documented escape hatch.

## Verified Working

- Kate / KWrite (paste with `Ctrl+V` or `Shift+Insert`)
- Konsole (paste with `Ctrl+Shift+V` or `Shift+Insert`)
- Web browsers, code editors, chat apps — anywhere clipboard paste is supported

`Shift+Insert` is the consistent universal habit because it pastes in both Konsole and GUI apps. `Ctrl+V` works in GUI apps but not Konsole; `Ctrl+Shift+V` works in Konsole but not most GUI apps.

## Status

Working. Documentation-only repo describing a live, in-use setup.

## Notes

- **Wayland blocks two related things.** First, app-level *global hotkeys* — that's why KDE owns the start/stop shortcut. Second, app-level *keystroke injection into other windows* — that's why paste is manual. Both are intentional Wayland security boundaries, not Handy bugs.
- **Tray icon is your recording-state HUD.** Hand = idle, ear = recording. Don't disable it.
- **Konsole's paste shortcut differs from GUI apps** (`Ctrl+Shift+V` vs `Ctrl+V`). `Shift+Insert` sidesteps that difference — make it the muscle memory.
- **Cost:** zero. No subscription, no API charges, no telemetry. All processing local.

## License

This documentation is released under [CC BY 4.0](LICENSE) — share, adapt, and reuse freely with attribution. Handy itself is MIT-licensed by its authors at https://github.com/cjpais/Handy.
