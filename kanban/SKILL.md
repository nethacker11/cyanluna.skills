---
name: kanban
description: Manage project tasks in a local SQLite DB (.claude/kanban.db). Supports 7-column AI team pipeline (Req → Plan → Review Plan → Impl → Review Impl → Test → Done), session context persistence, task CRUD, lifecycle documentation, and automated code review. Run with /kanban.
license: MIT
---

Manages project tasks in a project-local `.claude/kanban.db` SQLite database.
The DB lives inside the project directory, so it travels with the project and can be version-controlled.

## DB Path

```
{project_root}/.claude/kanban.db
```

Auto-creates the `.claude/` directory and DB file if missing.

## Table Schema

```sql
CREATE TABLE IF NOT EXISTS tasks (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  project TEXT NOT NULL,
  title TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'todo',
  priority TEXT NOT NULL DEFAULT 'medium',
  description TEXT,
  plan TEXT,
  implementation_notes TEXT,
  tags TEXT,
  review_comments TEXT,
  plan_review_comments TEXT,
  test_results TEXT,
  agent_log TEXT,
  current_agent TEXT,
  plan_review_count INTEGER NOT NULL DEFAULT 0,
  impl_review_count INTEGER NOT NULL DEFAULT 0,
  created_at TEXT DEFAULT (datetime('now')),
  started_at TEXT,
  planned_at TEXT,
  reviewed_at TEXT,
  tested_at TEXT,
  completed_at TEXT
);
```

| Column | Type | Description |
|--------|------|-------------|
| `project` | TEXT | Project identifier. Uses `basename "$(pwd)"` |
| `status` | TEXT | `todo` / `plan` / `plan_review` / `impl` / `impl_review` / `test` / `done` |
| `priority` | TEXT | `high` / `medium` / `low` |
| `description` | TEXT | **Requirements** in markdown - what needs to be done |
| `plan` | TEXT | **Implementation plan** in markdown - how to do it |
| `implementation_notes` | TEXT | **Implementation log** in markdown - what was actually done |
| `tags` | TEXT | JSON array string (e.g., `'["api","ui","db"]'`) |
| `review_comments` | TEXT | JSON array of impl review comment objects |
| `plan_review_comments` | TEXT | JSON array of plan review comment objects |
| `test_results` | TEXT | JSON array of test result objects |
| `agent_log` | TEXT | JSON array of agent activity log entries |
| `current_agent` | TEXT | Currently active agent name |
| `plan_review_count` | INTEGER | Number of plan review iterations |
| `impl_review_count` | INTEGER | Number of impl review iterations |
| `level` | INTEGER | Pipeline level: 1 (Quick), 2 (Standard), 3 (Full) |

## Pipeline Levels

Tasks have a `level` (1-3) that determines the pipeline path:

| Level | Path | Use Case |
|-------|------|----------|
| L1 Quick | `Req → Impl → Done` | File cleanup, config changes, typo fixes |
| L2 Standard | `Req → Plan → Impl → Review → Done` | Feature edits, bug fixes, refactoring |
| L3 Full | `Req → Plan → Plan Rev → Impl → Impl Rev → Test → Done` | New features, architecture changes |

Level is set when creating a task (`/kanban add`) and stored in the `level` column.

## 7-Column AI Team Pipeline

```
Req → Plan → Review Plan → Impl → Review Impl → Test → Done
```

| Column | Status | Agent | Model | Writes to |
|--------|--------|-------|-------|-----------|
| Req | `todo` | User | - | `description` |
| Plan | `plan` | Plan Agent | opus (Task) | `plan` |
| Review Plan | `plan_review` | Review Agent | gemini-3-pro-preview / copilot(opus) / sonnet | `plan_review_comments` |
| Impl | `impl` | Worker → TDD Tester (sequential) | opus → sonnet | `implementation_notes` |
| Review Impl | `impl_review` | Code Review Agent | gemini-3-pro-preview / copilot(opus) / sonnet | `review_comments` |
| Test | `test` | Test Runner | sonnet (Task) | `test_results` |
| Done | `done` | - | - | - |

### Valid Status Transitions

```
todo        → plan
plan        → plan_review, todo
plan_review → impl (approve), plan (reject)
impl        → impl_review
impl_review → test (approve), impl (reject)
test        → done (pass), impl (fail)
done        → (terminal)
```

