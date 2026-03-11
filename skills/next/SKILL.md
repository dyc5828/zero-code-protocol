---
name: next
description: Orchestrates task execution from TODO/. Analyzes dependencies, dispatches sub-agents one at a time with continuous reevaluation, coordinates completion and integration. Use when ready to work through the task backlog - "let's do some tasks", "work on backlog", "what's next", or just "/next".
metadata:
  author: danielchen
  version: 1.0.0
---

# Next (Task Orchestration)

The brain of the **Zero CODE** protocol - analyzes tasks, dispatches agents, coordinates completion. Handles both **O**rchestrate and **D**ispatch in the CODE flow.

## Skill Names

This document uses bare names (`/later`, `/next`, `/do`) for readability. Your invocation depends on install method:

| Install Method | Invocation |
|----------------|------------|
| Symlink or `npx skills add` | `/later`, `/next`, `/do` |
| Claude Code plugin | `/zero-code:later`, `/zero-code:next`, `/zero-code:do` |

## When to Use

- Ready to work through task backlog
- User says "let's do some tasks", "work on backlog", "what's next"
- Want to work through multiple tasks with intelligent prioritization
- After capturing several items with `/later`

## Responsibility

| Does | Does NOT |
|------|----------|
| Read and analyze TODO/ | Execute task work |
| Build dependency graph | Create worktrees |
| Dispatch agents with context | Verify code correctness |
| Track completion reports | Push/merge code directly |
| Archive completed tasks | Write code |
| Decide integration strategy | |

## Process

### Phase 1: Discovery

Use **todo-store** to detect and read from the TODO location:

```bash
# Detection algorithm (see todo-store for full details)
BASE=$(git worktree list --porcelain 2>/dev/null | grep '^worktree ' | head -1 | cut -d' ' -f2)
BASE="${BASE:-.}"

# Check for existing formats: TODO/, todo/, tasks/, TODO.md, etc.
# todo-store handles the detection and provides consistent interface
```

The todo-store skill handles various formats:
- **Folder-based:** `TODO/`, `todo/`, `tasks/` (one file per task)
- **File-based:** `TODO.md`, `todo.md`, `tasks.md` (all tasks in one file)

If no tasks found, inform user: "No tasks found. Use /later to capture work."

### Phase 2: Analysis

1. **Read each task file** - Parse frontmatter (priority, tags, blocks) and content
2. **Build dependency graph:**
   - Explicit: `blocks:` frontmatter creates edges
   - Inferred: Tasks referencing same files/features (light inference only)
3. **Identify task state:**
   - Available: Independent tasks (no blockers) - ready to dispatch
   - Blocked: Tasks waiting on dependencies
   - In-progress: Already dispatched to sub-agents

**Task selection bias (applied at each dispatch decision):**
- Newer tasks (recency signals relevance)
- Higher priority (from frontmatter)
- Unblocks others (enables more work)
- Not blocked by in-progress work

### Phase 3: Plan Presentation & Auto-Dispatch

Present brief task analysis, then **immediately begin dispatching work**:

```
=== Task Analysis ===
[X] available, [Y] blocked

Starting: task-name.md
```

**Auto-dispatch is the default behavior.** The user invoked `/next` to work through the backlog - that's authorization to proceed.

**ONLY pause for input if:**
- Conflicting dependencies need resolution
- Destructive actions detected in task content
- Ambiguous integration strategy needs clarification

Otherwise, start dispatching immediately without asking for confirmation.

### Phase 4: Continuous Dispatch with Full Reassessment Loop

**Architecture: Concurrent execution, sequential decisions with complete reevaluation**

This is NOT batch/wave dispatch. This IS:
- Read ENTIRE TODO list -> Evaluate current state -> Pick ONE task -> Dispatch it
- Read ENTIRE TODO list -> Reevaluate -> Pick ONE task -> Dispatch it
- Repeat while work remains

After each completion, **reassess the ENTIRE backlog** - don't just move to the next sequential task. Priorities shift, dependencies change, new context emerges.

Sub-agents run concurrently, but dispatch decisions are sequential and dynamic.

