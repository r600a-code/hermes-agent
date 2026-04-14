# Context Compaction Handoff — With vs Without the Skill

This artifact shows the practical difference between preserving only the active plan versus preserving both the active plan and a short recent transition timeline.

## Scenario

The agent is working through a todo-backed bug fix.

Initial plan:
- `inspect` — Inspect root cause
- `fix` — Patch todo compaction
- `validate` — Add regression test and verify behavior

Immediately before compaction:
- `inspect` moved from `pending` → `completed`
- `fix` moved from `pending` → `in_progress`
- `validate` remains `pending`

## Without the Skill

### Injected handoff

```text
[Your active task list was preserved across context compression]
- [>] fix. Patch todo compaction (in_progress)
- [ ] validate. Add regression test and verify behavior (pending)
```

### What the model still knows
- It is currently patching.
- Validation remains to be done.

### What the model just lost
- `inspect` already finished moments ago.
- The causal chain that explains why patching is the correct next step.
- Which transition happened most recently.

### Typical degraded continuation

> We should probably confirm the root cause before changing the compaction logic.

This is stale. The root-cause pass already finished; compaction only dropped the transition history.

## With the Skill

### Injected handoff

```text
[Your active task list was preserved across context compression]
- [>] fix. Patch todo compaction (in_progress)
- [ ] validate. Add regression test and verify behavior (pending)

[Recent task timeline]
- 2026-04-14 07:00:00Z · inspect. Inspect root cause · status · pending → completed
- 2026-04-14 07:00:05Z · fix. Patch todo compaction · status · pending → in_progress
```

### What the model now preserves
- The current working set.
- The immediate transition history.
- The reason patching is the right next move.

### Typical healthy continuation

> Root cause inspection is complete. Continue the in-progress patch, then run the regression test.

That is the intended post-compaction continuation.

## Behavioral Comparison

| Dimension | Without skill | With skill |
|---|---|---|
| Active task visibility | Good | Good |
| Awareness of just-completed work | Weak | Strong |
| Chance of redoing finished steps | Higher | Lower |
| Need to re-read old context | More likely | Less likely |
| Token overhead | Lower | Slightly higher but bounded |
| Continuation quality | Fragile | Stable |

## Why the Skill Wins

The skill introduces one additional design rule:

- preserve the active list as the live plan
- preserve a bounded timeline as recent causal memory

That small separation is enough to stop many post-compaction continuity failures without re-injecting the entire conversation.

## Recommended Probe Questions

Use these after compaction to verify the handoff quality:

1. What just finished?
2. What is currently in progress?
3. What should happen next?
4. Should we re-run investigation or continue implementation?

A strong handoff answers these correctly without reading older messages.