### Card Lifecycle (7 Phases)

Each card captures the full workflow. Clicking a card in the web board shows all phases in a modal:

```
Phase 1: Requirements       (description)            - What needs to be done
Phase 2: Plan                (plan)                   - How to approach it
Phase 3: Plan Review         (plan_review_comments)   - Plan verification
Phase 4: Implementation      (implementation_notes)   - What was actually changed
Phase 5: Implementation Review (review_comments)      - Code review results
Phase 6: Test                (test_results)            - Test execution results
Phase 7: Done                                          - Completed
```

### Comment Formats

#### review_comments / plan_review_comments Format
```json
[
  {
    "reviewer": "gemini",
    "status": "changes_requested",
    "comment": "## Review Findings\n\n1. Missing error handling\n2. Type safety issues",
    "timestamp": "2026-02-20T14:30:00.000Z"
  }
]
```

#### test_results Format
```json
[
  {
    "tester": "test-runner-agent",
    "status": "pass",
    "lint": "0 errors, 0 warnings",
    "build": "Build successful",
    "tests": "42 passed, 0 failed",
    "comment": "All checks passed",
    "timestamp": "2026-02-20T15:00:00.000Z"
  }
]
```

#### agent_log Format

**IMPORTANT**: Always include `model` field to track which AI model performed each step.

```json
[
  {
    "agent": "plan-agent",
    "model": "opus",
    "message": "Started planning for task #5",
    "timestamp": "2026-02-20T14:00:00.000Z"
  },
  {
    "agent": "review-agent",
    "model": "gemini",
    "message": "Plan review completed: approved",
    "timestamp": "2026-02-20T14:05:00.000Z"
  }
]
```

Standard agent + model combinations:
| Agent | Model |
|-------|-------|
| `plan-agent` | `opus` |
| `worker-agent` | `opus` |
| `tdd-tester` | `sonnet` |
| `review-agent` | `gemini-3-pro-preview` / `copilot-opus` / `sonnet` (whichever was used) |
| `code-review-agent` | `gemini-3-pro-preview` / `copilot-opus` / `sonnet` |
| `test-runner` | `sonnet` |

## DB Access

### Priority: HTTP API → sqlite3 CLI

1. **HTTP API** (preferred when kanban-board dev server is running): `http://localhost:5173`
2. **sqlite3 CLI** (when dev server is not running, or for direct queries/bulk ops): `sqlite3 .claude/kanban.db`

**IMPORTANT**: Do NOT use Python (`python3 -c "import sqlite3..."`) for DB access. Always use the `sqlite3` CLI command which is installed at `/usr/bin/sqlite3`.

```bash
# Quick read example
sqlite3 -json .claude/kanban.db "SELECT id, title, status, priority FROM tasks WHERE project='$(basename $(pwd))' ORDER BY id"

# Quick update example
sqlite3 .claude/kanban.db "UPDATE tasks SET status='impl', started_at=datetime('now') WHERE id=$ID"
```

Base URL: `http://localhost:5173` (default kanban-board port)

### API Endpoints

```bash
# Read task
curl -s http://localhost:5173/api/task/$ID | jq .

# Read board
curl -s "http://localhost:5173/api/board?project=$PROJECT" | jq .

# Update task (fields + status)
curl -s -X PATCH http://localhost:5173/api/task/$ID \
  -H 'Content-Type: application/json' \
  -d '{"plan": "...", "status": "plan_review"}'

# Create task
curl -s -X POST http://localhost:5173/api/task \
  -H 'Content-Type: application/json' \
  -d '{"title": "...", "project": "...", "priority": "medium", "description": "..."}'

# Plan review result
curl -s -X POST http://localhost:5173/api/task/$ID/plan-review \
  -H 'Content-Type: application/json' \
  -d '{"reviewer": "gemini", "status": "approved", "comment": "Plan looks good"}'

# Impl review result
curl -s -X POST http://localhost:5173/api/task/$ID/review \
  -H 'Content-Type: application/json' \
  -d '{"reviewer": "gemini", "status": "approved", "comment": "Code looks good"}'

# Test result
curl -s -X POST http://localhost:5173/api/task/$ID/test-result \
  -H 'Content-Type: application/json' \
  -d '{"tester": "test-runner", "status": "pass", "lint": "...", "build": "...", "tests": "...", "comment": "..."}'

# Reorder / drag-and-drop
curl -s -X PATCH http://localhost:5173/api/task/$ID/reorder \
  -H 'Content-Type: application/json' \
  -d '{"status": "plan", "afterId": null, "beforeId": null}'
```

