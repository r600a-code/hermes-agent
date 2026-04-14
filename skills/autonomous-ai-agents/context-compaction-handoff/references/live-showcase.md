# Context Compaction Handoff — Live Showcase

This reference is designed for PR reviewers and readers who want to see the skill's effect quickly.

## One-sentence claim

The skill helps an agent preserve not only what is active now, but also what just changed right before compaction.

## Minimal setup

Before compaction, the agent is working on a three-step bugfix:

- `inspect` — Inspect root cause
- `fix` — Patch compaction behavior
- `validate` — Add regression test and verify

Latest state transitions right before compaction:

- `inspect`: `pending → completed`
- `fix`: `pending → in_progress`
- `validate`: still `pending`

---

## Demo A — Without the skill

### Injected state after compaction

```text
[Your active task list was preserved across context compression]
- [>] fix. Patch compaction behavior (in_progress)
- [ ] validate. Add regression test and verify (pending)
```

### Ask the agent

1. What just finished?
2. What should happen next?
3. Should we re-run root cause inspection?

### Typical answer quality

1. What just finished?
   - Unclear / not recoverable from injected state alone
2. What should happen next?
   - Probably patch, but reasoning is less grounded
3. Should we re-run root cause inspection?
   - The agent may say yes, or hedge, because the completion event is missing

### Failure mode

The agent still knows the current plan, but it loses short-term causal memory.
That often produces:

- repeated investigation
- stale reasoning
- unnecessary re-orientation after compaction

---

## Demo B — With the skill

### Injected state after compaction

```text
[Your active task list was preserved across context compression]
- [>] fix. Patch compaction behavior (in_progress)
- [ ] validate. Add regression test and verify (pending)

[Recent task timeline]
- 2026-04-14 07:00:00Z · inspect. Inspect root cause · status · pending → completed
- 2026-04-14 07:00:05Z · fix. Patch compaction behavior · status · pending → in_progress
```

### Ask the agent

1. What just finished?
2. What should happen next?
3. Should we re-run root cause inspection?

### Expected answer quality

1. What just finished?
   - `inspect` just completed
2. What should happen next?
   - Continue the in-progress patch, then validate
3. Should we re-run root cause inspection?
   - No; inspection already completed right before compaction

### Win condition

The agent resumes execution cleanly without needing the full old transcript.

---

## Reviewer-friendly takeaway

Without the skill:
- active state survives
- recent transition context is lost

With the skill:
- active state survives
- recent transition context also survives
- completed tasks do not reappear as active work

That is the practical value of the skill.

---

## 15-second explanation for a PR description

This skill improves post-compaction handoff quality by preserving two layers separately:
1. active work
2. recent transition history

That small split reduces duplicate work and stale reasoning after `/compress` without re-injecting the full transcript.
