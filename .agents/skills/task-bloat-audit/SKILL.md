---
name: task-bloat-audit
description: Detect and remove task bloat — proposed tasks that are regression checks, QA steps, or "confirm X after Y" verifications that should have been inside the parent task's acceptance criteria instead of being separate tasks. Run this after any large batch of merges, or when the task list feels cluttered. Produces a categorised list of tasks to archive and a single consolidation task.
---

# Task Bloat Audit

Task bloat occurs when a task agent finishes a feature and proposes follow-up tasks that are really just verification steps that belong inside the original task's **Done looks like** criteria. The result is dozens of small "Confirm X still works" tasks that clog the backlog without producing new value.

## When to Run

- After a large batch of task merges
- When the user notices proposed tasks piling up
- Before a sprint or release to clean up the backlog
- When the user says "why is X a separate task?"

## The Core Rule

> Any task whose title starts with **Confirm, Verify, Make sure,** or **Check that** is a red flag. Ask: *"Does this require writing new code, or is it purely a QA step?"* If it's purely QA, it's bloat and belongs as a bullet in the parent task's Done looks like — not a standalone proposed task.

## Bloat Categories

### Category 1 — "Confirm X after Y was merged"
Regression checks written as tasks after the parent change landed.
- Pattern: "Confirm X still works after [change]", "Make sure X still looks right after [fix]"
- Signal: The parent task is already MERGED or QUEUED
- Action: **Archive** — should have been in the parent's acceptance criteria

### Category 2 — "Confirm X works end-to-end"
Playwright tests or integration checks spun off from the feature task that built X.
- Pattern: "Confirm the [form/flow/widget] works end-to-end", "Catch a broken [flow] before it ships"
- Signal: No new feature code — only test code. The feature is already shipped.
- Action: **Archive** — tests belong inside the feature task

### Category 3 — Launch checklist items
Pre-launch content checks dressed up as tasks.
- Pattern: "Confirm real [phone number / address / rate] is live before launch"
- Signal: No code needed — just verify a content value is correct
- Action: **Archive** — put in a single pre-launch checklist doc, not the task board

### Category 4 — Speculative future checks
Checks contingent on a hypothetical future change that hasn't been scheduled.
- Pattern: "Confirm X still works after a future [library] upgrade"
- Signal: The triggering change doesn't exist yet
- Action: **Archive** — reopen when the triggering change is actually planned

### Category 5 — Duplicates
- Pattern: Two tasks with identical or near-identical titles
- Action: **Archive** the newer one, keep the older one

### Category 6 — Should be absorbed into a sibling task
- Pattern: A tiny task whose scope is fully contained inside a larger proposed task that isn't merged yet
- Action: **Update the parent task** to include this scope, then **archive** the child

## Audit Procedure

```javascript
// Fetch all PROPOSED tasks in batches of 25 (larger batches cause TaskMan errors)
const batches = [];
for (let start = 1; start <= 500; start += 25) {
  batches.push(Array.from({length: 25}, (_, i) => `#${start + i}`));
}
const results = [];
for (let i = 0; i < batches.length; i += 4) {
  const chunk = batches.slice(i, i + 4);
  const chunkResults = await Promise.all(
    chunk.map(refs =>
      queryProjectTasks({ taskRefs: refs, states: ["PROPOSED"], maxDescriptionChars: 0 })
        .catch(() => ({ tasks: [] }))
    )
  );
  for (const r of chunkResults) results.push(...r.tasks);
}
results.sort((a, b) => parseInt(a.taskRef.replace('#','')) - parseInt(b.taskRef.replace('#','')));
console.log(`Total PROPOSED: ${results.length}`);
results.forEach(t => console.log(`${t.taskRef}: ${t.title}`));

// For any task needing full description:
const { task } = await getProjectTask({ taskRef: "#97" });
console.log(task.description);
```

## Output Format

Present findings as a table grouped by category:

| Task | Title | Category | Should have been inside |
|------|-------|----------|------------------------|
| #71 | Confirm fonts load after render-blocking fix | Cat 1 | Font render-blocking fix task |
| #97 | Confirm quote form submits on slow API | Cat 2 | Quote form submission task |
| #11 | Confirm real phone number is live | Cat 3 | Pre-launch checklist |

Total to archive: N tasks. Then surface them using `surfaceProjectTasks` (max 10 per call) so the user can batch-select and archive via the task panel UI. Cancellation is not available programmatically — the user must action it in the task panel.

## What NOT to Archive

These look like bloat but are genuine work — verify before archiving:

- Tasks where verification requires **writing new code** (retry logic, error boundaries, fallback UI)
- Tasks where the parent task was **never built** — the "confirm" is guarding against a gap, not a regression
- Tasks that are **open questions** requiring admin/team input before the code path is even known
- Tasks whose scope **goes beyond the parent** — e.g. "confirm X on city pages too" when the parent only fixed the homepage

## Prevention Rule (enforce on all future task agents)

When proposing follow-up tasks, each candidate must pass this gate:

> *"Does this task produce new code, content, or configuration that doesn't exist yet?"*

If no — it is a bullet in **Done looks like**, not a task. Regression tests and QA steps are always in scope of the task that shipped the feature.
