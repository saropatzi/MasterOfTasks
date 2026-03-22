# MasterOfTasks Plugin — Design Spec

## Overview

A Claude Code plugin that orchestrates plan implementation with automated code review loops. Takes one or more plans created by ClaudePlanners, implements each task via subagent, and reviews with ClaudeReviewers. Supports resume after interruption, progress tracking, and execution metrics.

**Repository:** git@github.com:saropatzi/MasterOfTasks.git

**Dependency:** Requires ClaudeReviewers plugin installed. At startup, verifies `/review` is available. If not, errors with: "ClaudeReviewers required. Install with: claude plugin add git@github.com:saropatzi/ClaudeReviewers.git"

## Plugin Structure

```
MasterOfTasks/
├── .claude-plugin/
│   └── plugin.json
├── README.md
├── LICENSE
├── .gitignore
├── skills/
│   ├── mot/
│   │   ├── SKILL.md                    # orchestrator (/mot, alias /master-of-tasks)
│   │   └── agents/
│   │       └── implementer.md          # implementation agent
│   ├── mot-status/
│   │   └── SKILL.md                    # /mot-status
│   └── mot-resume/
│       └── SKILL.md                    # /mot-resume
```

### plugin.json

```json
{
  "name": "MasterOfTasks",
  "version": "1.0.0",
  "description": "Orchestrates plan implementation with automated code review loops",
  "author": { "name": "saropatzi" },
  "license": "MIT"
}
```

## Installation

```bash
# From GitHub
claude plugin add git@github.com:saropatzi/MasterOfTasks.git

# From local path (for development/testing)
claude plugin add /path/to/MasterOfTasks
```

## Commands

### `/mot <plan1>,<plan2>,...` — Execute Plans

Takes a comma-separated list of plan file paths. Implements each task sequentially with review loops.

**Arguments:**
- Comma-separated list of plan paths (e.g., `/mot plans/auth.md,plans/sso.md`)

**Flow:**

```
1. INIT
   ├── Verify ClaudeReviewers is installed (test /review availability)
   ├── Read mot.yml (if exists) → merge with defaults
   ├── Parse plan paths from comma-separated argument
   ├── Confirm order: "Plans received in this order:
   │    1. plans/auth.md
   │    2. plans/sso.md
   │    Correct?"
   └── Check for existing status.json → if exists with matching plans,
       suggest using /mot-resume instead

2. PER PLAN (sequential)
   ├── Read plan file, identify all tasks
   ├── Create/update status.json with plan and tasks
   │
   ├── PER TASK (sequential)
   │    ├── Collect context:
   │    │    - Task description, files, TDD/non-TDD, verification criteria
   │    │    - Plan context (Overview, Context, Development Approach)
   │    │    - All previously completed tasks (description, files, commit)
   │    │
   │    ├── Launch implementer agent (subagent via Agent tool)
   │    │    - Pass all collected context
   │    │    - Model from: .mot/implementer.md frontmatter > mot.yml > default (opus)
   │    │
   │    ├── Review (light, post-task):
   │    │    /review --agents <light_agents> --plan <plan_file>
   │    │    ├── Exit 0 → task passed
   │    │    └── Exit 1 → STOP. Show findings to user, ask how to proceed.
   │    │
   │    ├── Update status.json:
   │    │    - task status: completed
   │    │    - commit hash, files, timestamp
   │    │    - review result, findings count, iterations
   │    │
   │    └── Update plan file: mark task checkbox [x]
   │
   ├── Review (full, post-plan):
   │    /review --plan <plan_file>
   │    ├── Exit 0 → plan passed
   │    └── Exit 1 → STOP. Show findings to user, ask how to proceed.
   │
   └── Mark plan completed in status.json

3. COMPLETION
   ├── Generate summary report: master-of-tasks-reports/YYYY-MM-DD-HHmm-summary.md
   └── Output: "All plans completed. Summary at <path>"
```

### `/mot-status` — Show Current Status