## Project Name Detection

Uses the basename of the current working directory:
```bash
basename "$(pwd)"
```

## Commands

### View Board (Default)
`/kanban` or `/kanban list`

Read the board via API and output as a markdown table:
```bash
BOARD=$(curl -s "http://localhost:5173/api/board?project=$PROJECT")
```

Output format:
```
### PROJECT Kanban Board

| ID | Status | Priority | Title |
|----|--------|----------|-------|
| 3  | impl | high | Category Rules UI |
| 7  | impl_review | medium | API Error Handling |
| 1  | todo | medium | Monthly Budget |
| 10 | done | - | Expense Flag |
```

If the kanban-board dev server is not running, fall back to sqlite3:
```bash
sqlite3 -header -column .claude/kanban.db \
  "SELECT id, title, status, priority FROM tasks WHERE project='PROJECT' ORDER BY CASE status WHEN 'impl' THEN 0 WHEN 'impl_review' THEN 1 WHEN 'plan' THEN 2 WHEN 'plan_review' THEN 3 WHEN 'test' THEN 4 WHEN 'todo' THEN 5 WHEN 'done' THEN 6 END, id"
```

### Context (Session Handoff)
`/kanban context`

**Run this first when starting a new session.** Shows pipeline state across all columns:
```bash
BOARD=$(curl -s "http://localhost:5173/api/board?project=$PROJECT")
```

Output format:
```
### Pipeline Status

🔨 Implementing
- [#3] Category Rules UI (high)
  Plan: ...
  Implementation Notes: ...

🔍 Plan Review
- [#5] New Feature (medium)
  Plan Review: approved by gemini

📝 Impl Review
- [#7] API Error Handling (medium)
  Latest review: changes_requested - "Need error handling"

🧪 Testing
- [#8] Auth Module (high)
  Test: pass - lint OK, build OK, 42/42 tests

✅ Recently Done
- [#10] Expense Flag (2026-02-20)

📋 Next To Do
- [#1] Monthly Budget (medium)
```

### Add Task
`/kanban add <title>`

1. Ask the user for priority, level (L1/L2/L3), description, and tags (use AskUserQuestion)
2. Create via API:
```bash
curl -s -X POST http://localhost:5173/api/task \
  -H 'Content-Type: application/json' \
  -d "{\"title\": \"$TITLE\", \"project\": \"$PROJECT\", \"priority\": \"$PRIORITY\", \"level\": $LEVEL, \"description\": \"$DESC\"}"
```
3. Output confirmation with the new task ID

### Move Task
`/kanban move <ID> <status>`

```bash
curl -s -X PATCH http://localhost:5173/api/task/$ID \
  -H 'Content-Type: application/json' \
  -d "{\"status\": \"$STATUS\"}"
```

The API enforces valid transitions. Invalid moves return 400 with allowed transitions.

### Run Pipeline
`/kanban run <ID>` — Execute the full AI team pipeline for a task

**Default mode**: Pauses for user confirmation at Plan Review approval and Impl Review approval.
**Auto mode**: `/kanban run <ID> --auto` — Fully automatic (no pauses except circuit breaker).

#### Pipeline Loop (Level-Aware)

The pipeline path depends on the task's `level`:

```
L1 Quick:
  1. todo → Worker(opus) implements → done
  Complete!

L2 Standard:
  1. todo → Plan Agent (opus) → impl (skip plan_review)
  2. impl → Worker(opus) then TDD Tester(sonnet) → impl_review
  3. impl_review → Code Review → user confirm → approve:done / reject:impl
  4. done → Complete!

L3 Full:
  1. todo → Plan Agent (opus) → plan_review
  2. plan_review → Review Agent → user confirm → approve:impl / reject:plan
  3. impl → Worker(opus) then TDD Tester(sonnet) → impl_review
  4. impl_review → Code Review → user confirm → approve:test / reject:impl
  5. test → Test Runner(sonnet) → pass:done / fail:impl
  6. done → Complete!

Circuit breaker: plan_review_count > 3 OR impl_review_count > 3 → stop and ask user
```

