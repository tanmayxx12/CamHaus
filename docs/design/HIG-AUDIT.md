# HIG audit — CamHaus-Designs-v2.html

Audited 2026-11-11 against the [Apple Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines)
(sections named per finding). Source: `CamHaus-Designs-v2.html` (2,008 lines), reviewed in
full plus a live pass through every Playground state, the Onboarding flow and the Specs tab.
Line numbers refer to that file.

Verdict up front: this is an unusually HIG-literate design file. The findings below are
mostly refinements and decisions to record, not rework. The three High items should be
settled before the shell is built, because code will inherit them.

---

## What the design already gets right

Worth listing so we don't accidentally regress any of it:

- **Hardware continuity** (HIG *Materials*, plus our own Geometry rule): pure `#000000`
  body in both appearances, concave ears, rounded bottom corners, open panel stays pure
  black, nothing ever drawn in the housing (`notchPath()`, the anatomy diagram, rule list
  at 826–832).
- **Motion** (HIG *Motion*): one small set of named springs with SwiftUI semantics
  (`response`/`dampingFraction`, 1014–1015, 1053–1078), interruptible by design, shape
  leads and content follows, and a real Reduce Motion fallback — instant size change +
  150 ms cross-fade (62–64, 1061), frozen EQ bars (79) and crab (236).
- **Forgiveness** (HIG *Undo and redo*, *Feedback*): delete is instant, never confirmed,
  always undoable — Undo chip in the closed-notch live activity (461–464), toast with
  Undo (567), ⌘Z (1712), batch-aware undo that restores order (1571–1579).
- **Permissions** (HIG *Privacy*): explain-first onboarding with a real "Not Now" path,
  ask-at-the-moment-of-need primer in the stash pane, a designed **denied** state that
  links to the right Settings pane (1493–1499), and the fineprint note that the Info.plist
  purpose string is onboarding copy too (805). This is exactly the HIG pattern.
- **Accessibility scaffolding** (HIG *Accessibility*): `aria-live` announcer for stash
  events (944, 1876), `listbox`/`option` semantics with multi-select on the strip (533),
  labelled tabs and controls, an Increase Contrast variant that raises on-notch opacities
  (65), shortcut recorder that requires a modifier (1927).
- **Typography** (HIG *Typography*): the type ramp in Specs (1976) matches the real macOS
  text-style sizes point for point; tabular numerals everywhere numbers change.
- **Color** (HIG *Color*, *Dark Mode*): the notch always uses the dark-variant system
  palette (`#30d158`, `#ff453a`, `#ff9f0a`, `#0a84ff`) regardless of appearance — correct,
  since the surface is always black. Battery green/red threshold at 20% matches the system.
- **Calm** (HIG *Feedback*, *Managing notifications*): transient live activities are
  2.4–3.6 s; only a waiting Claude session may hold one indefinitely (919); effects are
  opt-in and off by default; `cursor:default` on buttons is the correct macOS convention.
- **Artwork honesty**: low-res artwork is never stretched past its pixels — ambient blur
  fill + sharp native-size inset (119–120, 1276–1283). AirPods art is original vector work
  keyed off the product ID, no Apple assets.

---

## Findings

### High — settle before the shell is built

**H1. Tertiary on-notch text fails contrast in the default appearance.**
`--on-3: rgba(255,255,255,.38)` on pure black is ~**3.4:1**; WCAG AA (and HIG
*Accessibility > Color and effects*) wants 4.5:1 for text this size. It would be fine for
disabled or decorative text, but the design uses it for information-bearing text: scrub
times (128), stash footer shortcut hints (161), the AirPods footer line (199), Claude
context row (217), file sizes (184), settings footer (247). The Increase Contrast variant
(.66 → ~8.8:1) already fixes it — which proves the default is too low.
*Fix:* promote info-bearing uses to `--on-2` (.62, ~7.8:1 — passes), or raise `--on-3`
to ≥.48. Reserve true tertiary for genuinely decorative text. Encode this as a DesignSystem
rule: tertiary never carries information.

**H2. Bluetooth product ID shown in the UI.**
The AirPods pane footer renders `AirPods 4 · 0x2019 · Case charging` (1396). CLAUDE.md
explicitly bans product IDs in the UI, and HIG spirit agrees: never expose technical
internals. The Specs tab documenting the PID table is fine — that's documentation.
*Fix:* footer shows model name + charging state only; log the PID.

**H3. The keyboard model inside a non-activating panel is undesigned.**
The Keyboard spec (872, 1958–1962) promises arrows, ⌫, Space, ⌘A/⌘C/⌘Z inside the notch
while the app is "never frontmost". On macOS those are in tension: a `.nonactivatingPanel`
must become **key** to receive keystrokes, and while it is key the frontmost app's window
loses them. The design never says when the panel takes key status and when it doesn't.
*Fix (proposal to record in the architecture ADR):* hover-open never takes key focus
(pointer-only interactions work; keystrokes still go to the front app); shortcut-open
(⌃⌥S) and tabbing into the notch make the panel key, shown by the focus ring; closing
returns key to the previous app. This also decides how Space/Quick Look works (see M4).

### Medium

