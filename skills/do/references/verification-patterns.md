# Verification Patterns

Practical techniques for verifying application behavior during task execution. Part of the **Zero CODE** protocol.

> **This is design documentation.** It captures verification patterns and techniques that `/do` agents should lean into when building applications. The executable skills reference this for guidance.

## Why Verification Matters

Writing code is not enough. You need to prove it works. In the Zero CODE protocol:

- **`/do` agents must verify** - Status depends on verification passing, not just code existing
- **`/next` injects requirements** - Orchestrator ensures agents plan verification upfront
- **Verification unblocks work** - Dependent tasks trust predecessor's verification

Without verification, you get:
- Silent failures that cascade
- False "success" status that misleads orchestrator
- Integration issues discovered too late
- Lost time debugging issues that should have been caught

**The principle:** If you can't verify it works, you can't claim it's done.

## Core Pattern: Setup → Act → Verify → Teardown

Every verification follows this structure:

```
1. SETUP
   - Start necessary services (servers, databases, etc.)
   - Check resource availability (ports, files, permissions)
   - Prepare test environment

2. ACT
   - Perform the operation being verified
   - Trigger the feature/behavior
   - Execute user flows

3. VERIFY
   - Check expected outcomes (UI state, data changes, logs)
   - Validate error cases handled correctly
   - Correlate client/server behavior

4. TEARDOWN
   - Stop services you started
   - Clean up test data
   - Release resources (ports, temp files)
```

## Pattern 1: Background Server Management

### When to Use

Any task involving a server: web apps, APIs, dev servers, build watchers.

### The Pattern

**Start with `run_in_background`:**
```bash
# Start server in background
npm run dev &
SERVER_PID=$!

# Or using Task tool
# (Task tool with run_in_background: true)
```

**Health check before proceeding:**
```bash
# Wait for server to be ready
for i in {1..30}; do
    if curl -sf http://localhost:3000/health >/dev/null 2>&1; then
        echo "Server ready"
        break
    fi
    sleep 1
done
```

**Monitor during verification:**
```bash
# Check logs for errors
if grep -i "error\|exception\|fatal" server.log; then
    echo "Server errors detected"
fi

# Or use TaskOutput to check background task status
```

**Clean up when done:**
```bash
# Stop server gracefully
kill $SERVER_PID 2>/dev/null

# Verify it stopped
wait $SERVER_PID 2>/dev/null
```

### Multi-Agent Considerations

When other agents may be running servers:

```bash
# Don't assume port 3000 is free
for PORT in 3000 3001 3002 3003 3004; do
    if ! lsof -i :$PORT >/dev/null 2>&1; then
        export DEV_PORT=$PORT
        echo "Using port $PORT"
        break
    fi
done

# Start server on available port
PORT=$DEV_PORT npm run dev > server-$$.log 2>&1 &
SERVER_PID=$!
```

**Don't kill shared resources:**
```bash
# BAD: pkill -f "node"  (kills everyone's servers)
# GOOD: kill $SERVER_PID  (only your server)
```

## Pattern 2: Browser Automation with Playwright

### When to Use

Verifying web UI behavior: forms, navigation, user interactions, client-side validation.

### Available Tools

Claude has access to Playwright MCP tools:

| Tool | Purpose |
|------|---------|
| `browser_navigate` | Load pages, navigate to URLs |
| `browser_snapshot` | Get accessibility tree with element refs |
| `browser_click` | Click elements |
| `browser_type` | Type into inputs (one field at a time) |
| `browser_fill_form` | Fill multiple form fields efficiently |
| `browser_select_option` | Choose dropdown values |
| `browser_console_messages` | Check for JS errors |
| `browser_network_requests` | See HTTP requests/responses |
| `browser_take_screenshot` | Visual snapshot (for debugging) |
| `browser_wait_for` | Wait for text to appear/disappear |

### The Workflow

**1. Start your server (see Pattern 1)**

**2. Navigate to the page:**
```
browser_navigate(url: "http://localhost:3000")
```

**3. Get page structure:**
```
browser_snapshot()
```

Returns an accessibility tree with `ref` attributes for each interactive element. Use these refs for subsequent actions.

