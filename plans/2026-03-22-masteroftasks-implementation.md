# MasterOfTasks Plugin Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Claude Code plugin that orchestrates plan implementation with automated code review via ClaudeReviewers, supporting multi-plan execution, resume, status tracking, and metrics.

**Architecture:** Three user-invokable skills (`/mot` orchestrator, `/mot-status`, `/mot-resume`). The orchestrator reads plans in ClaudePlanners format, dispatches an implementer subagent per task, invokes ClaudeReviewers for light (post-task) and full (post-plan) reviews, tracks progress in `status.json`, and generates per-plan report files.

**Tech Stack:** Claude Code plugin (`.claude-plugin/plugin.json`), Markdown skills with YAML frontmatter, Agent tool for implementation, Skill tool for ClaudeReviewers invocation.

**Spec:** `docs/superpowers/specs/2026-03-22-masteroftasks-design.md`

---

## File Structure

```
MasterOfTasks/
├── .claude-plugin/
│   └── plugin.json                         # plugin metadata
├── .gitignore
├── LICENSE
├── README.md                               # full documentation
├── skills/
│   ├── mot/
│   │   ├── SKILL.md                        # orchestrator (/mot, alias /master-of-tasks)
│   │   └── agents/
│   │       └── implementer.md              # implementation agent
│   ├── mot-status/
│   │   └── SKILL.md                        # /mot-status
│   └── mot-resume/
│       └── SKILL.md                        # /mot-resume
```

---

### Task 1: Plugin Scaffold [non-TDD]

**Files:**
- Create: `.claude-plugin/plugin.json`
- Create: `.gitignore`
- Create: `LICENSE`

- [ ] **Step 1: Create plugin.json**

```json
{
  "name": "MasterOfTasks",
  "version": "1.0.0",
  "description": "Orchestrates plan implementation with automated code review loops",
  "author": { "name": "saropatzi" },
  "license": "MIT"
}
```

- [ ] **Step 2: Create .gitignore**

```
# OS
.DS_Store
Thumbs.db

# Editor
*.swp
*.swo
*~
.idea/
.vscode/
```

- [ ] **Step 3: Create LICENSE**

MIT license, copyright 2026 saropatzi.

- [ ] **Step 4: Commit**

```bash
git add .claude-plugin/plugin.json .gitignore LICENSE
git commit -m "feat: scaffold plugin with metadata and license"
```

---

### Task 2: Implementer Agent [non-TDD]

**Files:**
- Create: `skills/mot/agents/implementer.md`

- [ ] **Step 1: Write the implementer agent prompt**

```markdown
---
name: implementer
description: Implements plan tasks with TDD support, committing results
model: opus
---

You are a task implementer. You receive a task from a plan and implement it.

## Context You Receive

1. **Current task** — description, files involved, TDD/non-TDD indication, verification criteria
2. **Plan context** — Overview, Context, and Development Approach sections from the plan
3. **Previous tasks** — all completed tasks with description, files, and commit hash

## Implementation Rules

### TDD Tasks [TDD]
1. Write the test first
2. Run it — verify it fails
3. Implement the minimal code to make it pass
4. Run the test — verify it passes
5. Commit with: `git commit -m "feat: <task description>"`

### Non-TDD Tasks [non-TDD]
1. Implement the change
2. Write tests if applicable (not everything needs a test — config files, docs, scaffolding don't)
3. Run tests if they exist
4. Commit with: `git commit -m "feat: <task description>"`

## Scope

Stay within the scope of the assigned task:
- Files listed in the task
- Files directly required to integrate the change (e.g., updating a router to wire a new endpoint)
- Do NOT modify files unrelated to the task's goal

## Verification

If the task has verification criteria ("Verify: ..."), check them before committing:
- Run the specified checks
- If they fail, fix the implementation until they pass

## Commit Messages

- New functionality: `feat: <description>`
- Bug fixes: `fix: <description>`
- One commit per task (unless the task explicitly requires multiple steps)

## Output

When done, report:
- What was implemented
- Files created/modified
- Commit hash
- Test results (if applicable)
```

- [ ] **Step 2: Commit**

```bash
git add skills/mot/agents/implementer.md
git commit -m "feat: add implementer agent prompt"
```

---

### Task 3: Orchestrator Skill — `/mot` [non-TDD]

**Files:**
- Create: `skills/mot/SKILL.md`

- [ ] **Step 1: Write the orchestrator SKILL.md**

This is the core skill — the largest file. It must implement the full flow from the spec.

```markdown
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

10. **Update report file** (same directory as plan, same filename with `-reports` before `.md`. Example: `plans/2026-03-22-auth.md` → `plans/2026-03-22-auth-reports.md`):
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
```

- [ ] **Step 2: Commit**

```bash
git add skills/mot/SKILL.md
git commit -m "feat: add orchestrator skill (/mot)"
```

---

### Task 4: Status Skill — `/mot-status` [non-TDD]

**Files:**
- Create: `skills/mot-status/SKILL.md`

- [ ] **Step 1: Write the mot-status SKILL.md**

