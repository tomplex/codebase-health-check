# codebase-health-check

Analyze codebases for structural health issues and guide resolution through an interactive pipeline.

## Commands

- `/health-check [scope]` — Run a full analysis across 7 categories, producing a severity-ranked report
- `/resolve [path]` — Resume resolving findings from an existing health-check report

## Pipeline

```
/health-check → analysis → triage → resolve-batch (repeat) → complete
```

1. **Analysis** — 7 parallel agents scan for dead code, complexity, DRY violations, confusing implementations, extensibility concerns, inconsistent patterns, and naming/organization issues. Results are aggregated into `docs/health-checks/YYYY-MM-DD/report.md`.
2. **Triage** — Classify findings as mechanical vs. architectural, group into ordered batches, identify won't-fix candidates. Produces `plan.md` and `progress.md`.
3. **Resolve** — Execute batches one at a time. Mechanical batches use parallel subagents; architectural batches get interactive design discussion. Each batch is verified and committed individually.
4. **Complete** — Finalize progress tracking and present a summary of all resolved, won't-fix, and deferred findings.

Use `/resolve` to resume at any point — it detects the current pipeline state and picks up where you left off.
