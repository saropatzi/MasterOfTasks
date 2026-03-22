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
