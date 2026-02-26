---
description: "Resume resolving findings from an existing health-check report."
disable-model-invocation: true
---

Find the most recent health-check directory under docs/health-checks/.

The user's arguments are: ${ARGUMENTS}
If arguments were provided, use them as the path to the health-check directory.

Read the directory contents to determine state:
- If no plan.md exists → invoke codebase-health-check:triage with the directory path
- If plan.md exists and unresolved batches remain in progress.md → invoke codebase-health-check:resolve-batch with the directory path
- If all batches are resolved → invoke codebase-health-check:complete with the directory path