Shows the current state of plan execution without running anything.

**Flow:**
1. Read `master-of-tasks-reports/status.json`
2. If not found: "No execution in progress."
3. If found: show for each plan:
   - Status (pending / in_progress / completed)
   - Tasks completed / total
   - Last commit hash
   - Last review result
   - Duration so far

### `/mot-resume <plan1>,<plan2>,...` — Resume Execution

Resumes execution from where it was interrupted.

**Arguments:**
- Comma-separated list of plan paths (optional — if omitted, resumes from plan list in status.json)

**Flow:**
1. Read `status.json`
2. If not found: error "No previous execution found. Use /mot to start."
3. Show current state to user:
   - Which tasks completed, which in progress, which pending
   - Last commit, last review result
4. Ask: "Do you want to verify anything before resuming?"
   - If yes: user can inspect, run `/review`, check files. Then ask again when ready.
   - If no: proceed
5. Confirm: "Resume from Task N of plan X?"
6. Continue the `/mot` loop from the interrupted point

## Agents

### implementer

**File:** `skills/mot/agents/implementer.md`

**Receives:**
- Task description, files involved, TDD/non-TDD indication, verification criteria
- Plan context (Overview, Context, Development Approach)
- All previously completed tasks with description, files, commit hash

**Responsibilities:**
- If TDD: write test → verify failure → implement → verify success → commit
- If non-TDD: implement → test if applicable → commit
- Commit messages: `feat: <task description>` or `fix:` for fixes
- Stay within the scope of the assigned task — do not modify unrelated files

**Model:** opus (default, configurable via mot.yml or override frontmatter)

**Override:** `.mot/implementer.md` in project root replaces the built-in prompt. Frontmatter `model` field takes precedence over `mot.yml`.

**Model precedence:** override file frontmatter > `mot.yml` > default (opus)

## Review Integration

MasterOfTasks delegates all review work to ClaudeReviewers. No review logic is duplicated.

### Light Review (post-task)

```
/review --agents <review_agents_light> --plan <plan_file>
```

Default agents: `quality,implementation`. Configurable in `mot.yml`.

ClaudeReviewers runs its full flow: dispatch agents, collect findings, verify, fix if needed (fix loop). Returns exit code.

### Full Review (post-plan)

```
/review --plan <plan_file>
```

All default agents (quality, implementation, testing, simplification, documentation). Configurable in `mot.yml`.

### Handling Review Results

- **Exit 0:** review passed, proceed to next task/plan
- **Exit 1:** unresolved issues after ClaudeReviewers' fix loop. MasterOfTasks stops and shows findings to user. User decides how to proceed.

MasterOfTasks never fixes code itself — ClaudeReviewers handles all fixes via its internal fix loop.

## Per-Repo Configuration

### mot.yml (optional)

Placed in project root. Every field is optional.

```yaml
plan_dir: plans/                                    # default: plans/
status_dir: master-of-tasks-reports                 # default: master-of-tasks-reports/
review_agents_light: quality,implementation          # default: quality,implementation
review_agents_full: all                             # default: all
review_max_iterations: 3                            # default: 3

agents:
  implementer:
    model: opus                                     # default: opus
```

**Merge priority:** mot.yml > defaults

### Agent Override

`.mot/implementer.md` in project root replaces the built-in implementer prompt. Only the prompt body is replaced. `mot.yml` still controls model unless the override file has a frontmatter `model` field.

**Model precedence:** override file frontmatter > `mot.yml` > default (opus)

## Status File

**Path:** `master-of-tasks-reports/status.json` (configurable via `status_dir` in `mot.yml`)