```
Track state:
- task_id -> output_file (for in-progress agents)
- task_id -> completion_report (for finished work)

WHILE tasks exist (available OR in-progress OR blocked):

    # ============================================
    # FULL REASSESSMENT: Read entire TODO list
    # ============================================

    all_todos = read_all_todo_files()  # Fresh read every iteration

    # Parse frontmatter, content, dependencies for ALL tasks
    # This ensures we catch:
    # - New tasks added mid-execution
    # - Priority changes from completed work
    # - Unblocked dependencies
    # - Changed project context

    # ============================================
    # EVALUATE: What should be worked on next?
    # ============================================

    available_tasks = all_todos.filter(
        status == "pending" AND
        NOT blocked_by_in_progress_work AND
        dependencies_met
    )

    IF available_tasks.empty:
        # Nothing new to dispatch, wait for completions
        GOTO completions

    # ============================================
    # DECIDE: Pick ONE task using selection bias
    # ============================================

    next_task = select_best_task(available_tasks, bias=[
        newer_tasks,
        higher_priority,
        unblocks_others,
        low_resource_conflict_risk
    ])

    # ============================================
    # DISPATCH: Spawn single sub-agent via Task tool
    # ============================================

    base_worktree = determine_base(next_task, completed_tasks)
    predecessor_summary = get_summary(next_task, completed_tasks)

    # Build prompt that includes BOTH task content AND /do protocol
    agent_prompt = build_dispatch_prompt(next_task, base_worktree, predecessor_summary)

    # Use Task tool to spawn sub-agent
    # (Claude Code: Task tool; other tools: equivalent background agent mechanism)
    task_output = Task(
        subject: next_task.subject,
        description: agent_prompt,  # This prompt contains /do execution protocol
        activeForm: "Executing " + next_task.subject
    )

    track_agent(next_task.id, task_output.id)
    next_task.status = "in_progress"

    log("Dispatched: " + next_task.id)

    # ============================================
    # COMPLETIONS: Wait for agents to finish
    # ============================================
    # Primary: Task tool completes when agent finishes (natural completion)
    # Fallback: Can check output files to see intermediate progress

    FOR each in_progress_task:
        # Task tool will complete when agent is done
        # Meanwhile, check if any have finished and reported
        IF task_completed(in_progress_task):
            report = parse_completion_report(in_progress_task.output_file)

            archive_task(in_progress_task.id, report)
            completed_tasks[in_progress_task.id] = report

            update_dependency_graph(in_progress_task.id, report.status)

            log("Completed: " + in_progress_task.id + " [" + report.status + "]")

            # State changed - loop will do FULL REASSESSMENT
            # Not just "what's next", but "re-read everything and decide fresh"

    brief_pause()  # Avoid busy-wait, let agents work
    # Loop continues - FULL reassessment with complete TODO list re-read
```

**Key distinctions:**
- This is not "dispatch a wave, wait, dispatch next wave"
- This is not "pick next sequential task from a list"
- This IS "re-read everything -> evaluate -> pick one -> dispatch" on every iteration
- Completing Task A might make Task C more important than Task B, even if B was "next"

### Phase 5: Integration

After all tasks complete, choose integration strategy:

**Detection signals:**
- `.github/workflows/` or `.gitlab-ci.yml` -> CI exists
- Branch protection -> check via `gh api` if available
- Multiple contributors -> team context

**Strategy selection:**

| Context | Strategy | Rationale |
|---------|----------|-----------|
| CI + branch protection | PR-per-task | Safest, enables review |
| CI, related tasks | PR-per-chain | Cleaner for feature work |
| Solo dev, no CI | Direct merge | Fast iteration |
| Uncertain | PR-per-task | Default to safest |

**Execute chosen strategy:**
- PR-per-task: `gh pr create` for each branch
- PR-per-chain: Single PR from final task's branch
- Direct merge: `git merge` each branch to main

Report completion summary.

## Agent Prompt Template

**CRITICAL DISPATCH REQUIREMENT:**

Sub-agents spawned via Task tool DO NOT automatically inherit the /do skill. They are fresh Claude instances with no knowledge of the Zero CODE protocol.

**The orchestrator MUST embed the complete /do execution protocol** into every dispatch prompt. This includes:
- Worktree creation steps (with exact bash commands)
- Verification requirements and patterns
- Commit conventions
- Exact completion report format

Without embedding the protocol, sub-agents will:
- Edit files directly on main instead of using worktrees
- Skip verification and claim success without testing
- Report completion in inconsistent formats
- Not know to use Task tool for background processes

**The prompt structure:**
1. Task content (from TODO file)
2. Execution context (base worktree, predecessor info)
3. **Complete /do protocol** (worktree creation, verification, reporting format)

### Full Dispatch Prompt Template

