# /later - Capture

The inbox of Zero CODE. Captures deferred work so it's not forgotten and future agents can pick it up.

## Core Responsibility

**Capture, don't execute.** Write it down well enough that a fresh agent can act on it. Then you're done.

## Key Design Decisions

### Why Write for a Fresh Agent?

The agent that captures the task is not the agent that executes it. There might be hours, days, or sessions between capture and execution.

This means:
- Include context that seems obvious now (it won't be later)
- Write instructions as if for someone who wasn't in the conversation
- Add relevant file paths, code snippets, commands
- Don't assume shared memory

The TODO file is a handoff document, not a personal reminder.

### Why Semantic File Names?

`add-dark-mode-toggle.md` not `task-1.md` or `todo-2024-01-15.md`.

When scanning TODO/, the filename should tell you what the task is. When the orchestrator builds a dependency graph, it references tasks by name. Semantic names make everything more readable.

### Why Optional Frontmatter?

Not every task needs priority, tags, or dependency info. But when it does, there should be a place for it.

Frontmatter is optional - you can write a bare task file. But if you need to say "this is urgent" or "this blocks that other task," the structure is there.

### Why Proactive Invocation?

Users say things like:
- "Let's save that for later"
- "We'll handle that another time"
- "Not now but eventually"

These are implicit capture requests. The skill should recognize intent, not require explicit `/later` commands. Lower friction = better capture rate.

### Why Self-Improvement Capture?

When skills don't work well, that's valuable feedback. Rather than fixing everything immediately (which might be wrong), capture it:

```
/later "Improve /next: [what broke] -> [proposed fix]" --tag meta
```

This creates a backlog of improvements that can be reviewed and implemented thoughtfully.

## Mental Model

Think of `/later` as a careful note-taker:

```
hear_deferred_intent()       # User wants to capture something
understand_what()            # What are we actually capturing?
determine_where()            # Detect TODO store location
write_complete_handoff()     # Everything a future agent needs
confirm_capture()            # "Captured in TODO/add-thing.md"
```

The note-taker doesn't prioritize, schedule, or execute. Just captures accurately and completely.

## Boundaries

| /later does | /later doesn't |
|-------------|----------------|
| Write tasks to TODO/ | Decide execution order |
| Structure task format | Execute any work |
| Detect worktree base | Analyze dependencies |
| Ask for details if unclear | Dispatch agents |
| Recognize implicit capture intent | Track task status |

Once written, `/later` is done. The orchestrator (`/next`) takes over from there.

## Trade-offs Accepted

**Verbosity over brevity** - Better to capture too much context than too little. A verbose TODO can be skimmed; a cryptic one requires re-discovery.

**Interruption cost** - Stopping to capture properly takes a moment. Worth it to not lose the thought.

**No filtering** - Capture everything worth remembering, even if it might not get done. Let the orchestrator prioritize. Better to have it written down.