Read the task's `level` field first to determine which steps to execute.

#### Implementation

1. **Read current task state**:
```bash
TASK=$(curl -s http://localhost:5173/api/task/$ID)
STATUS=$(echo "$TASK" | jq -r '.status')
```

2. **Execute appropriate agent based on current status** (see Agent Dispatch below)

3. **Loop until done or blocked**:
   - After each agent completes, re-read task state
   - If status progressed, continue to next agent
   - If review rejected, loop back automatically
   - If circuit breaker triggers, stop and notify user

#### Agent Dispatch

Based on task status, dispatch the appropriate agent:

**`todo` → Plan Agent**:
```
Use Task tool: model="opus", subagent_type="general-purpose"
```

Plan Agent prompt:
```
You are a Plan Agent for Kanban task #<ID>.

## Task Info
- Title: <title>
- Requirements: <description>

## Your Job
1. Read the requirements carefully
2. Analyze the codebase to understand the current state
3. Create a detailed implementation plan in markdown
4. Write the plan to the task card via API

## Output
Write a markdown plan with:
- Files to modify/create
- Step-by-step approach
- Key design decisions
- Edge cases to handle

## Record Results
curl -s -X PATCH http://localhost:5173/api/task/<ID> \
  -H 'Content-Type: application/json' \
  -d '{"plan": "<PLAN_MARKDOWN>", "status": "plan_review", "current_agent": "plan-agent"}'

Also append to agent_log:
curl -s -X PATCH http://localhost:5173/api/task/<ID> \
  -H 'Content-Type: application/json' \
  -d '{"agent_log": "<UPDATED_LOG_JSON>"}'
```

**`plan_review` → Review Agent (external CLI or sonnet)**:

Detect available review CLI (priority: gemini → copilot → sonnet):
```bash
if command -v gemini &>/dev/null; then
  REVIEWER="gemini"
elif command -v copilot &>/dev/null; then
  REVIEWER="copilot"
else
  REVIEWER="sonnet"
fi
```

For gemini CLI (always use `--model gemini-3-pro-preview`):
```bash
TASK_JSON=$(curl -s http://localhost:5173/api/task/$ID)
REVIEW_RESULT=$(echo "$TASK_JSON" | jq -r '{title, description, plan}' | gemini --sandbox --model gemini-3-pro-preview <<'REVIEW_EOF'
Review this implementation plan. Evaluate:
1. Is the plan complete and addresses all requirements?
2. Are there missing edge cases?
3. Is the approach sound?

Respond with a JSON object:
{"status": "approved" or "changes_requested", "comment": "your review in markdown"}
REVIEW_EOF
)
```

For copilot CLI fallback (always use `--model claude-opus-4.6`):
```bash
TASK_JSON=$(curl -s http://localhost:5173/api/task/$ID)
REVIEW_INPUT=$(echo "$TASK_JSON" | jq -r '{title, description, plan}')
REVIEW_RESULT=$(copilot -p "Review this implementation plan: $REVIEW_INPUT. Evaluate: 1) Is the plan complete and addresses all requirements? 2) Are there missing edge cases? 3) Is the approach sound? Respond ONLY with a JSON object: {\"status\": \"approved\" or \"changes_requested\", \"comment\": \"your review in markdown\"}" --model claude-opus-4.6 2>&1)
```

For sonnet fallback (Task tool):
```
Use Task tool: model="sonnet", subagent_type="general-purpose"
```

Record result:
```bash
curl -s -X POST http://localhost:5173/api/task/$ID/plan-review \
  -H 'Content-Type: application/json' \
  -d "{\"reviewer\": \"$REVIEWER\", \"status\": \"$REVIEW_STATUS\", \"comment\": \"$REVIEW_COMMENT\"}"
```

Default mode: After review, ask user with AskUserQuestion whether to accept/reject.
Auto mode (`--auto`): Auto-accept the review agent's decision.

**`impl` → Worker Agent (opus) then TDD Tester (sonnet) — sequential**:

Step 1 - Worker Agent:
```
Use Task tool: model="opus", subagent_type="general-purpose"
```

