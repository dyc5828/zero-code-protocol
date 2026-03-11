---
name: todo-store
description: Shared foundation for TODO storage. Detects and works with various formats (TODO/, tasks/, todo.md, etc.). Used internally by /later, /next, and /do - not typically invoked directly.
user-invocable: false
metadata:
  author: danielchen
  version: 1.0.0
---

# TODO Store (Shared Foundation)

Single source of truth for TODO storage logic. Abstracts away the specific format so `/later`, `/next`, and `/do` work consistently regardless of how tasks are organized.

## Supported Formats

The TODO store detects and works with these formats:

### Folder-based (one task per file)
```
TODO/                    # or todo/, tasks/, TASKS/
├── add-dark-mode.md
├── fix-auth-bug.md
└── .done/
    └── completed-task.md
```

### File-based (all tasks in one file)
```
TODO.md                  # or todo.md, tasks.md, TASKS.md
```

With structure:
```markdown
# Tasks

## Pending
- [ ] Add dark mode toggle
- [ ] Fix auth bug

## Done
- [x] Refactor API
```

## Detection Algorithm

**Case-insensitive filesystem handling:** macOS (APFS/HFS+) is case-insensitive but case-preserving. `TODO/`, `todo/`, and `Todo/` all resolve to the same folder. The algorithm must:
1. Detect whether the folder/file exists (case-insensitive match is fine)
2. Return the **actual case** as it exists on disk (not the search pattern)
3. Avoid reporting false duplicates when multiple patterns match the same inode

```bash
# 1. Find base project directory (worktree-aware)
BASE=$(git worktree list --porcelain 2>/dev/null | grep '^worktree ' | head -1 | cut -d' ' -f2)
BASE="${BASE:-.}"  # Fallback to current dir if not in git

# Helper: Get actual name on disk (resolves case-insensitive match to real case)
# Usage: actual_name=$(get_actual_name "/path/to" "pattern")
get_actual_name() {
    local dir="$1" pattern="$2"
    # List directory, find entry matching pattern (case-insensitive)
    ls -1 "$dir" 2>/dev/null | grep -i "^${pattern}$" | head -1
}

# 2. Check for folder-based formats (in priority order)
# Only check one variant per logical name - filesystem handles case-insensitivity
for folder in "todo" "tasks"; do
    if [ -d "${BASE}/${folder}" ]; then
        # Get the actual case from disk
        actual=$(get_actual_name "$BASE" "$folder")
        echo "folder:${BASE}/${actual}"
        exit 0
    fi
done

# 3. Check for file-based formats
for file in "todo.md" "tasks.md"; do
    if [ -f "${BASE}/${file}" ]; then
        actual=$(get_actual_name "$BASE" "$file")
        echo "file:${BASE}/${actual}"
        exit 0
    fi
done

# 4. No existing format - return default (will be created on first write)
echo "folder:${BASE}/TODO"
```

**Why this works:**
- `[ -d "${BASE}/todo" ]` matches `TODO/`, `todo/`, or `Todo/` on case-insensitive systems
- `get_actual_name` then retrieves the real directory entry name from `ls` output
- We only check one pattern per logical name (`todo` not `TODO`+`todo`+`Todo`) to avoid duplicate detection
- The returned path uses the actual case, so writes go to the correct location

## Operations

### Detect Store

Returns the TODO store location and format.

**Output:** `{type: "folder"|"file", path: string, exists: boolean}`

```bash
# Example usage
STORE=$(detect_todo_store)
# Returns: "folder:/path/to/project/TODO" or "file:/path/to/project/todo.md"
```

### List Tasks

Returns all pending tasks.

**Folder-based:**
```bash
ls -t "${STORE_PATH}"/*.md 2>/dev/null | grep -v '.done'
```

**File-based:**
```bash
# Parse markdown, extract unchecked items under ## Pending
grep -E '^\s*-\s*\[ \]' "${STORE_PATH}"
```

### Read Task

Returns full content of a specific task.

**Folder-based:**
```bash
cat "${STORE_PATH}/${task_name}.md"
```

**File-based:**
```bash
# Extract task section from the file
# Tasks are delimited by headers or list items
```

### Write Task

Creates or updates a task.

**Folder-based:**
```bash
mkdir -p "${STORE_PATH}"
cat > "${STORE_PATH}/${task_name}.md" << 'EOF'
[task content]
EOF
```

**File-based:**
```bash
# Append to ## Pending section
# Or create file with initial structure if doesn't exist
```

### Archive Task

Moves task to done state with summary.

**Folder-based:**
```bash
mkdir -p "${STORE_PATH}/.done"
mv "${STORE_PATH}/${task_name}.md" "${STORE_PATH}/.done/"
# Append completion summary to the moved file
```

**File-based:**
```bash
# Change `- [ ]` to `- [x]`
# Move from ## Pending to ## Done section
# Append completion timestamp and summary
```

### List Completed

Returns archived/done tasks.

**Folder-based:**
```bash
ls -t "${STORE_PATH}/.done/"*.md 2>/dev/null
```

**File-based:**
```bash
grep -E '^\s*-\s*\[x\]' "${STORE_PATH}"
```

## Task Structure

Regardless of storage format, tasks have this logical structure:

```yaml
id: string           # Filename (folder) or generated (file)
title: string        # Task title
priority: low|normal|high|urgent
tags: string[]
blocks: string[]     # Task IDs this blocks
content:
  context: string    # Why this exists
  task: string       # What to do
  details: string    # Supporting info (optional)
status: pending|done
completion:          # Only if done
  summary: string
  date: string
  branch: string
```

### Folder Format (full)

```markdown
---
priority: normal
tags: [feature]
blocks: [other-task]
---

# Add Dark Mode

## Context
User requested dark mode support...

## Task
Implement dark mode toggle in settings...

## Relevant Details
- Design mockup at /docs/dark-mode.png
- Use CSS variables for theming
```

### File Format (compact)

```markdown
## Pending

- [ ] **Add dark mode** (high) #feature
  Context: User requested dark mode support
  Task: Implement toggle in settings

- [ ] **Fix auth bug** #bugfix
  Context: Login fails on Safari
  Task: Debug and fix Safari-specific issue
```

## Creating the Store

On first task write, if no store exists:

1. **Check project conventions:**
   - Does CLAUDE.md mention a task format?
   - Are there existing patterns in the repo?

2. **Default to folder-based:**
   - More flexibility for detailed tasks
   - Easier to track in git
   - Natural fit for agent handoffs

3. **Create with standard structure:**
   ```bash
   mkdir -p "${BASE}/TODO"
   mkdir -p "${BASE}/TODO/.done"
   ```

## Integration with Zero CODE

### /later uses:
- `detect_store()` - Find where to write
- `write_task()` - Create the task file/entry

### /next uses:
- `detect_store()` - Find task location
- `list_tasks()` - Get all pending tasks
- `read_task()` - Get task details
- `archive_task()` - Move completed with summary

### /do uses:
- `detect_store()` - Find task location
- `list_tasks()` - Show available tasks (manual mode)
- `read_task()` - Get task to execute
- `archive_task()` - Archive when done (manual mode)

## Why This Abstraction?

1. **Flexibility** - Projects can use whatever format fits their workflow
2. **Single source of truth** - Storage logic defined once, used everywhere
3. **Backwards compatible** - Works with existing TODO/ folders
4. **Future-proof** - Can add new formats without changing consumers
