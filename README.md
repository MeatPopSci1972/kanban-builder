# KanbanBuilder

A single-file, zero-dependency kanban board that runs straight from an HTML
file — no build step, no framework, no server. Open it locally or host it on
GitHub Pages.

**Status:** alpha · **Tests:** 81/81 across 10 suites (headless) · **Deps:** none

---

## Features

* **Columns** — add, rename, delete, and reorder. Reorder by drag *or* by
  keyboard (arrow keys on a focused column), so the board is usable without a
  mouse.
* **Tasks** — add to any column, edit, move between columns, delete, and
  reorder within a column (drag or keyboard).
* **Typed task properties** — attach `text`, `text (large)`, `date`, `link`,
  or `image` properties to a task. Each type is validated on the way in and
  formatted for readable display (a link shows its hostname, an image shows
  its filename, a date shows a localized date).
* **Sliding properties panel** — selecting a task opens a panel that slides in
  *right next to that task's column* rather than off in a fixed sidebar. Click
  the task again, or click outside, to dismiss it.
* **Save & load** — export the board to a canonical JSON file and re-import it,
  plus automatic save to `localStorage` on every change.
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
  URL limit; colors must be canonical 6-digit lowercase hex.

* **Pure, DOM-free helpers kept on the testable side of the line.** The
  drag/drop and keyboard-reorder math (`reorderIndexExcludingSelf`,
  `computeDropIndex`), color validation (`validColor`), the panel's two
  interaction decisions (`nextTaskSelection`, `shouldDismissPanel`), and the
  GitHub-link and display formatters are all pure functions. The render/event
  code stays thin; the logic stays pinned. The panel's *animation and DOM
  placement* deliberately stay in the UI (manual-verify), but the *decisions*
  behind them are extracted and tested.

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

The board ships with **81 behavioral pin-down tests across 10 suites**, and
they run **headless — no browser required.**

* **What they cover:** the full store contract (every action's accept/reject
  behavior), the serializer (including the export→import→export round-trip),
  and every pure helper (positioning math, color validation, panel decisions,
  GitHub-link building, display formatting).

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

* **Task color visual** — the store already validates and persists a per-task
  `color`, but the card doesn't paint it yet (the "gradient under a wire mesh"
  card treatment is unbuilt) and there's no picker wired up. This is UI work in
  the render path.
* **Multi-board workspace** — tabs, "New Board," and board name/color are
  designed as a UI + persistence layer on top of the existing import/export
  seam (the store stays single-board). Not built.
* **Rendering review** — the current renderer does a full teardown-and-rebuild
  on every change (no diffing, no event delegation). Whether to keep that
  hand-rolled simplicity or introduce reconciliation is an open architectural
  question, to be settled before the color visual and workspace layer land.

## Engineering philosophy

Built with a *proto2prod* discipline: validate before optimizing, single-file
prototypes first, visible errors over graceful degradation, naming consistency
treated as load-bearing, and every change gated by the test suite. A companion
`HANDOFF_kanbanbuilder_alpha.md` documents current state for both humans and a
fresh AI instance picking up the work.

## License

See [LICENSE](LICENSE).
