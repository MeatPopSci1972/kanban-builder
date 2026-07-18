# KanbanBuilder — Handoff

Attach the current `kanban-builder.html` to any thread you paste this into —
this doc is context, not a substitute for the file. Source control:
`github.com/MeatPopSci1972/kanban-builder` (`main`). Cross-thread reconciliation
is a git concern. **LF only.**

## What this is

Single-file, zero-dependency HTML/JS kanban board. No build, framework, or
server. Opens as a local file or on GitHub Pages. proto2prod discipline:
validate before optimize, single-file first, visible errors over graceful
degradation, document for a fresh instance as well as a human.

## The one rule

**Store owns state contracts. UI owns rendering.** All state changes go through
`store.dispatch({type, ...})` → `{ok:true,...}` or `{ok:false, error}`. Rejected
actions never mutate and never fire `change`. This is why the store is fully
testable without a DOM — keep tests on the store side of this line.

* **Store** (`createStore`): `dispatch`/`getState`/`on`. Actions: `ADD_COLUMN`,
  `RENAME_COLUMN`, `DELETE_COLUMN`, `REORDER_COLUMN`, `ADD_TASK`, `UPDATE_TASK`,
  `MOVE_TASK`, `DELETE_TASK`, `ADD_PROPERTY`, `UPDATE_PROPERTY`,
  `DELETE_PROPERTY`, `IMPORT_BOARD`.
* **Property types**: `text`, `text-large`, `date`, `link`, `image` — each
  store-validated.
* **Serializer**: `exportBoard`/`validateBoard`/`buildStateFromBoard`. Envelope
  `{app, schemaVersion, exportedAt, columns, tasks}`. Every import passes
  `validateBoard`.
* **Pure helpers** (DOM-free): `reorderIndexExcludingSelf`, `computeDropIndex`,
  `validColor`, `nextTaskSelection`, `shouldDismissPanel`, GitHub
  issue-link/display formatters. Pure on purpose — keeps render/event code thin.

## Current state

* **Task color — STORE-COMPLETE, UI-PENDING.** Store carries `color` (canonical
  6-digit lowercase hex; `''` = none; validated by `validColor`; tolerant on
  import so old saves load). The card does **not** paint it yet. Picker + card
  visual are open work in the render layer.
* **Properties panel — hybrid, complete.** `renderPanel` relocates it per-column
  via `$board.insertBefore($panel, nextColumn)`; the `.open` class drives the CSS
  reveal (width `0`→`var(--panel-w)`). Deselect → `appendChild` back to `#main`,
  remove `.open`. All placement lives in `renderPanel` only — click /
  click-outside handlers just set selection and call `render()`.
  Animation is settled — **don't undo these:** `#panel-head`/`#panel-body` are
  pinned to `calc(var(--panel-w) - 2px)` (`min-width` + `flex-shrink:0`) so
  content lays out once and the slide only clips it (no mid-slide reflow); the
  panel carries **no margin** — the board's flex `gap:14px` already spaces it, so
  re-adding `margin-left` double-spaces the left edge.

## Conventions (don't drift)

* **Visible errors over graceful degradation.** Reject with a message naming
  what's wrong; never silently coerce/truncate/default. (Watch: `loadSaved()`
  falls back to `defaultBoard()` on a bad save with only `console.warn` — wants
  a visible toast when persistence work happens.)
* **Shared validators, not copies.** `validColor` and `GITHUB_URL_BYTE_LIMIT`
  are module-level so store and UI reuse one definition.
* **Color = `#rrggbb` lowercase only.** Shorthand/uppercase/named/`rgb()`/`hsl()`
  rejected on purpose. Widen the gate deliberately, with tests — no silent
  normalization.
* **Images = relative path strings only.** No `data:`/base64. Enforced at store
  level.
* **Naming consistency is load-bearing.** Rename everywhere: UI, comments,
  tests, error strings. "No one sees it" is not a reason to skip a spot.
* **Gate every change** green before shipping. The suite counts dynamically —
  no hardcoded total to update.

## Gate (headless, no browser)

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

