# KanbanBuilder — Handoff

Read this before touching the file. Attach the current `kanban-builder.html`
to whatever thread you paste this into — this document is context, not a
substitute for the file itself.

**This revision exists to set up a fresh architectural review of rendering.**
See "Next: rendering architecture review" at the bottom — that is the agenda
for the thread this handoff is being carried into. Everything above it is the
state that review starts from.

## What this is

Single-file, zero-dependency HTML/JS kanban board. No build step, no
framework, no server. Opens as a local file or hosted on GitHub Pages.
Following the same proto2prod discipline as SequenceBuilder: validate before
optimizing, single-file first, visible errors over graceful degradation,
document for a fresh instance as well as a human.

Now under real source control: `github.com/MeatPopSci1972/kanban-builder`
(`main`). Reconciliation across threads is now a git concern, not a manual
chat merge — see the last section.

## Architecture — the one rule that matters most

**Store owns state contracts. UI owns rendering.** Never let UI code mutate
state directly — everything goes through `store.dispatch({type, ...})`,
which returns `{ok: true, ...}` or `{ok: false, error}`. Rejected actions
never mutate state and never fire `change`. This boundary is why the test
suite can pin down the store's behavior completely without touching a DOM —
and it is exactly what let the task-color work (store) and the panel-placement
work (UI) be built in two separate threads and merged with zero conflicts.

* **Store** (`createStore`): dispatch/getState/on. Actions: `ADD_COLUMN`,
  `RENAME_COLUMN`, `DELETE_COLUMN`, `REORDER_COLUMN`, `ADD_TASK`,
  `UPDATE_TASK`, `MOVE_TASK`, `DELETE_TASK`, `ADD_PROPERTY`,
  `UPDATE_PROPERTY`, `DELETE_PROPERTY`, `IMPORT_BOARD`.
* **Property types**: `text`, `text-large`, `date`, `link`, `image` — each
  has its own store-level validation (see Conventions).
* **Serializer**: `exportBoard`/`validateBoard`/`buildStateFromBoard`.
  Canonical envelope: `{app, schemaVersion, exportedAt, columns, tasks}`.
  `validateBoard` is the single gate every import must pass through.
* **Pure helpers (DOM-free, on the testable side of the line):**
  positioning (`reorderIndexExcludingSelf`, `computeDropIndex`), color
  (`validColor`), panel decisions (`nextTaskSelection`, `shouldDismissPanel`),
  and the GitHub issue-link + display formatters. These are pure on purpose so
  the render/event code stays thin and the logic stays pinned.

## What changed since the last handoff

Two independent feature lines, now merged into `main` + this file:

* **Task color (store).** Tasks carry a `color` field. Canonical stored form
  is a **6-digit lowercase hex** (`#3b82f6`) — exactly what a native
  `<input type="color">` emits, and a valid CSS color. Blank `''` means "no
  color" and is allowed. Validated by `validColor` (module-level, shared).
  `ADD_TASK` defaults it to `''`; `UPDATE_TASK` sets it through the gate;
  export/import/rehydrate all carry it; `validateBoard` checks it **only when
  present**, so old saves without a color still import (schema stayed at
  version 1 — see "deferred" below).
  **STORE-COMPLETE, UI-PENDING:** the store fully supports task color, but the
  card does **not** yet paint it. The "gradient under a wire mesh" card visual
  from the beta list is unbuilt. Wiring a picker + card rendering is open work,
  and it lands in the render layer — i.e. it depends on the rendering review.

* **Properties panel placement (UI).** The panel moved from a fixed
  right-hand `aside` to a sliding panel that **physically relocates in the
  DOM** to sit immediately after the selected task's column
  (`$board.insertBefore($panel, nextColumn)`), animated via an `.open` class
  (width/opacity/visibility transitions). Deselecting moves it back into
  `#main` and removes `.open` — and that placement logic lives in **exactly
  one place, `renderPanel`**. The card click handler and the click-outside
  handler now only decide *what* is selected (via the two pure functions) and
  call `render()`; they no longer duplicate placement.

## Conventions (don't drift from these)

* **Visible errors over graceful degradation.** Reject with a clear message
  naming what's wrong; never silently coerce, truncate, or default. (Note the
  one standing exception to watch: `loadSaved()` falls back to `defaultBoard()`
  on an invalid save with only a `console.warn` — a silent-ish path flagged
  for a visible toast when the schema/persistence work happens.)
* **Shared constants/validators, not duplicated logic.** `GITHUB_URL_BYTE_LIMIT`
  is one number with two uses. `validColor` is one validator meant for **both**
  task color (store) and the future board color picker (UI) — it's module-level
  specifically so the UI can call it without a second copy.
* **Color is canonical `#rrggbb` lowercase.** Shorthand (`#fff`), uppercase,
  named colors, and `rgb()`/`hsl()` are **deliberately rejected for now.**
  Widen the gate on purpose, with tests — do not add silent normalization.
* **Images are relative path strings only.** Never `data:` URIs, never base64
  in JSON. Enforced at the store level.
* **Naming consistency is load-bearing.** Rename everywhere: UI, comments,
  tests, error strings. "No one sees it" is not a reason to skip a spot.
