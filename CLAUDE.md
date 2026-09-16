# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page Kanban board demo for UOB's internal IT PMO, built as one self-contained file: [index.html](index.html). There is no build step, package manager, test suite, or server — the entire app (markup, `<style>`, `<script>`) lives in that one file.

## Running / testing changes

There are no commands to run. To try changes, open `index.html` directly in a browser (double-click it, or open it via `file://`). There is no lint or test tooling in this repo — verify changes manually in a browser.

## Hard constraints (do not violate when editing)

- **Vanilla only**: plain HTML/CSS/JS. No frameworks (React, Vue, jQuery, Tailwind), no build step, no bundler, no npm.
- **Single file**: all markup, styles, and script stay in `index.html`. Don't split into separate `.css`/`.js` files.
- **No external resources**: no CDN scripts, no Google Fonts, no image files. Icons are inline SVG or Unicode glyphs; fonts come from the system font stack (`--font-sans` in `:root`).
- **No persistence**: board state lives only in the in-memory `state` object. Never introduce `localStorage`, `sessionStorage`, `IndexedDB`, or cookies — a page refresh is expected to reset the board to the seeded demo data, and the UI's "Demo mode" note documents this intentionally.
- **No real UOB branding**: the header uses a plain "UOB IT PMO" text wordmark only — no logos, trademarks, or imitation of a real UOB system.
- **FormSubmit is the only backend integration**, used via its AJAX JSON endpoint (`fetch`, not a plain form POST, so the page never navigates away).

## Architecture

Everything is organized around one state object and a render-from-state loop:

```js
state = { tasks: [], filters: { project, assignee, priority }, confirmingDeleteId, nextIdCounter }
```

- **`renderBoard()`** is the only function that rewrites card DOM content. It filters `state.tasks` through `applyFilters()`, groups by status into the four fixed columns (Backlog / In Progress / Blocked / Done), and calls `renderSummary()`. Any state mutation must be followed by `renderBoard()` — there is no direct DOM patching of card contents elsewhere.
- **`renderCard(task)`** returns an HTML string for one card. All user-supplied fields are passed through `escapeHtml()` before interpolation — never build card HTML with raw `innerHTML` of unsanitized input.
- **Event delegation, not per-card listeners**: click/change handlers for delete, delete-confirm, and the "Move ▸" select are attached once to the `#board` container (`boardEl.addEventListener(...)`), keyed off `data-action`/`data-id` attributes, so re-rendering card HTML never orphans listeners. Drag/drop listeners (`dragover`/`dragleave`/`drop`) are attached once per `.column` element (columns themselves are static; only their inner `.card-list` is replaced on render).
- **Delete confirmation** is inline UI state (`state.confirmingDeleteId`), not `window.confirm()` — toggled via `request-delete` / `cancel-delete` / `confirm-delete` actions and re-rendered from state.
- **Add Task flow** (`addTaskForm` submit handler): validate synchronously with `validateForm()`/`displayFormErrors()` (no `alert()`) → optimistically push the task and call `renderBoard()` immediately → `await notifyNewTask(task)` wrapped in try/catch to send the FormSubmit email in parallel, showing a success or "email notification failed" warning toast either way, then reset/close the modal in `finally`. The card must never depend on the network call succeeding.
- **`FORMSUBMIT_ENDPOINT`** is the one config constant to change if the notification target email changes; it's called out in a comment above it in the script (FormSubmit requires the target address to click a one-time activation link before notifications deliver).
- **Toasts** go through `showToast(message, type)` into the `aria-live="polite"` `#toast-region`, auto-dismissing after ~4.5s.
