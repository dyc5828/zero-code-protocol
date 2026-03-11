---
name: do
description: Executes a single task from TODO/ in a worktree. Use for manual single-task work ("/do"), or as the execution protocol that sub-agents follow when dispatched by /next.
metadata:
  author: danielchen
  version: 1.0.0
---

# Do (Task Execution)

The hands of the **Zero CODE** protocol - executes a single task with worktree isolation. Handles the **E**xecute step in the CODE flow.

## When to Use

- **Manual mode:** User wants to work on a single task themselves
- **Sub-agent mode:** Dispatched by `/next` orchestrator (follows this protocol)
- User says "let's do a task", "pick something to work on", "/do"

## Responsibility

| Does | Does NOT |
|------|----------|
| Create worktree | Analyze other tasks |
| Execute task requirements | Dispatch other agents |
| Verify own work | Decide integration strategy |
| Report completion | Push/merge (unless manual mode) |
| Capture follow-up items | Archive own task |
| Manage own resources | Orchestrate dependencies |

## Manual Mode Process

When user invokes `/do` directly (not via `/next`):

### Step 1: Find Tasks

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

### Step 2: Present Options

Show available tasks with metadata:

```
Available tasks:
  1. add-dark-mode.md (high priority)
  2. fix-auth-bug.md
  3. theme-variants.md [blocked by: add-dark-mode]

Recommended: add-dark-mode.md (high priority, unblocks theme-variants)

Which task? (number or name)
```

**Selection bias:**
- Newer + higher priority
- Unblocks other tasks
- Not blocked by incomplete work

### Step 3: Execute Task

Follow the Worktree Protocol below.

### Step 4: Integration Decision (Manual Only)

After task completion, ask user:

```
=== Task Complete ===
Status: success
Branch: feature/add-dark-mode

How to integrate?
  1. Create PR now
  2. Push branch only (PR later)
  3. Direct merge to main
  4. Leave for /next to handle

Choice?
```

Execute chosen option:
- **PR now:** `git push -u origin [branch]` then `gh pr create`
- **Push only:** `git push -u origin [branch]`
- **Direct merge:** `git checkout main && git merge [branch]`
- **Leave:** Do nothing, worktree persists

### Step 5: Archive (Manual Only)

Move task to `TODO/.done/` with completion summary appended.

---

## Worktree Protocol

**REQUIRED BY DEFAULT** - All tasks MUST use worktree isolation unless user explicitly opts out (e.g., "edit directly on main", "no worktree", "skip worktree").

**Why worktrees are not optional:**
- Keeps main branch clean
- Enables natural PR workflow
- Prevents accidental commits to main
- Allows parallel agent execution
- Even "simple" tasks benefit from isolation

This is the core execution protocol. Followed in both manual and sub-agent mode.

### 1. Determine Base

```bash
# If orchestrator specified a base (for dependent tasks):
BASE="${SPECIFIED_BASE}"

# Otherwise, use main:
BASE="${BASE:-main}"
```

### 2. Create Worktree

```bash
# Branch name can be anything natural: feature/*, fix/*, chore/*, or plain names
BRANCH_NAME="[branch-name]"  # No prescribed format - name it however makes sense

# Get current directory name for worktree naming
REPO_NAME=$(basename "$PWD")

# Sanitize branch name for folder (replace / with -)
SAFE_BRANCH=$(echo "${BRANCH_NAME}" | tr '/' '-')

# Create worktree as SIBLING of current directory (same parent)
WORKTREE_PATH="../${REPO_NAME}-${SAFE_BRANCH}"

git worktree add "${WORKTREE_PATH}" -b "${BRANCH_NAME}" "${BASE}"
cd "${WORKTREE_PATH}"
```

If worktree already exists (from previous failed attempt):
- Check if it has uncommitted work
- If clean, remove and recreate: `git worktree remove --force`
- If dirty, inform user/orchestrator

### 3. Project Setup Phase

**CRITICAL:** Before starting task work, properly set up the project environment. Skipping this causes verification failures (missing builds, unknown setup requirements, no project context).

#### 3a. Read AI Memory/Instruction Files

These files contain critical project context that agents MUST read:

```bash
# Common AI instruction file locations
AI_FILES=(
    "CLAUDE.md"                          # Project root instructions
    ".claude/CLAUDE.md"                  # Claude-specific config
    ".cursorrules"                       # Cursor editor rules
    ".github/copilot-instructions.md"    # GitHub Copilot instructions
    ".aider.md"                          # Aider AI instructions
    "README.md"                          # Project overview
)

# Read all that exist
for file in "${AI_FILES[@]}"; do
    [[ -f "$file" ]] && echo "Reading project context: $file"
done
```

**What to extract from these files:**
- Setup/installation commands
- Build processes
- Environment configuration needs
- Secret/credential management
- Testing commands and requirements
- Project-specific conventions
- Known gotchas or prerequisites

**Use Read tool to actually read these files** - don't just check if they exist.

