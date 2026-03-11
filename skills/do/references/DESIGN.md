# /do - Execution

The hands of Zero CODE. Executes a single task in isolation, verifies its own work, and reports back.

## Core Responsibility

**Execute one task well.** Don't orchestrate, don't scope-creep, don't make integration decisions. Do the work, verify it, report.

## Key Design Decisions

### Why Worktree Isolation Is Mandatory?

**Every task MUST use worktree isolation unless the user explicitly opts out** (e.g., "edit directly on main", "no worktree", "skip worktree").

This isn't just for "complex" tasks. Even simple edits benefit because:
- **Keeps main clean** - No accidental commits to main branch
- **Natural PR workflow** - Branch already exists, push and PR is trivial
- **Parallel execution** - Multiple agents can work simultaneously without conflicts
- **Easy rollback** - Just delete the worktree if something goes wrong
- **No manual branch cleanup later** - Task is done in isolation from the start

**Real-world issue that drove this:** During a `/next` session, "simple" tasks were executed directly on main. This landed commits on main instead of feature branches, requiring manual cleanup to move them. Even simple tasks should live in worktrees to maintain clean workflow.

The cost is disk space and git overhead. Worth it for the isolation guarantees and workflow cleanliness.

### Why Project Setup Phase Is Critical?

**The problem:** Agents were jumping into task execution without properly setting up the project environment. This caused verification failures because:
- No build artifacts (didn't run build commands)
- Missing dependencies (didn't install packages)
- Agent lacked project-specific context (didn't read AI memory files)
- Environment not configured (missing secrets, config)

**The fix:** Mandatory Project Setup Phase that runs BEFORE task execution:

1. **Read AI memory/instruction files first** - These contain critical context:
   - `CLAUDE.md`, `.claude/CLAUDE.md`, `.cursorrules`, `.github/copilot-instructions.md`
   - Setup instructions, build processes, testing requirements
   - Project conventions and known gotchas

2. **Detect project type and dependencies** - Node/Python/Go/Rust/PHP/Ruby detection

3. **Run setup commands** - Follow AI instruction file guidance if present, otherwise use sensible defaults:
   - Install dependencies (`npm install`, `pip install -r requirements.txt`, etc.)
   - Run build scripts if they exist
   - Execute setup/postinstall hooks

4. **Check environment configuration** - Warn if `.env.example` exists but `.env` doesn't

5. **Verify setup worked** - Basic smoke tests before proceeding

**If setup fails, don't proceed to task execution.** Report as `failed` status with setup error in completion report.

This ensures agents work in a properly configured environment, just like a human developer would.

### Why Plan Verification Before Implementation?

Before writing any code, the agent should answer:
- What will prove this task is complete?
- What tests should pass?
- What resources do I need?

This prevents "I wrote the code but can't verify it" situations. Verification strategy is determined upfront, not improvised after.

**Verification requirements come from the orchestrator.** The TODO file may or may not specify how to verify - that's fine for capture. But the dispatch prompt from `/next` always includes verification requirements. The agent must:
1. Write a verification plan before implementing
2. Execute that plan after implementing
3. Report status based on verification results, not just code completion

"I wrote the code" is not done. "I wrote the code and verified it works" is done.

**For application tasks, lean into active verification patterns:**

See `verification-patterns.md` (in this references directory) for comprehensive techniques that agents should use when building applications:

- **Background server management** - Start dev servers in background, health check, monitor logs, clean up
- **Browser automation** - Use Playwright MCP tools to test UI like a real user
- **Client/server correlation** - Verify that UI state matches server logs (both sides agree)
- **Resource management** - Find available ports, use unique test DBs, don't conflict with other agents

### Why Multi-Agent Awareness?

When `/next` dispatches multiple agents concurrently, they share the same machine. Naive agents would:
- Fight over port 3000
- Kill each other's dev servers
- Corrupt shared test databases

Multi-agent awareness means:
- Check if resources are in use before claiming
- Use alternatives (port 3001, 3002...) rather than fighting
- Never kill processes you didn't start
- Clean up only what you created

This is defensive programming for agents.

### Why Structured Completion Reports?

The orchestrator needs to know:
- Did it work? (status)
- Where's the code? (worktree, branch)
- What was done? (summary)
- What should I tell dependent tasks? (context for chaining)

A structured report format makes this parseable and reliable. Free-form "I'm done!" messages don't give the orchestrator what it needs.

### Why Capture Follow-ups via /later?

During execution, agents notice things:
- "This function could be cleaner"
- "There's an unrelated bug here"
- "We should add more tests"

The temptation is to fix them. Don't. That's scope creep.

Instead, capture via `/later` and stay focused. The orchestrator will handle those items in future runs.

## Mental Model

Think of `/do` as a focused contractor:

```
receive_assignment(task, context)
setup_workspace()           # Worktree from appropriate base
plan_how_to_verify()        # Before implementing
do_the_work()               # Focused on task only
verify_it_works()           # Run the planned verification
commit_locally()            # Don't push - orchestrator decides
cleanup_resources()         # Release what you claimed
report_back()               # Structured completion report
```

The contractor doesn't decide what to work on next, doesn't push to production, doesn't handle integration. Just does their task well and reports back.

## Boundaries

| /do does | /do doesn't |
|----------|-------------|
| Create worktree | Analyze other tasks |
| Execute task requirements | Dispatch other agents |
| Verify own work | Decide integration strategy |
| Report completion | Push/merge (orchestrator does) |
| Capture follow-ups | Archive own task |
| Manage own resources | Know about the bigger picture |

## Two Modes

**Sub-agent mode:** Dispatched by `/next` with context. Just executes and reports back. No integration decisions.

**Manual mode:** User invokes `/do` directly. After completion, asks user how to integrate (PR? merge? leave it?). This mode handles its own archival.

The protocol is the same - the difference is who handles integration.

## Trade-offs Accepted

**Isolation costs disk space** - Every worktree is a near-full copy of the repo. For large repos, this adds up. We accept this for clean isolation.

**Structured reports are rigid** - Agents must format completion reports exactly. Free-form would be easier but less reliable for the orchestrator.

**No heroics** - If something is blocked or failing, report it rather than trying creative workarounds. Transparency over cleverness.
