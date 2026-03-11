# Protocol Variants

Alternative approaches to `/next` orchestration. We chose "concurrent execution with sequential dispatch" but these variants have their place.

## Current: Sequential Dispatch, Concurrent Execution with Full Reassessment

```
read ALL TODOs → evaluate → dispatch ONE → read ALL TODOs → evaluate → dispatch ONE → ...
                 (agents run concurrently in background)
```

**Why we chose this:**
- **Complete state refresh at each decision** - re-reading the entire TODO list ensures we catch priority changes, new tasks, and shifted context
- Fresh evaluation based on current reality, not stale snapshot
- Responsive to changing state (completion unblocks dependencies, reveals new priorities)
- Doesn't overwhelm with parallel failures
- But still gets concurrency benefits from background agents

---

## Variant A: Wave/Batch Dispatch

```
evaluate → dispatch BATCH → wait for wave → dispatch next BATCH → ...
```

Group independent tasks, dispatch all at once, wait for the wave to complete, then dispatch the next wave.

### Why It's Appealing

- Maximum parallelism for independent work
- Potentially faster throughput
- Simpler mental model ("wave 1, wave 2, wave 3")

### Why We Didn't Choose It

**The failure mode is overwhelming.**

If tasks aren't reliably one-shottable - if they fail, need clarification, require human intervention - you get *multiple* failing threads simultaneously. The human has to context-switch between:
- "Task A failed at step 3"
- "Task B needs permission for X"
- "Task C has a merge conflict"
- "Task D is waiting for input"

That's cognitively brutal. You're debugging N things at once instead of one at a time.

### What Would Unlock This

**Tooling needs to get better.** Specifically:

1. **More one-shottable execution** - Tasks succeed without intervention most of the time
2. **Better autonomous error recovery** - Agents can fix common issues without escalating
3. **Smarter resource isolation** - No port conflicts, DB collisions, etc.
4. **Reliable verification** - Agents can confidently say "this works" without human validation

As the stack matures and success rates climb, wave dispatch becomes viable. It's an efficiency play that requires reliability as a prerequisite.

### When It Might Make Sense Now

- Batch of truly independent, low-risk tasks
- Well-tested codebase with good CI
- Tasks that are similar in nature (e.g., "update 10 config files")
- User explicitly opts in knowing the trade-off

---

## Variant B: Blocking/Sequential Execution

```
evaluate → dispatch ONE → WAIT until complete → evaluate → dispatch ONE → ...
           (only one agent runs at a time)
```

No concurrency. Wait for each task to fully complete before starting the next.

### Why It's Appealing

- Simplest mental model
- Human can focus on one thing at a time
- Easy to reason about state (only one worktree active)
- Natural checkpoints for human review

### Why We Didn't Choose It (As Default)

**Leaves efficiency on the table.** If Task A takes 10 minutes and Task B is independent, why wait? You're serializing work that could overlap.

### When It Makes Sense

**High human-in-the-loop environments:**

1. **Frequent verification needs** - Human wants to validate each task before moving on
2. **Reconciliation complexity** - Merging worktrees requires thought and attention
3. **Learning/onboarding** - User is new to the system and wants to understand each step
4. **High-stakes changes** - Each task affects critical systems, caution warranted
5. **Unreliable execution** - Tasks often fail, so batching would just create a mess

**The blocking variant is a lower bar of entry.** If you expect lots of intervention, don't spawn concurrent work that will compete for your attention.

### Implementation

This is actually just the current system with a flag:

```
/next --blocking
```

Or detect it from context:
- No CI? Maybe default to blocking
- User has been intervening frequently? Suggest blocking mode
- First time using Zero CODE? Start blocking, graduate to concurrent

---

## Variant C: Adaptive Dispatch

```
start blocking → if N tasks succeed without intervention → switch to concurrent
```

A hybrid that earns trust over time.

### The Idea

Don't assume reliability. Prove it. Start with blocking execution, track success rate, and only "promote" to concurrent dispatch when the system demonstrates it can handle it.

### Promotion Criteria

- X consecutive tasks completed without human intervention
- Error rate below threshold
- No resource conflicts detected
- Verification passing reliably

### Demotion Triggers

- Task fails requiring manual fix
- Human intervenes mid-execution
- Resource conflict occurs
- User explicitly requests slowdown

### Why This Is Interesting

It's self-calibrating. New projects or unreliable environments stay blocking. Mature, stable projects earn concurrency. The system adapts to reality rather than assuming it.

### Complexity Cost

More state to track, more logic to maintain, harder to predict behavior. Might be over-engineering for most cases.

---

## Summary: Choosing a Variant

| Variant | Best For | Risk |
|---------|----------|------|
| **Sequential dispatch, concurrent execution** (current) | Balanced reliability + efficiency | Some overhead from continuous evaluation |
| **Wave/batch dispatch** | High-reliability environments, independent tasks | Overwhelming if things fail |
| **Blocking/sequential** | High human-in-the-loop, learning, high-stakes | Slower than necessary |
| **Adaptive** | Long-running projects that evolve | Complexity, unpredictable behavior |

The current default optimizes for the messy middle - things sometimes fail, humans sometimes need to intervene, but we still want reasonable throughput. As tooling matures, the dial can shift toward more aggressive concurrency.

---

## Future Considerations

### For Wave Dispatch
- Track success rates per project/task-type
- Let users opt-in with `--wave` flag
- Auto-suggest when conditions look favorable

### For Blocking Mode
- Detect high-intervention patterns and suggest it
- Make it the default for new/unknown projects
- Provide clear "graduate to concurrent" path

### For Adaptive
- Define clear promotion/demotion metrics
- Make the logic transparent ("you're in blocking mode because X")
- Let users override if they know better
