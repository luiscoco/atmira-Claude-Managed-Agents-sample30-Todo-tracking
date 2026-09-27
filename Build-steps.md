# Todo tracking

This file describes the **steps followed** to add Concept 30 (**Todo tracking**) to the Claude Agent SDK Lab: what
was read, what was probed, what was decided, how it was tested, and what the tests changed. To learn the concept
itself, read [Tab30-Todo-tracking.md](Tab30-Todo-tracking.md) (the same text is in `readme.md`).

| Concept | Topic | Routes | Explanation |
|---|---|---|---|
| 30 | How the agent keeps a todo list: the Task tools vs `TodoWrite`, which one each model gets, deferred tools, `tool_use_result`, the task files, a live board, a plan written by the host, resume, the `TaskCreated` / `TaskCompleted` hooks, a host check | `/api/c30/tools`, `/run`, `/code` | [Tab30-Todo-tracking.md](Tab30-Todo-tracking.md) |

## How to run it

```powershell
npm run dev        # server on http://localhost:3001, web on the Vite port
```

`node_modules` was copied from sample29, so `npm install` is not needed. Open the **30. Todo tracking** tab.
`ANTHROPIC_API_KEY` must be in `.env`: the lab uses a fake `CLAUDE_CONFIG_DIR`, so a Claude Code login is not found.
Start it from a normal terminal, not from inside Claude Code (see Tab16). Each scenario costs $0.05 or less.

> **Only one sample can run at a time.** Every sample's server uses port **3001**. Stop the other samples'
> `npm run dev` first.

## Step 1: Choose the feature

The request said "implement the following feature sample" with no feature text, as in sample28 and sample29. A grep of
the `Tab*.md` files and `server/concepts` showed that `TodoWrite`, `AskUserQuestion` and OpenTelemetry were never
covered, and the web tools only in passing. Four topics were offered, and **Todo tracking** was chosen.

## Step 2: Read what the course already said

sample30 already held a copy of sample29 without `node_modules`, so `node_modules` was copied from sample29.

| Read | To learn |
|---|---|
| `Build-steps.md`, `28-errors-retries.ts`, `Concept28ErrorsRetries.tsx`, the top of `29-images-files.ts` | The latest style: strict zod bodies, `relay()`, `#region` + `/code`, scenario buttons with hints, a fake `CLAUDE_CONFIG_DIR`, `short()` for paths |
| `server/index.ts`, `server/sse.ts`, `src/lib/sse.ts`, `src/App.tsx`, `src/styles.css` | Mounting, SSE, the tab list, CSS classes (`st-*` are tag backgrounds, so the board uses `todo-*`) |
| `Tab29-Images-and-file-input.md`, `Tab1-query().md` | The explanation's structure, and the table row format |

## Step 3: Check the types, and the binary

In `sdk-tools.d.ts` and `sdk.d.ts` (`0.3.281`):

| Found | Used for |
|---|---|
| `TodoWriteInput` / `TodoWriteOutput` (`oldTodos`, `newTodos`) | Scenario 2, the board from `TodoWrite` |
| `TaskCreateInput`, `TaskUpdateInput` (`status` + `"deleted"`, `addBlockedBy`, `owner`, `metadata`), `TaskListOutput`, `TaskUpdateOutput` (`statusChange`, `error`) | The tool rows and the board |
| `TaskCreatedHookInput`, `TaskCompletedHookInput` (`task_id`, `task_subject`, `task_description`) | The hooks |
| `SDKTaskUpdatedMessage` | Not a todo message: it is for background tasks (Concept 27) |

Two todo systems in one SDK raised the question of which one runs. A search of the Claude Code binary for the env
variables found `CLAUDE_CODE_ENABLE_TASKS` (the Task tools are on unless it is false), `CLAUDE_CODE_ENABLE_TODO_TOOLS`
next to a list of model IDs, and `CLAUDE_CODE_TASK_LIST_ID` (the list name, else the session ID). It also found the
hook code: a blocked `TaskCompleted` returns `success: false`, and a blocked `TaskCreated` deletes the task and throws.