**4. Interact with the page:**
```
# Single field
browser_type(ref: "input-username", text: "testuser")

# Multiple fields (more efficient)
browser_fill_form(fields: [
    {name: "username", type: "textbox", ref: "input-username", value: "testuser"},
    {name: "email", type: "textbox", ref: "input-email", value: "test@example.com"}
])

# Dropdowns
browser_select_option(ref: "select-country", values: ["US"])

# Buttons
browser_click(ref: "button-submit")
```

**5. Verify the outcome:**
```
# Get updated page state
browser_snapshot()

# Look for:
# - Success messages
# - Error messages
# - Changed content (watch for <changed> markers)
# - Navigation occurred
```

**6. Check console and network:**
```
# JavaScript errors
browser_console_messages(level: "error")

# Network requests
browser_network_requests(includeStatic: false)
```

### Example: Form Validation Testing

**Test Case 1: Client-side validation (empty submit)**
```
1. browser_navigate to form page
2. browser_snapshot to get refs
3. browser_click on submit (without filling)
4. browser_snapshot to check for validation errors
5. browser_console_messages to confirm no JS errors
6. browser_network_requests to confirm no server request
```

**Test Case 2: Server-side validation (invalid data)**
```
1. browser_fill_form with invalid data (e.g., existing username)
2. browser_click submit
3. browser_wait_for error message to appear
4. browser_snapshot to verify error shown in UI
5. browser_network_requests to see 409/422 response
6. Check server logs for error handling
```

**Test Case 3: Success path**
```
1. browser_fill_form with valid data
2. browser_click submit
3. browser_wait_for success message
4. browser_snapshot to verify form reset or redirect
5. browser_network_requests to see 200/201 response
6. Check server logs for success indication
```

### Browser Management

**Resize if needed:**
```
browser_resize(width: 1280, height: 720)
```

**Handle dialogs (alerts, confirms):**
```
browser_handle_dialog(accept: true)
```

**Close when done:**
```
browser_close()
```

Or let the session end (browser auto-closes).

## Pattern 3: Correlation (Client + Server)

### The Technique

Don't just verify one side. Check that client and server agree.

**Example: Form submission**

**Client-side check:**
- Snapshot shows success message
- Network request got 201 response
- Form fields were reset

**Server-side check:**
- Logs show "User created: testuser123"
- No error logs during request
- Request handled in reasonable time

**Correlation confirms:**
- Client saw success → Server logged success
- Client error → Server returned error code
- No client request → No server log (client validation worked)

### Reading Server Logs

If you started server with output redirection:
```bash
npm run dev > server-$$.log 2>&1 &
```

Then read the log:
```bash
# Check for errors
grep -i "error\|exception" server-$$.log

# Check for success indicators
grep -i "created\|success\|200\|201" server-$$.log

# Check request was received
grep -i "POST /api/users" server-$$.log
```

### Common Correlation Patterns

| Scenario | Client Check | Server Check |
|----------|--------------|--------------|
| Form submission | Success message in UI | "Created" in logs |
| Validation error | Error shown to user | 400/422 response logged |
| Client-side validation | No network request | No server log entry |
| Authentication | Redirect to dashboard | "User authenticated" log |
| API request | Data rendered in UI | Request logged with 200 |

## Pattern 4: Application Type Strategies

### Web Applications

**Primary verification:**
- Browser automation (Playwright)
- Visual/accessibility snapshots
- Network request validation
- Console error checking

**Correlation:**
- UI state + server logs
- Client-side errors + server responses

**Example task:** "Add user registration form"
- Fill form with valid data → verify success message + server log
- Fill form with invalid data → verify error message + 422 response
- Submit empty form → verify client validation + no server request

### REST APIs

**Primary verification:**
- `curl` or HTTP client (fetch, axios)
- Response status codes
- Response body validation
- Server logs

**Example task:** "Add POST /api/users endpoint"
```bash
# Valid request
response=$(curl -s -w "%{http_code}" -X POST http://localhost:3000/api/users \
    -H "Content-Type: application/json" \
    -d '{"username":"test","email":"test@example.com"}')

if [[ $response == *"201"* ]]; then
    echo "Created successfully"
fi

# Check server logs
grep "POST /api/users" server.log | grep "201"
```

