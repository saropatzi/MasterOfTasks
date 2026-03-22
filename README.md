# MasterOfTasks

A Claude Code plugin that orchestrates plan implementation with automated code review loops. Takes one or more plans created by ClaudePlanners, implements each task via a subagent, and reviews the result with ClaudeReviewers. Supports resume after interruption, progress tracking, and execution metrics.

## Prerequisites

MasterOfTasks requires the **ClaudeReviewers** plugin to be installed. At startup it verifies that `/review` is available. If not found, execution is blocked with install instructions.

Install ClaudeReviewers:

```bash
claude plugin add git@github.com:saropatzi/ClaudeReviewers.git
```

## Installation

### From GitHub

```bash
claude plugin add git@github.com:saropatzi/MasterOfTasks.git
```

### From a local path (development / testing)

```bash
claude plugin add /path/to/MasterOfTasks
```

Once installed, the `/mot` (alias `/master-of-tasks`), `/mot-status`, and `/mot-resume` commands are available in all Claude Code sessions.

## Quick Start

```
/mot plans/my-plan.md
```

MasterOfTasks will confirm the plan order, check for uncommitted changes, then work through every task: implement, light review, mark done. After all tasks are complete it runs a full review of the plan, then moves the plan and its report to `plans/completed/`.

To execute multiple plans in sequence:

```
/mot plans/auth.md,plans/sso.md
```

## Commands

### `/mot <plans>` — Execute Plans

**Alias:** `/master-of-tasks`

Takes a comma-separated list of plan file paths (whitespace around commas is trimmed). Paths containing literal commas are not supported.

**Startup sequence:**

1. Verify ClaudeReviewers is installed (`/review --skip-review`).
2. Read `mot.yml` from the project root and merge with defaults.
3. Check the working tree. If there are uncommitted changes, warn and wait for confirmation before continuing.
4. Show plan order and ask for confirmation. Reordering is accepted at this point.
5. Check for an existing `status.json` with matching plans. If found, suggest `/mot-resume` instead.
6. Create `status.json` to track progress.

**Execution loop — per task:**

1. Mark the task `in_progress` in `status.json`.
2. Collect context: task description, files, TDD/non-TDD tag, verification criteria, plan sections (Overview, Context, Development Approach), and all previously completed tasks with their commit hashes.
3. Launch the implementer agent with the collected context.
4. If the implementer fails: stop and present options — retry, manually fix then retry, skip task, or abort.
5. Run a light review via ClaudeReviewers.
6. If the review fails: stop and present the same four options.
7. On success: mark the task `completed`, update `status.json`, check off the checkbox in the plan file, and append a row to the report file.

**Execution loop — per plan:**

After all tasks pass, run a full review:

```
/review --plan <plan_file> --max-iterations <review_max_iterations>
```

On success: mark the plan completed, finalize the report (totals, review results, issues log), and move both the plan file and the report file to `<plan_dir>/completed/`.

**Completion message:**

```
All plans completed. Reports at plans/completed/
```

### `/mot-status` — Show Current Status

Reads `mot.yml` (for `plan_dir`) then reads `<plan_dir>/status.json` and displays a summary without running anything.

If no `status.json` is found: `No execution in progress.`

If found, shows for each plan:

- Status: `pending` / `in_progress` / `completed`
- Tasks completed / total
- Last commit hash
- Last review result
- Duration so far

Example output:

```
Plan: plans/2026-03-22-auth.md [in_progress]
  Tasks: 3/5 completed
  Last commit: abc1234
  Last review: pass
  Duration: 15m

Plan: plans/2026-03-22-sso.md [pending]
  Tasks: 0/4
```

### `/mot-resume` — Resume Interrupted Execution

Resumes from where an interrupted execution stopped.

**Arguments:** optional comma-separated plan paths. If omitted, the plan list is taken from `status.json`.

**Flow:**

1. Read `status.json`. If not found: `No previous execution found. Use /mot to start.`
2. Check the working tree for uncommitted changes; warn and wait for confirmation if any.
3. Show the current state: completed tasks, in-progress tasks, last commit, last review result.
4. Ask: "Do you want to verify anything before resuming?" If yes, the user can inspect, run `/review`, or check files. The question is repeated when they are ready.
5. Confirm the resume point: `Resume from Task N of plan X?`
6. Continue the `/mot` execution loop from that point.

**Resume granularity:** A task with status `in_progress` is re-executed from the beginning (implementation + review). The implementer works on top of the current working tree state and does not revert partial work from the interrupted run.

## Plan Format