## Step 4: Probe before designing

Six scratchpad scripts (in a temporary `.probe/` folder) ran real sessions with Haiku 4.5, each with its own
`CLAUDE_CONFIG_DIR`:

| Probe | Result | Decision |
|---|---|---|
| p1: "Reply ok" per model and env | Haiku: Task tools. `ENABLE_TASKS=false` (or `0`): `TodoWrite`. Sonnet 5: **none**; with `ENABLE_TODO_TOOLS=1`: Task tools | Button 0 and Step 2 of the explanation |
| p2: a 3-step job, "plan with your task list" | Haiku wrote the list as **text**, no `TaskCreate` | Name the tool in the prompt |
| p2: the prompt names `TaskCreate` | `ToolSearch` first (deferred), then 3 × `TaskCreate`, `TaskUpdate`s; both hooks fired; files in `cfg/tasks/<session id>/1.json` | Scenarios 1 and 3; the board reads the files |
| p2: `TodoWrite` mode | 5 calls, each the whole list; `tool_use_result` has `oldTodos` / `newTodos`; no files | Scenario 2 |
| p2: `TaskCompleted` returns `{ decision: "block", reason }` with a vague reason | The model **ignored** it; task 1 stayed `in_progress` | The reason must be an instruction; the host check |
| p2: no `allowedTools` for the todo tools, `canUseTool` logs | `canUseTool` never called for them | Stated in the explanation |
| p2: `ENABLE_TOOL_SEARCH=false`, no word about planning | No todo calls; the cost doubled ($0.042) | Scenario 4 |
| p3: host writes 3 task files, `CLAUDE_CODE_TASK_LIST_ID` | `TaskList` shows `[blocked by #1, #2]`; order respected; `fs.watch` saw each change | Scenario 5 |
| p3: resume, Task tools and `TodoWrite` | Same session ID, same folder; `TodoWrite` resumes from the transcript | Scenario 7 |
| p3: `claude_code` system prompt, a 5-step job, no word about planning | No todo calls | "If you want a list, ask for one" |
| p4: creation policy, a real completion check with a clear reason, a task added by the host | Refused create: `is_error`, ID skipped; refused completion: `success: false`, the model fixed the file; the host's task found by `TaskList` | Scenario 6 |
| p5: read the tool list without a turn (streaming input that never yields, `getContextUsage({ detail: "summary" })`) | No `system/init` before the first message; the summary lists no tool names | Button 0 pays for one short turn per setup |
| p6: an explicit `tools` list per model | Every model (Sonnet 5, Opus 5.5) gets the todo tools if listed; "Reply ok" costs $0.004 instead of $0.02 | The lab always passes `tools` |

## Step 5: Design the concept

- **One router, two SSE routes.** `POST /tools` runs 5 setups in parallel and reports the todo tools in each
  `system/init`. `POST /run` runs one of 8 scenarios: `tasks`, `todo`, `defaultTools`, `unprompted`, `seeded`,
  `gate`, `resume`, `custom` (your prompt + mode).
- **A folder per run** (`todo-lab/runs/<id>/` with `prices.csv`), so files from the last run are not there. One
  `CLAUDE_CONFIG_DIR` (`todo-lab/config`) holds the transcripts and the task lists.
- **The board**: a 250 ms poller on the task folder (Task tools) or `tool_use_result.newTodos` (`TodoWrite`). Each
  status change is also an event row.
- **The host check** after each run: the tasks left, and the files written.
- **Part C**: `/code` regions `options`, `board`, `hooks`, `seed`, `check`, `messages`. **Part D**: the comparison table.

## Step 6: Implement it