```
You are executing a task from the Zero CODE protocol.

## Task File
[Full content of the TODO file]

## Execution Context
- Base worktree: [main OR predecessor's worktree path]
- Predecessor summary: [Summary from completed task, if dependent]

---

# EXECUTION PROTOCOL (from /do skill)

You MUST follow this protocol to execute the task above.

## Verification Requirements (Orchestrator-Injected)

Before you implement anything, you MUST define how you'll verify this works.

**For application tasks (web apps, APIs, CLIs), lean into active verification:**

Verification techniques include:
- **Background server management** - Start servers, health checks, log monitoring
- **Browser automation** - Playwright tools for real user interactions
- **Client/server correlation** - Verify both sides agree on what happened
- **Resource management** - Find available ports, unique test DBs, cleanup

**Verification checklist:**

1. **What proves this works?**
   - Specific user action or API call?
   - Expected output or state change?
   - Test cases that should pass?

2. **What resources do you need?**
   - Servers? Which ports (check availability)?
   - Databases? Use unique names for isolation
   - Browser? What viewport or state?

3. **How will you verify?**
   - Start server -> interact -> check logs -> clean up?
   - Browser automation flow (navigate -> fill -> submit -> verify)?
   - API test with curl/fetch + response validation?
   - Existing test suite run?

4. **How will you know it worked?**
   - Client-side success indicators (UI messages, redirects)
   - Server-side success indicators (logs, status codes)
   - Correlation between client and server (do they agree?)

Write your verification plan BEFORE implementing. Execute it AFTER implementing.
Your status depends on verification passing, not just code being written.

## Step-by-Step Protocol

### 1. Create Worktree (MANDATORY)

Worktree isolation is REQUIRED unless the user explicitly opted out (e.g., "edit directly on main", "no worktree", "skip worktree").

```bash
TASK_SLUG="[task-name-in-kebab-case]"
WORKTREE_PATH="../wt-${TASK_SLUG}"
BRANCH_NAME="task/${TASK_SLUG}"
BASE="${BASE_WORKTREE:-main}"  # Use specified base or default to main

git worktree add "${WORKTREE_PATH}" -b "${BRANCH_NAME}" "${BASE}"
cd "${WORKTREE_PATH}"
```

If worktree already exists, check if clean and recreate if needed.

### 2. Plan Verification BEFORE Implementing

Ask yourself:
- What will prove this task is complete?
- What tests should pass?
- What resources do I need (ports, servers, databases)?

Write your verification plan now, execute it after implementation.

### 3. Execute Task Requirements

Do what the task asks, nothing more. Don't scope-creep.

If you notice other issues, capture them as TODO files (see step 7) - don't fix them now.

### 4. Verify Your Work

Run your verification plan. Your status depends on verification passing, not just code existing.

**For applications (web, API, CLI):**
- Start servers, test with real interactions (Playwright, curl)
- Check both client and server logs correlate
- Find available ports, use unique test DBs
- Clean up resources after

**For all tasks:**
- Run existing test suites if they exist
- Check edge cases, error handling
- Verify acceptance criteria met

### 5. Commit Locally (Don't Push)

```bash
git add .
git commit -m "feat: [description]"
```

Do NOT push or create PRs. The orchestrator handles integration.

### 6. Clean Up Resources

Stop servers you started, release ports, remove temp files.

### 7. Capture Follow-Ups

If you noticed out-of-scope issues, write them directly to the TODO store (you don't have the `/later` skill - you're a fresh sub-agent):

```bash
# Detect TODO location
BASE=$(git worktree list --porcelain 2>/dev/null | grep '^worktree ' | head -1 | cut -d' ' -f2)
BASE="${BASE:-.}"
TODO_DIR="${BASE}/TODO"
mkdir -p "${TODO_DIR}"

# Write follow-up task
cat > "${TODO_DIR}/followup-[descriptive-name].md" << 'TASK'
---
tags: [follow-up]
---

# [Title]

## Context
Noticed while working on [current task]: [what you observed]

## Task
[What needs to be done]
TASK
```

### 8. Report Completion

Use this EXACT format:

## Completion Report
- Status: success | partial | failed
- Worktree: ../wt-[task-slug]
- Branch: task/[task-slug]
- Base: [what you built on]

## Summary
[2-3 sentences: what was done, key decisions]

## Files Changed
- path/to/file.ts (created|modified|deleted)

## How I Verified This
- [Specific verification actions: server started, browser tests run, API calls made]
- [What passed: test cases, acceptance criteria]
- [Correlation checks: client/server logs, network requests, UI state]
- [Resource management: ports used, cleanup performed]

Examples:
- "Started dev server on port 3001, tested form with Playwright (empty/invalid/valid cases), all passed, server logs correlate with UI"
- "Ran curl tests against POST /api/users, verified 201 response and server logs, existing test suite passed"
- "Executed CLI with --format json, validated output with jq, tested error cases"

## Issues/Notes
[Any blockers, concerns, follow-ups captured]

---

**IMPORTANT:**
- Do NOT push, create PR, or archive. Orchestrator handles integration.
- Do NOT scope-creep. Capture out-of-scope items as TODO files (see step 7).
- Report status accurately. "partial" if some things worked. "failed" if nothing worked.
```