### CLIs

**Primary verification:**
- Run commands with test inputs
- Check exit codes
- Validate output format
- Test error cases

**Example task:** "Add --format json flag"
```bash
# Valid input
output=$(./cli list --format json)
echo "$output" | jq . >/dev/null  # Validates JSON

# Invalid input
./cli list --format invalid
if [[ $? -ne 0 ]]; then
    echo "Correctly rejected invalid format"
fi
```

### Database Migrations

**Primary verification:**
- Run migration up/down
- Check schema state
- Test data integrity
- Verify rollback works

**Example task:** "Add users.email_verified column"
```bash
# Run migration
npm run migrate:up

# Check column exists
psql -d mydb -c "\d users" | grep email_verified

# Test rollback
npm run migrate:down
psql -d mydb -c "\d users" | grep -v email_verified
```

## Pattern 5: Resource Management

### Before Starting Verification

**Check what resources you need:**
- Ports (servers, databases)
- Files (temp directories, test DBs)
- External services (APIs, queues)

**Check if resources are available:**
```bash
# Check port
if lsof -i :3000 >/dev/null 2>&1; then
    echo "Port 3000 in use, finding alternative"
fi

# Check file/directory
if [[ -f /tmp/test.db ]]; then
    echo "Test DB exists, using unique name"
fi
```

**Claim resources explicitly:**
```bash
# Use unique identifiers
export TEST_DB="test_${TASK_SLUG}_$$"
export SERVER_PORT=$(find_available_port 3000 3010)
export TEMP_DIR="/tmp/verify-${TASK_SLUG}-$$"
```

### During Verification

**Track what you started:**
```bash
# Save PIDs
echo $SERVER_PID > /tmp/server-$$.pid

# Track ports in use
echo "$SERVER_PORT" > /tmp/ports-$$.txt

# Track temp files
echo "$TEMP_DIR" >> /tmp/cleanup-$$.txt
```

### After Verification

**Clean up everything you claimed:**
```bash
# Stop servers
if [[ -f /tmp/server-$$.pid ]]; then
    kill $(cat /tmp/server-$$.pid) 2>/dev/null
    rm /tmp/server-$$.pid
fi

# Remove temp files
if [[ -f /tmp/cleanup-$$.txt ]]; then
    while read path; do
        rm -rf "$path"
    done < /tmp/cleanup-$$.txt
    rm /tmp/cleanup-$$.txt
fi

# Drop test databases
dropdb "$TEST_DB" 2>/dev/null
```

**Verify cleanup succeeded:**
```bash
# Check server stopped
if lsof -i :$SERVER_PORT >/dev/null 2>&1; then
    echo "WARNING: Server still running on port $SERVER_PORT"
fi
```

## Pattern 6: Test Suites (When Available)

### If Project Has Tests

**Run them as verification:**
```bash
# Unit tests
npm test

# Integration tests
npm run test:integration

# E2E tests (may overlap with your verification)
npm run test:e2e
```

**Regression check:**
- Did your changes break existing tests?
- If yes, is it intentional (API change)?
- If unintentional, fix before claiming success

### If No Tests Exist

**Don't add tests unless task requires it.** Focus on manual verification that proves the feature works.

**But consider capturing test gaps:**
```bash
/later "Add test coverage for user registration flow" --tag testing
```

## Pattern 7: Verification Planning

### Before Implementing

Ask yourself:

1. **What proves this works?**
   - Specific user action?
   - API endpoint returns correct data?
   - CLI output matches expected format?

2. **What resources do I need?**
   - Server? Which port?
   - Database? Test instance or unique name?
   - Browser? Any specific viewport size?

3. **How will I check it?**
   - Browser automation?
   - HTTP requests?
   - Log inspection?
   - Manual testing?

4. **What could go wrong?**
   - Edge cases?
   - Error conditions?
   - Resource conflicts?

### Document Your Plan

In your task execution, state your verification plan upfront:

