# Codebase Health Check

A Claude Code plugin that performs deep structural analysis of your codebase and then guides you through fixing what it finds.

## How it works

Run `/health-check` and the plugin dispatches 7 specialized agents in parallel, each scanning for a different category of structural issue: dead code, complexity hotspots, DRY violations, confusing implementations, extensibility concerns, inconsistent patterns, and naming/organization problems. Results are deduplicated, severity-ranked, and written to a report.

That's where most tools stop. This one keeps going.

After the analysis, the plugin offers to triage the findings — classifying each as mechanical (safe for automated fixes) or architectural (needs design discussion), identifying which findings are foundational (must be done first because others depend on them), and proposing won't-fix candidates where the cost exceeds the benefit. It groups everything into ordered batches and asks for your approval before touching anything.

Then it works through the batches one at a time. Mechanical fixes get dispatched to subagents in parallel. Architectural fixes get a design discussion — 2-3 approaches with trade-offs — before implementation. Every batch is verified, committed individually, and logged. If you run out of time, `/resolve` picks up exactly where you left off, even days later.

## Installation

```bash
claude plugin add /path/to/codebase-health-check
```

Or clone and add:

```bash
git clone git@github.com:tomplex/codebase-health-check.git
claude plugin add ./codebase-health-check
```

## Commands

- **`/health-check [scope]`** — Run analysis on a directory or the full repo. Produces a severity-ranked report and offers to start the resolution pipeline.
- **`/resolve [path]`** — Resume resolving findings from an existing report. Detects pipeline state automatically and picks up where you left off.

## The Pipeline

```
/health-check → analysis → triage → resolve-batch (repeat) → complete
```

### Analysis

Seven parallel agents scan for:

| Category | What it finds |
|----------|---------------|
| Dead Code | Unused exports, unreachable code, orphan files, phantom dependencies |
| Complexity | God functions, deep nesting, over-engineering, premature optimization |
| DRY Violations | Copy-pasted logic, parallel implementations, duplicated constants |
| Confusing Code | Misleading names, hidden side effects, non-obvious control flow |
| Extensibility | Tight coupling, hard-coded values, growing conditional chains |
| Inconsistencies | Mixed error handling, divergent patterns, style drift |
| Naming & Org | Misplaced files, junk-drawer modules, unclear boundaries |

Findings are deduplicated and written to `docs/health-checks/YYYY-MM-DD/report.md`.

### Triage

Classifies each finding along two axes:

- **Complexity:** Mechanical (renames, deletions, import cleanup — subagent-safe) vs. Architectural (decomposition, new abstractions — needs design)
- **Dependency:** Foundation (must be done first, other fixes depend on it) vs. Independent

Groups findings into ordered batches: foundations first, then mechanical by theme, then architectural individually. Identifies won't-fix candidates with cost-benefit rationale. You approve the plan before anything is touched.

Also asks for your fast verification command (something under a minute) which is used as the gate before and after every batch.

### Resolve

Executes one batch at a time:

- **Mechanical batches** — subagents fix each finding in parallel, then the main session runs verification independently (subagent reports are not trusted)
- **Architectural batches** — main session proposes 2-3 approaches, you pick one, then implementation proceeds incrementally

Each batch produces: one git commit, one batch log with verification evidence, and an updated progress tracker.

### Complete

Finalizes the health check with a summary of resolved, won't-fix, and deferred findings. Lists all commits produced.

## Output Structure

```
docs/health-checks/YYYY-MM-DD/
├── report.md           # Original findings (never modified)
├── plan.md             # Triage output: batches, won't-fix, verification command
├── progress.md         # Live progress tracker (updated after each batch)
├── batch-1-theme.md    # Batch 1 resolution log
├── batch-2-theme.md    # Batch 2 resolution log
└── ...
```

## Design Principles

This plugin encodes patterns discovered across real health-check sessions on production codebases:

- **Foundation-first ordering** — fix the things other fixes depend on before anything else
- **Theme-based batching** — group related fixes (all dead code together, all renames together) for coherent commits
- **Ceremony proportional to complexity** — mechanical fixes get subagents, architectural fixes get design discussion
- **Conservative won't-fix** — explicitly evaluate and document what you're choosing not to fix, with rationale
- **One commit per batch** — clean git history, easy to bisect or revert
- **Append-only resolution log** — batch files are written once, never edited; progress.md is the only file rewritten
- **Resumable at any point** — `/resolve` reads document state and routes to the right pipeline stage

## Dependencies

Works well with [Superpowers](https://github.com/obra/superpowers) but does not require it. The analysis pipeline is fully self-contained. The resolution pipeline uses its own triage and execution workflow rather than delegating to external skills.

## License

MIT
