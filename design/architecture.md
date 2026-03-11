# Zero C.O.D.E. Protocol - Architecture

A self-organizing, concurrent agent execution system for autonomous task completion with intelligent sequential dispatch.

**Zero** = the goal (inbox zero - clear the backlog)
**C.O.D.E.** = the method:
- **C**apture -> `/later`
- **O**rchestrate -> `/next`
- **D**ispatch -> `/next`
- **E**xecute -> `/do`

## Overview

Zero CODE transforms task management into a multi-skill orchestration system:

- **`/later`** = Capture (tasks to `TODO/`)
- **`/next`** = Orchestrate + Dispatch (analyze, coordinate, send agents)
- **`/do`** = Execute (what sub-agents follow, or manual single-task)

## Architecture

```
/later (capture)     /next (orchestrate)              /do (execute)
      |                     |                              |
      v                     v                              v
  Write TODO  --->  Analyze + Dispatch  ---------->  Work in worktree
                           ^                              |
                           |                              v
                           |                         Commit + Report
                           |                              |
                           <------------------------------+
                                  (completion + summary)
                                         |
                                         v
                              Unblock dependent tasks
                              Spin up next agent using
                              previous worktree as base
```

## Key Principles

### 1. Concurrent Execution with Sequential Decisions

**This is the core architectural pattern of `/next`:**

**What it is:**
- Orchestrator evaluates current state -> picks ONE task -> dispatches ONE agent
- Agent runs in background (concurrent with other agents)
- Orchestrator reevaluates -> picks next ONE task -> dispatches
- Continuous loop with dynamic prioritization

**What it is NOT:**
- NOT batch/wave dispatch (grouping independent tasks and dispatching all at once)
- NOT predetermined execution order (decisions adapt to changing state)
- NOT sequential execution (agents do run concurrently)

