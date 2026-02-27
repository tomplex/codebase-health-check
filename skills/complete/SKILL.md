---
name: complete
description: >
  Use when all batches in a health-check resolution plan are resolved
  and you need to finalize the health check.
---

# Complete — Health-Check Finalization

You are finalizing a completed health-check resolution. Follow every step below in strict order. Do NOT skip steps, reorder them, or assume a step's outcome without doing it.

---

## Phase 0: Locate and Verify

**Goal:** Find the health-check directory and confirm all batches are actually resolved.

1. Check if a path argument was provided (from resolve-batch handoff or the user).
2. If no path was provided, find the most recent directory under `docs/health-checks/` by sorting directory names (which are dates in `YYYY-MM-DD` format). Use the last entry in sorted order.
3. Read `plan.md` from the resolved directory. If it does not exist, stop with this error:
   > "No plan.md found in [path]. Nothing to complete."
4. Read `progress.md` from the resolved directory. If it does not exist, stop with this error:
   > "No progress.md found in [path]. Nothing to complete."
5. Check the Batch Status table in `progress.md`. If ANY batches have status other than `done`:

   **HARD GATE: Do NOT complete silently if batches remain — present and ask.**

   Present the unresolved batches to the user:
   > "N batches remain unresolved:
   > - Batch X: [theme] — [status]
   > - Batch Y: [theme] — [status]
   >
   > Close out anyway (remaining batches will be marked `deferred`), or continue resolving?"

   - If the user chooses to **continue resolving** — invoke `codebase-health-check:resolve-batch` with the health-check directory path and stop. Do NOT proceed to Phase 1.
   - If the user chooses to **close out** — proceed to Phase 1. Unresolved batches will be marked `deferred`.

---

## Phase 1: Final Progress Rewrite

**Goal:** Write the final version of `progress.md` reflecting the completed health check.

1. Count the final numbers from `plan.md` and the batch resolution logs:
   - **Resolved:** total findings across all batches with status `done`
   - **Won't Fix:** number of findings in the Won't Fix section of `plan.md`
   - **Deferred:** total findings across all batches marked `deferred` (zero if all batches completed)
   - **Total:** Resolved + Won't Fix + Deferred

2. Rewrite `progress.md` with this format:

```markdown
# Health Check Progress

**Completed:** YYYY-MM-DD

| Status | Count |
|--------|-------|
| Resolved | R |
| Won't Fix | W |
| Deferred | D |
| **Total** | **T** |

## Batch Status
| Batch | Theme | Type | Status |
|-------|-------|------|--------|
| 1 | [theme] | [mechanical/architectural] | done |
| 2 | [theme] | [mechanical/architectural] | done |
| 3 | [theme] | [mechanical/architectural] | deferred |
[repeat for all batches]
```

Use today's date for the `**Completed:**` value. Every batch must have status `done` or `deferred` — no batch should remain `next` or `pending`.

---

## Phase 2: Present Summary

**Goal:** Give the user a clear picture of what the health check accomplished.

Present the following in the conversation:

1. **Final counts:**
   > Resolved: R / Won't Fix: W / Deferred: D / Total: T

2. **Won't-fix items with rationale** — read these from the Won't Fix section of `plan.md`. List each finding number, short description, and the rationale. If none, say "None."

3. **Deferred items** — if any batches were deferred, list each deferred batch with its theme and the findings it contained. If none, say "None."

4. **Commits produced** — read the batch resolution logs (`batch-*.md` files in the health-check directory) and list each commit hash and its message. Present as a list:
   > - `abc1234` — fix: import cleanup (#3, #5, #7)
   > - `def5678` — refactor: App.tsx decomposition (#12)

5. **Health-check directory path** — the full path to the health-check directory for future reference.

---

## Phase 3: Commit

**Goal:** Commit the final progress update.

1. Stage `progress.md` (and any other health-check metadata files that were modified).
2. Commit with this message format:
   > `docs: complete health check — N resolved, M won't fix`

   Where N is the resolved count and M is the won't-fix count. If there are deferred items, append `, P deferred` to the message.

---

## Terminal State

None. The health check is done. There is no next skill to invoke.

---

## Hard Gates Summary

| Gate | Rule |
|------|------|
| G1 | Do NOT complete silently if batches remain — present and ask |
