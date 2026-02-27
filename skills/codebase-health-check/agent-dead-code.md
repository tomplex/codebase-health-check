---
name: agent-dead-code
description: Agent prompt for detecting unused exports, unreachable code, orphan files, and stale dependencies.
---

# Dead Code Analysis

You are analyzing a codebase for dead code. Dead code is any code that exists in the source but is never executed, never referenced, or serves no functional purpose. Dead code increases maintenance burden, confuses developers, and bloats the codebase.

Systematically work through every category below. For each category, actively search the codebase using the techniques described in the "How to Search" section.

---

## Categories to Check

### 1. Unused Exports & Declarations

- **Exported but never imported:** Functions, classes, constants, types, or interfaces that are exported from a module but never imported by any other file in the codebase. Pay special attention to named exports — default exports can be harder to trace.
- **Declared but never referenced:** Variables, functions, or classes declared in a file that are never used within that file or exported for use elsewhere.
- **Unused function parameters:** Parameters defined in a function signature that are never read or referenced in the function body. Ignore parameters that must exist for positional reasons (e.g., middleware `(req, res, next)` where only `next` is used) or to satisfy an interface contract.

### 2. Unreachable Code

- **Code after terminal statements:** Any code appearing after an unconditional `return`, `throw`, `break`, or `continue` statement within the same block. The statements below the terminal statement can never execute.
- **Impossible branches:** Conditional branches that can never be true given the logic. Examples: an `else` after a condition that is always true, a `case` in a switch that is unreachable, type checks that can never match.
- **Dead catch blocks:** `catch` blocks that handle exceptions which cannot be thrown by the code in the corresponding `try` block.

### 3. Commented-Out Code

- **Disabled code blocks:** Large blocks of commented-out code that are clearly disabled source code, not explanatory comments. Look for commented-out function definitions, class declarations, import statements, or multi-line logic blocks.
- **Stale TODO/FIXME references:** Comments containing TODO or FIXME that reference "temporarily" disabled code, especially if the surrounding git history or date references suggest the comment is old. These signal code that was meant to be re-enabled but never was.

Do NOT flag normal explanatory comments, documentation comments, or single-line clarifications. Only flag commented-out executable code.

### 4. Unused Dependencies

- **Unused imports:** Import statements at the top of a file where the imported name is never referenced in the rest of that file. Check for both named and default imports.
- **Phantom package dependencies:** Packages listed in manifest files (`package.json`, `requirements.txt`, `Cargo.toml`, `go.mod`, `Gemfile`, `pyproject.toml`, `build.gradle`, etc.) that are never imported or required anywhere in the source code. Be careful to check all common import patterns for the language.
- **Unused dev dependencies:** Dev/test dependencies listed in the manifest that are never imported in any test file, config file, or build script.

### 5. Stale Feature Flags & Configuration

- **Constant feature flags:** Feature flags or toggle variables that always evaluate to the same value (e.g., `const ENABLE_NEW_UI = true` set once and never changed, with no mechanism to toggle it). These indicate the flag has served its purpose and the conditional branches around it are dead.
- **Write-only configuration:** Configuration values that are set or defined but never read by any code. Check for config keys in config files that have no corresponding lookup in the source.
- **Orphan environment variables:** Environment variables referenced in config or `.env.example` files that are never actually read in the application code (no `process.env.X`, `os.environ["X"]`, or equivalent).

### 6. Dead Files & Modules

- **Orphan files:** Source files that are never imported, required, or referenced by any other file. These are entire modules that serve no purpose. Exclude entry points (e.g., `main`, `index`, `app`) and files that are loaded by convention (e.g., migration files, test files, config files).
- **Empty or stub modules:** Files that exist but contain no meaningful logic — only empty class bodies, pass-only functions, or placeholder comments.

---

## How to Search

Follow this systematic approach. Do not skip steps.

### Step A: Map the Codebase Structure

1. List all source files within the analysis scope using glob patterns appropriate for the detected languages.
2. Identify the primary languages and their import/require conventions.
3. Locate manifest files (package.json, requirements.txt, Cargo.toml, go.mod, etc.).

### Step B: Check for Orphan Files

1. For each source file, search the entire codebase for imports or references to that file's module name.
2. A file is an orphan if nothing imports from it AND it is not an entry point, config file, migration, or test file.
3. Focus on files in library/utility directories — these are most likely to become orphaned.

### Step C: Check Exports vs. Imports

1. For files that export symbols, grep the codebase for each exported name.
2. If an exported name appears only in its own file (the export declaration itself), it is unused.
3. Prioritize files with many exports — they are more likely to have unused ones.

### Step D: Check Package Manifests

1. Read the manifest file and extract all dependency names.
2. For each dependency, search the codebase for import/require statements that reference it.
3. Distinguish between runtime dependencies, dev dependencies, and peer dependencies. Check each against the appropriate file set (source files for runtime, test/build files for dev).

### Step E: Scan for Commented-Out Code

1. Search for multi-line comment blocks that contain code-like patterns: function calls, assignments, control flow keywords, import statements.
2. Use patterns like `// function`, `// if (`, `// import`, `/* ... return ...*/`, or language-appropriate equivalents.
3. Ignore single-line comments that are clearly explanatory.

### Step F: Scan for Unreachable Code

1. Search for patterns like `return ...;\n  ...` or `throw ...;\n  ...` where code follows a terminal statement at the same block level.
2. Look for feature flag constants and trace their usage in conditionals. If a flag is always `true`, the `else` branch is dead. If always `false`, the `if` branch is dead.

---

## Severity Guide

Use these severity levels when reporting findings:

### Critical

- **Entire dead files or modules** — source files that nothing imports from and are not entry points or convention-loaded files.
- **Dead feature flag branches** — large blocks of code guarded by a flag that always evaluates to the same value, meaning one entire branch is dead.

### Important

- **Unused exported functions, classes, or types** — public API surface that is never consumed.
- **Phantom package dependencies** — dependencies in the manifest that nothing in the codebase uses (unnecessary install and supply chain risk).
- **Large blocks of commented-out code** — more than ~10 lines of disabled code.
- **Unused dev dependencies** — test tooling listed but never used.

### Minor

- **Unused variables or function parameters** — small-scale dead declarations.
- **Single unused import statements** — one import line that is not referenced.
- **Short commented-out code** — a few lines of disabled code.
- **Unreachable code after a return/throw** — typically a few lines that cannot execute.
- **Orphan environment variables** — env vars defined but never read.
