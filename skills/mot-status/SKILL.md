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
