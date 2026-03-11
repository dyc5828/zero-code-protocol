# Zero CODE Protocol

A multi-agent task orchestration system for AI-assisted development. Capture ideas, orchestrate execution, and let agents work in parallel.

**Zero** = the goal (inbox zero - clear the backlog)
**C.O.D.E.** = the method:
- **C**apture (`/later`) - Never lose an idea
- **O**rchestrate (`/next`) - Analyze dependencies, plan work
- **D**ispatch (`/next`) - Spin up agents with full context
- **E**xecute (`/do`) - Work in isolation, verify, report back

## The Problem

Working with AI agents on coding tasks reveals friction:

1. **Context loss** - Ideas come up mid-session that can't be addressed now
2. **Sequential bottleneck** - One agent, one task at a time
3. **Handoff failures** - Fresh agents need everything re-explained
4. **Integration chaos** - Multiple branches, unclear merge strategy

## The Solution

Zero CODE turns deferred work into a managed pipeline:

```
You coding -> idea comes up -> /later "refactor auth module"
                                      |
                                      v
                              TODO/refactor-auth.md
                                      |
           Ready to process -> /next  |
                                      v
                              Analyze dependencies
                              Dispatch agents in worktrees
                              Coordinate completion
                              Integrate results
```

## Install

### Recommended: Claude Code Plugin

```bash
# Add the repo as a marketplace source
/plugin marketplace add dyc5828/zero-code-protocol

# Install the plugin
/plugin install zero
```

Installs all 4 skills (`/zero:later`, `/zero:next`, `/zero:do`, `/zero:todo-store`) plus the ZERO agent — a protocol guide that teaches, troubleshoots, and helps maintain the system.

### Other Options

<details>
<summary><strong>npx skills (cross-tool)</strong></summary>

Works with Cursor, Codex, GitHub Copilot, Windsurf, Gemini CLI, and [30+ other tools](https://agentskills.io):

```bash
npx skills add dyc5828/zero-code-protocol
```

Installs 4 core skills with bare names (`/later`, `/next`, `/do`).

To also install the ZERO protocol guide as a skill:
```bash
npx skills add dyc5828/zero-code-protocol -s zero --skill-path protocol/zero
```

</details>

<details>
<summary><strong>Manual clone + symlink (Claude Code)</strong></summary>

```bash
git clone https://github.com/dyc5828/zero-code-protocol.git
cd zero-code-protocol

# Symlink skills (bare names)
ln -s "$(pwd)/skills/later" ~/.claude/skills/later
ln -s "$(pwd)/skills/next" ~/.claude/skills/next
ln -s "$(pwd)/skills/do" ~/.claude/skills/do
ln -s "$(pwd)/skills/todo-store" ~/.claude/skills/todo-store

# Symlink ZERO agent
ln -s "$(pwd)/agents/zero.md" ~/.claude/agents/zero.md
```

Bare skill names + ZERO agent. Edits to the repo update your local install automatically. Best for contributors.

</details>

## Quick Start

**1. Capture** work as ideas come up:
```
/later "Add dark mode toggle"
/later "Fix Safari auth bug"
/later "Refactor API response format"
```

**2. Orchestrate** when ready to execute:
```
/next
```

`/next` analyzes your TODO backlog, dispatches agents to work in isolated worktrees, and coordinates integration.

**3. Or execute manually:**
```
/do
```

Pick and execute a single task yourself.

## How It Works

### Skills

| Skill | Role | Invocation |
|-------|------|------------|
| `/later` | Capture deferred work | Automatic on intent ("save for later") or explicit |
| `/next` | Orchestrate + dispatch | When ready to process backlog |
| `/do` | Execute single task | Manual single-task work |
| `todo-store` | Storage abstraction | Internal (used by other skills) |

### Key Concepts

- **Tasks are prompts** - Each TODO is a complete handoff for a fresh agent with zero context
- **Worktree isolation** - Every task runs in its own git worktree (branch + directory)
- **Worktree chaining** - Dependent tasks build on predecessor's worktree, not main
- **Sequential dispatch, concurrent execution** - Smart one-at-a-time decisions, parallel agent work
- **Structured reports** - Agents report status, summary, files changed in a parseable format
- **Intelligent integration** - PR strategy adapts to project context (CI, team, etc.)

### Recommended Workflow

**Two terminals** for optimal productivity:

- **Terminal 1**: `/next` - Dedicated orchestration session
- **Terminal 2**: `/later` for capture, `/do` for manual work

Start with a single session to learn the basics, then graduate to two terminals.

## Design Documentation

- [Architecture](design/architecture.md) - Protocol overview, key principles, task lifecycle
- [Dispatch Variants](design/variants.md) - Alternative strategies (wave, blocking, adaptive)
- [Verification Patterns](design/verification-patterns.md) - How agents verify their work
- [Interactive Diagram](diagram.html) - Visual system flow

Per-skill design rationale lives alongside each skill in `skills/[name]/references/DESIGN.md`.

## Repository Structure

```
skills/                    # Auto-discovered by Agent Skills-compatible tools
├── later/                 # Capture deferred work
│   ├── SKILL.md
│   └── references/DESIGN.md
├── next/                  # Orchestrate + dispatch
│   ├── SKILL.md
│   └── references/DESIGN.md
├── do/                    # Execute tasks in worktrees
│   ├── SKILL.md
│   └── references/
│       ├── DESIGN.md
│       └── verification-patterns.md
└── todo-store/            # Storage abstraction
    ├── SKILL.md
    └── references/DESIGN.md

agents/                    # For tools with agent support
└── zero.md                # ZERO protocol guide agent

protocol/zero/             # ZERO as portable skill
└── SKILL.md               # For tools without agent support

design/                    # Protocol-level documentation
├── architecture.md
├── variants.md
└── verification-patterns.md

diagram.html               # Interactive visualization
AGENTS.md                  # Cross-tool AI instructions
CLAUDE.md -> AGENTS.md     # Claude Code discovery
.claude-plugin/plugin.json # Claude Code plugin manifest
```

## License

MIT
