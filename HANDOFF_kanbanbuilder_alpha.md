# KanbanBuilder — Handoff

Read this before touching the file. Attach the current `kanban-board.html` to
whatever thread you paste this into — this document is context, not a
substitute for the file itself.

## What this is

Single-file, zero-dependency HTML/JS kanban board. No build step, no
framework, no server. Opens as a local file or hosted on GitHub Pages.
Following the same proto2prod discipline as SequenceBuilder: validate before
optimizing, single-file first, visible errors over graceful degradation,
document for a fresh instance as well as a human.

## Architecture — the one rule that matters most

**Store owns state contracts. UI owns rendering.** Never let UI code mutate
state directly — everything goes through `store.dispatch({type, ...})`,
which returns `{ok: true, ...}` or `{ok: false, error}`. Rejected actions
never mutate state and never fire `change`. This boundary is why the test
suite can pin down the store's behavior completely without touching a DOM.

* **Store** (`createStore`): dispatch/getState/on. Actions: `ADD\_COLUMN`,
`RENAME\_COLUMN`, `DELETE\_COLUMN`, `REORDER\_COLUMN`, `ADD\_TASK`,
`UPDATE\_TASK`, `MOVE\_TASK`, `DELETE\_TASK`, `ADD\_PROPERTY`,
`UPDATE\_PROPERTY`, `DELETE\_PROPERTY`, `IMPORT\_BOARD`.
* **Property types**: `text`, `text-large`, `date`, `link`, `image` — each
has its own store-level validation (see Conventions below).
* **Serializer**: `exportBoard`/`validateBoard`/`buildStateFromBoard`.
Canonical envelope: `{app, schemaVersion, exportedAt, columns, tasks}`.
`validateBoard` is the single gate every import (JSON file, or anything
else in the future) must pass through — reuse it, don't reinvent it.
* **Positioning helpers**: pure functions (`reorderIndexExcludingSelf`,
`computeDropIndex`) doing the drag/drop and keyboard-reorder math for
both columns and tasks. Kept DOM-free specifically so they're testable.
* **"Send to GitHub Issue"**: separate, zero-auth feature — opens a
prefilled `github.com/.../issues/new` URL. No token, no fetch, no
credentials. Uses `GITHUB\_URL\_BYTE\_LIMIT` (8191, GitHub's own documented
URL limit) as a truncation guard.
* **In-page dialogs** (`showPromptDialog`/`showConfirmDialog`): replace
native `prompt()`/`confirm()`. Necessary because Claude's in-chat file
preview runs in a sandboxed iframe without `allow-modals` — native
dialogs silently return `null` there instead of erroring. Works
everywhere regardless, so there's no reason to reintroduce native ones.
* **Image file picker** (`pickImagePath`): `showOpenFilePicker()` +
`showDirectoryPicker().resolve()` for a *verified* relative path.
Chromium-only, requires HTTPS — works once hosted on GitHub Pages, does
**not** work from a bare `file://` page. Falls back to a filename-only
guess (`./images/<name>`) on Firefox/Safari, clearly marked unverified.

## Conventions (don't drift from these)

* **Visible errors over graceful degradation.** Reject with a clear message
naming what's wrong; never silently coerce, truncate, or default.
* **Shared constants, not duplicated magic numbers.** `GITHUB\_URL\_BYTE\_LIMIT`
is reused for both the issue-link truncation guard *and* the text-large
property cap — one number, one place, two uses. Follow that pattern
rather than inventing a fresh constant that happens to match.
* **Images are relative path strings only.** Never `data:` URIs, never
base64 in JSON. Enforced at the store level (`\_validImagePath`), not just
the UI.
* **Naming consistency is load-bearing.** When something's renamed, rename
it everywhere — UI text, comments, tests, error strings. "No one sees it"
is not a reason to skip a spot.
* **Gate every change.** The pin-down test suite must be green before
anything ships. Currently: **68/68**, across 9 suites.

## How to run the gate (headless, no browser needed)

```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('kanban-board.html','utf8');
const start = html.indexOf(\\"const PROP\_TYPES\\");
const end = html.indexOf('── UI STATE ──');
const src = html.slice(start, html.lastIndexOf('/\*', end));
eval(src);
const results = runTests();
const failed = results.filter(r=>!r.pass);
console.log(results.filter(r=>r.pass).length + '/' + results.length + ' passed');
failed.forEach(f=>console.log('FAIL:', f.suite, '|', f.name, '|', f.msg));
process.exit(failed.length ? 1 : 0);
"
```

Also worth a full-script syntax check before shipping:

```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('kanban-board.html','utf8');
const start = html.indexOf(\\"'use strict';\\");
const end = html.lastIndexOf('</script>');
new Function(html.slice(start, end));
console.log('syntax OK');
"
```

## Explicitly rejected or removed — don't rebuild without a new reason

* **GitHub Gist as a shared-state backend.** Reviewed twice. Real, unresolved
concerns: a token in `localStorage` has bigger blast radius than it looks
(file:// origin sharing across unrelated local HTML files), a "secret"
Gist's *read* side needs no auth at all, and there's no built-in conflict
detection. Never built. If it's built later, it should use Gist's own
`history` revision ids as a version-guard from the start — same pattern
as the conflict check the GitHub-Issues-backend used before it was
removed (see below).
* **GitHub-Issues-backend (labels-as-columns, issues-as-tasks).** Fully
built — four ships: read-only mirror, move/create, column
create/rename/reorder via labels, conflict guard via `updated\_at`. Then
**deliberately deleted, cleanly, with a grep-verified sweep for orphaned
references**, after discovering GitHub's native Projects (v2) board view
already does this, better — real per-item drag ordering via GraphQL
instead of a hand-rolled issue-body comment encoding a manual order
field. **Do not rebuild this shape.** If GitHub Projects integration is
wanted again, target Projects v2's actual API (GraphQL,
`updateProjectV2ItemPosition`), not labels + issue bodies.
* **Restricting "+Task" to a single column.** Discussed, not built.
Conclusion: if wanted, it should be a real store-level "designated intake
column" flag with an honest rejection message — not just hiding a button,
which would be an undiscoverable, unstated rule. Open, undecided.

## Known limitations, stated plainly

* **WCAG target is 2.2 AA**, not 3.0 — checked current status directly:
WCAG 3.0 is still a W3C Working Draft (most recent update March 2026),
with a Recommendation not expected before 2028. 2.2 is what's actually in
force.
* **Nothing touching a real browser dialog or file picker is covered by the
automated gate** — the image picker and the in-page prompt/confirm
dialogs need manual verification in an actual browser; the headless
harness can only exercise pure functions and the store.
* Property types and their store-level gates: `image` (no `data:` URIs, ≤500
char path), `text-large` (≤8191 chars, shared with the GitHub Issue
guard), `date` (must parse, and must fall within 1900–2100 — added after
a native `<input type="date">` was found to allow typing garbage like
year `0006`).

## If you're reconciling multiple threads' work back into one file

There's no source control in play across these chat threads themselves —
if two threads both touch the store or the same render function, that's a
manual merge, not something either thread can know about in advance. Given
this is headed for a GitHub-hosted repo, it may be worth doing that
reconciliation as real git branches/PRs once it's there, rather than by hand
in chat — but that's your call on timing.

