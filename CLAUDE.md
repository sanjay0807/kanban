# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page IT project Kanban board for a fictional internal "UOB IT PMO" — a demo/training tool, not a
production system. The entire app is `index.html`: markup, one `<style>` block, one `<script>` block.

It is an internal demo only. Do not add real UOB logos, trademarks, or anything that imitates an official
UOB system. The header uses a plain text wordmark and a neutral corporate blue palette, and the demo
disclaimers in the header and footer are deliberate — keep them.

## Hard constraints

These are requirements of the deliverable, not style preferences. Violating any of them breaks it:

- **Vanilla only.** No React, Vue, jQuery, Tailwind, npm, bundler, or build step of any kind.
- **One file.** Everything stays in `index.html`. No sidecar `.css`/`.js` files.
- **Must run from `file://`** by double-clicking. Nothing may assume a server or an origin.
- **No persistence.** No `localStorage`, `sessionStorage`, IndexedDB, or cookies. Board state lives only in
  the in-memory `state.tasks` array; a refresh resetting the board to seed data is intended behaviour and
  the header note says so.
- **No external resources.** No CDN scripts, no web fonts, no image files. Icons are inline SVG strings in
  the `ICONS` map or Unicode glyphs; typography is a system font stack.
- **No `alert()` or `confirm()`.** Validation errors render inline under each field; delete confirmation is
  an inline "Delete? Yes / No" row inside the card.
- **No `!important`** in CSS.

## Running and testing

Double-click `index.html`, or open it directly in a browser. There is no test suite, no linter, and no
build — changes are verified by using the board.

For browser automation (the pane refuses `file://` URLs), serve the directory and drive `localhost`:

```bash
python -m http.server 8731
```

Delete any `.claude/launch.json` you create for that server afterwards — the deliverable is one file.

Worth re-checking after any change to rendering or state: drag a card between columns (highlight appears
and clears), drag to reorder inside a column while sort is set to manual, the Move select on a card, the
delete Yes/No flow and where focus lands, the command palette (Ctrl+K) finding both tasks and commands,
opening a card into the drawer and saving an edit, undo after a move/edit/delete, submitting the form
empty, submitting with a past due date, the filters and quick chips composing, and the layout at 375px.

## Architecture

**Single source of truth.** One `state` object holds `tasks`, `filters`, `sort`, `theme`, `density`,
`idCounter` and the transient UI flags (`pendingDeleteId`, `dragTaskId`, `openTaskId`, `editing`, `undo`,
`palette`). Every mutation goes through `addTask()`, `moveTask()`, `updateTask()` or `deleteTask()`, and
each one ends by calling `renderBoard()`.

**Undo is snapshot-based.** Each mutator calls `snapshot()` first, which deep-copies the task array into
`state.undo`. `undoLast()` restores it wholesale, so undo works for moves, edits, adds and deletes without
per-operation inverse logic. Any new mutator must call `snapshot()` before it changes anything.

**Reordering only applies in manual sort.** `sortTasks()` re-sorts on every render, so a dragged position
would be immediately overridden under any other mode. The drag handler therefore only draws the insertion
indicator and passes `beforeId` when `state.sort === "manual"`.

**One-way render.** `renderBoard()` → `renderColumn()` → `renderCard()` rebuilds the board's `innerHTML`
from `state` on every change. Nothing outside that chain may mutate card contents. Transient UI that
survives a re-render lives in `state`, not in the DOM — this is why `pendingDeleteId` (which card is
showing its delete confirmation) and `dragTaskId` are state fields rather than CSS classes.

**Delegated events.** Because cards are replaced wholesale on every render, per-card listeners would go
stale. All card interaction is delegated from the `#board` element and dispatched on `data-action`
attributes (`ask-delete`, `cancel-delete`, `confirm-delete`, `move`). Add new card controls the same way —
never attach a listener to a card node.

**Escaping is mandatory.** Cards are built by string concatenation into `innerHTML`, so every interpolated
value — including ones that look safe, like `task.id` — must pass through `escapeHtml()`. There is a test
for this: adding a task titled `<img src=x onerror=...>` must render as literal text.

**Dates.** `todayIso()` builds `YYYY-MM-DD` from local calendar parts; do not substitute `toISOString()`,
which shifts to UTC and makes due dates wrong near midnight. Overdue and past-date checks rely on
`YYYY-MM-DD` strings comparing correctly with `<`.

**Vocabularies.** `STATUSES`, `PROJECTS`, `CATEGORIES`, `PRIORITIES`, `WIP_LIMITS` and `QUICK_FILTERS`
drive the selects, the chips, the rail and the board. Adding a status also needs `STATUS_HINTS`,
`STATUS_HUE` and a `.column[data-status]` rule; adding a priority also needs `PRIORITY_RANK`, a `--pri-*`
custom property, a `.card[data-priority]` rule and a `.pill-priority` rule. Priority is never signalled by
colour alone — the pill always carries its text label.

**Two views, one state.** `state.view` is `"board"` or `"dashboard"`, mirrored into the URL hash so the
dashboard is linkable. `renderApp()` is the single entry point: it picks the view, then always refreshes
the rail, chips and drawer. Filters and search apply to both views — the dashboard reads the same
`applyFilters()` result the board does.

**RAG is derived, never stored.** `ragStatus()` computes Red/Amber/Green from status, due date and
priority on every render, so it cannot drift from the task. Red is late or blocked, Amber is due within
seven days or critical, Green is on track or done. Change the thresholds there and the KPIs, health bars
and register all follow.

**CSS traps that have already bitten.** Two rules to check when adding UI:
- `input[type="text"]` is more specific than a bare class, so a class restyling a text input must be
  scoped through an ancestor (`.search-wrap .search-input`) or the generic rule silently wins.
- A component that sets `display` (`.field`, `.drawer`) beats the UA rule for `[hidden]`, so `el.hidden`
  stops working. A `[hidden] { display: none; }` rule at the very end of the stylesheet restores it —
  keep it last.
- Grid items default to `min-width: auto`, so a wide child (the register table) stretches the whole page.
  Grid containers holding wide content use `minmax(0, 1fr)` and the items carry `min-width: 0`.

## FormSubmit

`FORMSUBMIT_ENDPOINT` at the top of the script is the only place the recipient address appears; keep it
there. FormSubmit needs a one-time activation per address — the first submission sends a confirmation
email and nothing is delivered until that link is clicked.

`notifyNewTask()` is fire-and-forget by design: `handleSubmit()` adds the card, closes and resets the form
*before* awaiting it, and a rejection only raises a warning toast. A notification failure must never block
or roll back the board. Over `file://` the request carries a `null` origin and may be rejected on CORS
grounds even after activation — that path is expected to fail gracefully.

Never send the user's email address anywhere except this endpoint.
