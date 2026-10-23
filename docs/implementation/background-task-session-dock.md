# Session-scoped background tasks and a shared Task Dock

Design plan and execution record for the change that makes background commands
belong to their session and lets Todo and background tasks share one composer
container.

## Problem

Background commands were host-global in three separate ways:

1. `backgroundTasks.list` returned every task with no session filter, and the
   renderer showed all of them in one bar. A task started in session B was
   visible — and stoppable — from session A.
2. Todo and the task bar were two stacked sibling surfaces in the composer, so a
   running command competed with the plan and ask decision rows for the same
   vertical space, and the bar had no empty state of its own.
3. Deleting a session removed its scratch directory, review records, and file
   hashes, but never its background processes. A deleted session's dev server
   kept running until the whole app exited.

Two smaller defects compounded these:

- The concurrency limit was a single process-wide constant of 32, which is both
  too high to be a per-session rule and unrelated to what a user sees.
- "Removal" of a finished row was a renderer-only action with no state-layer
  guard, so a caller holding a stale snapshot could drop a running task from the
  list while its process lived.

## Decisions

| # | Decision | Rationale |
| --- | --- | --- |
| D1 | Concurrency limit is **per session, 10** | It is the limit a user can actually hit; refusals name it explicitly |
| D2 | A **process-wide ceiling** stays as a host safety net | Many sessions must not add up to an exhausted host; reported separately so the two causes are distinguishable |
| D3 | `backgroundTasks.list` stays **session-agnostic**; the renderer filters | Filtering is presentation. Adding a required parameter would force every consumer — including a future AI read path — to change |
| D4 | A spawn is **refused when the session is already gone** | Otherwise the task belongs to no visible container and outlives its owner |
| D5 | **Deleting** a session stops its running tasks; **archiving** does not | Archiving is a reversible marker; killing its processes would be a destructive side effect |
| D6 | The stop is issued from `drop_session_side_data` | That is the one place all three deletion paths already converge |
| D7 | Stop is **signal-and-return** | Deletion must not wait for a task to settle, and must not fail when a task cannot be killed |
| D8 | Todo and tasks share **one container, tabs in the header** | Collapsing then hides only panels, so tabs stay reachable; the tab is never a second row competing with decisions |
| D9 | The header is a row of **siblings**, not one button | A tab is a button; a button may not nest in a button and the click target would be ambiguous |
| D10 | Tabs appear only when **both sides have content** | With one populated side, its panel renders alone and the tab list is not rendered at all |
| D11 | Tab choice is **per-session memory**, and only a user action writes it | A tab the container picked must never overwrite what the user wanted |
| D12 | A remembered tab whose side emptied **falls back** to the other side | Never render an empty panel |
| D13 | A new task **raises a badge** on the tasks tab; it never steals the tab | The user's current tab is a deliberate choice |
| D14 | Entering the tasks tab is **the only** badge clear point | There is exactly one path, so the badge cannot clear behind the user's back |
| D15 | Clicking a tab while collapsed **also expands** | Otherwise the click appears to do nothing |
| D16 | Removal is gated on a **terminal state in the state layer** | Hiding the control is not a guarantee; a stale snapshot must be refused too |
| D17 | Removal is **view-only and issues no RPC** | The host keeps the task, its output, and its id; readability by id is unchanged |
| D18 | Grace window, 256KB per stream, 16-entry history **unchanged** | They already bound the surface correctly; this change is about ownership, not retention |

## Execution

### P1 — host behaviour (`crates/host-core`)

- Split the limit into `MAX_RUNNING_TASKS_PER_SESSION` (10) and
  `MAX_RUNNING_TASKS_TOTAL` (64), with distinct refusal messages.
- Add `running_count_for(session)` and `stop_session_running(session)` to the
  registry. The stop takes no output lock, so a caller holding the state mutex
  cannot deadlock against a task mid-write.
- Add `sessions::session_exists` (one indexed `EXISTS`) and refuse a spawn for a
  session that is gone.
