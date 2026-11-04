# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

An internal demo/training IT PMO Kanban board for a **fictitious** bank. The whole app is one file, `index.html`, with all markup, a `<style>` block and a `<script>` block. Branding is a neutral "IT PMO" text wordmark in a corporate blue palette. Do not add real bank logos or trademarks, or imitate any official system. The `UOB-ITPM-####` task-ID format and the `[UOB IT PMO]` email subject are text strings required by the spec and are intentional.

## Hard constraints (from the original spec — keep them)

- Vanilla HTML/CSS/JS only: no frameworks, libraries, build step, bundler or npm.
- Single file that must work when opened by double-click (`file://`), with no server required.
- No external resources: no CDNs, web fonts or image files. Use the system font stack and Unicode or inline SVG for icons.
- **No persistence.** Never use localStorage, sessionStorage, IndexedDB, cookies or any storage API. A refresh resetting to the seed data is intended, and the header note says so.
- The only network call is FormSubmit's AJAX endpoint. The user's email goes nowhere else. The one other URL is `WHATSAPP_URL`, a link the chatbot panel opens when the user clicks it.
- No `alert()`, `confirm()` or `!important`. Use CSS custom properties for colours and spacing.

## Running / checking

There is no build, lint or test tooling. To run it, open `index.html` in a browser:

```bash
start index.html
```

If FormSubmit calls fail CORS from `file://` (origin `null`), serve the folder locally instead:

```bash
python -m http.server 8000
```

Quick static check that the constraints still hold. It should only match the `FORMSUBMIT_ENDPOINT` line, the `WHATSAPP_URL` line and the CSP `<meta>` (`connect-src https://formsubmit.co`):

```bash
rg -n "localStorage|sessionStorage|indexedDB|document\.cookie|!important|https?://|alert\(|confirm\(" index.html
```

**CSP hash — update after every script edit.** The `Content-Security-Policy` `<meta>` in `<head>` allows the inline `<script>` only by its SHA-256 hash, so any change to the script blocks it until the hash is updated (CI fails on a stale hash). Print the current hash and paste it into `script-src`:

```bash
node -e "const s=require('fs').readFileSync('index.html','utf8');const m=s.match(/<script>([\s\S]*?)<\/script>/);console.log('sha256-'+require('crypto').createHash('sha256').update(m[1],'utf8').digest('base64'))"
```

Keep a single inline `<script>` and no inline `on*=` handlers. `.gitattributes` pins `index.html` to LF so the hash is the same on every platform. The script also refuses to run inside a frame (clickjacking guard).

## Architecture (inside `<script>`)

- **Config at top:** `FORMSUBMIT_ENDPOINT` is the single place to change the email address. `FORMSUBMIT_TIMEOUT_MS` aborts a hung request. `STATUSES`, `PROJECTS`, `CATEGORIES` and `PRIORITIES` drive both the columns and every `<select>`, which are populated in `init()`.
- **Single source of truth:** `state = { tasks, filters, ui }` plus the `idCounter` module variable. `state.ui` holds transient view state: `pendingDeleteId` (inline "Delete? Yes/No"), `openMoveId` (keyboard "Move ▸" menu) and `focusSelector`.
- **One-way render:** mutators (`addTask`, `moveTask`, `deleteTask`) change `state` and then call `renderBoard()`. `renderBoard()` is the only function that writes card DOM. It runs `applyFilters()` (pure), builds cards from `renderCard()` HTML strings, and updates the column count badges and the header summary via `renderSummary()`. After a re-render it restores keyboard focus from `state.ui.focusSelector`, so set that before re-rendering whenever focus should land somewhere specific.
- **Escaping:** every interpolated value in `renderCard`, `buildColumns` and `optionsHtml` goes through `escapeHtml()`. Keep it that way for any new markup.
- **Events are delegated on `#board`:** clicks dispatch on `data-action` (`move-toggle`, `move-to`, `delete`, `delete-yes`, `delete-no`) together with `data-id`. Native HTML5 drag-and-drop listeners (`dragstart`, `dragover`, `dragleave`, `drop`, `dragend`) are attached to the board too. Column shells are built once by `buildColumns()`; only their `.card-list` contents are re-rendered.
- **IT meeting reminder + chatbot:** `IT_MEETING` / `REMINDER_DELAY_MS` drive a one-time `role="alertdialog"` (`#reminderBackdrop`) shown after 10 s of *visible* tab time, deferred while the Add Task modal is open. The floating `#chatbot` button toggles a small panel, speaks `CHATBOT_GREETING` with the built-in `speechSynthesis` API (no network), and links to `WHATSAPP_URL`. These are page features, not Claude Code hooks, since hooks only fire on Claude Code events.
- **Dates:** use the local-date helpers (`toISODate`, `todayISO`, `daysFromToday`) and compare ISO strings. Seed due dates are relative to today, so the Overdue badges always show. `isOverdue` means the due date is before today and the status is not `Done`.
- **Add-task flow (`handleSubmit`):** `validateForm()` writes inline errors under each field (`#err-<key>`, linked via `aria-describedby`). The flow is optimistic: `addTask`, then reset and close the modal, then a success toast. `notifyNewTask()` then runs while the submit button shows "Sending…". On any failure, including a timeout or a FormSubmit `success: "false"` response, it shows the warning toast "Card added locally — email notification failed", and the board is never affected.
- **FormSubmit activation:** the first submission to a new address sends a confirmation email, and nothing is delivered until its link is clicked. The HTML comment above `<script>` documents this.
