---
name: mot
description: Orchestrates plan implementation with automated code review loops. Takes plans from ClaudePlanners, implements tasks via subagent, reviews with ClaudeReviewers. Use when user invokes /mot or /master-of-tasks.
allowed-tools: [Read, Glob, Grep, Bash, Agent, Edit, Write, Skill]
argument-hint: <plan1>,<plan2>,... (comma-separated plan paths)
---

# MasterOfTasks — Plan Execution Orchestrator

You orchestrate plan implementation with automated code review. Follow these steps precisely.

## Step 0: Parse Arguments

Parse the comma-separated plan paths from the argument. Trim whitespace around each path.
Paths containing commas are not supported.

## Step 1: INIT

### 1a. Verify ClaudeReviewers

Invoke `/review --skip-review` via the Skill tool. If it fails (skill not found), error:
"ClaudeReviewers required. Install with: claude plugin add git@github.com:saropatzi/ClaudeReviewers.git"

### 1b. Read Configuration

Read `mot.yml` in the project root (if it exists) via Read tool. Parse in-context.
If mot.yml exists but is not parseable, warn: "Warning: mot.yml could not be parsed. Using defaults."

**Defaults:**
- plan_dir: plans/
- review_agents_light: quality,implementation
- review_agents_full: all
- review_max_iterations: 3
- agents.implementer.model: opus

### 1c. Check Working Tree

Run `git status --porcelain` via Bash. If output is non-empty:
"Working tree has uncommitted changes. Commit or stash before proceeding?"
Wait for user confirmation.

### 1d. Confirm Plan Order

Show the plans in order:
"Plans received in this order:
1. plans/auth.md
2. plans/sso.md
Correct?"

Wait for confirmation. If user wants to reorder, accept the new order.

### 1e. Check for Existing Status

Read `<plan_dir>/status.json`. If it exists with matching plans:
"Previous execution found. Use /mot-resume to continue, or delete `<plan_dir>/status.json` to start fresh."

### 1f. Initialize Status File

Create `<plan_dir>/status.json` with:
```json
{
  "started": "<current ISO timestamp>",
  "last_updated": "<current ISO timestamp>",
  "completed": null,
  "plans": [<for each plan: {"path": "<path>", "status": "pending", "tasks": []}>]
}
```

Create plan_dir if it doesn't exist (Bash: `mkdir -p`).

## Step 2: Execute Plans

For each plan in order:

### 2a. Read Plan

Verify the plan file exists. If not: error "Plan not found: `<path>`. Check the file path."

Read the plan file. Parse tasks from checkbox items (`- [ ] Task N: ...`).
If no tasks found: error "No tasks found in `<path>`. Ensure the plan follows ClaudePlanners format with checkbox items."

Extract plan context sections (Overview, Context, Development Approach) for passing to the implementer.

Update status.json: set plan status to "in_progress", populate tasks array.

### 2b. Execute Tasks

For each task (sequential):

1. **Mark task in_progress** in status.json, update last_updated.

2. **Collect context for implementer:**
   - Current task: description, files, TDD/non-TDD, verification criteria
   - Plan context: Overview, Context, Development Approach sections
   - All previously completed tasks: description, files, commit hash (from status.json)

3. **Read implementer prompt:** Check if `mot/implementer.md` exists in project root (override). If yes, use it. Otherwise use built-in `skills/mot/agents/implementer.md`.

4. **Resolve model:** Override file frontmatter `model` > mot.yml `agents.implementer.model` > default (opus).

5. **Launch implementer agent** via Agent tool with collected context and resolved model.

6. **Handle implementer failure:** If the agent errors or fails, STOP. Show error to user. Present options:
   (a) retry task
   (b) manually fix, then retry
   (c) skip task and continue
   (d) abort entire execution

7. **Record implementation result:** Get commit hash from implementer output. If no commit produced, log warning "Task N produced no changes", set commit to null.

8. **Light review:** Invoke ClaudeReviewers via Skill tool:
   `/review --only <review_agents_light> --plan <plan_file> --max-iterations <review_max_iterations>`

9. **Handle review result:**
   - Exit 0: task passed. Update status.json: task completed, record commit, findings, fixed, iterations, duration, timestamp. Update plan file: mark checkbox `[x]`. Update report file.
   - Exit 1: STOP. Show findings to user. Present options:
     (a) retry review
     (b) manually fix, then retry
     (c) skip task and continue
     (d) abort entire execution

10. **Update report file** (same directory as plan, same filename with `-reports` before `.md`. Example: `plans/2026-03-22-auth.md` -> `plans/2026-03-22-auth-reports.md`):
   Append task metrics row to the report. Create the file if it doesn't exist with the full report header (title, Plans Executed section). Each task adds a row to the Metrics table.

### 2c. Full Review (post-plan)

After all tasks complete:

If `review_agents_full` is not `all`: invoke `/review --only <review_agents_full> --plan <plan_file> --max-iterations <review_max_iterations>`
If `review_agents_full` is `all`: invoke `/review --plan <plan_file> --max-iterations <review_max_iterations>` (no --only, all default agents run)

- Exit 0: plan passed.
- Exit 1: STOP. Show findings. Present options:
  (a) retry review (b) manually fix (c) skip plan (d) abort

### 2d. Complete Plan

1. Mark plan completed in status.json, update last_updated.
2. Finalize report file: add Totals section (total duration, findings, fixed, unresolved, commits, avg iterations), Review Results section (light reviews passed/failed, full reviews passed/failed), and Issues Log section (list unresolved findings with task reference and description).
3. Move plan file to `<plan_dir>/completed/` (create dir if needed via Bash: `mkdir -p`).
4. Move report file to `<plan_dir>/completed/`.

## Step 3: Completion

After all plans:
1. Set `completed` timestamp in status.json.
2. Output: "All plans completed. Reports at `<plan_dir>/completed/`"

## Adapt Language

Respond in the same language the user uses.