```json
{
  "started": "2026-03-22T14:30:00",
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
          "iterations": 0,
          "duration_seconds": 120,
          "timestamp": "2026-03-22T14:35:00"
        },
        {
          "number": 2,
          "description": "Add auth middleware",
          "status": "completed",
          "files": ["src/middleware/auth.go", "src/middleware/auth_test.go"],
          "commit": "def5678",
          "review": "pass",
          "findings": 3,
          "iterations": 2,
          "duration_seconds": 480,
          "timestamp": "2026-03-22T14:42:00"
        }
      ]
    },
    {
      "path": "plans/2026-03-22-sso.md",
      "status": "pending",
      "tasks": []
    }
  ]
}
```

Created at the start of `/mot`. Updated after each task completion. Read by `/mot-status` and `/mot-resume`.

The `status_dir` directory is created automatically if it doesn't exist.

## Metrics

### Per-Task Metrics (in status.json)

Each task entry includes:
- `duration_seconds` — implementation + review time
- `findings` — number of review findings
- `iterations` — how many times the review fix loop ran
- `commit` — implementation commit hash (fix commits tracked by ClaudeReviewers)

### Summary Report (post-completion)

Generated at `master-of-tasks-reports/YYYY-MM-DD-HHmm-summary.md` when all plans complete.

```markdown
# MasterOfTasks Summary — 2026-03-22 16:30

## Plans Executed
- plans/2026-03-22-auth.md — completed
- plans/2026-03-22-sso.md — completed

## Metrics

| Task | Plan | Duration | Findings | Fixed | Iterations | Commits |
|---|---|---|---|---|---|---|
| Task 1: scaffold | auth | 2m | 0 | 0 | 0 | 1 |
| Task 2: auth middleware | auth | 8m | 3 | 3 | 2 | 3 |
| Task 3: user model | auth | 5m | 1 | 1 | 1 | 2 |

## Totals
- Total duration: 45m
- Total findings: 12
- Total fixed: 11
- Unresolved: 1
- Total commits: 18
- Average iterations per task: 1.4

## Review Results
- Light reviews (post-task): 8 passed, 1 failed
- Full reviews (post-plan): 2 passed

## Issues Log
- Task 5 (sso/token-refresh): 1 unresolved finding — description from review report
```

The summary report aggregates metrics from status.json into a human-readable format.

## Error Handling

**ClaudeReviewers not installed:** Error at startup: "ClaudeReviewers required. Install with: claude plugin add git@github.com:saropatzi/ClaudeReviewers.git"

**Plan file not found:** Error: "Plan not found: `<path>`. Check the file path."

**Status dir doesn't exist:** Create automatically.

**Review fails (exit 1):** Stop execution. Show findings to user. Ask how to proceed. Do not continue to next task.

**Implementer agent fails:** Stop execution. Show error to user. Ask how to proceed.

**Existing status.json for same plans:** When starting `/mot`, if status.json exists with matching plans, suggest: "Previous execution found. Use /mot-resume to continue, or delete `<status_dir>/status.json` to start fresh."

**User cancellation:** Status.json retains progress. User can resume later with `/mot-resume`.

## Agent Invocation

All agents are invoked via the Claude Code `Agent` tool:

- **Implementer:** dispatched with task context + plan context + previous task results. Model resolved from override frontmatter > mot.yml > default.
- **Review:** invoked by calling `/review` via the Skill tool (delegates to ClaudeReviewers plugin).

**Configuration parsing:** `mot.yml` is read via the Read tool and parsed by the LLM in-context.

## Future Work

### Parallelism + Git Worktrees (v2)
Independent tasks (touching different files) executed in parallel, each in its own git worktree. Merge back to main branch on completion. Requires: dependency analysis between tasks, worktree management, concurrent status.json updates, coordinated review.

### Specialized Implementer Agents
Multiple implementer agents selectable per task: backend, frontend, infrastructure, full-stack. Task in the plan specifies which agent to use. Override via `.mot/implementer-<type>.md`.

### ClaudePlanners Integration
`/mot --create` flag that first invokes `/plan` to create a plan, then immediately executes it. Single command for the full cycle.

### Enhanced Metrics
Token consumption tracking per task, cost estimation, historical trends across executions, performance comparison between plans.