| File | What was done |
|---|---|
| `server/concepts/30-todo-tracking.ts` | New: `newRun()`, `baseOptions()` (explicit `tools`, `canUseTool` for file tools only, both hooks), `writeTask()` / `seedPlan()`, `taskHook()` / `gate()`, `readDisk()` / `showBoard()` / `watchDisk()`, `relay()`, `check()`, the routes |
| `server/index.ts` | Mounted on `/api/c30` |
| `src/concepts/Concept30TodoTracking.tsx` | New: `TodoBoard`, `Timeline`, `ToolsTable`, Parts A to D, the custom job form |
| `src/App.tsx`, `src/styles.css` | Tab; `.tag-todo`, `.tool-call.todo`, `.board-change`, `.todo-*` |
| `.gitignore`, `Tab1-query().md` | `todo-lab/`, table row |

`npx tsc --noEmit -p .` found one error (`gate()` used `emit` without receiving it); after the fix it passed.

## Step 7: Test the routes

Port 3001 was left free for the other samples. A scratchpad script (`serve.ts`) mounted **only** the Concept 30 router
on port **3130**, with `.env` loaded and the `CLAUDE*` variables removed. A second script (`drive.ts`) posted each
body the way the tab does and printed each SSE event on one line.

| Test | Result |
|---|---|
| `{ scenario: "rm" }`, `{ scenario: "tasks", prompt: "x" }`, `{ scenario: "custom" }` | Refused with a clear message, nothing started |
| `POST /tools` | The 5 rows, as in the probes |
| Scenarios 1 to 7 | As in the probes; every host check 3/3 or 4/4 completed except scenario 4 ("No todo list was made") |
| `custom` in `TodoWrite` mode, with "also try to write ../escape.txt" | The write was denied by `canUseTool`; the board followed `newTodos` |

## Step 8: What the tests changed

- **A refused completion did not look refused.** The blocked `TaskUpdate` has `is_error: false`, so the row was not
  highlighted. The tab now also marks a result whose `tool_use_result.success` is `false`.
- **The `system/init` row listed `ToolSearch` as a todo tool** in scenario 3. It is now shown apart, as "some tools are
  deferred".
- **The rows were too tall.** Pretty-printed `tool_use_result` made the event list about 10,000 px high. It is now one
  line.
- **"blocked by #1, #2" stayed after #1 and #2 were done.** The board now shows only blockers that are not completed,
  as `TaskList` does.
- **A new fact for the explanation:** after a `TodoWrite` call in which every item is completed, the next call's
  `oldTodos` is `[]`.

## Step 9: Run it in the real app

Vite was started on port **5199** with a scratchpad config whose proxy pointed to 3130. Headless Edge was driven
through the DevTools protocol (`shot.mjs`, with its profile in the scratchpad): open the page, click tab 30, click a
scenario, wait for "done", read the board and the host check, take a screenshot.

| Scenario | Page |
|---|---|
| 6 · Hooks gate the plan | 90 event rows, 4 highlighted refusals, board 4/4 from `todo-lab\config\tasks\plan-…/*.json`, host check "Every task is completed", no console error |
| 2 · TodoWrite | Board 4/4 from `TodoWrite tool_use_result.newTodos`, green board rows, no console error |
| 0 · Which todo tools | The 5-row table; with a warm prompt cache the whole button cost $0.014, no console error |

All test processes were stopped, and `todo-lab/` and `.probe/` were deleted.

Costs: about $1.30 for all the probes, route tests and browser runs.

## Files added or changed

| File | Change |
|---|---|
| `server/concepts/30-todo-tracking.ts` | New: the Concept 30 routes, the board, the hooks |
| `server/index.ts` | Mounts `/api/c30` |
| `src/concepts/Concept30TodoTracking.tsx` | New: the Todo tracking tab |
| `src/App.tsx` | Tab |
| `src/styles.css` | Todo board and row styles |
| `.gitignore` | `todo-lab/` |
| `Tab1-query().md` | Adds Concept 30 to the table |
| `Tab30-Todo-tracking.md` | Explanation of the concept |
| `readme.md` | Same content as `Tab30-Todo-tracking.md` |
| `Build-steps.md` | This file |