Worker Agent prompt:
```
You are a Worker Agent implementing Kanban task #<ID>.

## Task Info
- Title: <title>
- Requirements: <description>
- Plan: <plan>
- Plan Review Comments: <plan_review_comments>

## Your Job
1. Follow the plan to implement the changes
2. Write clean, well-tested code
3. Document what you changed

## Record Results
After implementation, update the task:
curl -s -X PATCH http://localhost:5173/api/task/<ID> \
  -H 'Content-Type: application/json' \
  -d '{"implementation_notes": "<NOTES_MARKDOWN>", "current_agent": "worker-agent"}'

Also append to agent_log.
Do NOT change the status - the orchestrator handles that.
```

Step 2 - TDD Tester (runs after Worker completes):
```
Use Task tool: model="sonnet", subagent_type="general-purpose"
```

TDD Tester prompt:
```
You are a TDD Tester for Kanban task #<ID>.

## Task Info
- Title: <title>
- Requirements: <description>
- Implementation Notes: <implementation_notes>

## Your Job
1. Read the implementation notes to understand what was changed
2. Write or update tests for the new/modified code
3. Ensure test coverage for edge cases
4. Append your test notes to implementation_notes

## Record Results
curl -s -X PATCH http://localhost:5173/api/task/<ID> \
  -H 'Content-Type: application/json' \
  -d '{"implementation_notes": "<UPDATED_NOTES>", "current_agent": "tdd-tester"}'

Do NOT change the status.
```

After both complete, move to impl_review:
```bash
curl -s -X PATCH http://localhost:5173/api/task/$ID \
  -H 'Content-Type: application/json' \
  -d '{"status": "impl_review", "current_agent": null}'
```

**`impl_review` → Code Review Agent**:

Same reviewer detection as plan_review (gemini → copilot → sonnet).

For gemini CLI (always use `--model gemini-3-pro-preview`):
```bash
TASK_JSON=$(curl -s http://localhost:5173/api/task/$ID)
REVIEW_RESULT=$(echo "$TASK_JSON" | jq -r '{title, description, plan, implementation_notes}' | gemini --sandbox --model gemini-3-pro-preview <<'REVIEW_EOF'
Review this code implementation. Evaluate:
1. Code quality: readability, duplication, naming
2. Error handling: proper try-catch, error messages
3. Type safety: TypeScript types, minimize any usage
4. Security: SQL injection, XSS, input validation
5. Performance: unnecessary queries, memory usage

Respond with a JSON object:
{"status": "approved" or "changes_requested", "comment": "your review in markdown"}
REVIEW_EOF
)
```

For copilot CLI (always use `--model claude-opus-4.6`):
```bash
TASK_JSON=$(curl -s http://localhost:5173/api/task/$ID)
REVIEW_INPUT=$(echo "$TASK_JSON" | jq -r '{title, description, plan, implementation_notes}')
REVIEW_RESULT=$(copilot -p "Review this code implementation: $REVIEW_INPUT. Evaluate: 1) Code quality: readability, duplication, naming 2) Error handling: proper try-catch, error messages 3) Type safety: TypeScript types, minimize any usage 4) Security: SQL injection, XSS, input validation 5) Performance: unnecessary queries, memory usage. Respond ONLY with a JSON object: {\"status\": \"approved\" or \"changes_requested\", \"comment\": \"your review in markdown\"}" --model claude-opus-4.6 2>&1)
```

Record result:
```bash
curl -s -X POST http://localhost:5173/api/task/$ID/review \
  -H 'Content-Type: application/json' \
  -d "{\"reviewer\": \"$REVIEWER\", \"status\": \"$REVIEW_STATUS\", \"comment\": \"$REVIEW_COMMENT\"}"
```

Default mode: Ask user with AskUserQuestion whether to accept/reject.
Auto mode: Auto-accept the review agent's decision.

**`test` → Test Runner Agent**:
```
Use Task tool: model="sonnet", subagent_type="general-purpose"
```

Test Runner prompt:
```
You are a Test Runner Agent for Kanban task #<ID>.

## Task Info
- Title: <title>
- Implementation Notes: <implementation_notes>

## Your Job
1. Run lint checks
2. Run build
3. Run tests
4. Report results

## Record Results
curl -s -X POST http://localhost:5173/api/task/<ID>/test-result \
  -H 'Content-Type: application/json' \
  -d '{"tester": "test-runner", "status": "pass" or "fail", "lint": "...", "build": "...", "tests": "...", "comment": "..."}'
```

