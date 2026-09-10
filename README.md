# Boox Note Max as a thin client for coding agents

Notes from setting up a **Boox Note Max** (13.3" e-ink tablet, Android 13, Snapdragon 855, Onyx firmware 4.x)
as the display and keyboard for coding agents (Claude Code, Codex) that run on a home computer. Nothing runs
on the tablet except Termux (ssh), a browser, and the Claude app. Most of it should carry over to other
recent Boox models (same Onyx firmware quirks); the coordinates in a few scripts assume the Note Max's
3200×2400 screen in landscape.

## What's here

- **[boox-agent-setup-runbook.md](boox-agent-setup-runbook.md)** — the runbook. Sections 1–4 are the plan
  (computer, tablet, phone, acceptance test). Section 5 is what actually happened, in order, including every
  Onyx-specific gotcha: sideloaded apps silently disabled (`enabled=3`), the auto-freeze and App Startup
  allowlists, Tailscale dying on sleep, firmware updates reverting state, a battery audit, and a
  root assessment.
- **`bin/`** — computer-side helper scripts referenced in the runbook (copy to `~/bin`):
  - `eink-check` — re-applies every adb-fixable tablet setting after a firmware update or a freeze. Idempotent.
  - `eink-capslock` — re-selects the Caps Lock→Ctrl keyboard layout by driving the AOSP Settings UI
    with `uiautomator dump` and taps by text (no fixed coordinates). Reusable pattern for any GUI-only setting.
  - `eink-refresh` — forces a full e-ink (GC16) refresh over ssh via the Onyx `REFRESH_SCREEN` broadcast.
  - `diagrams` — serves `~/diagrams` as static SVGs on the Tailscale interface only.
- **`keymapper-ring.json`** — [Key Mapper](https://github.com/keymapperorg/KeyMapper) backup implementing a
  fixed app ring (Alt+H/L previous/next, Alt+J home, Alt+K recents, Alt+U last app) for a hardware keyboard.
  Ring is Termux → Claude → ChatGPT → Brave; edit the package names to taste.

## Initial setup: get adb working first

Almost everything in the runbook was done from the computer over adb, with no typing on the e-ink screen.
Do this before anything else.

1. **On the tablet, enable Developer options.** Settings → About Tablet → tap **Build Number** seven times
   (Onyx moves this menu around between firmware versions; it is always under About). Then
   Settings → System → Developer options → turn on **USB debugging**. Leave "Stay awake" on too.
2. **On the computer, install adb.** macOS: `brew install --cask android-platform-tools`.
   Linux: `apt install android-tools-adb` or the platform-tools zip from Google. Windows: the same zip.
3. **Connect the USB-C cable** and run `adb devices -l`. The tablet shows `unauthorized` until you tap
   **Allow** on the "Allow USB debugging?" dialog on the tablet (tick "Always allow from this computer").
   It then shows `device` and `adb shell` gives you a shell.
4. **Optional: adb without the cable.** Once per boot, with USB attached: `adb tcpip 5555`. Then from
   anywhere that can reach the tablet: `adb connect <tablet-ip>:5555`. Only works while the tablet is awake;
   its Wi-Fi is off during sleep.

Two Onyx behaviours will bite immediately after installing anything, so read runbook 5.1 and 5.16 first:
sideloaded apps get disabled (`adb shell pm enable --user 0 <pkg>` fixes it), and apps are "frozen" a minute
after leaving the foreground unless you exempt them in Settings → Apps → Freeze settings.

## What is optional

The runbook uses one specific stack. Swap parts freely:

| Piece | What it does here | If you don't use it |
|---|---|---|
| **Tailscale** | Lets the tablet and phone reach the computer from anywhere, and Taildrop sketches | Any route that gives the tablet ssh to the computer works: same LAN IP, WireGuard, ZeroTier, an ssh jump host. Replace `<laptop-magicdns>` with that address. Skip the Taildrop and `tailscale ip` bits (`bin/diagrams` binds to the Tailscale IP; change it to your LAN IP). |
| **herdr** | Persistent terminal sessions with an agent-status sidebar, reattached from the tablet | tmux or zellij do the persistence part. `ssh host -t tmux attach` instead of `ssh lap -t herdr`. You lose the agent status sidebar. |
| **Claude Code Remote Control / Codex remote control** | Drive the same session from the browser or phone app | Not needed if you only use the terminal over ssh. |
| **macOS specifics** | Remote Login for ssh, SuperMirror for mirroring, `caffeinate` | On Linux use `tailscale up --ssh` or plain sshd, and `logind.conf` for lid-closed. Runbook 1.1–1.2 gives both. |
| **Key Mapper, ExKeyMo, FUTO Voice Input** | Hardware-keyboard app switching, Caps Lock→Ctrl, offline dictation | All independent; skip any you don't want. |

Placeholders like `<laptop-magicdns>`, `<laptop-user>` and `<tablet-ts-ip>` stand for values specific to
one setup. Dates in the runbook are when a step was done; the firmware was Onyx 4.2 (Android 13,
TKQ1.230721.002) at the time.
