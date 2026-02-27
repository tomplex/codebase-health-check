---
description: "Resume resolving findings from an existing health-check report. Usage: /resolve or /resolve docs/health-checks/2026-02-27/"
disable-model-invocation: true
---

Find the most recent health-check directory under docs/health-checks/.

The user's arguments are: ${ARGUMENTS}
If arguments were provided, use them as the path to the health-check directory.

Read the directory contents to determine state:
- If no plan.md exists → invoke health-check:triage with the directory path
- If plan.md exists and unresolved batches remain in progress.md → invoke health-check:resolve-batch with the directory path
- If all batches are resolved → invoke health-check:complete with the directory path