#### 3b. Detect Project Type and Dependencies

```bash
# Detect project type by checking for config files
if [[ -f "package.json" ]]; then
    PROJECT_TYPE="node"
    PACKAGE_MANAGER="npm"  # Default

    # Detect which package manager
    [[ -f "pnpm-lock.yaml" ]] && PACKAGE_MANAGER="pnpm"
    [[ -f "yarn.lock" ]] && PACKAGE_MANAGER="yarn"
    [[ -f "bun.lockb" ]] && PACKAGE_MANAGER="bun"

elif [[ -f "pyproject.toml" || -f "setup.py" || -f "requirements.txt" ]]; then
    PROJECT_TYPE="python"

elif [[ -f "go.mod" ]]; then
    PROJECT_TYPE="go"

elif [[ -f "Cargo.toml" ]]; then
    PROJECT_TYPE="rust"

elif [[ -f "composer.json" ]]; then
    PROJECT_TYPE="php"

elif [[ -f "Gemfile" ]]; then
    PROJECT_TYPE="ruby"
fi
```

#### 3c. Run Setup Commands

**Follow AI instruction file guidance first.** If they specify setup steps, do exactly that. Otherwise, use these defaults:

**Node.js projects:**
```bash
# Install dependencies
${PACKAGE_MANAGER} install

# Check for build scripts in package.json
if grep -q '"build"' package.json; then
    ${PACKAGE_MANAGER} run build
fi

# Check for setup/postinstall scripts
if grep -q '"setup"' package.json; then
    ${PACKAGE_MANAGER} run setup
fi
```

**Python projects:**
```bash
# Check for virtual environment
if [[ -d "venv" ]] || [[ -d ".venv" ]]; then
    source venv/bin/activate 2>/dev/null || source .venv/bin/activate
fi

# Install dependencies
if [[ -f "requirements.txt" ]]; then
    pip install -r requirements.txt
elif [[ -f "pyproject.toml" ]]; then
    pip install -e .
fi
```

**Go projects:**
```bash
# Download dependencies
go mod download

# Build if needed
go build ./...
```

**Rust projects:**
```bash
# Build project
cargo build
```

**Other projects:**
```bash
# Look for Makefile
if [[ -f "Makefile" ]]; then
    # Check for setup target
    if make -n setup >/dev/null 2>&1; then
        make setup
    fi
fi
```

#### 3d. Environment Configuration

**Check for environment file templates:**
```bash
# Look for environment templates
if [[ -f ".env.example" ]] && [[ ! -f ".env" ]]; then
    echo "Warning: .env.example exists but .env does not"
    echo "You may need to create .env with required values"
    # Don't auto-copy - might need real secrets
fi

# Check if AI instruction files mention environment setup
# (Already read in step 3a - apply their guidance here)
```

#### 3e. Verify Setup Worked

```bash
# Basic smoke tests based on project type
case "$PROJECT_TYPE" in
    node)
        # Check node_modules exists
        [[ -d "node_modules" ]] || echo "Warning: node_modules missing"
        ;;
    python)
        # Check imports work
        python -c "import sys; print(sys.version)" >/dev/null
        ;;
    go)
        # Check go.mod is valid
        go list ./... >/dev/null 2>&1
        ;;
esac
```

**If setup fails:**
- Don't proceed to task execution
- Report in completion report as `failed` status
- Include setup error in Issues/Notes section

### 4. Plan Verification BEFORE Implementing

Ask yourself:
- What will prove this task is complete?
- What tests should pass?
- What resources do I need (ports, servers, databases)?

**Multi-agent awareness checks:**

```bash
# Check if port is in use
lsof -i :3000 >/dev/null 2>&1 && echo "Port 3000 in use"

# Find available port
for port in 3000 3001 3002 3003; do
    lsof -i :$port >/dev/null 2>&1 || { echo "Using port $port"; break; }
done
```

If resources are in use:
- Don't terminate others' processes
- Find alternatives (different port, different test DB prefix)
- Document what you're using

### 4. Execute the Task

Do whatever the task requires:
- Write code
- Modify configuration
- Create files
- Run setup commands

**Stay focused:**
- Do what the task asks, nothing more
- Don't scope-creep
- If you notice other issues -> capture with `/later`, don't fix now

### 5. Verify the Work

**Verification is not optional.** Your status depends on verification passing, not just code existing.

**For application tasks (web apps, APIs, CLIs):**

Lean into active verification patterns. See `references/verification-patterns.md` for comprehensive techniques:

- **Background server management** - Start dev servers, health check, monitor logs
- **Browser automation** - Use Playwright tools to test UI behavior as a real user
- **Client/server correlation** - Check that UI state matches server logs
- **Resource management** - Find available ports, use unique test DBs, clean up after

**Common verification workflows:**

