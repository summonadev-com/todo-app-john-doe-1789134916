---
status: pending
title: Personal Todo App with Due Dates and Reminders
---

## Scope

A single-user, local-only daily task app. One person, one device, no accounts, no sync.
Emphasis is on due dates: knowing what is overdue, what is due today, and what is coming up.

In scope:
1. Add, edit, complete, and delete tasks.
2. Optional due date (and optional time) per task.
3. Grouping and filtering by Overdue / Today / Upcoming / No date / Completed.
4. In-app reminder surfacing for tasks that are due or overdue, plus optional browser notifications behind an explicit permission prompt.
5. Persistence across reloads via `localStorage`.
6. Empty states and keyboard-friendly quick add.

Non-goals (explicitly do not build):
1. Authentication, user profiles, sharing, or any multi-user concept.
2. Backend, database, or network requests of any kind.
3. Recurring tasks, subtasks, attachments, tags/projects, priority levels, stats/streaks.
4. Drag-and-drop reordering, calendar month view, offline service worker.
5. Dark/light theme switching (pick one light, minimal look and stay there).

## Data model

Defined in `src/types/task.ts`:
1. `Task`: `id` (string, crypto.randomUUID), `title` (string, required, trimmed, non-empty), `notes` (string, optional, short single-line only), `dueAt` (ISO string or null), `hasTime` (boolean — distinguishes an all-day date from a timed due moment), `completed` (boolean), `completedAt` (ISO string or null), `createdAt` (ISO string).
2. `TaskDraft`: the subset used by the add/edit form (`title`, `notes`, `dueAt`, `hasTime`).
3. `DueBucket`: union of `'overdue' | 'today' | 'upcoming' | 'someday'`.
4. `TaskFilter`: union of `'all' | 'today' | 'upcoming' | 'completed'`.
5. Persisted shape: `{ version: 1; tasks: Task[] }` so future migrations have a hook.

Expected outcome: every other module imports task shapes from `@/types/task`; no inline structural types for tasks anywhere else.

## Routes and screens

File-based routes under `src/routes/`:
1. `src/routes/__root.tsx` — app shell: centered max-width column, minimal header with app name and the reminder bell/permission control, `<Outlet />`, and a global toast/reminder region.
2. `src/routes/index.tsx` — the main list screen. Owns the filter tab state via search params (`?filter=today` etc.) so a filter survives reload and back/forward.
3. `src/routes/settings.tsx` — a small page for reminder permission status, a "clear completed" action, and a "delete all data" action with confirm.

Expected outcome: two navigable pages, filter state reflected in the URL, generated `src/routeTree.gen.ts` untouched.

## Component breakdown

Under `src/components/`:
1. `AppHeader.tsx` — title, link to settings, reminder-permission indicator.
2. `QuickAddInput.tsx` — always-visible single-line input pinned at the top of the list. Enter submits and keeps focus; Escape clears. Includes an inline due-date affordance so a date can be attached without leaving the keyboard.
3. `TaskForm.tsx` — used for editing an existing task (title, notes, due date, time toggle). Rendered inline in place of the row being edited rather than in a modal, to keep the UI minimal.
4. `DueDatePicker.tsx` — quick presets (Today, Tomorrow, Next week, No date) plus a native `<input type="date">` and an optional `<input type="time">` revealed by a "add time" toggle.
5. `TaskItem.tsx` — checkbox, title, due-date chip, edit and delete controls (controls appear on hover/focus only). Completed tasks render with muted text and strikethrough.
6. `TaskGroup.tsx` — section header (label + count) wrapping a list of `TaskItem`s.
7. `TaskList.tsx` — takes the filtered tasks, applies grouping, renders `TaskGroup`s or the empty state.
8. `FilterTabs.tsx` — All / Today / Upcoming / Completed, driven by the route search param.
9. `EmptyState.tsx` — per-filter copy (first-run vs. "nothing due today" vs. "no completed tasks").
10. `DueChip.tsx` — small label rendering relative due text ("Overdue by 2 days", "Today 5:00 PM", "Fri 14 Mar") with an emphasis variant for overdue.
11. `ReminderBanner.tsx` — dismissible in-app banner listing tasks that just became due or are overdue.
12. `ConfirmButton.tsx` — two-step confirm used by delete actions (click once to arm, again to confirm) to avoid building a modal system.

Expected outcome: each component is presentational except `TaskList`/route files; all task mutations flow through the store hook below.

## State and persistence

1. `src/lib/storage.ts` — typed `loadTasks()` / `saveTasks()` around `localStorage` under key `todo.v1`. Guards against absent storage, malformed JSON, and wrong `version`, returning an empty list rather than throwing.
2. `src/hooks/useTasks.ts` — the single source of truth. Holds `tasks` in `useState`, lazily initialized from storage, and writes back in a `useEffect` on every change. Exposes `tasks`, `addTask`, `updateTask`, `toggleComplete`, `deleteTask`, `clearCompleted`, `clearAll`.
3. Share the hook through a small context provider in `src/components/TasksProvider.tsx`, mounted in `__root.tsx`, so the list route and settings route mutate the same state.
4. `src/hooks/useNow.ts` — returns a `Date` that ticks every 30 seconds, so relative due labels and overdue transitions update without a reload.