MasterOfTasks expects plans following the ClaudePlanners format. Tasks are parsed from the "Implementation Steps" section (or equivalent) as markdown checkbox items:

```markdown
- [ ] Task 1: Create plugin scaffold [non-TDD]
- [ ] Task 2: Add auth middleware [TDD]
  - Files: src/middleware/auth.go, src/middleware/auth_test.go
  - Verify: auth middleware rejects unauthenticated requests with 401
```

**Parsing rules:**

- Tasks are `- [ ]` or `- [x]` checkbox items with a `Task N:` prefix.
- TDD/non-TDD tag: `[TDD]` or `[non-TDD]` at the end of the task line.
- Files: optional indented line starting with `Files:` — lists the files the task touches.
- Verification criteria: optional indented line starting with `Verify:` — passed to the implementer and the review.
- Plan sections (Overview, Context, Development Approach, etc.) are identified by `##` headings and passed as context to the implementer.

If a plan file exists but contains no parseable tasks (no checkbox items):

```
No tasks found in `<path>`. Ensure the plan follows ClaudePlanners format with checkbox items.
```

## Execution Loop

### Per-Task Flow

```
1. Mark task in_progress
2. Collect context (task + plan sections + previous task results)
3. Launch implementer agent
   - TDD: write test → verify failure → implement → verify success → commit
   - non-TDD: implement → test if applicable → commit
   - Commit message: feat: <task description>
4. Light review: /review --only quality,implementation --plan <file> --max-iterations N
5. On pass: update status.json, mark checkbox [x], update report file
6. On fail: stop, show findings, ask user how to proceed
```

### Per-Plan Flow

```
1. After all tasks pass: full review (/review --plan <file> --max-iterations N)
2. On pass: finalize report, move plan + report to <plan_dir>/completed/
3. On fail: stop, show findings, ask user how to proceed
```

## Review Integration

All review work is delegated to ClaudeReviewers. MasterOfTasks never fixes code itself.

### Light Review (post-task)

Runs only the agents listed in `review_agents_light` (default: `quality,implementation`):

```
/review --only quality,implementation --plan <plan_file> --max-iterations 3
```

ClaudeReviewers runs its full flow: dispatches agents, collects findings, verifies, and applies fixes through its internal fix loop. Returns an exit code.

### Full Review (post-plan)

Runs all default agents (quality, implementation, testing, simplification, documentation):

```
/review --plan <plan_file> --max-iterations 3
```

If `review_agents_full` in `mot.yml` is set to a specific list (not `all`), `--only` is used instead.

### When Review Fails (exit 1)

MasterOfTasks stops and shows the unresolved findings to the user. Four options are offered:

- **(a) Retry review** — run ClaudeReviewers again on the current state.
- **(b) Manually fix, then retry** — the user edits the code and then triggers another review pass.
- **(c) Skip task / plan and continue** — mark as completed despite findings and move on.
- **(d) Abort entire execution** — stop everything; `status.json` retains progress for a later `/mot-resume`.

### `--only` vs `--agents`

These are ClaudeReviewers flags with different semantics:

- `--only` is **restrictive**: runs exclusively the listed agents, ignoring all defaults. Used by MasterOfTasks for light reviews.
- `--agents` is **additive**: adds the listed agents to the default set (e.g., `--agents conventions` adds the conventions agent to the five default agents).

## Configuration

### `mot.yml`

Optional file placed in the project root. All fields are optional; omitted fields use defaults.

```yaml
plan_dir: plans/                          # default: plans/
review_agents_light: quality,implementation  # default: quality,implementation
review_agents_full: all                   # default: all
review_max_iterations: 3                  # default: 3

agents:
  implementer:
    model: opus                           # default: opus
```

`mot.yml` is read and parsed in-context by the LLM via the Read tool. Keep the file simple — complex or deeply nested YAML may not parse reliably.

**Merge priority:** `mot.yml` > defaults.

### `mot/implementer.md` — Agent Prompt Override

Place `mot/implementer.md` in the project root to replace the built-in implementer prompt. Only the prompt body is replaced; `mot.yml` still controls the model unless the override file contains a frontmatter `model` field.

**Model precedence (highest to lowest):**

1. `model` field in the frontmatter of `mot/implementer.md` (if override file exists)
2. `agents.implementer.model` in `mot.yml`
3. Default: `opus`

## Status and Reports

### `status.json`

**Path:** `<plan_dir>/status.json` (default: `plans/status.json`). Lives alongside the plan files.

Created at the start of `/mot`. Updated after each task completes. Read by `/mot-status` and `/mot-resume`. Persists after completion — it serves as a historical record.