* **Gate every change.** The pin-down suite must be green before anything
  ships. Currently: **81/81, across 10 suites** (1, 2, 3, 3b, 4, 5, 6, 7, 8, 9).
  The tests modal counts dynamically — there is no hardcoded total to update.

## How to run the gate (headless, no browser needed)

```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('kanban-builder.html','utf8');
const start = html.indexOf('const PROP_TYPES');
const end = html.indexOf('── UI STATE ──');
const src = html.slice(start, html.lastIndexOf('/*', end));
eval(src);
const results = runTests();
const failed = results.filter(r=>!r.pass);
console.log(results.filter(r=>r.pass).length + '/' + results.length + ' passed');
failed.forEach(f=>console.log('FAIL:', f.suite, '|', f.name, '|', f.msg));
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

Line endings: `main` and this file are **LF**. Keep them LF — a CRLF/LF
mismatch makes `git merge-file` treat every line as changed (learned the hard
way during the color+panel merge).

## Deferred / parked (designed, not built — don't rebuild without reading)

* **Multi-board workspace layer.** Decided architecture: the **store stays
  single-board** and never learns boards exist. "Multi-board" is a UI +
  persistence layer riding the existing `IMPORT_BOARD`/`exportBoard` seam —
  a workspace = a list of board envelopes + an active id; switching a tab
  imports that board into the one store; "active board in UI" also scopes
  Save (for now). Board **name** and board **color** are workspace-layer
  metadata, **not** store state (that's why board color reuses `validColor`
  from the UI side). Tabs, "New Board," and the pickers are all unbuilt.
* **Schema version / migration.** Deliberately deferred. Task color was made
  additive + tolerant to avoid a bump. When multi-board or any newly-required
  field forces `SCHEMA_VERSION` 1 → 2, that needs an explicit, visible upgrade
  path (old single-board saves adopted as board #1), not a silent wipe.
* **"Send to bot" / `ENSURE_COLUMN`.** Parked pending vetting. `ENSURE_COLUMN`
  (idempotent "column exists → ok; absent → create, `created:true`") was
  designed as the honest, contracted way for a bot flow to spawn a
  pending-review column without an out-of-band mutation. Not built.

## Explicitly rejected or removed — don't rebuild without a new reason

* **GitHub Gist as a shared-state backend.** Token-in-localStorage blast
  radius across `file://` origins; a "secret" Gist's read side needs no auth;
  no conflict detection. Never built. If revisited, use Gist `history`
  revision ids as a version-guard from the start.
* **GitHub-Issues-backend (labels-as-columns).** Fully built, then deleted
  after finding GitHub Projects v2 does it better. If wanted again, target
  Projects v2's real API (GraphQL `updateProjectV2ItemPosition`), not labels +
  issue bodies.
* **Restricting "+Task" to a single column.** Discussed, not built. If wanted,
  a real store-level "designated intake column" flag with an honest rejection —
  not a hidden button.

## Known limitations, stated plainly

* **WCAG target is 2.2 AA**, not 3.0 (3.0 is still a W3C Working Draft).
* **Nothing touching a real browser is covered by the automated gate.** The
  headless harness exercises pure functions and the store only. The panel's
  CSS slide, the DOM `insertBefore` placement, the image picker, and the
  in-page dialogs all need manual browser verification. Extracting the two
  panel *decisions* into `nextTaskSelection` / `shouldDismissPanel` pulled the
  testable logic across the line; the DOM half remains manual by design.
* **`renderPanel` does `document.querySelector('[data-col-id=...]').nextSibling`.**
  Safe today because the store guarantees every task's `columnId` is a real
  column, so the element is always present — but it would throw if that ever
  stopped holding. A defensive guard is a cheap future hardening.

## Next: rendering architecture review (the reason for this handoff)

The user wants a **fresh-context look at the overall architecture, focused on
rendering**, before any more key changes. Google's Gemini (which authored the
panel-placement work on `main`) called out rendering opportunities — **bring
those specific callouts into the review; they are not captured here.**

Current rendering approach, stated factually as the starting point (these are
observations for the review to weigh, NOT decisions already made):

* `render()` runs on **every** store `change` and rebuilds everything:
  `renderBoard(state)` recreates every column and card — and re-attaches every
  drag/click/keydown listener — from scratch; `renderPanel(state)` clears
  `$panelBody.innerHTML` and rebuilds all fields.
* There is **no reconciliation/diffing and no event delegation** — it's a full
  teardown-and-rebuild each time. This is simple and correct (proto2prod:
  validate before optimize), which is why it shipped this way.
* The panel is a single persistent DOM node moved around via
  `insertBefore`/`appendChild` between renders, rather than re-created.

Open questions a rendering review would likely take up: whether full-rebuild
is a real problem at expected board sizes or a premature-optimization trap;
event delegation vs. per-element listeners; reconciling changed
columns/tasks vs. rebuilding all; where the not-yet-built task-color card
visual and the multi-board tab strip should hook in; and whether any of this
warrants a render abstraction or stays deliberately hand-rolled. Decide the
rendering posture **before** building the color card visual or the workspace
layer, since both live in the render path.
