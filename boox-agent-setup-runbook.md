# Boox + Laptop Agent Setup Runbook

Topology: Boox and phone each run Tailscale and talk to the laptop directly. The phone is only a hotspot (away from home) and an optional voice input. All agent compute lives on the laptop.

```
Boox ──(Tailscale / Wi-Fi or hotspot)──▶ Laptop ── herdr ── Claude Code / Codex ── AWS / repos
Phone ──(Claude app, Remote Control)───▶ Laptop
```

---

## 1. Laptop (do this first, before the Boox arrives)

### 1.1 Keep it awake with the lid closed
- **macOS:** `caffeinate -dimsu &` for a quick test; install Amphetamine (App Store) for a persistent, configurable version. Clamshell mode needs power connected.
- **Linux:** in `/etc/systemd/logind.conf` set `HandleLidSwitch=ignore` and `HandleLidSwitchExternalPower=ignore`, then `sudo systemctl restart systemd-logind`.
- Confirm Tailscale reconnects after a sleep/wake cycle: `tailscale status` from another device.
- 2026-09-08: on this MacBook, lid-closed on power already stayed awake without caffeinate/Amphetamine; nothing installed for this.

### 1.2 SSH over Tailscale
You already have Tailscale, so no port-forwarding or public keys on the internet.
- **Linux:** `sudo tailscale up --ssh` — Tailscale handles auth; no key management needed.
- **macOS:** the App Store client doesn't run Tailscale SSH, so enable System Settings → General → Sharing → Remote Login, and use key auth (step 2.4 puts the Boox key in `~/.ssh/authorized_keys`).
- Note your laptop's MagicDNS name (`tailscale status` shows it, e.g. `laptop.tailXXXX.ts.net`) — you'll use it everywhere below.