### Example: How /next Dispatches a Sub-Agent

When the orchestrator dispatches a task, it uses the Task tool like this:

```python
# Read the TODO file
todo_content = read_file("TODO/003-add-dark-mode.md")

# Determine execution context
base = "main"  # or a predecessor's worktree path
predecessor_summary = None  # or summary from dependent task

# Build the complete dispatch prompt (task + /do protocol)
dispatch_prompt = f"""
You are executing a task from the Zero CODE protocol.

## Task File
{todo_content}

## Execution Context
- Base worktree: {base}
- Predecessor summary: {predecessor_summary or "None (independent task)"}

---

# EXECUTION PROTOCOL (from /do skill)

You MUST follow this protocol to execute the task above.

## Verification Requirements (Orchestrator-Injected)
[... complete verification section from template above ...]

## Step-by-Step Protocol
[... complete protocol steps from template above ...]

## Completion Report
[... exact format from template above ...]
"""

# Spawn the sub-agent with Task tool
task = Task(
    subject="Add dark mode toggle",
    description=dispatch_prompt,  # Contains BOTH task AND protocol
    activeForm="Adding dark mode toggle"
)

# Track the task
track_agent(task_id="003-add-dark-mode", task_output_id=task.id)
```

**The key insight:** The `description` parameter contains the ENTIRE execution protocol, not just the task content. The sub-agent receives a complete instruction set for how to work.


## Archival

When archiving completed tasks:

1. Move file from `TODO/` to `TODO/.done/`
2. Append completion summary to the file:

```markdown
---
## Completion Summary
**Status:** success
**Completed:** 2024-01-15
**Branch:** task/add-dark-mode

[Agent's summary from completion report]
```

This preserves context for dependent tasks and future reference.

## Permission Requirements

On first run, verify these permissions are available:
- `Bash(git:*)` - worktree management, commits
- `Task` - spawn sub-agents (Claude Code); equivalent mechanism in other tools
- `Read`, `Write`, `Edit` - file operations
- `Glob`, `Grep` - task discovery

If missing, inform user:
```
Missing required permissions for /next orchestration:
- Bash(git:*) - needed for worktree management

Add to .claude/settings.json or run in permissive mode.
```

Don't proceed until permissions are available.

## Graceful Degradation

- If agent reports "blocked by permission": Surface to user, pause that task
- If agent fails: Log failure, skip dependent tasks, continue others
- If orchestrator crashes: Worktrees persist, can resume by re-running `/next`

## Self-Improvement Protocol

When orchestration reveals issues or inaccuracies, improve them:

### Immediate Fixes (Preferred)

If you can fix it now, locate and update the relevant file:

| Issue Type | Update |
|------------|--------|
| Orchestration logic wrong | Locate and edit this skill's `SKILL.md` (see detection below) |
| TODO storage issues | Locate and edit `todo-store/SKILL.md` |
| Project conventions inaccurate | Edit project's `CLAUDE.md` |
| Global preferences wrong | Edit `~/.claude/CLAUDE.md` |

**Locating skill/agent files (install-aware):**

Skills and agents may be installed as symlinks (pointing to the source repo) or as copies (standalone). To find and edit the actual source:

1. Scan known install locations in priority order:
   - Project-level: `.claude/skills/[skill]/SKILL.md`, `.agents/skills/[skill]/SKILL.md`
   - User-level: `~/.claude/skills/[skill]/SKILL.md`, `~/.agents/skills/[skill]/SKILL.md`
2. Resolve symlinks to find the actual file: `readlink -f <path>`
3. Edit the resolved path
   - If it resolved to a different location (symlinked install), your edit updates the source repo
   - If it resolved to the same path (copied install), the edit is local-only

**Ask the user first:** "I noticed [issue]. Should I update [file] to fix this?"

### Deferred Improvements

For larger changes that need more thought:

```
/later "Improve /next: [observation] -> [proposed fix]" --tag meta
```

Examples:
- "Improve /next: dispatched conflicting tasks concurrently -> add resource conflict detection to selection bias"
- "Improve /next: dependency detection missed explicit mention -> scan task content for task references"

### What to Watch For

- Tasks that should have been dispatched sooner but weren't picked
- Dependency chains that weren't detected
- Integration strategy that didn't fit the project
- Agent coordination issues (resource conflicts, blocking)
- Selection bias not reflecting actual priority

## Boundary

`/next` dispatches and coordinates. It does NOT do the actual work. That's `/do`'s job.
