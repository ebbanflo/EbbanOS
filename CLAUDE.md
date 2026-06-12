# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

EbbanOS ("second brain") is an offline-first personal organizer PWA for capturing tasks, notes, and events. The **entire application is a single file**, `index.html` — markup, CSS, and JavaScript are all inline. `sw.js` is the only other file (a network-first service worker for offline caching). There is no build system, package manager, framework, or dependencies.

## Running & developing

- **Run locally:** serve over HTTP (the service worker and PWA manifest require a server, not `file://`). E.g. `python3 -m http.server 8000` then open `http://localhost:8000`. Editing `index.html` and reloading is the entire dev loop.
- **No build, lint, or test commands exist.** Don't look for `package.json`, npm scripts, or a test runner.
- **Service worker is network-first** (`sw.js`): it always fetches the latest version and only falls back to cache offline. Bump the `CACHE` constant (`secondbrain-v2`) when you need to force-evict old caches. During local dev, disable the SW or hard-reload to avoid serving stale code.
- The app is built for iOS PWA / mobile (safe-area insets, touch gestures, `100dvh`). Test in a mobile viewport or device.

## Architecture

Everything lives in the inline `<script>` in `index.html`. Mental model:

- **Persistence — IndexedDB** (`secondbrain` DB, version 1) with three object stores: `tasks`, `notes`, `events`, each keyed by `id`. `openDB`, `dbPut`, `dbDel`, `dbGetAll` are thin promise wrappers around it.
- **In-memory cache** — `cache = {tasks, notes, events}` is the read model. `refreshCache()` reloads all three stores from IndexedDB. **The canonical write cycle is: `dbPut`/`dbDel` → `await refreshCache()` → `render()`.** Follow this pattern for any mutation; the UI never reads IndexedDB directly during render.
- **State** — a single `state` object holds all UI state (current `tab`, `filter`, `search`, `view` list/cal, calendar month, selected day, capture/edit context). There is no reactive framework: mutate `state`, then call `render()`.
- **Rendering** — `render()` is the single entry point that rebuilds `#list`'s innerHTML from `cache` + `state`. Two modes: list view (`visibleItems()` → `cardHTML()`) and calendar view (`calHTML()` → month grid + selected-day agenda). After re-rendering HTML, render re-attaches gesture handlers (`attachSwipe`, `attachCalGestures`) since the DOM was replaced.
- **Areas** — `AREAS` (top of script) defines the color-coded categories (`personal`/`work`/`music`); the `inbox` tab shows everything. This is the one intended customization point ("rename areas here if forking"). Each item has an `area` field; tab selection filters by it.

### Item shapes

- **task:** `{id, title, area, dueDate|null, status:'active'|'archived', createdAt}`
- **note:** `{id, title, body, area, isDaily, date, createdAt}`
- **event:** `{id, title, area, startDate, endDate, createdAt}`

Only tasks support archive/restore; notes and events support delete only.

### Interaction model

- **Card swipe** (`attachSwipe`) on cards: left reveals archive+delete (delete for notes/events), right archives/restores tasks; full swipes trigger the action directly, with damped rubber-band resistance past the bounds. Pointer events with a horizontal/vertical lock heuristic.
- **List-level gestures** — a single pointer arbiter on `#list` (IIFE near `switchTab`) routes by direction lock: horizontal off a card → **swipe between tabs** (`TAB_ORDER`, animates via `renderSwap`); vertical-down at `scrollTop===0` → **pull-to-refresh** (`#ptr` spinner → `refreshCache`). It bails to mode `'no'` when the drag starts on a `.card-wrap` so per-card swipe keeps its own gesture.
- **Bottom sheet** (`#sheet`) is the capture/edit form, reused for create (`openCapture`) and edit (`openEdit`). `setSheetType` toggles which fields show per type; drag-down-to-dismiss is wired on `#grabZone`.
- **Action sheet** (`actionSheet(title, actions)`) — reusable iOS-style modal returning a Promise of the picked value; used by import to choose merge/replace.
- **Haptics** (`haptic(kind)`) — fires the iOS system haptic by toggling a rendered `<input type="checkbox" switch id="hapticSwitch">` (the only thing that works in iOS Safari; `navigator.vibrate` is a no-op there and used only as the Android fallback). Called on swipe commit, save, tab switch, sheet/action-sheet open, and pull-to-refresh.
- **Motion** — spring/overshoot easing on the sheet and card snap; `@keyframes cardIn` staggered mount, gated behind a transient `#list.anim-in` class added only by `renderSwap()` (deliberate tab/view swaps) so search/filter re-renders stay static. All new animations respect the `prefers-reduced-motion` block.
- **OS handoffs** — saving a task fires `handoffReminder()` (deep-links to Apple Reminders via `x-apple-reminderkit://`); saving/editing an event generates an `.ics` file shared via the Web Share API (`shareFile`, falling back to a download link).
- **Backup** — export (`#exportBtn`) writes all data as JSON; **import** (`#importBtn` → hidden `#importFile`) parses a backup, validates item shape, and applies it as a **merge** (`dbPut` over existing, overwriting matching ids) or **replace** (`dbClear` all stores first, behind a confirm).

## Conventions

- `$` / `$$` are `querySelector` / `querySelectorAll` aliases. `esc()` HTML-escapes all user content interpolated into template strings — use it for anything user-provided rendered via innerHTML.
- Dates are stored as ISO strings; `dayKey()` produces local `YYYY-MM-DD` keys used throughout the calendar.
- Accent color is driven by CSS custom property `--accent`, set per active tab in `render()`.