Syntax check before shipping:

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

**LF only** — CRLF makes `git merge-file` treat every line as changed.

## Deferred / parked (don't rebuild without reading)

* **Multi-board workspace.** Store stays single-board and never learns boards
  exist. Multi-board = UI + persistence on the `IMPORT_BOARD`/`exportBoard`
  seam: workspace = board envelopes + active id; tab switch imports into the one
  store; active board scopes Save. Board name + color are workspace metadata,
  not store state (board color reuses `validColor`). Tabs / New Board / pickers
  unbuilt.
* **Schema version / migration.** Deferred; task color was made additive to
  avoid a bump. First newly-required field forcing `SCHEMA_VERSION` 1→2 needs an
  explicit, visible upgrade (old saves → board #1), not a silent wipe.
* **"Send to bot" / `ENSURE_COLUMN`.** Parked. Idempotent "exists → ok; absent →
  create `created:true`" as the contracted way a bot spawns a review column.
  Not built.

## Rejected (don't rebuild without a new reason)

* **Gist as shared-state backend.** Token-in-localStorage blast radius on
  `file://`; a secret Gist's read side needs no auth; no conflict detection. If
  revisited, use Gist `history` revision ids as a version guard from day one.
* **GitHub-Issues backend (labels-as-columns).** Built, then deleted — Projects
  v2 does it better. If wanted, target Projects v2 GraphQL
  (`updateProjectV2ItemPosition`), not labels + issue bodies.
* **"+Task" restricted to one column.** Not built. If wanted, a real store-level
  "intake column" flag with an honest rejection — not a hidden button.

## Known limitations

* **WCAG target 2.2 AA** (3.0 is still a Working Draft).
* **Nothing touching a real browser is gated.** The headless harness covers pure
  functions + store only. Panel slide, `insertBefore` placement, image picker,
  and in-page dialogs need manual browser verification.
* **`renderPanel` column lookup is guarded.** Resolves the column via
  `querySelector('[data-col-id=...]')`. The store guarantees `columnId` is real,
  so it's present in normal operation; if that invariant breaks, `renderPanel`
  fires a named error toast and bails to closed instead of throwing on
  `.nextSibling`. `selectedTaskId` is left intact — selection valid, DOM
  inconsistent.

## Rendering posture (decided)

`renderBoard` tears down and rebuilds every column and card on each `change`
(`render()` → `renderBoard` → `$board.innerHTML=''`). No diffing, no event
delegation — full rebuild each time. Simple and correct; validate before
optimize.

**Panel = persistent by reference, not in place.** `$board.innerHTML=''`
detaches it every render; the `$panel` reference keeps the node alive;
`renderPanel` re-inserts it. Node identity survives; DOM position and running
transitions do not.

**Tripwire — graduate to keyed reconciliation (`data-id` + delegation) when, not
before:**

1. *Nearer* — any animation/transition **on or inside the panel** must survive a
   `change`. Detach cancels it every render. Invisible today (panel eases open
   once, then re-inserts already-`.open`). Breaks the moment an in-panel element
   must animate *across* a state change.
2. *Later* — a *non-panel* node's correctness depends on surviving a rebuild
   (multi-board tab strip active state; in-place editing keeping focus). Likely
   first real trip = the tab strip; re-decide posture when it starts.

## Open work

**Next task — task-color card visual + picker.** Store side is done
(STORE-COMPLETE); the card just doesn't paint `color` yet. Wire a color picker
into the panel (reuse `validColor` — don't add a second validator) and paint the
card: the "gradient under a wire mesh" visual from the beta list. This lands in
the render layer, and the render posture it depended on is now **decided**
(below), so it is unblocked.

**Still open:** event delegation as a standalone win — **hold** until a tripwire
trips (see Rendering posture), not now; multi-board workspace layer (see
Deferred); where the tab strip hooks into the render path.

*(Gemini's panel summary claimed "entirely CSS, dropped `insertBefore`" — false;
`main` is the hybrid above. Its "infinite loop" and "hardware-accelerated"
reasons are both wrong — don't canonize them.)*
