# CamHaus

A native macOS app that lives in the MacBook notch (Apple's term: the "camera housing").
My workflow tool first: a stash for screenshots that reach a Claude Code session without a
drag-and-drop dance, plus a few notch features borrowed from other apps and rebuilt properly.
Quality bar: it should feel like Apple shipped it.

The design is settled before the code is. `docs/design/CamHaus-Designs.html` is the spec —
open it in a browser. Every state in its Playground is a state the app must have, and its
Specs tab holds the geometry, motion, type, colour and routing rules. Numbers in it are
estimates unless Specs says otherwise; real geometry always comes from `NSScreen` at runtime.
The design will keep changing, so the codebase has to make re-skinning cheap. That is the
main constraint on every structural decision below.

## Before any app code: we pick the architecture together

Do NOT scaffold features until this is settled. First session: read this file, the design
file and the reference repos, then write `docs/adr/0001-architecture.md` comparing these,
and stop for my approval.

| Option | Shape | The usual catch |
|---|---|---|
| MV | `@Observable` models in the environment, views read them directly. Apple's current sample-code style. | Logic leaks into views unless you're disciplined |
| MVVM | One `@Observable` view model per pane | Boilerplate; view models often end up passthroughs that fight SwiftUI's own state |
| Unidirectional store | One app state, actions, reducers, hand-rolled | Plumbing you maintain yourself |
| TCA | Point-Free's Composable Architecture | Large dependency, steep curve, slow builds |
| Layered / Clean with use cases | Explicit application layer | Ceremony far beyond this app's size |

Score them against what this app actually is, not against general good taste:

1. Cost of a design change — the design churns; what does a re-skin touch?
2. Fit for the notch's presentation state, which is a real state machine
   (closed / peek / live activity / open×pane) with interruptible animations.
3. Testability of the pure parts (geometry, stash order and undo, routing, battery
   thresholds) with no Mac in the loop.
4. Handling of OS-facing async side effects: screenshot watcher, Bluetooth, media,
   Claude routing.
5. Swift 6 strict-concurrency fit.
6. Build time and dependency weight for one person.
7. How many files a one-line feature change touches.

Recommend one, say plainly what it costs, and note what would make you change your mind.
Record the outcome in the ADR and keep every later ADR in `docs/adr/`.

These hold whichever option wins, so structure can start before the decision lands:
layering, dependencies behind protocols, views as pure functions of state, design tokens
in one place.

## SwiftUI first; AppKit only where it is the only way

This is macOS, so the fallback is AppKit, not UIKit. SwiftUI is the default for everything.

A notch app cannot be written in pure SwiftUI — these are known-necessary and already
settled, so don't re-litigate them, just keep them contained:

- `NSPanel`, borderless and non-activating, above the menu bar on every Space.
  SwiftUI has no window type that does this.
- `NSScreen` geometry (`safeAreaInsets`, `auxiliaryTopLeftArea`, `auxiliaryTopRightArea`,
  `backingScaleFactor`) and `NSApplication.didChangeScreenParametersNotification`.
- `NSEvent` monitors for hover and hotkeys while the app is never frontmost.
- `NSPasteboard` and drag sessions for dragging screenshots out.
- `NSWorkspace`, `SMAppService`, `NSOpenPanel`.
- ImageIO / Core Graphics for downsampling artwork.
- `NSHostingView` to put SwiftUI inside the panel.

For anything else, before writing AppKit: name the SwiftUI API you tried and why it fell
short. If you can't, it isn't necessary. Every such use carries this comment and a matching
line in `docs/APPKIT.md`:

```swift
// AppKit: <type or API>
// SwiftUI can't: <the specific limitation, naming the SwiftUI API that fell short>
// Contained to: <what is wrapped; what stays SwiftUI>
// Delete when: <the SwiftUI capability that would let this go>
```

Wrap it in a SwiftUI-shaped type — a `View`, a `ViewModifier`, or a service behind a
protocol. `NS*` types never appear in a feature's public API.

## Built to be re-skinned

- Local Swift packages so the compiler enforces layering: `Core` (pure Swift, no AppKit),
  `DesignSystem`, `Services` (protocols plus live implementations), `NotchShell`,
  `Features/*`. The app target wires them together. `NotchShell` must not import `Features`:
  the shell knows about panes, never about stashes or AirPods.
- Every visual constant lives in `DesignSystem`: colour roles, spacing scale, type roles,
  motion tokens, geometry constants. No raw numbers or colours in a feature view. A re-skin
  should mostly be a token change.
- A pane is data, not a special case. One `NotchPane` protocol (id, symbol, badge,
  preferred size, body); adding, removing or reordering tabs edits an array.
- Views are pure functions of state. No view reaches for a singleton or a global.
- Every pane has SwiftUI previews for every state the design file shows, including empty,
  denied-permission, and Reduce Motion.
- When the design file changes: diff it, list what moved, change tokens first, then layout.
  Say which parts of the app the change doesn't reach.

## Coding principles

- Swift 6 strict concurrency, `MainActor` by default; background work in actors or
  explicitly `nonisolated`.
- State is value types. `@Observable` reference types only where identity or lifetime
  genuinely matters.
- Every OS-facing dependency is a protocol with a live, a preview and a test
  implementation. `Core` compiles without AppKit.
- One type per file. A file past ~250 lines gets split or earns an explanation.
- Typed errors; no empty catch. Anything swallowed gets a log line and a visible fallback.
- No force unwraps outside previews and tests.
- Swift API Design Guidelines for naming.
- Don't add a third-party dependency without asking; say what it costs and what it replaces.
- Unit-test the pure parts: geometry, stash order and undo, routing, battery thresholds,
  shortcut matching. No UI tests for the notch.
- Idle CPU near zero: event-driven, no polling where a notification exists, no work while
  the notch is closed and nothing is playing.
- No network calls or analytics unless a feature needs them and I've approved it.

## Project setup (done; don't change without asking)

- `CamHaus.xcodeproj`: target and scheme `CamHaus`, SwiftUI app with test targets.
- Agent app (`LSUIElement` = YES): no Dock icon, no app menu.
- Not sandboxed. Revisit before any App Store plan; log it in `docs/adr/`.
- Signed with my Apple Development identity so macOS keeps granted permissions across builds.
- Synchronized folders: create files on disk under the source folders and Xcode picks them
  up. Don't hand-edit `project.pbxproj`. If a build setting, capability or target needs
  changing, tell me exactly what to change in Xcode.
- Build and test with
  `xcodebuild -project CamHaus.xcodeproj -scheme CamHaus -destination 'platform=macOS' build test`
  before calling anything done.
- Keep a hidden debug overlay that draws the computed notch rect on a pixel grid.

## Reference repos (read-only)

In `../CameraHouseTinkers/`, attached with `--add-dir`. Never modify them or lift files.

- `boring.notch-main` (GPL-3.0): animations, permission onboarding, notch sizing.
- `Notchmeister-main` (BSD-3-Clause): only the Expando and EyeMac effects.
- `NotchDrop-main` (MIT): an anti-reference for UI; its shelf and drag internals may help.
- `nook-main`: confirm what it is before using it. Feature ideas only.

Licensing is not negotiable. Never paste GPL code into this repo: read it, describe the
behaviour in your own words in `docs/research/`, then write a fresh implementation.
MIT/BSD code may be adapted with its copyright notice kept in the file and listed in
`THIRD_PARTY_NOTICES.md`. No other app's name, icon or assets. Log every borrowed idea in
`docs/PROVENANCE.md`: source, what we took, how we changed it.

## Scope, in priority order

1. Notch shell: closed state indistinguishable from the hardware, correct on every display
   setup (see Edge cases).
2. Screenshot stash and the capture tray, including routing into a live Claude Code session.
3. Claude sessions: the Home card and the full pane.
4. Now Playing, with artwork that is never stretched past its pixels.
5. Onboarding and permissions.
6. AirPods battery.
7. Expando and EyeMac: opt-in, off by default.

## Design rules (Apple HIG)

Source of truth: https://developer.apple.com/design/human-interface-guidelines
Name the HIG section you followed in commit messages for UI changes.

- Hardware continuity: the notch body is pure black with concave ears and rounded bottom
  corners. Opening animates size and shape, never opacity.
- Pixel-perfect geometry: one type owns it, computed from `NSScreen`, snapped to device
  pixels, recomputed on screen changes. No per-model hard-coded sizes.
- Native look: system font through text styles, SF Symbols, semantic colours, standard
  controls before custom ones. AirPods art is SF Symbols, never a fake product render, and
  no Bluetooth product IDs in the UI.
- Motion: springs from the named tokens, interruptible, honouring Reduce Motion, Reduce
  Transparency and Increase Contrast.
- Accessibility: VoiceOver labels and actions on every control, full keyboard navigation in
  the open notch, comfortable hit targets.
- Permissions: ask at the moment of need with a one-line reason first. The app keeps
  working, degraded, when one is denied, and links to the right Settings pane.
- Calm: no modal alerts for routine actions. Destructive actions are instant and undoable
  rather than confirmed.

## Edge cases (definition of done for the shell)

No notch (external display, lid closed, non-notched Mac); multiple displays and rearranging
them; per-display scale factors; scaled resolutions; full-screen apps, Spaces, Mission
Control, Stage Manager, auto-hiding menu bar; sleep/wake, lock, fast user switching; a
pointer flicking past (hover intent); a drag hovering the notch opens it for drops; never
stealing focus from the frontmost app; light and dark menu bars; Reduce Motion, Reduce
Transparency, Increase Contrast, VoiceOver.

## Decisions still open (ask me; record in docs/adr/)

- Architecture (the session above).
- Sandbox, distribution and CamHaus's own licence.
- Stash: move originals out of the screenshots folder, or copy them?
- Default screenshot routing: the `UserPromptSubmit` hook or the MCP server.
- Which Nook-inspired features make the cut.

## How we work

- Plan first and wait for my OK, then build. One feature per session and branch; small commits.
- Keep `docs/` current: the ADRs, `APPKIT.md`, `PROVENANCE.md`, `PLAN.md`.
- Taste calls (visuals, motion feel, shortcuts) are mine — show me options.
  Correctness calls are yours — decide and explain why.
- If I correct you on the same thing twice, propose a line for this file.