```json
{
  "started": "2026-03-22T14:30:00",
  "last_updated": "2026-03-22T14:42:00",
  "completed": null,
  "plans": [
    {
      "path": "plans/2026-03-22-auth.md",
      "status": "completed",
      "tasks": [
        {
          "number": 1,
          "description": "Create plugin scaffold",
          "status": "completed",
          "files": [".claude-plugin/plugin.json", ".gitignore", "LICENSE"],
          "commit": "abc1234",
          "review": "pass",
          "findings": 0,
          "fixed": 0,
          "iterations": 0,
          "duration_seconds": 120,
          "timestamp": "2026-03-22T14:35:00"
        }
      ]
    }
  ]
}
```

Per-task fields in `status.json`:

| Field | Description |
|---|---|
| `commit` | Commit hash produced by the implementer (`null` if no changes) |
| `review` | `pass` or `fail` |
| `findings` | Number of review findings reported |
| `fixed` | Number of findings fixed by ClaudeReviewers |
| `iterations` | Number of fix-loop iterations ClaudeReviewers ran |
| `duration_seconds` | Total time for implementation + review |

### Report Files

Each plan has a corresponding report file. The report filename is derived by appending `-reports` before the `.md` extension:

- Plan: `plans/2026-03-22-auth.md`
- Report: `plans/2026-03-22-auth-reports.md`

The report file is created when the first task completes and updated after each subsequent task. When the plan is completed, both files are moved to `<plan_dir>/completed/`:

- `plans/completed/2026-03-22-auth.md`
- `plans/completed/2026-03-22-auth-reports.md`

Report format:

```markdown
# MasterOfTasks Summary — 2026-03-22 16:30

## Plans Executed
- plans/2026-03-22-auth.md — completed

## Metrics

| Task | Plan | Duration | Findings | Fixed | Iterations | Commits |
|---|---|---|---|---|---|---|
| Task 1: scaffold | auth | 2m | 0 | 0 | 0 | 1 |
| Task 2: auth middleware | auth | 8m | 3 | 3 | 2 | 3 |

## Totals
- Total duration: 10m
- Total findings: 3
- Total fixed: 3
- Unresolved: 0
- Total commits: 4
- Average iterations per task: 1.0

## Review Results
- Light reviews (post-task): 2 passed, 0 failed
- Full reviews (post-plan): 1 passed

## Issues Log
(none)
```

## Error Handling

| Situation | Behavior |
|---|---|
| ClaudeReviewers not installed | Error at startup: `ClaudeReviewers required. Install with: claude plugin add git@github.com:saropatzi/ClaudeReviewers.git` |
| Plan file not found | Error: `Plan not found: <path>. Check the file path.` |
| Plan directory does not exist | Created automatically (`mkdir -p`) |
| Malformed plan (no checkbox items) | Error: `No tasks found in <path>. Ensure the plan follows ClaudePlanners format with checkbox items.` |
| Invalid `mot.yml` | Warning: `Warning: mot.yml could not be parsed. Using defaults.` Execution continues with all defaults. |
| Review fails (exit 1) | Stop execution. Show findings. Present four options: retry, manually fix then retry, skip, abort. |
| Implementer agent fails | Stop execution. Show error. Present four options: retry, manually fix then retry, skip, abort. |
| No-op implementation (no commit produced) | Mark task completed with `commit: null`. Log: `Task N produced no changes.` |
| Dirty working tree at startup | Warn: `Working tree has uncommitted changes. Commit or stash before proceeding?` Wait for confirmation. |
| Existing `status.json` for same plans | Suggest: `Previous execution found. Use /mot-resume to continue, or delete <plan_dir>/status.json to start fresh.` |
| User cancellation mid-run | `status.json` retains progress. User can resume later with `/mot-resume`. |
| Plan paths with commas | Not supported. Paths are split on commas; use unambiguous paths. |

## Resume

When an execution is interrupted (cancelled by the user, or aborted due to a review or implementer failure), `status.json` preserves the full state. `/mot-resume` reads it and continues from the first incomplete task.

**How tasks are resumed:**

- Tasks with status `completed` are skipped.
- The first task with status `in_progress` is re-executed from the beginning (implementation + review). Any partial work it left in the working tree is not reverted — the implementer builds on the current state.
- Tasks with status `pending` are executed in order after the resumed task.

**Before resuming**, `/mot-resume` shows the current state and asks whether the user wants to verify anything (inspect files, run `/review`, check git log) before proceeding. Execution does not restart until the user confirms.

## License

MIT
