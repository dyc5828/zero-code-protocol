# todo-store - Storage Abstraction

The foundation of Zero CODE. Provides a consistent interface for TODO storage regardless of how a project organizes tasks.

## Core Responsibility

**Abstract storage details.** The other skills shouldn't care whether tasks live in `TODO/` folder, `tasks.md` file, or something else. They just read and write tasks.

## Key Design Decisions

### Why Abstract Storage?

Different projects organize differently:
- Some use `TODO/` folders with one file per task
- Some use a single `tasks.md` with checkboxes
- Some use `tasks/`, some use `TASKS/`, some use lowercase

Without abstraction, every skill would need logic for every format. With abstraction, storage logic lives in one place.

### Why Folder-Based as Default?

When creating a new store, we default to folder-based (`TODO/`) because:
- One file per task = cleaner git diffs
- Easier to archive (move file to `.done/`)
- More room for detailed task content
- Natural fit for agent handoffs (each file is a complete prompt)

File-based (`todo.md`) works for simpler task lists but doesn't scale as well for detailed agent instructions.

### Why Worktree-Aware Detection?

In a git worktree setup, the TODO store should live in the main worktree, not scattered across worktrees. This ensures:
- Single source of truth for tasks
- Agents in different worktrees see the same backlog
- No confusion about which TODO is canonical

Detection starts by finding the base worktree, then looks for TODO there.

### Why Support Multiple Formats?

Respecting existing conventions > imposing new ones.

If a project already has a `tasks/` folder, we use it. If they have `todo.md`, we work with that. Zero CODE adapts to the project, not the other way around.

### Why .done/ for Archival?

Completed tasks shouldn't disappear - they're valuable context:
- What was done and when
- Completion summaries from agents
- Audit trail for the work
- Context for dependent tasks

Moving to `.done/` keeps the main TODO clean while preserving history.

## Mental Model

Think of todo-store as a database adapter:

```
detect_store()    -> Find where tasks live, what format
list_tasks()      -> Get pending tasks
read_task(id)     -> Get full task content
write_task(id)    -> Create or update task
archive_task(id)  -> Move to done with summary
```

The consuming skills (`/later`, `/next`, `/do`) don't know or care about the underlying format. They just use these operations.

## Boundaries

| todo-store does | todo-store doesn't |
|-----------------|-------------------|
| Detect TODO location | Execute tasks |
| Read/write tasks | Dispatch agents |
| Handle format differences | Decide priorities |
| Archive completed tasks | Orchestrate anything |

This is infrastructure, not intelligence. It doesn't make decisions about *what* to do with tasks - just provides reliable storage.

## Trade-offs Accepted

**Detection overhead** - Every operation starts with format detection. Small cost for flexibility.

**Lowest common denominator** - The abstraction can only expose what all formats support. Rich features of one format might not be usable through the abstraction.

**Format lock-in per project** - Once a project has a TODO format, we stick with it. Migration between formats isn't supported (do it manually if needed).

## Not User-Invocable

This skill is internal infrastructure. Users invoke `/later`, `/next`, `/do` - those skills use todo-store under the hood. There's no `/todo-store` command.
