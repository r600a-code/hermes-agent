---
name: context-compaction-handoff
description: Preserve task continuity across `/compress` and automatic context compaction. Use when an agent relies on todo state, recent progress, or short-term execution history that would otherwise vanish during handoff.
version: 1.0.0
author: AIAD
license: MIT
metadata:
  hermes:
    tags: [context, compaction, compression, handoff, continuity, todo, session-state]
    related_skills: [hermes-agent]
---

# Context Compaction Handoff

Context compaction should save tokens without making the agent lose its immediate sense of progress.

The failure mode is subtle: the agent keeps the current active plan, but loses the short-term transition history that explains what just happened. After compaction, it may know that task B is in progress, but forget that task A was already completed one turn earlier. That is enough to cause re-orientation drift, duplicate work, or stale reasoning.

This skill teaches the agent to preserve a compact handoff layer across `/compress` or automatic compaction:

- current active work
- bounded recent transition history
- enough continuity to resume without re-fetching or re-deciding
- without re-injecting completed work as if it were still active

## When to Use

Use this skill when:
- an agent session is approaching compaction and is tracking work with a todo/task list
- the user asks to improve `/compress` quality or handoff continuity
- the agent loses recent progress details after compaction
- you are designing summaries for long-running coding or research sessions
- active state and recent transitions both matter

Do not use this as a generic long-session summary skill when a broad narrative summary is enough. This skill is specifically for preserving execution continuity around task state.

## Core Principle

Preserve two layers, not one:

1. **Active state** — what is still pending or in progress right now.
2. **Recent transition timeline** — what changed just before compaction.

If you preserve only active state, the agent loses the immediate causal chain.
If you preserve everything, you waste tokens and may reintroduce finished work.

The right design is:
- active list stays minimal
- recent history is bounded
- completed work is excluded from the active section
- recent changes remain visible in a separate timeline section

## Handoff Format

Recommended post-compaction injection shape:

```text
[Your active task list was preserved across context compression]
- [ ] inspect. Inspect root cause (pending)
- [>] fix. Patch todo compaction (in_progress)

[Recent task timeline]
- 2026-04-14 07:00:00Z · inspect. Inspect root cause · status · pending → completed
- 2026-04-14 07:00:05Z · fix. Patch todo compaction · status · pending → in_progress
```

Why this works:
- the active section gives the current working set
- the timeline preserves immediate progress context
- completed items are visible as history, not as active work

## Procedure

### 1. Identify the state that must survive compaction

Before changing compression behavior, list the exact state the agent must preserve:
- active tasks
- most recent completed task
- task transitions (`pending → completed`, `pending → in_progress`)
- item rename or removal events if they affect continuation
- any short-lived execution fact that would otherwise be lost

If the answer is only “the general gist of the conversation,” use a normal summary instead.

### 2. Split current state from recent history

Do not inject one blended blob.

Use two sections:
- **Active section**: only pending / in-progress items
- **Recent timeline**: bounded history of recent task events

This prevents the model from treating historical completion records as active work.

### 3. Bound the timeline aggressively

Keep only a small recent window.

Good defaults:
- retain the last 6–12 history events
- keep each event to one line
- drop old events first

The handoff layer should be cheap enough that keeping it is obviously better than re-discovering the same state later.

### 4. Record semantic events, not raw snapshots

Track compact events such as:
- `planned`
- `renamed`
- `status`
- `removed`

Examples:
- `inspect · planned · set to pending`
- `inspect · status · pending → completed`
- `fix · status · pending → in_progress`
- `draft-pr · removed · dropped from plan (was pending)`

Event-style history is far smaller and more informative than storing full copies of every todo list revision.

### 5. Keep completed work out of the active list

This is the guardrail.

Completed or cancelled work should not appear in the active list section after compaction. Otherwise the model may redo finished work.

If a completed task still matters for continuity, show it only in the timeline section.

### 6. Add regression tests before you trust the fix

A good continuity regression test should verify both sides:

- the recent transition history survives compaction injection
- completed items do **not** reappear in the active section

Minimum test shape:
1. Create multiple pending items.
2. Update one to `completed` and another to `in_progress`.
3. Build the post-compaction injection string.
4. Assert that:
   - the timeline exists
   - the transition lines are present
   - the completed task is absent from the active section

### 7. Evaluate with continuation probes

After implementing the handoff, test it with questions like:
- “What just finished?”
- “Which task is currently in progress?”
- “What should happen next?”
- “Should we re-run root cause inspection or proceed to patching?”

If the agent answers these correctly after compaction without re-reading old context, the handoff is doing its job.

## With vs Without This Skill

### Without the skill

Post-compaction injection only preserves active items:

```text
[Your active task list was preserved across context compression]
- [>] fix. Patch todo compaction (in_progress)
```

Likely result:
- the agent knows what it is doing now
- but loses the fact that `inspect` just completed
- may re-open investigation, ask redundant questions, or hedge about next steps

Example continuation:

> We should probably verify whether the root cause has been identified before patching.

This is stale reasoning. The root cause step already finished; the transition just got lost.

### With the skill

Post-compaction injection preserves active items plus the recent transition timeline:

```text
[Your active task list was preserved across context compression]
- [>] fix. Patch todo compaction (in_progress)

[Recent task timeline]
- 2026-04-14 07:00:00Z · inspect. Inspect root cause · status · pending → completed
- 2026-04-14 07:00:05Z · fix. Patch todo compaction · status · pending → in_progress
```

Likely result:
- the agent recognizes that investigation is done
- it continues directly with the patch
- it preserves continuity without needing the full pre-compaction transcript

Example continuation:

> Root cause inspection is complete. I should continue the in-progress patch and then validate it with a regression test.

That is the target behavior.

## Design Rules

1. Optimize for continuation quality, not just token reduction.
2. Preserve active work and recent transitions separately.
3. Never reintroduce completed work into the active section.
4. Prefer compact semantic events over full-state snapshots.
5. Bound the history window so the handoff stays cheap.
6. Add regression tests for both continuity and anti-regression.
7. Evaluate with post-compaction continuation probes, not only summary readability.

## Example Bug-Fix Pattern

A concrete implementation pattern for todo-backed sessions:

1. Keep the canonical todo list in memory.
2. On each write/merge, compare previous vs current items.
3. Append compact history events for status changes, renames, additions, and removals.
4. Trim history to a small fixed window.
5. During compaction injection:
   - inject only active items in the main list
   - append a separate recent timeline block
6. Add a regression test proving that a completed item stays out of the active section while its recent transition remains visible.

## Verification Checklist

Before finalizing a compaction-continuity fix, verify:
- active items survive compaction
- recent transition history survives compaction
- completed items do not appear in the active section
- the timeline is short and bounded
- the agent can answer “what happened just before now?” after compaction
- regression tests cover the edge case that originally failed

## Related Files

See `references/with-vs-without.md` for a compact comparison artifact you can attach to PRs or design discussions.