Expected outcome: closing and reopening the tab restores the exact task list; no duplicated state between routes.

## Due dates and reminders

1. `src/lib/dates.ts` — pure helpers: `startOfDay`, `isSameDay`, `bucketFor(task, now): DueBucket`, `formatDueLabel(task, now)`, `presetDate(preset)`. Unit-friendly, no React imports.
2. Bucketing rules: `dueAt` before now (timed) or before today's start (all-day) → `overdue`; same calendar day as now → `today`; later → `upcoming`; `dueAt === null` → `someday`.
3. Sorting: within a group, ascending `dueAt`, undated tasks by `createdAt` descending. Completed tasks always sort last and are hidden unless the Completed filter is active.
4. In-app reminders (the primary mechanism, always works): `src/hooks/useReminders.ts` watches `tasks` against the ticking `now` and surfaces any incomplete task that crossed into due/overdue during the session via `ReminderBanner`. Dismissals are tracked per task id in component state so a banner does not reappear each tick.
5. Browser notifications (secondary, opt-in): NOTE — `Notification` requires explicit user permission and must be requested from a user gesture. Implement in `src/lib/notifications.ts` with `getPermission()`, `requestPermission()`, and `notify(title, body)`. Never request on page load; only from the button in `AppHeader` / `settings.tsx`. If permission is `denied` or the API is unavailable, degrade silently to in-app reminders only and show that state in settings.
6. Notifications only fire while the tab is open — no service worker, no background scheduling. Say this plainly in the settings copy so the behavior is not surprising.

Expected outcome: overdue tasks are visually obvious at a glance; reminders never depend on permission being granted.

## Visual design direction

1. Clean and minimal, light only. Background `white`/`neutral-50`, text `neutral-900`, secondary text `neutral-500`, hairline borders `neutral-200`.
2. One accent colour only, used for the active filter tab, focus rings, and primary buttons. Overdue is the single exception, rendered in a restrained red (`red-600` text, no filled badges).
3. Typography: system font stack, `text-sm` body, `text-base`/`font-medium` task titles, `text-xs uppercase tracking-wide text-neutral-500` group headers. No decorative fonts.
4. Layout: single centered column, `max-w-2xl`, generous vertical rhythm, list rows separated by hairline dividers rather than cards or shadows.
5. Interaction: no animations beyond a subtle opacity/colour transition on hover and completion. Visible focus rings everywhere — the app must be fully usable from the keyboard.
6. `src/styles/global.css` contains exactly `@import "tailwindcss";` plus, at most, a base layer setting the body background and antialiasing.

## Build order

Phase 1 — Foundation
1. Scaffold the Vite + React + TypeScript project; add `@tanstack/react-router`, `@tanstack/router-plugin`, `tailwindcss`, `@tailwindcss/vite`.
2. Configure `vite.config.ts` with the router plugin and Tailwind plugin, and the `@/` → `src/` alias in both `vite.config.ts` and `tsconfig.json`.
3. Create `src/styles/global.css`, import it once in `src/main.tsx`, mount the router.
4. Create `src/routes/__root.tsx` and a placeholder `src/routes/index.tsx`.
   - Checkpoint: `npm run dev` serves a styled empty shell with no console errors and `src/routeTree.gen.ts` is generated.

Phase 2 — Data layer
1. Write `src/types/task.ts`, `src/lib/storage.ts`, `src/lib/dates.ts`.
2. Write `src/hooks/useTasks.ts` and `src/components/TasksProvider.tsx`; mount the provider in `__root.tsx`.
   - Checkpoint: tasks seeded manually in code persist across a hard reload.

Phase 3 — Core CRUD
1. Build `TaskItem`, `TaskList`, `EmptyState`, `QuickAddInput` (title only, no date yet).
2. Wire add, toggle complete, and delete on `src/routes/index.tsx`.
   - Checkpoint: a task can be added with the keyboard alone, completed, deleted, and survives reload.

Phase 4 — Due dates
1. Build `DueDatePicker`, `DueChip`, `TaskGroup`; extend `QuickAddInput` with the inline date affordance.
2. Add `TaskForm` inline editing.
3. Apply bucketing and sorting in `TaskList`.
   - Checkpoint: a task dated yesterday appears under Overdue in red; one dated today appears under Today; grouping and labels are correct.

Phase 5 — Filters and reminders
1. Add `FilterTabs` backed by the `?filter=` search param on `src/routes/index.tsx`.
2. Add `useNow`, `useReminders`, `ReminderBanner`.
3. Add `src/lib/notifications.ts` and the opt-in permission button.
   - Checkpoint: switching filters updates the URL and survives reload; a task due one minute out raises the in-app banner without any permission granted.

Phase 6 — Settings and polish
1. Build `src/routes/settings.tsx` with permission status, clear-completed, and delete-all via `ConfirmButton`.
2. Pass over empty states, focus rings, hover-only controls, and responsive behaviour down to 360px width.
3. Verify: full keyboard walkthrough, `npm run build` clean, no TypeScript errors, no unused files.
   - Checkpoint: app is usable end to end with the mouse never touched.
