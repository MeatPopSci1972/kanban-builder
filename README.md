# KanbanBuilder

A single-file, zero-dependency kanban board that runs straight from an HTML
file — no build step, no framework, no server. Open it locally or host it on
GitHub Pages.

**Status:** alpha · **Tests:** 100/100 across 12 suites (headless) · **Deps:** none

---

## Features

* **Columns** — add, rename, delete, and reorder. Add is per-column: a `+`
  button beside each column's move handle inserts a new column immediately to
  its right. An empty board shows an "add your first column" affordance so it's
  never a dead end. Reorder by drag *or* by keyboard (arrow keys on a focused
  column), so the board is usable without a mouse.
* **Tasks** — add to any column, edit, move between columns, delete, and
  reorder within a column (drag or keyboard). Adding a task is prompt-free: it
  drops a card in immediately and puts your cursor straight in the panel's Title
  field (text pre-selected) to name it inline. The auto-assigned default is
  de-duped — repeated adds read `New task`, `New task 2`, `New task 3`… rather
  than a pile of identical cards. A title you type yourself is never touched, so
  duplicate names are still allowed where you want them.
* **Typed task properties** — attach `text`, `text (large)`, `date`, `link`,
  or `image` properties to a task. Each type is validated on the way in and
  formatted for readable display (a link shows its hostname, an image shows
  its filename, a date shows a localized date).
* **Task color** — give any task a color from a native swatch, or clear it back
  to none. Colored cards paint a *gradient under a wire mesh* — a soft diagonal
  glow of the color beneath a faint wireframe of the same hue — with a left
  accent stripe. The blank-swatch default follows the active theme's accent.
* **Sliding properties panel** — selecting a task opens a panel that slides in
  *right next to that task's column* rather than off in a fixed sidebar. Click
  the task again, or click outside, to dismiss it. When the column sits far
  right, the board scrolls just enough to keep the whole panel on-screen.
* **Save & load** — export the board to a canonical JSON file and re-import it,
  plus automatic save to `localStorage` on every change. Each *copy* of the file
  keeps its own storage slot (see the per-deployment ULID key below), and a
  **Clear Store** button wipes the saved board and resets to a fresh default
  after a confirm.
* **Send to GitHub Issue** — turn a task into a pre-filled
  `github.com/.../issues/new` link (title, body, and `kanban` + column labels),
  opened in a new tab. **Zero credentials** — no token, no API call; your
  existing browser session does the rest.
* **Accessible** — keyboard-operable throughout, targeting **WCAG 2.2 AA**.
* **Portable** — one `.html` file. Works from `file://` or GitHub Pages.

## Notable engineering

* **Single file, zero dependencies, no build.** All markup, styles, logic, and
  tests live in one `kanban-builder.html`. Nothing to install; nothing to
  compile.

* **A hard store / UI boundary.** All state lives behind a store. UI code never
  mutates state directly — every change goes through
  `store.dispatch({type, ...})`, which returns `{ok: true, ...}` or
  `{ok: false, error}`. Rejected actions never mutate state and never fire a
  `change` event. This one rule is what lets the entire store be pinned down by
  tests without ever touching a DOM — and it's why two features built in
  separate sessions (task color in the store, the panel redesign in the UI)
  merged with **zero conflicts**: they never touched the same lines.

* **A canonical serializer with a round-trip guarantee.** `exportBoard`,
  `validateBoard`, and `buildStateFromBoard` share one fixed envelope
  (`{app, schemaVersion, exportedAt, columns, tasks}`). Export → import →
  export is byte-identical, and `validateBoard` is the single gate every
  import must pass — one door, not one per entry point.

* **Visible errors over graceful degradation.** Validation lives at the store
  level and *rejects with a message naming what's wrong* rather than silently
  coercing, truncating, or defaulting. Examples: images must be relative path
  strings (never `data:` base64 in JSON, ≤ 500 chars); dates must parse and
  fall within 1900–2100 (added after a native date input was found to accept
  a year like `0006`); large text is capped at GitHub's documented 8191-char
  URL limit; colors must be canonical 6-digit lowercase hex; and a saved board
  that can't be loaded surfaces a toast rather than silently resetting to a
  fresh one.

* **Pure, DOM-free helpers kept on the testable side of the line.** The
  drag/drop and keyboard-reorder math (`reorderIndexExcludingSelf`,
  `computeDropIndex`), color validation (`validColor`), the de-duped default
  task title (`uniqueDefaultTitle` — applied only to the system-generated
  placeholder, never to a typed title), the panel's interaction decisions
  (`nextTaskSelection`, `shouldDismissPanel`, `computePanelScroll`), and the
  GitHub-link and display formatters are all pure functions. The render/event
  code stays thin; the logic stays pinned. The panel's *animation and DOM
  placement* deliberately stay in the UI (manual-verify), but the *decisions*
  behind them are extracted and tested.

* **Two subtle correctness details worth knowing.** *(1) Keeping the panel
  on-screen computes against its **final** width, not its live one.* The panel
  opens with a `0 → 340px` width transition, so measuring its width right after
  insertion would read ~0 and under-scroll; using the known `--panel-w` makes
  the reveal correct mid-slide — and also when re-opening onto an
  already-open panel, where there's no transition to wait on. *(2)
  Click-outside dismissal ignores a detached target.* A panel button whose
  dispatch rebuilds the panel body detaches the very element you clicked before
  the document-level handler runs; that handler early-returns when the click
  target is no longer in the document, so in-app controls (Clear, a
  mouse-clicked **+ Add**) never misread as outside clicks and collapse the
  panel.

