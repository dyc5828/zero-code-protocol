# Zero CODE Protocol

**Zero CODE** (Capture, Orchestrate, Dispatch, Execute) is a multi-agent task orchestration protocol for AI-assisted development.

## For AI Tools Working in This Repo

This repository contains the Zero CODE protocol definition - a system of skills and agents for orchestrating multi-agent task execution.

### Structure

- **`skills/`** - Core skills following the [Agent Skills spec](https://agentskills.io). Auto-discovered by compatible tools.
  - `later/` - Capture deferred work as TODO files
  - `next/` - Orchestrate task execution, dispatch agents
  - `do/` - Execute single tasks in worktree isolation
  - `todo-store/` - Shared TODO storage abstraction (internal)

- **`agents/`** - Agent definitions for tools with agent support (e.g., Claude Code)
  - `zero.md` - ZERO protocol guide: teaches, guides, advises, maintains

- **`protocol/zero/`** - ZERO as a portable skill for tools without agent support
  - `SKILL.md` - Protocol guide in skill format

- **`design/`** - Protocol design documentation
  - `architecture.md` - Protocol overview, key principles, task lifecycle
  - `variants.md` - Alternative dispatch strategies and trade-offs
  - `verification-patterns.md` - Application verification techniques

- **`diagram.html`** - Interactive visualization of the system flow

### Quick Start

1. **Capture** work: `/later "Add feature X"`
2. **Orchestrate** when ready: `/next`
3. **Execute** manually: `/do`

### Key Concepts

- Tasks are **prompts for fresh agents**, not tickets
- Each task runs in a **git worktree** for isolation
- Dependent tasks **chain** on predecessor's worktree
- Agents report back with **structured completion reports**
- Integration strategy adapts to **project context** (CI, team size, etc.)

### Design Docs

For protocol rationale and architecture details, see `design/architecture.md`.