**Web applications:**
```bash
# Start dev server in background
npm run dev > server-$$.log 2>&1 &
SERVER_PID=$!

# Health check
for i in {1..30}; do
    curl -sf http://localhost:3000 >/dev/null && break
    sleep 1
done

# Use Playwright to test feature
# browser_navigate, browser_snapshot, browser_fill_form, browser_click
# Verify UI state changes, check console errors, inspect network requests

# Correlate with server logs
grep -i "POST /api/users" server-$$.log

# Clean up
kill $SERVER_PID 2>/dev/null
```

**REST APIs:**
```bash
# Start server
npm run dev > server-$$.log 2>&1 &
SERVER_PID=$!

# Test endpoints
curl -X POST http://localhost:3000/api/users \
    -H "Content-Type: application/json" \
    -d '{"username":"test"}' \
    -w "%{http_code}\n"

# Check server logs
grep "201" server-$$.log

# Clean up
kill $SERVER_PID 2>/dev/null
```

**For all task types:**

If project has existing test suites, run them:

```bash
# Unit tests
npm test  # or pytest, go test, etc.

# Linting
npm run lint

# Type checking
npm run typecheck  # or tsc --noEmit
```

**Acceptance criteria:**
- Does the feature work as described?
- Are edge cases handled?
- Did you verify it works (not just visually inspect code)?
- Did existing tests still pass?

### 6. Commit Locally

```bash
git add .
git commit -m "feat: [description from task]"
```

**Do NOT push yet** (orchestrator decides integration strategy).

### 7. Clean Up Resources

```bash
# Stop any servers you started
# Release any ports you claimed
# Remove temp files you created
```

### 8. Capture Follow-ups

If you noticed things out of scope:

```bash
/later "Noticed: [issue] while working on [task]" --tag follow-up
```

Examples:
- "This function could be cleaner, but out of scope"
- "Found unrelated bug in auth flow"
- "Should add more test coverage later"

### 9. Report Completion

**EXACT FORMAT REQUIRED:**

```markdown
## Completion Report
- Status: success | partial | failed
- Worktree: ../[repo-name]-[sanitized-branch-name]
- Branch: [branch-name]
- Base: [main or predecessor worktree]

## Summary
[2-3 sentences: what was done, key decisions made]

## Files Changed
- path/to/file1.ts (created)
- path/to/file2.ts (modified)
- path/to/file3.ts (deleted)

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
[Any blockers, concerns, or follow-up items captured]
```

**Status definitions:**
- `success`: Task fully complete, all verification passed
- `partial`: Some parts done, some issues remain
- `failed`: Could not complete, blocked or errored

---

## Multi-Agent Awareness

When running in parallel with other `/do` agents:

### Don't Assume Exclusive Access

- Ports may be occupied by other agents
- Dev servers may be running
- Databases may have concurrent connections

### Isolate Your Environment

```bash
# Try ports sequentially until one is free
for PORT in 3000 3001 3002 3003 3004; do
    if ! lsof -i :$PORT >/dev/null 2>&1; then
        export DEV_PORT=$PORT
        break
    fi
done

# Use unique test database prefix
export TEST_DB="test_${TASK_SLUG}_$$"
```

### Don't Kill Shared Resources

If you find a server running, don't kill it:
```bash
# BAD: pkill -f "node server.js"
# GOOD: Start your own on different port
```

### Clean Up After Yourself

Track what you started, stop only those:
```bash
# Save PID when starting
node server.js &
SERVER_PID=$!

# Kill only your server when done
kill $SERVER_PID 2>/dev/null
```

---

## Permission Handling

If a permission is denied mid-work:

1. **STOP immediately** - Don't retry blindly
2. **Don't fail silently** - Make it visible
3. **Report in completion:**

```markdown
## Completion Report
- Status: failed

## Issues/Notes
Blocked by permission: Bash(npm test) was denied.
Could not verify the implementation.
```

This surfaces to user via `/next` orchestrator.

---

## Self-Improvement Protocol

When execution reveals issues or inaccuracies, improve them:

### Immediate Fixes (Preferred)

If you can fix it now, locate and update the relevant file:

| Issue Type | Update |
|------------|--------|
| Execution protocol wrong | Locate and edit this skill's `SKILL.md` (see detection below) |
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

```bash
/later "Improve /do: [what broke] -> [proposed fix]" --tag meta
```

Examples:
- "Improve /do: verification approach didn't catch async bug -> add integration test requirement"
- "Improve /do: port conflict with parallel agent -> add port scanning before server start"
- "Improve /do: worktree cleanup failed on dirty state -> add stash-or-warn step"

### What to Watch For

- Verification that didn't catch actual issues
- Resource conflicts with parallel agents
- Worktree management edge cases
- Missing project context that would help execution

---

## Sub-Agent Mode Notes

When dispatched by `/next`:

- You receive: task content, base worktree, predecessor summary
- You follow this protocol exactly
- You report completion in the exact format
- You do NOT: push, create PR, archive task
- The orchestrator handles all integration decisions

---

## Boundary

`/do` does ONE task, verifies it, reports back. It doesn't know about other tasks, the dependency graph, or integration strategy. That's `/next`'s job.