### 1.3 herdr (persistent sessions)
Install per https://herdr.dev (single binary; `cargo install herdr` or the release tarball). Don't run it inside tmux — it replaces tmux.
```bash
herdr                        # first run: onboarding, then a default session
# ctrl+b n   new workspace (one per repo)
# ctrl+b v   split pane      ctrl+b c   new tab
# ctrl+b q   detach          herdr      reattach
```
Run `claude` and `codex` in their own panes. herdr detects the agents and shows blocked/working/done in the sidebar, which is the thing you actually want on a small e-ink screen. Pane processes survive detach; after a full laptop reboot the panes come back as bare shells in the right directories and you re-run the agents (or enable `resume_agents_on_restore` in herdr's config).

### 1.4 Claude Code Remote Control
- Update: `claude update` (Remote Control needs a recent version).
- In a herdr pane in your project dir: `claude remote-control` — the machine should then appear as a device card in the Code tab of the Claude app / claude.ai/code. Check `claude --help` if the subcommand name has changed.
- Docs: https://code.claude.com/docs/en/remote-control

### 1.4b Codex CLI remote control
- Enable the feature flag in `~/.codex/config.toml`:
  ```toml
  [features]
  remote_control = true
  ```
- In a herdr pane: `codex remote-control` (check `codex features list` / `codex --help` — the flag was still marked under-development mid-2026).
- Surface: the ChatGPT mobile app only (Codex tab → your machine). No browser equivalent of claude.ai/code, so on the Boox that means the ChatGPT Android app. Known rough edges: locally generated images referenced by path don't render in the mobile view, and restarting the bridge can leave stale "run failed" banners on completed threads.
- For the e-ink-friendly path, drive the Codex TUI in its herdr pane over SSH (section 2.4). That's the primary Codex route; the phone app is secondary.

### 1.5 Diagram server (static SVGs for the e-ink screen)
```bash
mkdir -p ~/diagrams
npm i -g @mermaid-js/mermaid-cli     # provides `mmdc`; pulls a headless Chromium
# serve only on the Tailscale interface, not the LAN:
TS_IP=$(tailscale ip -4)
cat > ~/bin/diagrams <<EOF
#!/bin/bash
cd ~/diagrams && python3 -m http.server 8080 --bind $TS_IP
EOF
chmod +x ~/bin/diagrams
```
Run `diagrams` in a spare herdr pane or tab. Add to your global `~/.claude/CLAUDE.md`:
```
When explaining architecture, data flow, or system design, write a diagram
(Mermaid rendered to SVG via `mmdc -i x.mmd -o ~/diagrams/x.svg`, or hand-written SVG)
and give me the URL http://<laptop-magicdns>:8080/x.svg. Prefer high-contrast
black-on-white, no gradients — it will be viewed on an e-ink display.
```

### 1.6 Receive sketches from the Boox (Taildrop)
- **macOS:** Taildrop files land in `~/Downloads` automatically.
- **Linux:** run a receiver loop in a herdr pane: `mkdir -p ~/inbox && while true; do tailscale file get ~/inbox; sleep 5; done`
- Add to CLAUDE.md: `Sketches I send arrive in ~/inbox (or ~/Downloads). When I say "look at my sketch", read the newest PNG there. (Put the same lines in ~/.codex/AGENTS.md for Codex.)`

---

## 2. Boox (Note Max)

### 2.1 First boot
- Skip the Boox account if you don't want their cloud; do sign into Google: Settings → Apps → Enable Google Play, register the device ID when prompted, reboot.
- Settings → Power / Apps: disable "auto-freeze" or "kill background apps" for Tailscale and Termux (name varies by firmware). Otherwise the VPN and terminal get killed after a few minutes idle.

### 2.2 Install
| App | Source | Purpose |
|---|---|---|
| Tailscale | Play Store | Reach the laptop; Taildrop for sketches |
| Termux | GitHub releases or F-Droid (**not** Play Store — that build is abandoned) | SSH → herdr; primary route for the Codex TUI |
| Chrome or Firefox | Play Store | claude.ai/code, diagram server |
| FUTO Voice Input | futo.org / F-Droid | Offline Whisper dictation as a keyboard |
| Claude (Android) | Play Store | Alternative to the browser for Remote Control |
| ChatGPT (Android) | Play Store | Codex remote control (optional) |

### 2.3 Tailscale
- Sign in to the same tailnet. Confirm the laptop shows online in the device list.
- Optional: allow it to run as an always-on VPN (Android Settings → VPN → Tailscale → Always-on).

### 2.4 Termux
```bash
pkg update && pkg install openssh
ssh-keygen -t ed25519            # macOS laptops only; Tailscale SSH on Linux needs no key
cat ~/.ssh/id_ed25519.pub        # paste into laptop ~/.ssh/authorized_keys
cat >> ~/.ssh/config <<'EOF'
Host lap
  HostName <laptop-magicdns>
  User <your-username>
  ServerAliveInterval 30
EOF
termux-wake-lock                 # keeps the session alive when the screen sleeps
ssh lap -t herdr                 # attaches to the herdr session from 1.3
```
Bump the Termux font size (pinch-zoom) and set the Boox per-app refresh mode for Termux and the browser to the fastest mode (X-mode / "Fast") to reduce typing lag. herdr is mouse-native, so tapping panes and the sidebar works with the pen; if the sidebar eats too much screen, toggle it off and rely on notifications.

### 2.5 Browser
- Open `https://claude.ai/code`, log in, add to home screen. Verify the laptop appears as a Remote Control device.
- Bookmark `http://<laptop-magicdns>:8080/` for diagrams.
- Set the browser's per-app refresh mode to Fast, and enable "request desktop site" for claude.ai/code if the mobile layout is cramped.

### 2.6 Dictation
- Enable FUTO Voice Input as an input method; pick the Whisper "base" or "small" English model. First use downloads the model — do it on Wi-Fi.
- Use the mic key in the keyboard bar when the claude.ai/code message box is focused.
- Whitelist FUTO from background killing too, or the first dictation after idle will take several seconds to reload the model.

### 2.7 Sketch → Claude loop
1. Draw in the Boox Notes app (or a screenshot of a diagram you've marked up).
2. Share/Export → PNG → **Tailscale** → pick the laptop. It lands in `~/Downloads` (mac) or `~/inbox` (Linux).
3. In claude.ai/code: "Look at my latest sketch and update the design." Or attach the PNG directly with the paperclip — both work; Taildrop keeps it out of the transcript uploads.

### 2.8 Peripherals
- Pair the Bluetooth keyboard and headset to the Boox (Settings → Bluetooth). If you'd rather dictate through the phone's Claude app instead, pair the headset to the phone.

---

## 3. Phone
- Install the Claude app; confirm the laptop appears in the Code tab (Remote Control). This gives you voice input into the same session the Boox is displaying.
- Hotspot on when away from home. Both the Boox and phone still route to the laptop through Tailscale; no other config.

---

## 4. Acceptance test (15 minutes)
1. Laptop lid closed, on power. From the Boox: `tailscale status` shows the laptop online.
2. Termux `ssh lap -t herdr` attaches to herdr; both the `claude` and `codex` panes show in the sidebar.
3. claude.ai/code on the Boox shows the laptop's Remote Control session; send a message, get a reply.
4. Ask Claude for a diagram; open the URL in the Boox browser.
5. Draw a box on the Boox, Taildrop it, ask Claude to describe it.
6. Dictate a paragraph with FUTO; check accuracy.
7. Turn off home Wi-Fi on the Boox, join the phone hotspot, repeat step 3.

---

## 5. What actually happened (2026-09-08, first setup)

Laptop facts: MagicDNS `<laptop-magicdns>` (`<laptop-ts-ip>`), user `<laptop-user>`. Boox adb serial `<serial>`, Android 13 / SDK 33, arm64, Play Store present and enabled out of the box.

### 5.1 adb fast path (replaces most of 2.2 and 2.4)
With USB debugging on and the RSA prompt accepted, everything below ran from the laptop with no typing on the e-ink screen.
```bash
brew install --cask android-platform-tools
adb devices -l                                # "unauthorized" until you tap Allow on the Boox
adb install -r termux-app_v0.118.3+github-debug_arm64-v8a.apk   # github.com/termux/termux-app releases
adb install -r tailscale-android-universal-1.102.3.apk         # github.com/tailscale/tailscale-android releases
```
**Gotcha:** the Boox marks sideloaded APKs `enabled=3` (disabled for user 0) right after install, so they don't show up in the launcher and `am start` says the activity doesn't exist. Fix:
```bash
adb shell pm enable --user 0 com.termux
adb shell pm enable --user 0 com.tailscale.ipn
adb shell am start -n com.termux/.app.TermuxActivity           # first launch unpacks the bootstrap (~5s)
```
Doze / background-kill exemptions (the generic Android layer; the Onyx-specific per-app "freeze" toggle is still GUI-only):
```bash
for p in com.termux com.tailscale.ipn; do
  adb shell dumpsys deviceidle whitelist +$p
  adb shell cmd appops set --uid $p RUN_ANY_IN_BACKGROUND allow   # --uid matters, see 5.26
done
```
Wi-Fi was off after first boot; `adb shell svc wifi enable` brought it up and it rejoined the saved network.

### 5.2 Driving Termux from the laptop
The GitHub "debug" build is debuggable, so `adb shell run-as com.termux sh` works. Termux binaries need the Termux env, and `$VARS` get expanded by the device shell before `run-as` sees them, so pipe a script over stdin instead:
```bash
cat > tx-env.sh <<'EOF2'
export PREFIX=/data/data/com.termux/files/usr
export HOME=/data/data/com.termux/files/home
export PATH=$PREFIX/bin:/system/bin
export LD_PRELOAD=$PREFIX/lib/libtermux-exec.so
export TMPDIR=$PREFIX/tmp
export TERM=xterm-256color
export DEBIAN_FRONTEND=noninteractive
EOF2
{ cat tx-env.sh; echo 'cd $HOME; pkg install -y openssh; ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519'; } | adb shell run-as com.termux sh
```
Done this way on 2026-09-08: openssh installed, key `boox-notemax` generated and appended to laptop `~/.ssh/authorized_keys`, `~/.ssh/config` written with `Host lap`, and `alias lap='termux-wake-lock; ssh lap -t herdr'` added to Termux `~/.bashrc`. SSH from Termux to the laptop verified over LAN (key auth). Tailscale sign-in done later the same evening: the Boox is `note-max` (`<tablet-ts-ip>`) and `ssh lap` from Termux was verified over MagicDNS with a direct connection.

### 5.3 Boox as an external Mac display
Chosen: **SuperMirror** (supermirror.app, $29 one-time after a 7-day trial, macOS 14+ Apple Silicon). USB only, uses adb, nothing to install on the Boox, mirror-only (no extended desktop). Set E Ink Center on the Boox to Speed mode while mirroring. Installed to /Applications (v2.4.10) on 2026-09-08; the dmg is also in `~/Downloads`. Free alternative if the latency is acceptable: Deskreen over Tailscale in the browser, but it needs a virtual display (BetterDisplay) on the Mac and is Wi-Fi only.

### 5.4 Open todos
- Pair the Targus Bluetooth keyboard (deferred on 2026-09-08) (`adb shell am start -a android.settings.BLUETOOTH_SETTINGS`, then pair from the GUI).
- Interactive cleanup of unused apps / background items (inventory via `adb shell pm list packages -3` and `ps -A`).
- Battery: check `dumpsys batterystats` after a day of real use before disabling things blindly.

### 5.5 Landscape lock (2026-09-08)
Standard Android settings work on the Boox; this rotates everything, including Termux:
```bash
adb shell settings put system accelerometer_rotation 0   # disable auto-rotate
adb shell settings put system user_rotation 1            # 1 = 90°, 3 = 270° (flip if upside down), 0 = portrait
```

### 5.6 Drop into herdr on start (2026-09-08)
Two layers. herdr runs only on the laptop; the Boox is a thin SSH client.
1. **Termux opens → herdr.** `~/.bashrc` on the Boox auto-runs `termux-wake-lock; ssh lap -t herdr` in interactive, non-ssh, non-nested shells (guarded by `HERDR_AUTO`). When the session ends you land in the local shell; `lap` reconnects.
2. **Boot → Termux opens.** Termux:Boot (github.com/termux/termux-boot, same-signature github debug build) installed via adb, `pm enable --user 0 com.termux.boot`, opened once so its BOOT_COMPLETED receiver registers. Script `~/.termux/boot/00-open-termux.sh` waits 15 s then runs `am start -n com.termux/.app.TermuxActivity`. Use Termux's own `am` (termux-am over a socket); `/system/bin/am` exits 255 from an app uid on Android 13.

**Gotcha, again:** the Boox re-disabled com.termux.boot (`enabled=3`) some seconds *after* the first `pm enable`. Re-check with `dumpsys package <pkg> | grep 'User 0'` and enable a second time if needed. Verify after a reboot that all three packages stay `enabled=0/1`.

### 5.7 Termux with a hardware keyboard (2026-09-08)
`~/.termux/termux.properties` on the Boox: `hide-soft-keyboard-on-startup = true` and `extra-keys = []`. Volume Up + K toggles the soft keyboard when needed. Aliases `keys-on` / `keys-off` in `~/.bashrc` restore or hide the ESC/CTRL/arrow row and call `termux-reload-settings`.

**Font size (2026-09-09):** adjust from the keyboard with **Ctrl+Alt+=** / **Ctrl+Alt+-** (or pinch-zoom). Termux writes it to `/data/data/com.termux/shared_prefs/com.termux_preferences.xml` as a *string* entry (`<string name="fontsize">38</string>`, px; default 22 at this 300 dpi screen) and it persists across launches. Don't edit the file by hand: an `<int>` entry is ignored, the app only re-reads the file on process start, and killing the Termux process over ssh also kills its sshd (same uid; sshd only returns via the boot script or a manual `sshd`). Currently 38.

### 5.8 Keyboard ergonomics (2026-09-08)
- **Newline in Claude Code over ssh:** Termux can't send a distinct Shift+Enter. Use **Ctrl+J** (default `chat:newline` binding) or `\` then Enter. `/terminal-setup` doesn't help here; a different key can be bound to `chat:newline` in `~/.claude/keybindings.json` on the laptop if wanted.
- **Caps Lock → Ctrl, no root:** ExKeyMo prebuilt layout APK (github.com/ris58h/exkeymo-server README, "CapsLock to Ctrl"). The KCM is just `type OVERLAY` + `map key 58 CTRL_LEFT`. `adb install`, `pm enable --user 0 ris58h.exkeymo_keyboard_layout` (twice, see 5.6 gotcha), then Settings → Physical keyboard → Targus Folding Ergonomic Bluetooth Keyboard → pick "ExKeyMo Layout" (`adb shell am start -a android.settings.HARD_KEYBOARD_SETTINGS` opens the page). Layout choice is GUI-only.
- **`keys-on` / `keys-off`** are aliases in the Boox-local Termux shell, not on the laptop. From herdr: `ctrl+b q` detaches (ssh exits, you land in the local shell), run the alias, then `lap`. Or swipe in from the left edge → New session: since 5.6's guard, a second Termux session is a plain local shell.

### 5.9 Reading comfort on e-ink (2026-09-08)
- **Claude Code animations:** `~/.claude/settings.json` now has `prefersReducedMotion: true` (no spinner/shimmer/flash) and `spinnerTipsEnabled: false`. `tui` was already `"fullscreen"`, which bounds redraws to the visible area. Streaming can't be disabled; the closest thing to chunked output is screen-reader mode, per launch: `CLAUDE_AX_SCREEN_READER=1 CLAUDE_AX_PREPARK_MS=0 claude` (plain text, no borders, batched updates). Docs: code.claude.com/docs/en/settings-reference, /fullscreen, /env-vars.
- **herdr scrollback by keyboard:** the prefix here is **ctrl+a** (set in config.toml), not the default ctrl+b. PageUp/PageDown scroll the focused pane directly. `ctrl+a [` enters copy mode: j/k lines, ctrl+u/ctrl+d half pages, ctrl+b/ctrl+f pages, `/` search, Esc leaves. `ctrl+a e` opens the scrollback in $EDITOR.
- **Full e-ink refresh from a command:** no documented Onyx API. Candidates found in the system APKs and test-fired via `adb shell am broadcast -a <action>`: `onyx.android.intent.action.REFRESH_SCREEN` (handled by the Notes app, may be note-only) and `action.view.epd.update` (toolbar app). Both deliver without error, including when sent from Termux's own `am`. Whether either produces the toolbar-button GC16 flash was unconfirmed at time of writing. Fallback that always works on e-ink: paint the pane black then white (the AutoHotkey trick), or assign "Refresh" to a Boox swipe gesture in Settings → Gesture management.

### 5.10 `eink-refresh` (2026-09-08) — confirmed working
`onyx.android.intent.action.REFRESH_SCREEN` is the toolbar-button full refresh (confirmed by watching the screen flash; `action.view.epd.update` was not the one). Any app may send it, including Termux's `am`.
Plumbing: Termux `sshd` on the tablet (port 8022, `PasswordAuthentication no`, laptop `id_ed25519.pub` in Termux `~/.ssh/authorized_keys`, started from `~/.termux/boot/00-open-termux.sh`). Laptop `~/.ssh/config` has `Host note-max` (MagicDNS, port 8022, user = Termux's uid, e.g. `u0_a123`; check with `id` in Termux). The laptop command `eink-refresh` runs `ssh note-max am broadcast -a onyx.android.intent.action.REFRESH_SCREEN`. From inside Claude Code type `!eink-refresh`. Note `am` on the tablet is termux-am and needs the Termux app alive, which the boot script guarantees.
The same ssh path is a general laptop→tablet channel (scp sketches, `am start`, etc.) that doesn't need USB.

### 5.11 Sleep kills Tailscale (2026-09-08)
Symptom: tablet shows "offline" on the tailnet, `ssh note-max` times out, Wi-Fi still fine. logcat shows `OnyxPowerManager: setKernelPowerModeImp: freeze` at screen-off; the Tailscale process is gone afterwards (`ps -A | grep tailscale` → 0). Not caused by any app setting; it's the Boox sleep path.
Fix: `adb shell am start -n com.tailscale.ipn/.MainActivity` brings it back in ~5 s. Mitigations applied:
- `settings put global stay_on_while_plugged_in 7` (never sleep while on USB/AC).
- `settings put secure always_on_vpn_app com.tailscale.ipn` plus confirm the toggle in Settings → VPN → Tailscale → Always-on VPN (GUI). Android then restarts the VPN after wake.
- `settings put global wifi_on_when_device_boot 1` (it was 0, which is why Wi-Fi was off after first boot).
Tunnel interface on the tablet is `tun0`, not `tailscale0`. Screen timeout is 5 min (`screen_off_timeout=300000`); Boox "auto power off" is disabled.
Still to test: does Tailscale come back on its own after a screen-off/on cycle on battery, and does Termux sshd survive it (it holds a wake lock).

### 5.12 Termux light theme (2026-09-08)
`~/.termux/colors.properties` on the Boox: white background, black foreground, dark saturated ANSI colors (red #A00000, green #005A00, blue #0000A8, ...), bright variants only slightly lighter so they survive grayscale. `termux-reload-settings` applies live. Revert by deleting the file and reloading. herdr's own chrome (sidebar, borders) follows herdr's theme on the laptop (`catppuccin`), not Termux's palette; for a light herdr use `name = "catppuccin-latte"` or `auto_switch = true` with `light_name` set, then `herdr server reload-config`.

### 5.13 Batch of 2026-09-08 evening
- **Refresh hotkey:** `ctrl+a f` in herdr runs `eink-refresh` (custom `[[keys.command]]` in herdr config.toml).
- **FUTO Voice Input** sideloaded from voiceinput.futo.org (`org.futo.voiceinput`), enabled, doze-exempt, IME registered via `adb shell ime enable org.futo.voiceinput/.VoiceInputMethodService`. Model download is a tap inside the app (Wi-Fi).
- **Sketch loop:** CLAUDE.md and ~/.codex/AGENTS.md now say sketches arrive in `~/Downloads` via Taildrop; "look at my sketch" reads the newest PNG.
- **Diagram server:** `mmdc` 11.17 installed (`npm i -g @mermaid-js/mermaid-cli`), `~/bin/diagrams` serves `~/diagrams` on the Tailscale IP only, running in herdr tab "diagrams" in the eink-screen workspace. Verified: `http://<laptop-magicdns>:8080/test.svg` returns 200, LAN IP refuses. CLAUDE.md/AGENTS.md instruct agents to render diagrams there. Render: `mmdc -i x.mmd -o ~/diagrams/x.svg -b white`.
- **Claude Remote Control:** `claude remote-control --name <project>` running in herdr tab "remote-control" in that project's workspace (spawn mode: same-dir). Existing sessions can join with `/remote-control` typed inside them.
- **Codex remote control:** `remote_control = true` added to ~/.codex/config.toml, but codex 0.153.4 (npm) reports the feature "removed" and `codex remote-control start` demands the native install (`curl -fsSL https://chatgpt.com/codex/install.sh | sh`). Left for later; needs a manual install.
- **Laptop awake:** lid-closed on power already works on this MacBook; caffeinate/Amphetamine not needed.

### 5.14 App cleanup (2026-09-08)
Disabled with `adb shell pm disable-user --user 0 <pkg>` (re-enable with `pm enable`): com.onyx.aiassistant, appmarket, easytransfer, mail, musicplayer, voicerecorder, calculator, dict, igetshop. Kept: clock, gallery, floating toolbar (refresh button), reader (kreader), notes.
FUTO Voice Input model names are Whisper sizes by parameter count: English-39 = tiny.en, English-74 = base.en, English-244 = small.en. 74 for speed, 244 for accuracy on the Snapdragon 855.

### 5.15 herdr on the tablet (2026-09-08)
Topology today: `lap` = `ssh lap -t herdr` runs a herdr **client on the laptop** inside the ssh session; Termux only shows the bytes. Everything (server, panes, agents) lives on the laptop.
Alternative: herdr's release binaries are static, so `herdr-linux-aarch64` runs in Termux as-is (`cp` into `$PREFIX/bin`). Installed 0.8.0 to match the laptop. `lapr` = `herdr --remote lap` runs the **client on the tablet** and attaches to the laptop server over ssh: local keybindings (`--remote-keybindings local`), local scrolling, less round-trip per keystroke. Non-interactive ssh into the laptop does find herdr (`/opt/homebrew/bin` is in the sshd PATH). Untested interactively at time of writing; if it works, point the bashrc auto-attach at it. Keep client and server on the same version when running `herdr update` on the laptop.
Result 2026-09-08: `lapr` (client on tablet, 0.8.0 both sides) was much slower on input than `lap`, and it tried and failed to restart the host herdr server (handoff). Not pursued; `lap` stays the default. Revisit only if herdr documents the remote-attach handoff.

### 5.16 The `enabled=3` mystery = Onyx app auto-freeze (2026-09-08)
Boox "freezes" apps by literally disabling the package (`setApplicationEnabledSettingAsUser enable:false` from `ApplicationFreezeHelper`) about a minute after they leave the foreground, and re-enables them when tapped in the launcher. That is why sideloaded apps showed `enabled=3`, why Termux vanished from the app switcher, and why a message typed in the Claude app was lost. Newly installed apps default to auto-freeze (`auto_freeze_newly_install_app`). Tailscale (VPN), FUTO (IME) and Chrome (preinstalled) were exempt.
Fix (GUI only, no adb setting): `adb shell am start -a onyx.settings.action.APP_FREEZE_MANAGEMENT` opens Settings → Apps → freeze management. Turn auto-freeze OFF for Termux, Termux:Boot, Claude, and untick "auto-freeze newly installed apps". The doze whitelist from 5.1 does not cover this.
Recovery if it happens again: `adb shell pm enable --user 0 <pkg>`.

### 5.17 Boot chain after first reboot (2026-09-08)
Observed: Tailscale came up by itself (always-on VPN; Onyx EAC killed it once at boot, Android restarted it). Termux:Boot never received BOOT_COMPLETED: the Onyx system app has a per-app **Auto Start** allowlist (EAC "auto start settings", third-party apps default off). Fix is GUI: `adb shell am start -a onyx.settings.action.app.management` → Auto-start → allow Termux:Boot (and Termux). Also granted Termux `SYSTEM_ALERT_WINDOW` (needed to open its window from a background service on Android 10+), set `bt_on_when_device_boot=1` for the keyboard, and rewrote `~/.termux/boot/00-open-termux.sh` to start sshd first, retry `am start`, and log to `~/.termux/boot.log`. `adb shell am broadcast BOOT_COMPLETED` cannot be used to test (protected); a real reboot is the test, then read boot.log.
Note: with Termux, Termux:Boot and Claude excluded from auto-freeze, a "keep Tailscale open" workaround is no longer needed: the VPN is always-on and exempt from freeze.
Update: the allowlist is Settings → Apps & Notifications → **App Startup** (every third-party app defaults OFF, Tailscale included). Set ON for Termux:Boot, Termux, Tailscale on 2026-09-08. Trick for GUI-only Boox settings: `adb exec-out screencap -p > shot.png`, read the image, then `adb shell input tap X Y` (coordinates in the 3200x2400 native resolution).
NeoBrowser (`org.chromium.chrome`, system app) disabled for user 0 on 2026-09-08 (`pm disable-user --user 0`; the uninstall variant was blocked by the agent sandbox). Brave is the browser. Restore with `adb shell pm enable --user 0 org.chromium.chrome`.

### 5.18 App-switching hotkeys (2026-09-08)
Android 13 has no user-defined global shortcuts and no non-root API to cycle real recents (the Boox recents grid is tap-only; double-recents "last app" doesn't work there either). Implemented a fixed ring, like macOS spaces, with **Key Mapper** (github.com/keymapperorg/KeyMapper, foss APK, sideloaded; accessibility service enabled via `settings put secure enabled_accessibility_services <onyx toolbar>:io.github.sds100.keymapper/.system.accessibility.MyAccessibilityService`).
Ring: Termux → Claude → ChatGPT → Brave → Termux. **Alt+L** next, **Alt+H** previous (each is an "open app" rule constrained on which app is in the foreground; from the launcher `com.onyx`, L→Termux, H→Brave). **Alt+K** recents, **Alt+J** home, **Alt+U** previous app: Key Mapper's built-in "go to last app" only flashes the Boox switcher, so Alt+U is instead "open recents → 400 ms → tap (1600,592)", the second tile of the Boox recents grid in landscape. Breaks if rotation changes or Onyx redesigns the grid.
Config is `keymapper-ring.json` in this folder, generated from Key Mapper's backup schema (db version 22). Load: push to `/sdcard/Download/`, Key Mapper ⋮ → Import → Downloads → file → Replace. adb-injected key combos (`input keycombination`) do NOT trigger Key Mapper; test with the real keyboard. Alt was chosen over Ctrl because Ctrl+H/J/K/L are terminal control characters (Ctrl+J is Claude Code's newline).

### 5.19 Fewer animations, tablet-wide (2026-09-08)
`adb shell settings put global window_animation_scale 0; transition_animation_scale 0; animator_duration_scale 0` (same as Developer options → animation scales). Kills Android UI transitions in every app, and Chromium/Brave reports `prefers-reduced-motion` to websites when the animator scale is 0. In Brave itself, `brave://flags/#smooth-scrolling` → Disabled removes scroll interpolation (the worst offender on e-ink). **Applied 2026-09-08 23:50** via adb: `am start -d brave://flags` only foregrounds Brave (Chromium refuses internal URLs from external intents), so tap the omnibox (1024,274), `input text 'brave://flags/#smooth-scrolling'`, Enter, then tap the dropdown (2360,630) → Disabled (2248,1006) → Relaunch (3002,2130). Flag persists across relaunch; stored in Brave's private `Local State`, not adb-readable. Set Brave's per-app refresh mode to Speed/X in E-ink Center.
**Gotcha:** the X on a Boox recents tile is a *force stop*, and Android drops an accessibility service when its app is force-stopped, so closing Key Mapper that way silently kills all the hotkeys (Key Mapper's own UI does not need to stay open). Recovery: `adb shell settings put secure enabled_accessibility_services "com.onyx.floatingbutton/com.onyx.floatingbutton.service.FloatButtonAccessibilityService:io.github.sds100.keymapper/.system.accessibility.MyAccessibilityService"`. Granted `android.permission.WRITE_SECURE_SETTINGS` to Key Mapper via `pm grant` so it can re-enable its own service from its Fix button without adb.
The floating "accessibility ball" that appears after enabling Key Mapper's service is Android's accessibility button (`accessibility_button_targets` pointing at the service). Not needed; remove with `adb shell settings delete secure accessibility_button_targets`. The service stays enabled.

### 5.20 Firmware 4.2 post-update (2026-09-08 23:37)
Onyx incremental 103 → 226, still Android 13 / TKQ1.230721.002. **Boot chain worked**: Tailscale (always-on VPN) direct, Termux:Boot ran, sshd up, `am start` succeeded on the 2nd try (first try fails with a Parcel error while the app socket is coming up; the retry loop handles it).
Survived: App Startup allowlist, Freeze settings, Key Mapper service + rules, always-on VPN, animation scales, stay-awake, Wi-Fi/BT at boot, FUTO IME, sshd config, all sideloaded apps.
Reset by the update: rotation lock (back to auto), **8 of the 10 disabled system apps re-enabled** (OTA reprocesses system packages), ExKeyMo disabled again (`enabled=3`, cause still not visible in logs; needs `pm enable` and a manual re-select of the layout).
`~/bin/eink-check` on the laptop (tablet on USB) re-applies all of the adb-fixable state idempotently and prints what it changed; run it after any update or whenever something "reverts". The Caps Lock layout re-select and the App Startup/Freeze toggles remain GUI-only.

### 5.21 Laptop/phone → tablet file transfer (2026-09-09 00:16)
`tailscale file cp <f> note-max:` from the laptop prints "note-max is not replying; trying anyway" and hangs (WireGuard ping is fine, 14 ms; the peerapi side is what's not answering). Tailscale Android 1.102.3. Most likely cause: the Android app needs a Taildrop save directory chosen once in its settings (SAF picker, GUI-only, tablet must be free). Not yet fixed. Phone → tablet uses the same receive path, so it is blocked by the same thing.
Working today: `scp <f> note-max:/sdcard/Download/` over Termux sshd. Needed `pm grant com.termux android.permission.{READ,WRITE}_EXTERNAL_STORAGE` + `appops set com.termux LEGACY_STORAGE allow` (Termux targets SDK 28, so the legacy grants give full /sdcard). `termux-setup-storage` did not create `~/storage` symlinks from an ssh session; not needed since /sdcard is directly readable.

### 5.22 High-contrast herdr chrome on e-ink (2026-09-09)
The Boox is run in the recommended fast-refresh mode ("deep" color), where mid-grays smear; catppuccin's muted tab bar and sidebar text were unreadable. Fix in the laptop `~/.config/herdr/config.toml` (the `lap` path runs the herdr client on the laptop, so that config governs): `[theme] auto_switch = true`, `dark_name = "catppuccin"`, `light_name = "gruvbox-light"`. herdr queries the host terminal background (OSC 11; Termux answers it), so the tablet gets the light theme while laptop terminals stay catppuccin. Also removed the hard-coded catppuccin `fg = "#cdd6f4"` / `"#a6adc8"` from the `[ui.sidebar.agents]` rows (kept `dim = false`) so row text follows the active theme. Backup: `config.toml.bak-20260909`. `herdr server reload-config` applies it; reattach (`lap`) to see it.

Rejected: `light_name = "terminal"` (herdr chrome in the host ANSI palette). It draws the selected tab in ANSI bright white, which `~/.termux/colors.properties` deliberately maps to near-black (#1A1A1A) so bright-white TUI text stays visible on the white background, so the selected tab came out black-on-black. `[theme.custom]` tokens (accent, panel_bg, surface0/1, surface_dim, overlay0/1, text, subtext0, mauve, green, yellow, red, blue, teal, peach) apply to *both* themes, so they can't patch the light side alone. Other built-in light names: catppuccin-latte, one-light, solarized-light, kanagawa-lotus, tokyo-night-day, rose-pine-dawn.

Android status bar: Termux paints it solid black. `night-mode = false` in `~/.termux/termux.properties` did **not** lighten it (tried, with an app restart). Went with `fullscreen = true` instead, which hides the bar and gives the terminal the extra rows; `termux-reload-settings` applies it (occasionally needs one more app restart).

Sidebar width (2026-09-09): `[ui] sidebar_width = 25`, `sidebar_max_width = 38` (were 42 / 64, cut ~40% for more pane text on the tablet). One config governs laptop and tablet alike; `prefix+b` toggles the sidebar to the compact rail when more room is needed.

### 5.23 Battery audit (2026-09-09 20:30, `dumpsys batterystats` since the 00:22 charge)
Numbers: 982 mAh of 3858 used in 19h56m. Screen on 1h36m cost 672 mAh (~420 mAh/h, ~11%/h of use). Screen off 18h20m cost 310 mAh (~17 mAh/h, ~8%/day). Active use is the problem, idle is acceptable. Foreground time: Brave 45m, Termux 37m, Claude app 2m. CPU: Termux 11m32s (≈30% of a core while in front; main/UI thread), Brave ≈5m, Tailscale 7m33s, system_server 10m, keymaster-strongbox HAL 32m43s (see idle below).

Findings, active use:
- **herdr pane repaints ~3 fps while idle** (Termux `gfxinfo`: 44 frames / 15 s with nothing happening; matching packet trickle on `tun1`). Each frame = software render of the 2400x3200 terminal + an EPD partial refresh; Termux burns ~7% of a core doing nothing. Source is the *focused* pane's agent while it works: with the eink-screen session idle (another agent still working in an unfocused pane), the stream fell to 2 packets / 30 s and Termux to 2 ticks. `prefersReducedMotion` was already on, so it is Claude Code's status line / streaming output, not the spinner. Unfocused working agents cost nothing. Typing test (x+backspace injected): Termux CPU ≈ 3x the whole display pipeline (surfaceflinger + composer + crtc_commit), so the app render dominates, not the refresh computation. `commit_epdc` kernel thread: 0 ticks.
- **CPU floor pinned**: `scaling_min_freq` little=1478400 big=1804800 (max 1785600 / 2419200), prime 825600. Stays pinned with screen off and with fake unplug. Matches Qualcomm perf hint 0x1086 in `/vendor/etc/perf/perfboostsconfig.xml` (no timeout; stock msmnile uses it as the package-install boost). Nothing installed today, so the holder is unknown. Not writable without root. Reboot test (20:41): floor is identical at 62 s and 200 s after a clean boot, so it is Onyx firmware policy, not a leaked lock. Nothing to do without root. Launch/scroll boosts likewise: configs live on /vendor and the hints come from the framework, no non-root switch.
- Prime core spent 35 min at 2841600 (max): launch boost 0x1081 pins all clusters to max for 2 s on every app switch (the Alt+HJKL ring included) plus scroll boost 0x1080.
- The Tailscale path is direct, Wi-Fi drain is negligible (1.7 mAh); Tailscale cost is CPU per packet.

Findings, idle:
- `onyx_dream_refresh` wakeup alarm (com.onyx, 220 wakes/day) redraws the screensaver clock every 5 min. Each wake also runs the keymaster/StrongBox + `secnvm` flash commits (`SPL` log burst of ~124 lines per wake) — that is the keymaster HAL's 32 min CPU. Known and accepted; it is the screensaver clock. A static screensaver avoids it.
- Wi-Fi is disconnected 92% of the time (sleep), so Tailscale/ssh are unreachable while the tablet sleeps.
- Termux wake lock held ~44 min screen-off (the `lap` alias + boot script). Awake time 2h43m total vs 1h36m screen on.

Tooling: adb-only. Termux cannot read `dumpsys`, `/proc/stat`, or `/sys/class/power_supply` (hidepid + SELinux). Scripts pushed to `/data/local/tmp/` run fine from `adb shell sh`.

### 5.24 Caps Lock layout re-select, scripted (2026-09-09)
The reverted state after a reboot/update: ExKeyMo package `enabled=3` → Android drops the Targus layout to Default (or the next enabled layout) and does not restore it on re-enable. The selection lives in `/data/system/input-manager-state.xml` (system, 0600) behind the signature permission SET_KEYBOARD_LAYOUT; no `settings` key, no `cmd input` subcommand on Android 13. So: `~/bin/eink-capslock` drives the AOSP Settings page with `uiautomator dump` and taps by text (no fixed coordinates). Screens: Physical keyboard page → keyboard row → "Choose keyboard layout" (enabled layouts, or "Default" when none, + "Set up keyboard layouts") → picker, alphabetical Switch rows, scrolled with swipes until the layout is visible → tick → back = current. Tested 2026-09-09 both ways (chooser path, picker path after unticking, idempotent). `eink-check` now calls it (`EINK_SKIP_CAPSLOCK=1` to skip when the tablet is in use). `EINK_LAYOUT="English (US)"` selects another layout; `EINK_UNTICK=1` reproduces the reverted state for testing.

### 5.25 Root: shelved, notes for later (2026-09-09)
This unit already boots with `ro.boot.verifiedbootstate=orange`, `ro.boot.flash.locked=0`, `vbmeta.device_state=unlocked` (bootloader unlocked from the factory; Onyx build TKQ1.230721.002, SoC msmnile = Snapdragon 855, active slot `_b`). Community route (MobileRead / XDA "Onyx Boox Note Max Root (Magisk/APatch)", jdkruzr's Palma 2 guide): `adb reboot edl` → `edl` tool + SD855 firehose loader → `edl r boot_b boot_b.img` → patch with Magisk (or APatch) on the tablet → `edl --memory=ufs w boot_b <patched>` → `edl reset`. No wipe, no unlock step. Costs: incremental OTAs stop (full updates still work, then re-root), Magisk app hangs on 4.1+ without the `boox-ams-fix` module, brick if the wrong image is written (revert = write the saved original back). Would fix the CPU frequency floor (5.23) and make the keyboard layout / Freeze / App Startup state plain file or DB writes. Decision: not worth it now; everything that hurts is scripted.

Adb without the cable: `adb tcpip 5555` once over USB per boot, then `adb connect <tablet-ts-ip>:5555` over Tailscale (laptop key already authorized). Only while the tablet is awake; Wi-Fi is off during sleep.

Still GUI-only, to script with the 5.24 uiautomator pattern when needed: Onyx App Startup allowlist and Freeze settings (`am start -a onyx.settings.action.APP_FREEZE_MANAGEMENT`), Taildrop save directory (5.21).

### 5.26 Termux killed a few minutes after backgrounding = Android "background restricted" at the uid level (2026-09-09)
Symptom: Termux (and its sshd) dies 2-10 min after leaving the foreground even though the package stays `enabled=0`, so it is not the 5.16 freeze. `adb shell logcat -b events -d | grep am_kill` shows `am_kill: [0,<pid>,com.termux,905,cached idle & background restricted]` and `dumpsys activity exit-info com.termux` shows `reason=13 (OTHER KILLS BY SYSTEM)`. Cause: `cmd appops get com.termux` printed `Uid mode: RUN_ANY_IN_BACKGROUND: ignore` above a package-level `RUN_ANY_IN_BACKGROUND: allow`. Android checks the uid mode first, and that is the flag Settings → Battery → "Restricted" (and, apparently, Onyx's power tooling) writes; the 5.1 command without `--uid` only set the package mode. Tailscale had the same uid flag. Fix: `adb shell cmd appops set --uid com.termux RUN_ANY_IN_BACKGROUND allow` (same for com.tailscale.ipn); `eink-check` now checks and re-applies this plus the doze whitelist. Under a restriction, a foreground service does not keep the process out of the cached state, so Termux's "Acquire wakelock" would not have helped either; on this firmware the PowerManager also logs `WakeLock tag:termux:service-wakelock from [com.termux] is forbidden`, so Termux's wake lock is refused outright (Android's own UndimDetector is refused too; Onyx policy, not fixable without root).
Unknown: what flipped the uid mode. It was only set on the two packages from 5.1, so either the original command never covered it and the 5.16 freeze fix masked the kills for a day, or an Onyx power/optimize path restricts non-system apps. Re-run `eink-check` if it comes back and note the date here.
Knock-on: every kill also took Termux's sshd with it, and only the boot script restarts sshd, so `eink-refresh` (and herdr's `ctrl+a f`) failed with "could not reach note-max" while Tailscale was fine. Restarted it over adb (`run-as com.termux sh` with the 5.2 env, then `sshd`) and added `pgrep -x sshd >/dev/null 2>&1 || sshd` near the top of Termux `~/.bashrc` (before the herdr auto-attach), so any Termux launch respawns sshd. Backup `~/.bashrc.bak-20260909`. If `eink-refresh` fails and the tablet is on the tailnet, check `adb shell "ps -A | grep sshd"` first.
