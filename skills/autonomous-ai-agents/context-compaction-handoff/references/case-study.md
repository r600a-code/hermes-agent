# Context Compaction Handoff — Case Study

This case study shows why the `context-compaction-handoff` skill exists and what practical problem it solves.

## The problem

After `/compress` or automatic context compaction, an agent often keeps the current active plan but loses the short-term transition history that explains what just changed.

That means the agent may still know:
- what is currently in progress
- what is still pending

But it may lose:
- which task just completed
- which task just moved to `in_progress`
- why the current next step is justified

This weakens post-compaction continuation quality.

## Minimal scenario

Initial plan:
- `inspect` — Inspect root cause
- `fix` — Patch compaction behavior
- `validate` — Add regression test and verify

Immediately before compaction:
- `inspect`: `pending → completed`
- `fix`: `pending → in_progress`
- `validate`: remains `pending`

## Without the skill

### Injected handoff

```text
[Your active task list was preserved across context compression]
- [>] fix. Patch compaction behavior (in_progress)
- [ ] validate. Add regression test and verify (pending)
```

### Consequence

The agent still knows the current working set, but it no longer knows that `inspect` just finished.

Typical degraded continuation:

> We should probably confirm the root cause before changing the compaction logic.

That is stale reasoning. The root-cause step already finished; only the transition history was lost.

## With the skill

### Injected handoff

```text
[Your active task list was preserved across context compression]
- [>] fix. Patch compaction behavior (in_progress)
- [ ] validate. Add regression test and verify (pending)

[Recent task timeline]
- 2026-04-14 07:00:00Z · inspect. Inspect root cause · status · pending → completed
- 2026-04-14 07:00:05Z · fix. Patch compaction behavior · status · pending → in_progress
```

### Consequence

The agent now preserves:
- the live plan
- the recent transition context

Typical healthy continuation:

> Root cause inspection is complete. Continue the in-progress patch, then run the regression test.

That is the desired behavior.

## Why this matters

The value of the skill is not “more context.”

It is preserving the right two layers:
1. active state
2. recent transition timeline

That small split is enough to improve handoff continuity without re-injecting the full transcript.

## Reviewer takeaway

Without the skill:
- active tasks survive
- recent causal memory is lost

With the skill:
- active tasks survive
- recent causal memory also survives
- completed work does not reappear as active work

## Related references

- `SKILL.md`
- `references/with-vs-without.md`
- `references/live-showcase.md`
