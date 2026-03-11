---
name: later
description: Captures deferred work as a TODO for a future Claude agent to pick up. Use when something comes up that should be addressed later - ideas, improvements, bugs, tasks, or anything worth remembering. IMPORTANT - Proactively invoke when user says things like "save that for later", "let's do that later", "park this for now", "we'll handle that another time" - don't wait for explicit /later command.
metadata:
  author: danielchen
  version: 1.0.0
---

# Later (Deferred Work)

Captures deferred tasks as TODO files that serve as prompts for future Claude agents. Part of the **Zero CODE** protocol (Capture -> Orchestrate -> Dispatch -> Execute).

## Skill Names

This document uses bare names (`/later`, `/next`, `/do`) for readability. Your invocation depends on install method:

| Install Method | Invocation |
|----------------|------------|
| Symlink or `npx skills add` | `/later`, `/next`, `/do` |
| Claude Code plugin | `/zero-code:later`, `/zero-code:next`, `/zero-code:do` |

## When to Use

- User wants to capture something for later
- An idea or improvement comes up mid-session
- Something is out of scope but worth remembering
- User says "later", "todo", "remind me", "we should", "let's defer", etc.
- User indicates deferring work with phrases like "save that for later", "let's save the work for later", "we'll do that another time", "not now but eventually", "park this for now"
- **Self-improvement:** When noticing issues with any Zero CODE skill (`/later`, `/next`, `/do`)

**Important:** Proactively invoke this skill when the user expresses intent to defer - don't wait for explicit "/later" command. If the user says something like "let's save that for later" or "we're going to do X later", invoke this skill automatically.

## Storage

Use the **todo-store** skill to detect and write to the appropriate location. The TODO store handles various formats:

- **Folder-based:** `TODO/`, `todo/`, `tasks/`, `TASKS/` (one file per task)
- **File-based:** `TODO.md`, `todo.md`, `tasks.md` (all tasks in one file)

```bash
# Detection algorithm (see todo-store for full details)
BASE=$(git worktree list --porcelain 2>/dev/null | grep '^worktree ' | head -1 | cut -d' ' -f2)
BASE="${BASE:-.}"

# Check for existing formats, default to TODO/ folder
```

The todo-store skill is the single source of truth for storage logic shared by `/later`, `/next`, and `/do`.

## File Naming

Use semantic, descriptive names in kebab-case:
- `worktree-doppler-setup.md`
- `add-dark-mode-toggle.md`
- `refactor-auth-flow.md`

Derive the name from the core topic. Ask the user if unclear.

## File Format

Write as a prompt/handoff for a future Claude agent. Include optional YAML frontmatter for metadata:

```markdown
---
priority: normal  # low | normal | high | urgent (optional)
tags: [feature]   # optional tags for categorization
blocks: []        # task names this blocks (optional, for dependencies)
---

# [Title]

## Context

[Why this TODO exists - what came up, what was discussed, any relevant background from the session.]

## Task

[Clear description of what needs to be done. Written as instructions for another agent.]

## Relevant Details

[Any code snippets, file paths, commands, or specifics that would help. Optional - include if available.]
```

### Frontmatter Fields

All frontmatter is optional. Only include fields that are relevant:

- **priority**: `low` | `normal` | `high` | `urgent` - Default is `normal`. Use when task urgency is clear.
- **tags**: Array of strings for categorization. Use `meta` tag for self-improvement items.
- **blocks**: Array of task names (filenames without .md) that cannot start until this task completes.

## Process

1. Understand what the user wants to capture from the conversation
2. Ask user for a semantic name if not obvious
3. Detect TODO store location (using todo-store logic):
   - Find base project directory (worktree-aware)
   - Detect existing format or default to `TODO/` folder
4. Determine if any metadata applies (priority, tags, blocks)
5. Write the task using the detected format
6. Confirm to user with full path: "Captured in [path]"

## Key Principle

Write for a fresh agent with no prior context - include enough detail to act without re-discovery.

## Self-Improvement Protocol

When workflows don't work well or you notice inaccuracies, improve them directly:

### Immediate Fixes (Preferred)

If you can fix it now, locate and update the relevant file:

| Issue Type | Update |
|------------|--------|
| Skill behavior wrong | Locate and edit the skill's `SKILL.md` (see detection below) |
| Project conventions inaccurate | Edit project's `CLAUDE.md` |
| Global preferences wrong | Edit `~/.claude/CLAUDE.md` |
| Agent behavior issues | Locate and edit the agent definition (see detection below) |

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
/later "Improve [skill]: [what broke] -> [proposed fix]" --tag meta
```

Examples:
- `"Improve /later: task format wasn't clear -> add acceptance criteria section"` --tag meta
- `"Improve /next: dispatch caused resource conflicts -> check file overlap"` --tag meta
- `"Improve /do: verification missed edge case -> add integration test"` --tag meta

### What to Watch For

- Repeated course-corrections from user
- Instructions that don't match reality
- Missing context that would help future sessions
- Patterns that should be codified

## Boundary

Once written to `TODO/`, `/later` is done. It doesn't track, prioritize, or execute tasks. The `/next` orchestrator handles that.
