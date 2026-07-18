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
  `validColor`, `nextTaskSelection`, `shouldDismissPanel`, `isUlid`,
  `boardStorageKey`, GitHub issue-link/display formatters. Pure on purpose —
  keeps render/event code thin.

## Current state

* **Task color — COMPLETE (store + UI).** Store carries `color` (canonical
  6-digit lowercase hex; `''` = none; validated by `validColor`; tolerant on
  import so old saves load). Panel has a native `<input type="color">` swatch +
  explicit **Clear**; the blank swatch seeds from the live `--accent` token via
  `themeSwatchSeed()` (guarded by `validColor`, with a constant fallback). The
  card paints it as the "gradient under a wire mesh" treatment
  (`.card.has-color` background layers + left `::before` stripe), keyed off
  `--card-color` set in `renderCard`. All UI DOM, so manual-verify — not gated.
* **Per-deployment storage key + Clear Store — SHIPPED.** Storage key is now
  `boardStorageKey(_BOARD_ULID)` → `kanbanbuilder_board_v1_<ULID>`, where
  `_BOARD_ULID` is a constant baked per copy of the file (boot region). `v1`
  keeps the schema-version namespace; the ULID isolates deployments so two
  copies on one origin don't stomp one key. `loadSaved()` prefers this
  deployment's slot and, if empty, adopts the pre-ULID `_LEGACY_BOARD_KEY`
  (`kanbanbuilder_board_v1`) once — next `save()` writes it into the new slot;
  legacy key is never deleted. Malformed `_BOARD_ULID` → fall back to legacy key
  + `_bootLoadNotice`. `isUlid`/`boardStorageKey` are pure + gated (Suite 10);
  the baked constant lives at boot, outside the gate. The **Clear Store** button
  (toolbar, next to Tests, `danger` class) confirms via `showConfirmDialog`,
  then dispatches `IMPORT_BOARD` of `exportBoard(defaultBoard())` so the reset
  flows through the store (`change` → render + autosave) — UI DOM, manual-verify.
  *Isolates deployments, not tabs: two tabs of the same file share one ULID =
  one board (still the frozen multi-board question).*
* **Panel dismiss — guarded against mid-bubble detach.** The document
  click-to-dismiss handler early-returns when `event.target.isConnected` is
  false. A panel control whose dispatch re-renders (`$panelBody.innerHTML=''`)
  detaches the clicked node before the global handler runs, which used to read
  as an outside click and wrongly collapse the panel — hit both **Clear** and a
  mouse-clicked **+ Add**. Guard lives in the DOM-gathering half;
  `shouldDismissPanel` stays pure (Suite 9 untouched).
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
* **Panel stays on-screen — SHIPPED.** Placement is consistent: the panel is
  always inserted *after* the selected column (no left-flip for the last column —
  consistency chosen over contextual placement). To keep a far-right/last-column
  panel visible, `renderPanel` scrolls the real scroller (`#board-wrap`, not
  `#board`) after insert. Decision is pure: `computePanelScroll(panelStart,
  panelWidth, viewStart, viewWidth) → newScrollLeft | null` (Suite 9). The DOM
  half measures the panel's left edge in scroller-content coords but uses the
  panel's **final** `--panel-w` (+2px borders), not its animating `offsetWidth`,
  so it's correct mid-slide and when re-opening while already open.

## Conventions (don't drift)

* **Visible errors over graceful degradation.** Reject with a message naming
  what's wrong; never silently coerce/truncate/default. (`loadSaved()` now
  surfaces a deferred toast — `_bootLoadNotice` — when a saved board exists but
  can't be loaded, shown after boot since loadSaved runs before the toast system
  is live. A clean first run stays silent.)
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

* **Blank-swatch seed couples to `--accent`.** `themeSwatchSeed()` derives the
  default task color from the theme accent token. Fine now, but task-default and
  UI-accent are different contracts; when theming lands, give the task-color
  default its own token so a branding change to `--accent` doesn't drag every
  blank swatch with it.
* **Multi-board workspace — FROZEN (a future idea must be imagined first).** Not
  a priority; do not build before the future concept shapes it. Browser tabs +
  export/import cover casual multi-board today — but tabs of the same file share
  one `kanbanbuilder_board_v1_<ULID>` key (same board across tabs,
  last-write-wins on save), so tabs are multiple *views*, not multiple *boards*.
  The per-deployment ULID key (shipped) isolates *different copies* of the file,
  not tabs of one — it does not thaw this. Real multi-board within a single file
  still needs per-board keys or the envelope design that follows. *Prior design,
  retained for when it thaws:* store stays single-board and never learns boards
  exist;
  multi-board = UI + persistence on the `IMPORT_BOARD`/`exportBoard` seam
  (workspace = board envelopes + active id; tab switch imports into the one
  store; active board scopes Save; board name + color are workspace metadata, not
  store state, board color reuses `validColor`). Tabs / New Board / pickers
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

**Recently shipped.** Per-deployment ULID storage key (`isUlid` /
`boardStorageKey`, Suite 10; baked `_BOARD_ULID` + non-destructive legacy
migration in `loadSaved()`); **Clear Store** button; task color (store + UI,
wire-mesh visual); `loadSaved()` visible-toast; panel dismiss `isConnected`
guard; panel stays on-screen (`computePanelScroll`). All gated where gate-able;
DOM behavior manual-verify. Gate is now 91/91 across 11 suites.

**Next task — OPEN.** No committed next feature. Multi-board (the prior "next
frontier") is **FROZEN** pending a future concept that must be imagined first
(see Deferred / parked). When that concept is named, drop a one-line placeholder
here so a fresh thread knows what gates the thaw.

**Candidates if you want a small win in the meantime:** none pressing. Remaining
parked items are deliberately held, not queued — see below.

**Still open / hold:** event delegation as a standalone win — **hold** until a
tripwire trips (see Rendering posture), not now; schema migration (see
Deferred); "Send to bot" / `ENSURE_COLUMN` (parked).

*(Gemini's panel summary claimed "entirely CSS, dropped `insertBefore`" — false;
`main` is the hybrid above. Its "infinite loop" and "hardware-accelerated"
reasons are both wrong — don't canonize them.)*