**Why this matters:**
- State changes between decisions (completions, new info, unblocked work)
- Selection bias applies fresh to current state, not stale snapshot
- Blocked-by-in-progress work is respected (don't pick tasks that would conflict)
- More responsive to completion order (doesn't wait for slowest in wave)

**Analogy:** Like a smart work queue processor that continuously evaluates "what should I work on next?" rather than batching "do these 5 things in parallel."

### 2. Separation of Concerns

| Skill | Responsibility | Does NOT |
|-------|----------------|----------|
| `/later` | Capture work items | Execute, prioritize, or dispatch |
| `/next` | Analyze, dispatch, coordinate | Execute task work, create worktrees |
| `/do` | Execute single task | Orchestrate, decide integration strategy |
| `todo-store` | Abstract TODO storage | User-facing operations |

The **todo-store** skill is a shared foundation that handles TODO discovery regardless of format (folder-based or file-based, various naming conventions). All three main skills use it.

### 3. Worktree Chaining

Dependent tasks build on predecessor's work:

```
Task A (independent):
  base: main
  creates: my-app-feature-auth (sibling of my-app/)
  reports: "done, worktree at ../my-app-feature-auth, summary: [what was done]"

Task B (depends on A):
  base: ../my-app-feature-auth  <-- NOT main!
  creates: my-app-feature-dashboard (sibling of my-app/)
  builds on A's work
```

This enables:
- Incremental feature building
- Natural PR chains (A->B->C becomes one coherent change)
- Avoiding merge conflicts between related tasks

### 4. Completion Detection

Sub-agents complete naturally and report back:

**Primary mechanism:**
- Agents spawned with background task mechanism
- Task completes when sub-agent finishes
- Orchestrator receives structured completion report

**Fallback/supplementary:**
- Can check output files for intermediate progress
- Useful for long-running tasks or monitoring

**Completion report contains:**
- Status (success/failed/partial)
- Worktree location and branch
- Summary of work done
- Files changed
- Issues encountered

The orchestrator uses this to chain dependent tasks and make integration decisions.

### 5. Intelligent Integration

Integration strategy is chosen based on context, not hardcoded:

| Strategy | Best For |
|----------|----------|
| PR-per-task | Team review, CI-required contexts |
| PR-per-chain | Related features, cleaner history |
| Batch merge | Rapid prototyping, low-risk changes |
| Direct merge | Solo dev, fast iteration |

### 6. Intelligent Task Selection

Each dispatch decision considers:
- Priority (from task frontmatter)
- Recency (newer tasks often signal relevance)
- Impact (unblocks other work)
- Resource conflicts (avoid stepping on in-progress work)

The selection happens dynamically - not predetermined at the start.

### 7. Multi-Agent Awareness

Agents must be aware they operate concurrently with other agents:
1. **Plan verification BEFORE implementing** - Check for resource needs
2. **Check for resource conflicts** - Ports, servers, databases may be in use
3. **Isolate test environments** - Use unique ports, DB prefixes
4. **Clean up after verification** - Release resources you claimed

### 8. Self-Improvement

Every skill can learn and improve. **Prefer immediate fixes when possible.**

Skills and agents may be installed as symlinks or copies. To edit:
1. Scan known install locations (project-level, then user-level)
2. Resolve symlinks (`readlink -f`) to find actual source
3. Edit the resolved path (symlink = updates source repo; copy = local-only)

For larger changes that need more thought:
```bash
/later "Improve [skill]: [what broke] -> [proposed fix]" --tag meta
```

## Task Lifecycle

```
[/later]                [/next]                      [/do]
   |                       |                            |
   v                       v                            v
Write to TODO/  --->  Analyze tasks              Agent in worktree
                           |                            |
                           v                            v
                    Evaluate state               Do the work
                    Pick ONE best task                  |
                           |                            v
                           v                       Commit locally
                    Dispatch agent                      |
                           |<---------------------------+
                           |         (report: status, summary,
                           v          worktree location)
                    Archive + Update deps
                           |
                           v
                    Reevaluate state
                    Pick ONE best task
                    (may now include previously
                     blocked work)
                           |
                           v
                    Dispatch agent
                    "base: ../my-app-feature-auth"
                           |
                           v
                    [continuous loop]
                    Multiple agents run
                    concurrently, but
                    decisions are sequential
                           |
                           v
                    Integration decision
                    (PR? merge? chain?)
```

## File Structure

The **todo-store** skill auto-detects various formats:

### Folder-based (one task per file)
```
TODO/                    # or todo/, tasks/, TASKS/
├── add-dark-mode.md
├── fix-auth-bug.md
└── .done/
    └── refactor-api.md  # includes completion summary
```

### File-based (all tasks in one file)
```
TODO.md                  # or todo.md, tasks.md
```

With structure:
```markdown
## Pending
- [ ] Add dark mode toggle
- [ ] Fix auth bug

## Done
- [x] Refactor API
```

The folder-based format is default and recommended for complex tasks.

## Task File Format

```markdown
---
priority: normal  # low | normal | high | urgent
tags: [feature, auth]
blocks: [other-task-name]  # optional dependencies
---

# [Title]

## Context
[Why this TODO exists - background from the session]

## Task
[Clear instructions for an agent to execute]

## Relevant Details
[Code snippets, file paths, commands]
```

## Required Permissions

For concurrent agent execution, these permissions are needed:

| Permission | Why |
|------------|-----|
| `Bash(git:*)` | Worktree create/remove, commit, branch |
| `Bash(npm:*, yarn:*, pnpm:*)` | Run tests, install deps |
| `Bash(node:*, python:*, etc)` | Run verification scripts |
| `Read`, `Write`, `Edit` | File operations |
| `Task` | Dispatch sub-agents |
| `Glob`, `Grep` | Find files, search code |

## Usage

### Recommended Workflow (Two Terminals)

Zero CODE is designed for a two-terminal workflow:

**Terminal 1 - Orchestration (dedicated):**
```bash
/next  # Analyzes TODO/, dispatches agents, coordinates completion
```
This session continuously monitors agent progress, coordinates integration, and manages the overall task execution flow.

**Terminal 2 - Capture/Execution (on-demand):**
```bash
/later "Add dark mode toggle to settings"  # Capture work as ideas arise
/do    # Manually execute a single task if needed
```
This session is for quick capture without interrupting orchestration, or manual task execution.

### Getting Started (Single Session)

For newcomers learning the protocol, start with a single session:

```bash
# Capture work items
/later "Add feature X"
/later "Fix bug Y"

# Orchestrate when ready
/next  # Analyzes, dispatches, coordinates

# Or manually execute a single task
/do
```

Once comfortable with the flow, transition to the two-terminal workflow for optimal productivity.

## Design Decisions

1. **Dispatch pattern** - Continuous sequential decisions with concurrent execution (NOT batch/wave)
2. **Task selection** - Dynamic evaluation with bias toward newer + higher impact + unblocks others
3. **Worktrees** - Always isolate work; chain for dependencies
4. **Archival** - Move to `.done/` with summary for audit trail
5. **Orchestration** - Active coordination with completion reports (not fire-and-forget)
6. **Integration** - Intelligent strategy based on project context

## Further Reading

- [Dispatch Variants](variants.md) - Alternative dispatch strategies and when they make sense
- [Verification Patterns](verification-patterns.md) - Application verification techniques
- [Interactive Diagram](../diagram.html) - Visual system flow

Per-skill design rationale:
- [/later](../skills/later/references/DESIGN.md) - Capture design
- [/next](../skills/next/references/DESIGN.md) - Orchestration design
- [/do](../skills/do/references/DESIGN.md) - Execution design
- [todo-store](../skills/todo-store/references/DESIGN.md) - Storage design