```markdown
## Verification Plan

**What proves it works:**
- User can submit registration form with valid data
- Form shows error for invalid data (existing username)
- Form shows client-side validation for empty fields

**Resources needed:**
- Dev server on port 3000 (will find alternative if occupied)
- Browser automation (Playwright)

**Verification approach:**
1. Start dev server in background
2. Test case 1: Empty submit → client validation
3. Test case 2: Invalid data → server error
4. Test case 3: Valid data → success
5. Correlate UI state with server logs
6. Stop server and clean up

**Success criteria:**
- All 3 test cases pass
- No console errors
- Server logs correlate with UI behavior
```

## Pattern 8: Failure Handling

### When Verification Fails

**Don't fake success.** Report accurately:

```markdown
## Completion Report
- Status: partial

## How I Verified This
- Started dev server on port 3001
- Tested form submission with Playwright
- Test case 1 (empty submit): PASS - client validation worked
- Test case 2 (invalid data): FAIL - server returned 500 instead of 422
- Test case 3 (valid data): NOT RUN (blocked by test 2 failure)

## Issues/Notes
Server error handling needs work. Captured follow-up task:
- TODO/fix-registration-error-handling.md
```

### Partial Success Is Valid

If some parts work and some don't:
- Report what works
- Document what doesn't
- Status = "partial"
- Let orchestrator decide if dependent work can proceed

### Total Failure

If nothing works:
- Status = "failed"
- Explain what you tried
- Explain what blocked you
- Don't leave broken code (git reset if needed)

## Anti-Patterns

### Don't Do This

**Skipping verification entirely:**
```
# BAD: "I wrote the code, it looks right, claiming success"
Status: success
How I Verified: Visually inspected the code
```

**Verification without cleanup:**
```
# BAD: Leave server running on port 3000
# (blocks other agents from using that port)
```

**Fake verification:**
```
# BAD: Mock out the thing you're trying to verify
# (tests the mock, not the real implementation)
```

**Over-verification:**
```
# BAD: Test every edge case not mentioned in task
# (scope creep, delays completion)
```

**Assuming exclusive access:**
```
# BAD: pkill -f "node"  (kills all Node processes)
# BAD: dropdb test_db  (might be in use by another agent)
```

## Integration with Zero CODE

### How `/do` Uses This

When executing a task, `/do` agents:

1. **Plan verification before implementing** (Pattern 7)
2. **Choose appropriate patterns** (web? API? CLI?)
3. **Set up resources** (Pattern 5)
4. **Execute verification** (Patterns 1-4)
5. **Clean up** (Pattern 5)
6. **Report with verification details** (structured format)

### How `/next` Enforces This

The orchestrator injects verification requirements into agent prompts:

```
## Verification Requirements (Orchestrator-Injected)

Before you implement anything, you MUST define how you'll verify this works.

See design/verification-patterns.md for techniques:
- Background server management
- Browser automation with Playwright
- Client/server correlation
- Resource management

Your status depends on verification passing, not just code being written.
```

### How Tasks Can Specify Verification

Task authors can suggest verification approaches in the TODO file:

```markdown
## Task
Add user registration form with email validation.

## Verification Approach (Suggested)
- Test empty form submission (client-side validation)
- Test invalid email format (client-side validation)
- Test duplicate email (server-side validation)
- Test successful registration (end-to-end)

Use Playwright to automate form interaction.
```

But even if not specified, `/next` requires the agent to plan verification.

## Summary

**Verification is not optional.** It's what makes "success" meaningful.

**Key principles:**

1. **Plan before implementing** - Know how you'll verify before you write code
2. **Use appropriate patterns** - Web apps need browsers, APIs need HTTP clients
3. **Correlate client and server** - Both sides should agree on what happened
4. **Manage resources carefully** - Don't conflict with other agents
5. **Clean up after yourself** - Release what you claimed
6. **Report accurately** - Partial success is better than fake success

**When in doubt:**
- Start a server, interact with it, check the logs
- If it's a UI, use Playwright to act like a user
- If it's an API, curl it and check responses
- If it's a CLI, run it and check output

**The goal:** At the end of verification, you should be confident the feature works. If you're not confident, don't claim success.
