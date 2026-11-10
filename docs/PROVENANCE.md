# Provenance

Every borrowed idea or snippet: source, what we took, how we changed it.

| Date | Source | License | What we took | How we changed it |
|---|---|---|---|---|
| 2026-10-29 | Nook (`../CameraHouseTinkers/nook-main`, github.com/oa1mgo/nook-notch) — verified: it is Nook, GPL-3.0 | GPL-3.0 | Idea only: agent sessions in the notch (hooks → local socket, transcript parsing, phase model working/waiting/idle), in-notch settings list | No code read into ours. CamHaus redesigns it: calmer two-card layout, pure-black panel, adds the `/usage` gauges (5-hour session, weekly, per-model weekly) which Nook doesn't have |
| 2026-10-29 | Apple iOS AirPods case-open card (see user screenshot) | — | The layout idea: device name, buds + case imagery, battery rings with charging bolt | Recreated from scratch in the notch's visual language; original CamHaus vector artwork per model, no Apple assets |
| 2026-10-29 | The Apple Wiki Bluetooth PIDs, furiousMAC/continuity | wiki/docs | Facts: product-ID → model table; proximity-pairing message layout | Facts only, recorded in docs/research/airpods-and-claude-usage.md |
| 2026-10-29 | librepods (GPL-3.0), AirBattery (AGPL-3.0) | GPL/AGPL | Behavior references: BLE advert scanning viability, ioreg battery keys | Described in our own words; no code or assets reused |
| 2026-10-29 | Claude Code statusline docs, ccusage (PyPI) | docs | Facts: statusline JSON schema; the OAuth usage endpoint, fields and auth | Facts only; our reader will be a fresh Swift implementation |
| 2026-10-29 | LobeHub lobe-icons `@lobehub/icons-static-svg` (lobehub.com/icons/claudecode) | MIT packaging; the mark itself is Anthropic's | The Claude Code pixel mark's exact SVG path (`claudecode.svg`), used to label Claude Code sessions in CamHaus | Rebuilt in layers (body/eyes/legs) for our own walk-and-blink animation — no official animated asset is published; brand terracotta #D97757 kept; frozen under Reduce Motion |