### Step (Single Step)
`/kanban step <ID>` — Execute only the next pipeline step for a task

Same as `/kanban run` but exits after one step instead of looping.

### Agents
`/kanban agents` — Show available agents

Detect and display:
```bash
echo "## Available Agents"
echo ""
echo "| Agent | Model | Available |"
echo "|-------|-------|-----------|"

if command -v gemini &>/dev/null; then
  echo "| Review Agent | gemini | ✅ |"
else
  echo "| Review Agent | gemini | ❌ |"
fi

if command -v copilot &>/dev/null; then
  echo "| Review Agent | copilot (claude-opus-4.6) | ✅ (fallback #1) |"
else
  echo "| Review Agent | copilot | ❌ |"
fi

echo "| Plan Agent | opus (Task) | ✅ |"
echo "| Worker Agent | opus (Task) | ✅ |"
echo "| TDD Tester | sonnet (Task) | ✅ |"
echo "| Review Agent | sonnet (Task) | ✅ (fallback #2) |"
echo "| Test Runner | sonnet (Task) | ✅ |"
```

### Review
`/kanban review <ID>`

When a task is in `impl_review` status, trigger a Code Review agent (same as impl_review step in the pipeline).

### Edit Task
`/kanban edit <ID>`

Ask the user which fields to modify, then update via API.

### Delete Task
`/kanban remove <ID>`

```bash
# Via API (if dev server running)
curl -s -X DELETE http://localhost:5173/api/task/$ID

# Via sqlite3 CLI (if dev server not running)
sqlite3 .claude/kanban.db "DELETE FROM tasks WHERE id=$ID;"
```

### Stats
`/kanban stats`

```bash
BOARD=$(curl -s "http://localhost:5173/api/board?project=$PROJECT")
echo "$BOARD" | jq '{
  todo: (.todo | length),
  plan: (.plan | length),
  plan_review: (.plan_review | length),
  impl: (.impl | length),
  impl_review: (.impl_review | length),
  test: (.test | length),
  done: (.done | length),
  total: ((.todo + .plan + .plan_review + .impl + .impl_review + .test + .done) | length)
}'
```

## Error Handling

### Agent Failure
- 1 retry on first failure
- 2nd failure: keep current status, log error to `agent_log`, notify user

### External CLI Failure
- `which gemini` not found → try `copilot -p --model claude-opus-4.6` → fallback to `sonnet` (Task tool)
- CLI execution error (including rate limit) → log to `agent_log`, try next fallback
- gemini rate limit → copilot fallback → sonnet fallback

### Review Rejection Loop (Circuit Breaker)
- `plan_review_count > 3`: stop loop, ask user for guidance
- `impl_review_count > 3`: stop loop, ask user for guidance
- In `--auto` mode: circuit breaker still fires, loop stops, user intervention required

### Mid-Pipeline Crash
- Current status is preserved (no partial transitions)
- Error logged to `agent_log`
- User notified of the failure

## Agent Context Flow (Card = Communication Channel)

Each agent reads all card fields and writes to its designated field:

```
Plan Agent   → reads: description
              → writes: plan
              → moves: todo → plan_review

Review Agent → reads: description, plan
              → writes: plan_review_comments
              → moves: plan_review → impl (approved) or plan (rejected)

Worker Agent → reads: description, plan, plan_review_comments
              → writes: implementation_notes
              → (no status change)

TDD Tester   → reads: description, implementation_notes
              → writes: implementation_notes (appends)
              → moves: impl → impl_review (after both complete)

Code Review  → reads: description, plan, implementation_notes
              → writes: review_comments
              → moves: impl_review → test (approved) or impl (rejected)

Test Runner  → reads: implementation_notes
              → writes: test_results
              → moves: test → done (pass) or impl (fail)

All agents   → append to: agent_log
```

## Initial Setup