- Call `stop_session_running` from `drop_session_side_data`, which covers
  `session.delete`, the sidebar batch delete, and `projects.remove`.

### P2 — renderer data layer

- `dock-state.ts` (new): per-session `chosenTab` / `newCount` / `collapsed`,
  `resolveDockTab`, `observeSessionTasks`, `clearBadge`, `chooseTab`,
  `toggleCollapsed`, `pruneDockView`.
- `bar-visibility.ts`: `dismissTask` now takes the task and refuses `running`;
  `canDismiss` exposes the same rule to the view.
- `useBackgroundTasks(sessionId)`: filters to the session, owns the badge
  accounting, and exposes `remove(id)` which looks the task up rather than
  trusting the caller.

### P3 — UI

- `TaskDock.tsx` (new): one container, one header carrying progress summary,
  live count, tab list, and disclosure.
- `TodoDock.tsx`: keeps the standalone dock (and its recovery + disclosure) and
  additionally exports `TodoPanel` / `todoProgress` for the container.
- `Composer.tsx`: mounts `TaskDock` under the existing
  `activeSessionId && localToolsAllowed` gate.
- `composer.css`: `.dock*` and `.task-*` replace `.background-tasks-bar` /
  `.background-task*`. The task row is a vertical stack — `.task-band` (command +
  action, never wrapping) above an optional `.task-output` — so a long command
  ellipsizes on one line instead of pushing the metadata and its action onto a
  second line.

### P4 — i18n and docs

- New `taskDock` block plus `headerCount` / `removeFailed` in all 9 locales.
- `03-tools-and-permissions.md` (en + zh-CN): the "at most 32 tasks" and the
  fixed "Todo, background commands, Plan/Ask" stack-order clauses are replaced by
  the per-session rule, the shared container, the badge rule, the removal guard,
  and the deletion linkage.
- `06-host-rpc-protocol.md`: notes that `list` stays session-agnostic and that
  removal issues no RPC.

## What the demos show

`docs/demos/background-task-dock/` holds the interactive prototypes used to
settle the layout. Open `index.html`.

| Demo | Question it answers |
| --- | --- |
| 1 | Empty-side and empty-container rules (tab hidden vs. container absent) |
| 2 | Per-session tab memory, and the fallback when a remembered side empties |
| 3 | Badge accumulation without stealing the tab; clearing on entry |
| 4 | Container-level collapse, and removal refused for a live or settling task |
| 5 | Strict session scoping, delete-stops-tasks vs. archive-does-not |
| 6 | Three header layouts side by side (inline / adaptive / stacked) |
| 7 | Narrow-width stress: which copy degrades first, measured at 420–760px |
| 8 | The chosen inline layout running the complete flow with a live spec check |
| final | The shipped layout as a clean product surface, no instrumentation |

`docs/demos/background-task-dock/tokens.css` copies the real token values from
`apps/desktop/src/styles/tokens.css`, so the prototypes read like the product.

## Verification

Local:

- `cargo test -p host-core --locked background_tasks` — 7 passed, including three
  new tests: the per-session limit and its refusal message, session-scoped stop
  that spares other sessions, and the deleted-session spawn refusal.
- `cargo fmt -p host-core -- --check`, `cargo clippy -p host-core --all-targets`
- `pnpm build:js`, `pnpm --filter @pi-desktop/desktop typecheck`, `pnpm lint`
- `node --test apps/desktop/test/background-tasks*.test.mjs`

Delegated to CI: the full desktop suite, because it contains macOS
signing/notarization and Electron-binary cases that need the CI environment.

## Compatibility

- No wire parameter added; `backgroundTasks.list/output/stop` and the
  `backgroundTasks.changed` payload are unchanged.
- No new RPC. Removal stays renderer-local.
- The 10s grace window, per-stream 256KB cap, and 16-entry finished history are
  unchanged.
- Removing the standalone bar is a UI change only: the container presents the
  same rows, states, and actions.