**M1. 9 px badge text breaks the design's own floor.**
`.tab-badge` is 9px (93) against the Specs rule "nothing smaller than 10 pt" (849).
*Fix:* 10 pt minimum, or drop the number for a dot when the count isn't essential
(HIG treats badges as noncritical, glanceable info).

**M2. Hover-only destructive affordances; shelf delete has no keyboard path.**
The per-thumb ✕ (155–156), per-file ✕ (185–186) and Quick Look ✕ appear only on hover at
16–18 px. The stash has ⌫ as the real keyboard path (good), but shelf files have none, and
16 px is below a comfortable click target (HIG *Accessibility > Buttons and controls*;
CLAUDE.md "comfortable hit targets").
*Fix:* keep hover reveal, but make targets ≥20×20 with padding, give shelf files focus +
⌫, and make sure VoiceOver exposes the remove action via accessibilityActions (the custom
rotor/actions pattern), not a hidden hover button.

**M3. "Open at login" is pre-checked** (789). HIG *Settings* and plain courtesy: launch
items are the user's call, and macOS posts a "login item added" notification that will
feel like a surprise if the user never chose it. The toggle on the final step is the right
place; it should just default to **off** — flipping it on is one click for someone who
wants it. (Taste call is yours; this one is cheap to change now.)

**M4. Custom Quick Look instead of the system one.**
The mock draws its own QL window (280–285). The real app will have actual files on disk;
`QLPreviewPanel` gives zoom, markup, share and correct behavior for free, and is what users
expect Space to do (HIG *File management*). It also sidesteps H3 for the preview itself
(the QL panel is a normal key window). Keep the custom overlay only if QLPreviewPanel
genuinely can't be summoned from a non-activating panel — if so, note it in APPKIT.md.

**M5. Toast with an action auto-dismisses in 4.2 s** (1589). WCAG 2.2.1 (timing) and HIG
*Feedback*: an actionable control shouldn't race the user. ⌘Z persisting after the toast
(it does — the undo stack survives) is the saving grace.
*Fix:* ≥6 s when the toast carries a button, pause the timer while hovered, and mention
⌘Z in the toast's VoiceOver announcement.

**M6. Missing states** (the design file's Playground is the state inventory, so gaps here
become untested code paths):
- **Home with nothing playing** — the Home pane always shows a track. What's the
  empty Home? (Calendar card full-width? A quiet placeholder?)
- **Claude pane with no sessions** / Claude Code not installed / usage endpoint down
  (Specs says "hide the card rather than show stale numbers" (918) — design what shows
  instead).
- **Previous stacks**: "New stack" starts Stack n+1, but nothing shows where Stack n
  went or how to get it back. Even a one-line answer ("stacks are just history; the strip
  always shows the current one") should be written down, because it decides the data model.
- **AirPods pane when disconnected** (pane is reachable only while connected? say so).
- **AirDrop in progress / done / failed** on the shelf.
- **Non-notched display presentation** — CLAUDE.md's edge-case list requires behavior on
  external displays and non-notched Macs; the design has no answer yet (fake notch strip?
  menu-bar item? nothing?). This one shapes the shell architecture.

### Low

**L1. Mock menu bar is 4 px taller than the notch** (`--menubar-h: notchH + 4`, 1146).
On real notched Macs `safeAreaInsets.top` *is* the menu bar height; they're the same
number. Harmless in the mock — just make sure the geometry type never copies it.

**L2. Non-text-style font sizes.** 10.5/11.5/12.5 px sprinkled through the CSS. Fine in
HTML; in SwiftUI every one of these should snap to a named text style (Body 13, Callout 12,
Subheadline 11, Footnote 10) so Dynamic Type-style scaling and consistency come free.
Map them once in DesignSystem; never `.font(.system(size:))` in a feature view.

**L3. Hidden scrollbars** on the strip (150) and settings pane (240). macOS overlay
scrollbars already fade; fully hiding them removes the only scrollability affordance.
Prefer the system behavior, or add edge fades that imply more content.

**L4. Always-running animations while visible.** The EQ bars and the crab walk animate
continuously (73–78, 228–235). Both honor Reduce Motion (good). For idle-CPU discipline,
drive them with a slow `TimelineView` tick (the crab is already `steps(1)` at ~2 fps —
cheap; keep it that cheap in Swift) and pause the EQ when the notch strip is occluded by
a full-screen app.

**L5. Blue unseen-count badge** (93). System notification badges are red; blue reads as
"information", which may be exactly the calm you want. Taste call — noting it so it's
chosen, not inherited.

---

## Decisions this audit adds to the ADR agenda

1. **Key-window policy** for the panel (H3) — when does the notch take keyboard focus?
2. **QLPreviewPanel vs custom preview** (M4).
3. **Stack lifecycle** (M6) — are previous stacks recoverable, and from where?
4. **Non-notched display behavior** (M6) — it changes what `NotchShell` even is.

## Suggested token changes (cheap now, expensive later)

| Token | Now | Change |
|---|---|---|
| `--on-3` | `rgba(255,255,255,.38)` | `.48`, and never on info-bearing text |
| badge text | 9 px | 10 pt min, or a dot |
| hover ✕ targets | 16–18 px | ≥20 px with inset padding |
| toast timeout | 4.2 s | ≥6 s with a button; pause on hover |
| login item default | on | off |