Auto-creates DB if missing. Use `sqlite3` CLI (NOT python):
```bash
mkdir -p .claude
sqlite3 .claude/kanban.db "
  CREATE TABLE IF NOT EXISTS tasks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    project TEXT NOT NULL,
    title TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'todo',
    priority TEXT NOT NULL DEFAULT 'medium',
    description TEXT,
    plan TEXT,
    implementation_notes TEXT,
    tags TEXT,
    review_comments TEXT,
    plan_review_comments TEXT,
    test_results TEXT,
    agent_log TEXT,
    current_agent TEXT,
    plan_review_count INTEGER NOT NULL DEFAULT 0,
    impl_review_count INTEGER NOT NULL DEFAULT 0,
    level INTEGER NOT NULL DEFAULT 3,
    attachments TEXT,
    notes TEXT,
    rank INTEGER NOT NULL DEFAULT 0,
    created_at TEXT DEFAULT (datetime('now')),
    started_at TEXT,
    planned_at TEXT,
    reviewed_at TEXT,
    tested_at TEXT,
    completed_at TEXT
  );
"
```

## .gitignore

Choose whether to track the DB in git:
```bash
# To exclude (personal use):
echo ".claude/kanban.db" >> .gitignore
echo ".claude/kanban.db-wal" >> .gitignore
echo ".claude/kanban.db-shm" >> .gitignore
```

## Agent Workflow (Lifecycle Documentation)

The implementation agent MUST record documentation at each phase. Summarize what you would normally output in chat as markdown and write it to the card.

### Step 1: Start Pipeline

```bash
# Move to plan
curl -s -X PATCH http://localhost:5173/api/task/$ID \
  -H 'Content-Type: application/json' \
  -d '{"status": "plan", "current_agent": "plan-agent"}'
```

### Step 2: Record Plan

```bash
# Record plan via API
curl -s -X PATCH http://localhost:5173/api/task/$ID \
  -H 'Content-Type: application/json' \
  -d '{"plan": "## Implementation Plan\n\n### Files to Modify\n- src/lib/xxx.ts\n\n### Approach\n1. First modify XXX\n2. Then add YYY", "status": "plan_review"}'
```

### Step 3: Plan Review

```bash
# Submit plan review
curl -s -X POST http://localhost:5173/api/task/$ID/plan-review \
  -H 'Content-Type: application/json' \
  -d '{"reviewer": "gemini", "status": "approved", "comment": "Plan is thorough and complete."}'
```

### Step 4: Implementation

```bash
# Record implementation
curl -s -X PATCH http://localhost:5173/api/task/$ID \
  -H 'Content-Type: application/json' \
  -d '{"implementation_notes": "## Changes\n\n### Modified Files\n- src/lib/xxx.ts: Added feature\n\n### Tests Added\n- test/xxx.test.ts: 5 new tests"}'

# Move to impl_review
curl -s -X PATCH http://localhost:5173/api/task/$ID \
  -H 'Content-Type: application/json' \
  -d '{"status": "impl_review"}'
```

### Step 5: Code Review

```bash
# Submit code review
curl -s -X POST http://localhost:5173/api/task/$ID/review \
  -H 'Content-Type: application/json' \
  -d '{"reviewer": "gemini", "status": "approved", "comment": "Code quality is good."}'
```

### Step 6: Test

```bash
# Submit test results
curl -s -X POST http://localhost:5173/api/task/$ID/test-result \
  -H 'Content-Type: application/json' \
  -d '{"tester": "test-runner", "status": "pass", "lint": "0 errors", "build": "OK", "tests": "42 passed", "comment": "All checks pass."}'
```

### Summary

| Phase | Field | Content | Written By |
|-------|-------|---------|------------|
| Requirements | `description` | What needs to be done | User |
| Plan | `plan` | How to approach it | Plan Agent (opus) |
| Plan Review | `plan_review_comments` | Plan verification | Review Agent (gemini-3-pro-preview / copilot-opus / sonnet) |
| Implementation | `implementation_notes` | What was changed + tests | Worker (opus) + TDD Tester (sonnet) |
| Impl Review | `review_comments` | Code review results | Code Review Agent (gemini-3-pro-preview / copilot-opus / sonnet) |
| Test | `test_results` | Lint/build/test results | Test Runner (sonnet) |

## Web Board Viewer

Run `/kanban-init` to scaffold the web board in any project. It creates a `kanban-board/` directory with all files.

```bash
cd kanban-board && pnpm dev
```
Default port: 5173 (auto-increments if in use). Open the 7-column board (Req, Plan, Review Plan, Implement, Review Impl, Test, Done) with drag-and-drop (valid transitions only), card lifecycle modal with 7-step progress bar, add card form, agent log viewer, and 10s auto-refresh.