* **Per-deployment storage isolation via a baked ULID.** The `localStorage` key
  is `kanbanbuilder_board_v1_<ULID>`, where the ULID is a constant baked into
  each copy of the file. Two different deployments on one origin — a fork, or a
  second copy on the same GitHub Pages site — land in separate slots instead of
  stomping one shared key. The `v1` prefix keeps the schema-version namespace
  intact for a future migration; the ULID only adds uniqueness. On a new
  deployment whose slot is empty, the pre-ULID `kanbanbuilder_board_v1` board is
  adopted once (the next save writes it into the new slot) and the legacy key is
  never deleted, since another tab or deployment may still read it. A malformed
  baked ULID falls back to the legacy key and surfaces a boot notice rather than
  keying off garbage. *(This isolates deployments, not tabs: two tabs of the
  **same** file share one ULID, hence one board — that's the frozen multi-board
  question, not this.)* The `isUlid` / `boardStorageKey` logic is pure and
  tested; the constant lives at boot.

* **Shared constants and validators, not duplicated magic.** One
  `GITHUB_URL_BYTE_LIMIT` serves both the issue-link truncation guard and the
  large-text cap. One `validColor` is designed to serve both task color and a
  future board-color picker. Reuse the existing number/validator rather than
  inventing a fresh one that happens to match.

* **Sandbox-safe in-page dialogs.** `showPromptDialog` / `showConfirmDialog`
  replace native `prompt()`/`confirm()`, which silently return `null` inside
  sandboxed iframes (e.g. in-chat file previews). The custom dialogs work
  everywhere regardless.

* **Verified relative image paths where the platform allows.** When hosted
  over HTTPS on a supporting (Chromium) browser, the image picker uses the
  File System Access API to resolve a *verified* relative path; elsewhere it
  falls back to a clearly-marked `./images/<name>` guess rather than pretending
  it's confirmed.

## Tests

The board ships with **100 behavioral pin-down tests across 12 suites**, and
they run **headless — no browser required.**

* **What they cover:** the full store contract (every action's accept/reject
  behavior, including column insertion to the right of a given column via
  `ADD_COLUMN`'s `afterId`), the serializer (including the export→import→export
  round-trip), and every pure helper (positioning math, color validation, the
  de-duped default-title picker `uniqueDefaultTitle`, panel decisions,
  GitHub-link building, display formatting, and the deployment storage-key
  helpers `isUlid` / `boardStorageKey`).

* **What they deliberately don't cover:** anything that touches a real browser —
  the panel's CSS slide, the actual DOM insertion, the file picker, and the
  in-page dialogs. Those are manual-verify by design. The pure-function
  extraction exists precisely so the *logic* around those UI pieces is still
  pinned even though the DOM behavior isn't.

* **The gate:** the suite must be green before anything ships. The test modal
  in the UI reports the count dynamically (there's no hardcoded total to drift).

Run the gate from the repo root:

```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('kanban-builder.html','utf8');
const start = html.indexOf('const PROP_TYPES');
const end = html.indexOf('── UI STATE ──');
const src = html.slice(start, html.lastIndexOf('/*', end));
eval(src);
const results = runTests();
const failed = results.filter(r => !r.pass);
console.log(results.filter(r => r.pass).length + '/' + results.length + ' passed');
failed.forEach(f => console.log('FAIL:', f.suite, '|', f.name, '|', f.msg));
process.exit(failed.length ? 1 : 0);
"
```

Full-script syntax check before shipping:

```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('kanban-builder.html','utf8');
const start = html.indexOf(\"'use strict';\");
const end = html.lastIndexOf('</script>');
new Function(html.slice(start, end));
console.log('syntax OK');
"
```

## Running it

No install. Either:

* **Open locally** — double-click `kanban-builder.html` (works from `file://`;
  the verified image picker and any HTTPS-only APIs are the only things that
  need hosting).
* **Host on GitHub Pages** — serve the file over HTTPS to unlock the full
  image-picker path.

## Roadmap (designed, not yet built)

Called out honestly so contributors don't assume these exist:

* **Multi-board workspace — frozen.** Tabs, "New Board," and board name/color
  were designed as a UI + persistence layer over the import/export seam (the
  store stays single-board), but this is **parked pending a future concept that
  must be imagined first.** One caveat for anyone tempted to skip it: browser
  tabs won't stand in for real multi-board — every tab of the *same* file shares
  one `localStorage` key (the file's baked ULID), so they show the *same* board
  and overwrite each other on save. The per-deployment ULID key isolates
  *different* copies of the file, not tabs of one; genuine multi-board within a
  single file still needs per-board keys or the envelope design. For separate
  boards today, use export/import to JSON, or a second copy of the file (its own
  ULID).
* **Rendering posture — decided (for now).** The renderer does a full
  teardown-and-rebuild on every change (no diffing, no event delegation), kept
  deliberately under a validate-before-optimize discipline. The plan graduates
  to keyed reconciliation only when a concrete tripwire trips — an in-panel
  element that must animate *across* a state change, or a non-panel node whose
  correctness depends on surviving a rebuild (e.g. a future tab strip's active
  state) — not before.

## Engineering philosophy

Built with a *proto2prod* discipline: validate before optimizing, single-file
prototypes first, visible errors over graceful degradation, naming consistency
treated as load-bearing, and every change gated by the test suite. A companion
`HANDOFF_kanbanbuilder_alpha.md` documents current state for both humans and a
fresh AI instance picking up the work.

## License

See [LICENSE](LICENSE).