```markdown
---
name: mot-status
description: Show current MasterOfTasks execution status. Use when user invokes /mot-status.
allowed-tools: [Read, Bash]
argument-hint: (no arguments)
---

# MasterOfTasks — Status

Show the current state of plan execution.

## Steps

### 1. Read Configuration

Read `mot.yml` (if exists) to resolve `plan_dir` (default: `plans/`).

### 2. Read Status File

Read `<plan_dir>/status.json` via Read tool.

If not found: output "No execution in progress." and exit.

### 3. Display Status

For each plan in status.json:

Show:
- Plan path and status (pending / in_progress / completed)
- Tasks: completed / total
- Last commit hash (from most recent completed task)
- Last review result
- Duration so far (from `started` to `last_updated`)

Format as a clear, readable summary. Example:

```
Plan: plans/2026-03-22-auth.md [in_progress]
  Tasks: 3/5 completed
  Last commit: abc1234
  Last review: pass
  Duration: 15m

Plan: plans/2026-03-22-sso.md [pending]
  Tasks: 0/4
```

## Adapt Language

Respond in the same language the user uses.
```

- [ ] **Step 2: Commit**

```bash
git add skills/mot-status/SKILL.md
git commit -m "feat: add mot-status skill (/mot-status)"
```

---

### Task 5: Resume Skill — `/mot-resume` [non-TDD]

**Files:**
- Create: `skills/mot-resume/SKILL.md`

- [ ] **Step 1: Write the mot-resume SKILL.md**

```markdown
---
name: mot-resume
description: Resume interrupted MasterOfTasks execution. Use when user invokes /mot-resume.
allowed-tools: [Read, Glob, Grep, Bash, Agent, Edit, Write, Skill]
argument-hint: [plan1,plan2,...] (optional — resumes from status.json if omitted)
---

# MasterOfTasks — Resume

Resume an interrupted plan execution from where it stopped.

## Step 0: Parse Arguments

Optional comma-separated plan paths. If omitted, resume from the plan list in status.json.

## Step 1: Read Status

Read `mot.yml` (if exists) to resolve `plan_dir` (default: `plans/`).
Read `<plan_dir>/status.json`.

If not found: error "No previous execution found. Use /mot to start."

## Step 1b: Check Working Tree

Run `git status --porcelain` via Bash. If output is non-empty:
"Working tree has uncommitted changes. Commit or stash before proceeding?"
Wait for user confirmation.

## Step 2: Show State

Display current state to user:
- For each plan: status, tasks completed/total
- For in_progress tasks: show which task was interrupted
- Last commit, last review result

## Step 3: Pre-Resume Verification

Ask: "Do you want to verify anything before resuming?"

If yes: let the user inspect, run commands, check files. When they're ready, ask again.
If no: proceed.

## Step 4: Confirm Resume Point

Identify the first incomplete task (status: in_progress or pending).

"Resume from Task N of plan X?"

Wait for confirmation.

## Step 5: Resume Execution

**Resume granularity:** If a task has status `in_progress`, re-execute it from the beginning (implementation + review). The implementer works on top of the current working tree state — it does not revert partial work.

Continue with the same `/mot` flow (Steps 2a-2d from the orchestrator) from the resume point.

After all remaining plans complete, follow the same completion flow (Step 3 from orchestrator).

## Adapt Language

Respond in the same language the user uses.
```

- [ ] **Step 2: Commit**

```bash
git add skills/mot-resume/SKILL.md
git commit -m "feat: add mot-resume skill (/mot-resume)"
```

---

### Task 6: README.md [non-TDD]

**Files:**
- Create: `README.md` (overwrite existing minimal file)

- [ ] **Step 1: Write README**

IMPORTANT: Read the existing README.md first, then overwrite.

The README must cover (based on the spec):

1. **Title and description** — MasterOfTasks, orchestrates plan implementation with code review
2. **Prerequisites** — ClaudeReviewers must be installed (with install command)
3. **Installation** — from GitHub, from local path
4. **Quick Start** — basic `/mot plans/my-plan.md` example
5. **Commands:**
   - `/mot <plans>` — full flow, comma-separated plans, confirmation, execution loop
   - `/mot-status` — show current status
   - `/mot-resume` — resume interrupted execution, verification before resume
   - Alias: `/master-of-tasks` = `/mot`
6. **Plan Format** — expected format from ClaudePlanners, parsing rules (checkbox items, TDD tags, Files, Verify)
7. **Execution Loop** — per-task flow (implement → light review → update), per-plan flow (full review → complete → move to completed/)
8. **Review Integration:**
   - Light review (post-task): `--only quality,implementation`
   - Full review (post-plan): all agents
   - When review fails: options (retry, manual fix, skip, abort)
   - Difference between `--only` (restrictive) and `--agents` (additive)
9. **Configuration:** `mot.yml` with all fields, `mot/implementer.md` override, model precedence
10. **Status and Reports:**
    - `status.json` in plan_dir — progress tracking, persists after completion
    - `*-reports.md` alongside plans — metrics per task
    - Both move to `plans/completed/` when done
11. **Error Handling** — all error cases from spec
12. **Resume** — how interruption/resume works, granularity (in_progress tasks re-executed)
13. **License** — MIT

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add comprehensive README"
```

---

### Task 7: Test and Push [non-TDD]

- [ ] **Step 1: Install the plugin locally**

```bash
claude plugin add /c/Users/carru/Documents/ProgettiSviluppo/MasterOfTasks
```

- [ ] **Step 2: Verify `/mot` is available**

In a Claude Code session, verify the skill loads.

- [ ] **Step 3: Verify `/mot-status` and `/mot-resume` are available**

Test both commands.

- [ ] **Step 4: Push to remote**

```bash
cd /c/Users/carru/Documents/ProgettiSviluppo/MasterOfTasks
git push origin main
```
