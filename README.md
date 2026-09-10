# Boox Note Max as a thin client for coding agents

Notes from setting up a Boox Note Max (Android 13 e-ink tablet) as the display and keyboard for
Claude Code and Codex running on a laptop, connected over Tailscale and multiplexed with
[herdr](https://herdr.dev). Nothing runs on the tablet except Termux (ssh), a browser, and the Claude app.

- **[boox-agent-setup-runbook.md](boox-agent-setup-runbook.md)** — the runbook. Sections 1–4 are the plan
  (laptop, tablet, phone, acceptance test). Section 5 is what actually happened, in order, including every
  Onyx-specific gotcha: sideloaded apps silently disabled (`enabled=3`), the auto-freeze and App Startup
  allowlists, Tailscale dying on sleep, firmware updates reverting state, a battery audit, and a
  root-without-root assessment.
- **`bin/`** — laptop-side helper scripts referenced in the runbook (install to `~/bin`):
  - `eink-check` — re-applies every adb-fixable tablet setting after a firmware update or a freeze. Idempotent.
  - `eink-capslock` — re-selects the Caps Lock→Ctrl keyboard layout by driving the AOSP Settings UI
    with `uiautomator dump` and taps by text (no fixed coordinates). Pattern for any GUI-only setting.
  - `eink-refresh` — forces a full e-ink (GC16) refresh over ssh via the Onyx `REFRESH_SCREEN` broadcast.
  - `diagrams` — serves `~/diagrams` as static SVGs on the Tailscale interface only.
- **`keymapper-ring.json`** — [Key Mapper](https://github.com/keymapperorg/KeyMapper) backup implementing a
  fixed app ring (Alt+H/L previous/next, Alt+J home, Alt+K recents, Alt+U last app) for a hardware keyboard.
  Ring is Termux → Claude → ChatGPT → Brave; edit the package names to taste.

Placeholders like `<laptop-magicdns>` and `<tablet-ts-ip>` stand for values specific to one tailnet.
Dates are when a step was done; the firmware was Onyx 4.2 (Android 13, TKQ1.230721.002) at the time.
