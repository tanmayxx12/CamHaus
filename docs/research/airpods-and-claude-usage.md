# Research: AirPods detection/artwork and Claude usage stats

Written for the v2 design file (`../CamHaus-Designs/CamHaus-Designs-v2.html`). Everything below is
described in our own words; no code was copied from GPL/AGPL projects.

## AirPods: which model is connected?

- Paired AirPods expose an Apple **Bluetooth product ID** (vendor `0x004C`). Read it from
  IOBluetooth paired-device info, or `system_profiler SPBluetoothDataType -json`
  (`device_productID`).
- **Verified on this Mac** (2026-10-29): "T's AirPods" (AirPods 4) report `device_productID 0x2019`,
  `device_vendorID 0x004C`, plus `device_batteryLevelLeft/Right` while connected.
- Product-ID table (see sources):
  | PID | Model | Confirmed by |
  |---|---|---|
  | 0x2002 | AirPods (1st gen) | The Apple Wiki |
  | 0x200F | AirPods (2nd gen) | The Apple Wiki |
  | 0x2013 | AirPods (3rd gen) | The Apple Wiki |
  | 0x200E | AirPods Pro | The Apple Wiki; furiousMAC (as LE `0x0e20`) |
  | 0x2014 | AirPods Pro 2 (Lightning) | The Apple Wiki |
  | 0x2024 | AirPods Pro 2 (USB-C) | MacRumors (reported) |
  | 0x2019 | AirPods 4 | this Mac |
  | 0x200A / 0x201F | AirPods Max (Lightning / USB-C) | The Apple Wiki |
- Unknown `0x20xx` → show generic artwork, log the ID (AirPods 4 ANC and Pro 3 will have their own
  IDs; capture them when seen).

## Battery

- **While connected**: `ioreg` class `AppleDeviceManagementHIDEventService` exposes
  `BatteryPercentLeft/Right/Case` (AirBattery's approach); `system_profiler` mirrors them as
  `device_batteryLevel{Left,Right,Case}`. Event-driven preferred: IOKit notifications over polling.
- **Case-open moment** (the iPhone-style card): AirPods broadcast an unencrypted BLE Continuity
  **proximity-pairing message** (type `0x07`, manufacturer data vendor `0x004C`): model ID,
  left/right battery nibbles, case level, charging bits, lid state. Scan with CoreBluetooth —
  no connection needed. This is why the iPhone card appears the instant the lid opens.

## Artwork (the fix for the ugly AirPods image)

- macOS does **not** ship reusable photoreal per-model AirPods images: checked
  `Bluetooth.appex` and `HeadphoneSettingsUI.framework` asset catalogs with `assetutil`
  (only `B788_airpod_left/right` exists). The iPhone's photoreal images arrive via MobileAsset
  downloads, not in reusable catalogs.
- **SF Symbols ships model-specific glyphs** (verified in CoreGlyphs on this Mac): `airpods`,
  `airpods.gen3`, `airpods.gen4` (+ `.left/.right`), `airpods.pro`, `airpods.max`, and
  charging-case variants (`airpods.gen4.chargingcase.wireless`, …). Legal to use in-app; right for
  small sizes (live activity, status items).
- For the big card, CamHaus uses **its own original vector renders** per model (in the design
  file: parameterized bud/case/Max SVGs keyed by product ID). No Apple screenshots, no other
  apps' assets — consistent with CLAUDE.md licensing rules.

## Claude usage stats (context, 5-hour session, weekly, Fable weekly)

- **Context %**: Claude Code pushes statusline JSON to a configured command:
  `context_window.used_percentage`, `total_input_tokens`, `context_window_size`, plus
  `model.display_name`, `cost.*`, `transcript_path`. Official docs list the schema.
- **Limits** (what `/usage` shows): `GET https://api.anthropic.com/api/oauth/usage` with
  `Authorization: Bearer <token>` (token from `~/.claude/.credentials.json` →
  `claudeAiOauth.accessToken`, short-lived, refresh via `refreshToken`) and header
  `anthropic-beta: oauth-2025-04-20`. Response: `limits[]` entries with `kind`, `percent`,
  `resets_at`; model-scoped entries carry `scope.model.display_name` — that row is the
  Fable/Opus weekly gauge. Legacy flat keys (`five_hour`, `seven_day*`) are being nulled.
  **Not a public API** — poll gently, expect changes, hide the card on failure.
- **Sessions + phases**: Claude Code hooks → local socket + transcript JSONL parsing. This is
  Nook's architecture (GPL-3.0) — reimplement fresh, idea only.

## Claude Code icon

- Official pixel mark fetched from LobeHub's MIT-packaged icon set:
  `https://unpkg.com/@lobehub/icons-static-svg/icons/claudecode.svg` (mono, `currentColor`) and
  `claudecode-color.svg` (brand terracotta `#D97757`). Catalog page: https://lobehub.com/icons/claudecode
  (also mirrored at dashboardicons.com, listed MIT). The mark itself is Anthropic's — fine here,
  since it labels Anthropic's own tool inside a personal app.
- Geometry decoded (24×24 viewBox, 1.5-unit pixel grid): body x3–21 y5–17.08, arms x0–3 / x21–24 at
  y10.95–14.05, eyes x6/x16.5 y8.1–10.95, four legs y17.08–20. **No animated version exists
  anywhere** (checked LobeHub, dashboardicons, the npm package), so CamHaus's walk-and-blink is our
  own: separate body/eye/leg layers, steps() leg alternation + eye scaleY blink, disabled with
  Reduce Motion. In SwiftUI: `TimelineView` or keyframe animations over the same layers.

## Sending stash screenshots into a running Claude Code session

There is **no public API to inject into a running interactive session** — it's terminal input:

- Claude Code reads **image paths typed into the prompt**; drag-and-drop into its terminal inserts
  the path, and **⌃V pastes an image** from the clipboard.
- Hooks tell us which sessions exist (session id, cwd, and the tmux pane when run inside tmux —
  Nook routes "Go to Terminal" this way).
- So "Send to Claude": inside tmux → `tmux send-keys` the stack's file paths (in order) into the
  session's pane; otherwise focus the terminal app and paste the paths (AppleScript/Accessibility),
  or put the image on the clipboard for ⌃V. Alternative for *non-interactive* follow-ups:
  `claude --resume <session-id> -p "..."` appends to a session without its UI, but that is not the
  same as feeding the live prompt.

## Sources

- The Apple Wiki, Bluetooth PIDs: https://theapplewiki.com/wiki/Bluetooth_PIDs
- furiousMAC continuity (proximity-pairing layout): https://github.com/furiousMAC/continuity
- librepods (AGPL/GPL-3.0; protocol behavior reference only): https://github.com/librepods-org/librepods
- AirBattery (AGPL-3.0; ioreg approach reference only): https://github.com/lihaoyun6/AirBattery
- apple-ble (advert parser reference): https://pypi.org/project/apple-ble/
- MacRumors on 0x2024: https://www.macrumors.com/2025/06/26/apple-codebase-unreleased-audio-product
- Claude Code statusline docs: https://code.claude.com/docs/en/statusline
- ccusage (PyPI; documents the usage endpoint + auth): https://pypi.org/project/ccusage/
- claude-limitline (same endpoint, statusline): https://www.pkgstats.com/pkg:claude-limitline
- Nook (GPL-3.0): https://github.com/oa1mgo/nook-notch
