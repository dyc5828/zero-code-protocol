# /next - Orchestration

The brain of Zero CODE. Analyzes the task backlog, decides what to work on, dispatches agents, and coordinates completion.

## Core Responsibility

**Orchestrate, don't execute.** The orchestrator never does task work itself - it coordinates others who do.

## Key Design Decisions

### Why Sequential Dispatch with Concurrent Execution?

The original instinct was "batch dispatch" - identify all independent tasks, dispatch them all at once in a wave, wait for the wave to complete, then dispatch the next wave.

**Why that's wrong:**

State changes constantly. By the time you've dispatched task 3, task 1 might have completed and unblocked task 5. If you predetermined "wave 1 = tasks 1,2,3" you miss the opportunity to start task 5 earlier.

**The better model:**

Read ALL TODOs -> Evaluate -> Pick ONE -> Dispatch -> Read ALL TODOs -> Reevaluate -> Pick ONE -> Dispatch...

Each decision is fresh and based on complete current state. You're not locked into a predetermined batch OR a sequential list. The orchestrator stays maximally responsive to changing reality.

**Agents still run concurrently** - the sequential part is the *decision*, not the execution. Multiple agents can be working simultaneously, but dispatch decisions happen one at a time based on current state.

**Why read everything every time?**

Completing Task A doesn't just unblock dependencies - it changes the entire priority landscape:
- Task C might now be more urgent than Task B based on what A revealed
- New tasks might have been added via `/later` while agents worked
- Project context might have shifted (tests revealed new issues)

The loop is: **Full reassessment -> Best available task -> Dispatch -> Full reassessment**

Not: "Work through the list sequentially"

### Why Natural Completion Over Polling?

When you spawn a sub-agent with a background task mechanism (e.g., Claude Code's Task tool), it naturally completes when the agent finishes. You don't need to poll for completion - you're notified.

Polling is a fallback for:
- Checking intermediate progress on long tasks
- Monitoring when you want visibility before completion
- Edge cases where natural completion doesn't work

But the primary model is: dispatch -> agent works -> agent reports back -> you receive the report.

### Why Worktree Isolation Is Mandatory?

**ALL tasks dispatched by `/next` MUST use worktree isolation.** There is no "simple task exception" unless the user explicitly opts out (e.g., "edit directly on main", "no worktree", "skip worktree").

**Why this is non-negotiable:**

Without mandatory worktrees, agents edit directly on main. This creates:
- Commits landing on main instead of feature branches
- Manual cleanup required to move work to proper branches
- Broken PR workflow (no branch to push from)
- Risk of polluting main with incomplete work

**Real-world failure case:** During a `/next` session, tasks were dispatched without worktrees because they seemed "simple." This resulted in multiple commits on main that had to be manually moved to a feature branch. Even simple tasks benefit from worktree isolation because it keeps main clean and makes the PR workflow natural.

**The orchestrator enforces this** by including explicit worktree creation instructions in every dispatch prompt. Agents should not be given the option to skip worktrees unless the user specifically requested it.

### Why Sub-Agents Must Receive the /do Protocol?

**CRITICAL REALIZATION:** Sub-agents spawned via background task mechanisms are fresh instances. They DO NOT automatically have access to the /do skill definition.

**What happens without embedding the protocol:**
- Sub-agents don't know to use worktrees (edit directly on main)
- No verification happens (agents claim success without testing)
- Reports come back in inconsistent formats (orchestrator can't parse them)
- Resource management is ignored (port conflicts, server collisions)

**The solution:** The orchestrator must embed the COMPLETE /do execution protocol into every dispatch prompt. This includes:
- Exact bash commands for worktree creation
- Verification requirements and patterns
- Commit conventions
- Exact completion report format

**The dispatch prompt structure:**
1. Task content (from the TODO file)
2. Execution context (base worktree, predecessor summary)
3. **Complete /do protocol** (embedded in full)

This is not optional. Without it, sub-agents have no idea how Zero CODE works and will execute tasks incorrectly.

### Why Worktree Chaining?

If Task B depends on Task A, B shouldn't start from `main` - it should start from A's completed worktree. This:
- Avoids merge conflicts between related tasks
- Creates natural PR chains (A->B becomes one coherent change)
- Lets dependent work build incrementally

The orchestrator tracks which tasks completed in which worktrees and passes that context to dependent tasks.

### Why Intelligent Integration Strategy?

Different projects need different integration approaches:
- Team with CI? PR-per-task for review
- Related feature work? PR-per-chain for cleaner history
- Solo dev iterating fast? Direct merge

Rather than hardcode one approach, detect project context and choose appropriately.

### Why Orchestrator-Injected Verification?

TODO files may or may not include verification criteria. That's fine for capture - `/later` shouldn't force a rigid format. But when dispatching, the orchestrator must ensure agents verify their work.

**The problem with optional verification:**

If verification is "do it if you feel like it," agents will skip it when they're confident. Confidence isn't correctness. We've all written code that "obviously works" and didn't.

**The solution:**

The orchestrator's dispatch prompt *always* includes verification requirements:
1. What specific check proves the task is complete?
2. How do you ensure nothing regressed?
3. What resources do you need to verify?

The agent must write a verification plan before implementing, then execute it after. Status depends on verification passing, not just code being written.

**This separates concerns properly:**
- `/later` captures work (may include verification hints, doesn't have to)
- `/next` ensures verification happens (injects requirements into every dispatch)
- `/do` executes and verifies (follows the injected plan)

The TODO is a handoff document. The dispatch prompt is a contract.

### Why Auto-Dispatch by Default?

When a user runs `/next`, they're explicitly choosing to work through the backlog. Asking "should I start?" adds unnecessary friction.

**The invocation IS the authorization.**

The orchestrator should only pause for exceptional cases:
- Conflicting dependencies that need human resolution
- Destructive actions detected in task content
- Ambiguous integration strategy

Otherwise, analyze the backlog and immediately begin dispatching. The user can always interrupt if needed.

This makes `/next` feel responsive and autonomous - more like "start the factory" than "show me the plan and ask permission for each step."

## Mental Model

Think of `/next` as a work queue processor with judgment:

```
while work_exists:
    read_all_todos()             # Full reassessment
    look_at_current_state()      # What's done? Running? Blocked?
    pick_best_available_task()   # Selection bias: priority, impact, recency
    dispatch_one_agent()         # Not a batch - just one
    # Loop continues - complete state reassessment
```

It's not fire-and-forget. The orchestrator stays engaged, responds to completions, unblocks dependent work, and makes fresh decisions continuously based on the full backlog state.

## Boundaries

| /next does | /next doesn't |
|------------|---------------|
| Read and analyze TODO/ | Execute task work |
| Build dependency graph | Create worktrees |
| Make dispatch decisions | Write code |
| Track completion reports | Verify code correctness |
| Archive completed tasks | Push/merge directly |
| Choose integration strategy | |

The actual work happens in `/do`. The orchestrator coordinates.

## Trade-offs Accepted

**Sequential decisions add latency** - There's a small delay between "task completes" and "next task dispatched" because we reevaluate. We accept this for better responsiveness to changing state.

**No predetermined plan** - Users don't see "here's exactly what will happen." They see "here's what's available, I'll dispatch continuously." This is intentional - the plan emerges from state, not from upfront prediction.

**Orchestrator must stay alive** - If the orchestrator crashes mid-execution, agents keep running but new dispatch stops. Worktrees persist, so you can resume by re-running `/next`.
