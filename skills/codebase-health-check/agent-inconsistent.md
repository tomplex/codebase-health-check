# Inconsistent Patterns Analysis

**Your task:** Find places where the same concern is handled differently across the codebase without justification — mixed error handling strategies, inconsistent API styles, divergent coding patterns, and conventions that most modules follow but a few violate. Search systematically using the methodology below and report findings with severity ratings.

Systematically work through every category below. For each category, actively search the codebase using the techniques described in the "How to Search" section.

**Important distinction:** Some inconsistency is intentional and appropriate. Different parts of a codebase may have genuinely different needs — a high-performance data pipeline may use a different error strategy than a user-facing API layer, or a legacy module may follow older conventions while newer modules follow updated ones. Only flag inconsistency where:

- The same problem is being solved in the same context with no reason for the approaches to differ.
- A clear convention exists in the majority of the codebase but is not followed everywhere.
- The inconsistency would confuse a new developer trying to understand how the codebase handles a particular concern.

If two modules use different approaches because they have genuinely different requirements (e.g., a CLI tool using result types for user-facing errors and an internal library using exceptions for programmer errors), that is an intentional design choice, not a finding.

---

## Categories to Check

### 1. Mixed Error Handling

- **Different error handling mechanisms across modules doing the same kind of work:** Some modules use try/catch with thrown exceptions, others return result types or error tuples, others use callbacks with error parameters — all for the same class of operation (e.g., database access, API calls, input validation). When a developer moves from one module to another, they must discover which error mechanism is in use before they can safely call any function. Look for modules at the same architectural layer (all services, all controllers, all repositories) that handle errors differently from each other.
- **Inconsistent error message formats:** Some errors include error codes, others include only a message string, others include a stack trace or context object. Some error messages are user-facing ("Invalid email address"), others are developer-facing ("VALIDATION_ERR: field 'email' failed regex check"), and these two styles are mixed without separation. When error messages are logged or displayed, the inconsistency makes them harder to parse, search, and monitor.
- **No clear error propagation policy:** Some functions log the error and return a default value, others re-throw the error for the caller to handle, others swallow the error silently, others wrap the error in a new error type and throw that. Within the same module, adjacent functions may use different strategies. There is no discernible rule for when to log-and-continue versus when to propagate, meaning a developer cannot predict whether a called function will throw or silently fail.
- **Inconsistent error types or error class hierarchies:** Some parts of the codebase define custom error classes (e.g., `NotFoundError`, `ValidationError`), while other parts throw generic `Error` objects with a message string, and still others use string error codes or numeric status codes. Catching and handling errors becomes unreliable because the consumer cannot use a consistent mechanism (catch by type, check by code, pattern-match on result) across the codebase.

### 2. Inconsistent API Styles

- **Different naming conventions across modules at the same level:** Some modules use camelCase for function names, others use snake_case, others use PascalCase for non-class identifiers. Variable naming follows different conventions in different files: some use abbreviated names (`usr`, `cfg`, `req`), others use full names (`user`, `config`, `request`). File naming may also diverge: some files use kebab-case (`user-service.ts`), others use camelCase (`userService.ts`), others use PascalCase (`UserService.ts`). When conventions differ within the same layer or directory, it signals a lack of agreed standards.
- **Inconsistent parameter ordering across similar functions:** Some functions take `(options, callback)`, others take `(callback, options)`, others take `(data, options, callback)`, and still others take `(options)` and return a promise. Functions that serve the same purpose in different modules accept their parameters in different orders, forcing callers to check the signature every time rather than relying on a consistent convention. Similarly, some functions use positional parameters while parallel functions use a single options object for the same set of values.
- **Different return type conventions for the same semantic situation:** Some functions return `null` or `undefined` when an item is not found, others throw an exception, others return a result object with an `error` field, others return an empty array or a sentinel value like `-1`. The caller must know which convention each specific function uses, and using the wrong assumption (e.g., checking for `null` when the function actually throws) causes bugs. Look especially for `getById`/`findById`-style functions across different modules and compare how they handle the not-found case.
- **Mixed async patterns:** Some modules use callbacks, others use Promises with `.then()`, others use `async/await`, and some mix two or more styles within the same file. Functions that perform the same kind of async operation (database queries, HTTP requests, file I/O) use different async mechanisms in different modules, making it impossible to compose them without adapter code.

### 3. Mixed Coding Styles

- **Different formatting patterns within the same module or directory:** Inconsistent indentation (tabs in some files, spaces in others, or different indentation widths), inconsistent brace placement (opening brace on same line in some functions, next line in others within the same file), inconsistent use of semicolons (present in some files, omitted in others in languages where they are optional), and inconsistent string quoting (single quotes in some files, double quotes in others). These may seem cosmetic but they signal a lack of shared tooling or configuration and make diffs noisy when different developers auto-format differently.
- **Inconsistent use of language features across files at the same level:** Some files use modern syntax (`const`/`let`, arrow functions, destructuring, template literals, `async/await`, optional chaining) while other files in the same directory use legacy syntax (`var`, `function` declarations, string concatenation, callbacks, manual null checks). This is not about a gradual migration (where old code is being updated over time) but about new code being written in different styles depending on which developer wrote it. Look for recently-modified files that still use patterns abandoned elsewhere.
- **Mixed paradigms without clear boundaries:** Some modules are written in an object-oriented style (classes with methods, inheritance hierarchies, encapsulated state), others in a functional style (pure functions, immutable data, composition), and others in a procedural style (scripts with sequential logic, global state, imperative loops) — all within the same layer of the application, with no architectural rationale for the difference. When a developer needs to add a feature, they cannot predict whether to write a class, a function, or a procedural script because the existing code gives conflicting signals.
- **Inconsistent module structure:** Some modules export a single default class, others export a bag of named functions, others export a singleton instance, others export a factory function. Some modules have their tests alongside the source, others have tests in a separate tree. Some modules have an `index` file that re-exports, others are imported by direct file path. The structural inconsistency means a developer cannot rely on convention to navigate unfamiliar modules.

### 4. Pattern Divergence

- **A clear pattern in most modules but violated in a few:** The majority of the codebase follows a recognizable convention (e.g., all services use dependency injection, all API responses follow a standard envelope format, all database access goes through a repository layer), but a handful of modules deviate from that pattern with no documented reason. These outliers are confusing because a developer who has learned the convention will assume it applies everywhere, and the exception will cause incorrect assumptions.
- **Utilities or helpers that exist but are not used everywhere they should be:** A shared utility function or module exists for a specific concern (logging, error wrapping, input sanitization, date formatting, response building), but some modules use it and others implement the same concern inline or with their own local version. The shared utility was created to standardize the approach, but adoption was incomplete. Look for utility modules and then search for modules that implement the same concern without using the utility.
- **Conventions established by example but not consistently followed:** The first module written established a pattern (file structure, naming, how to register routes, how to define models, how to write tests), and most subsequent modules follow it, but some do not. This is common in growing codebases where patterns are communicated through example rather than enforced through tooling or documentation. Look for the most common structural pattern and identify outliers.
- **Inconsistent test patterns:** Some modules have comprehensive unit tests with descriptive names, others have sparse integration tests, others have no tests. Test file naming varies (`*.test.ts`, `*.spec.ts`, `*_test.go`, `test_*.py`) within the same project. Test setup patterns differ (some use factories, others build objects inline, some use shared fixtures, others are fully self-contained). The inconsistency makes it unclear what level of testing is expected and how new tests should be written.

---

## How to Search

Follow this systematic approach. Do not skip steps.

### Step A: Map the Codebase Structure and Identify Peer Modules

1. List all source files within the analysis scope using glob patterns appropriate for the detected languages.
2. Identify modules, services, controllers, handlers, and other components that serve parallel purposes. These are the peer groups you will compare. Files in the same directory or at the same architectural layer (all controllers, all services, all repositories, all handlers) are the most important peers.
3. Note the most common file naming pattern, directory structure, and export style. This establishes the baseline convention.

### Step B: Compare Error Handling Across Peer Modules

1. Search for error handling patterns across the codebase: `try`, `catch`, `throw`, `throws`, `raise`, `rescue`, `except`, `Result`, `Err`, `Ok`, `Either`, `Left`, `Right`, `.catch(`, `on('error'`, `callback(err`, `(err, `, `if err !=`, `if (error)`. Read the surrounding code to understand the mechanism in use.
2. For each peer group (e.g., all service files, all controller files), compare how errors are handled. Note which mechanism each module uses: try/catch, result types, callbacks with error parameters, or something else.
3. Search for error creation patterns: `new Error(`, `new CustomError(`, `createError(`, `AppError(`, `throw new`, `raise `. Compare the error formats across modules: do they include error codes? Status codes? Context objects? Are the message formats consistent?
4. Search for catch blocks and examine what each one does with the caught error: re-throws it, logs and re-throws, logs and swallows, wraps and re-throws, returns a default value. Compare these strategies across peer modules to identify inconsistency in error propagation policy.

### Step C: Compare Naming and API Conventions Across Peer Modules

1. For each peer group, compare function and method names. Note whether they use camelCase, snake_case, PascalCase, or a mix. Check whether abbreviation usage is consistent (do all modules say `config` or do some say `cfg` and others `configuration`?).
2. Compare the parameter signatures of functions that serve analogous purposes across different modules. Do they take parameters in the same order? Do they use positional parameters or options objects? Do parallel functions have consistent arity?
3. Compare return type conventions for analogous operations. For "get" or "find" functions, what happens when the item is not found? Compare across modules: null, undefined, throw, empty result, result type. For "create" or "save" functions, what is returned: the created object, a boolean, void, a result type?
4. Compare async patterns. Search for callback-style function signatures `(err, result)`, Promise chains `.then(`, `async/await` patterns. Note which pattern each module uses for the same kind of operation (database access, HTTP calls, file I/O).

### Step D: Compare Coding Style and Language Feature Usage

1. Check formatting consistency across files in the same directory: indentation (tabs vs. spaces, width), brace style, semicolon usage, string quoting style. You do not need to exhaustively audit every file — sample several files from each directory and compare them.
2. Look for language feature inconsistency by searching for legacy patterns alongside modern equivalents. For example, in JavaScript/TypeScript: search for `var ` alongside `const `/`let `, `function(` alongside `=>`, string concatenation with `+` alongside template literals. In Python: search for `%` string formatting alongside `f"` strings alongside `.format(`. Note whether old and new patterns coexist in the same directories or same files.
3. Compare paradigm usage across peer modules. Are some modules class-based while their peers are function-based? Do some use inheritance where peers use composition? Do some use mutable state where peers use immutable patterns? Note whether the paradigm differences align with architectural boundaries (acceptable) or cut across them (inconsistent).

### Step E: Identify Established Patterns and Their Exceptions

1. After reading several modules in each peer group, identify the dominant pattern — the convention that the majority follow. This is the baseline.
2. Search for exceptions to the dominant pattern. For each exception, read the outlier module and determine whether the deviation is justified by a different requirement or whether it appears to be accidental (e.g., written by a different developer who was unaware of the convention, or written before the convention was established and never updated).
3. Search for shared utility modules: files in directories named `utils`, `helpers`, `common`, `shared`, `lib`, `core`. Read what each utility provides. Then search the codebase for modules that implement the same concern inline instead of using the shared utility. For example, if a `formatError` utility exists, search for modules that format errors without importing it.
4. Compare test patterns across modules. Check test file naming, test structure (describe/it vs. test functions vs. class-based test cases), setup patterns (factories vs. inline construction vs. fixtures), and assertion styles. Note divergences from the most common pattern.

### Step F: Check for Mixed Async Patterns and Module Structures

1. For each module, note the async pattern used for I/O operations. Search for modules where different async patterns are used for the same kind of operation and compare them.
2. Compare module export patterns across peer files. Are some exporting a default class while peers export named functions? Are some exporting singleton instances while peers export factory functions?
3. Check for consistent use of index/barrel files. Do some directories have them and others do not? Do some modules require importing specific files while peers allow importing from the directory?

---

## Severity Guide

Use these severity levels when reporting findings:

### Critical

- **Core patterns inconsistent across the codebase** — error handling, data access, or request/response patterns that differ fundamentally between modules at the same architectural layer. When error handling is try/catch in half the services and result types in the other half, every cross-module call is a potential bug because the caller may use the wrong error handling style. When data access patterns differ (some modules go through the repository, others query the database directly), there is no single place to add cross-cutting concerns like caching, logging, or access control. These inconsistencies affect every developer on every task because they cannot rely on any convention holding across the codebase.
- **Mixed return type conventions for the same operation across public interfaces** — when "get" functions return null in some modules and throw in others, or when "create" functions return the created entity in some modules and a boolean in others, the inconsistency in public interfaces means every caller must check the specific function's behavior, and getting it wrong causes silent bugs (unchecked null) or unhandled exceptions.

### Important

- **Clear conventions violated in several places** — a recognizable pattern that 70-80% of the codebase follows but 20-30% deviates from without justification. The convention is strong enough that developers will assume it holds everywhere, but the exceptions are numerous enough to cause real confusion and bugs. Examples: most modules use the shared logger but several use `console.log` directly, most API responses follow an envelope format but several return raw data, most services are injected but several are imported directly.
- **Mixed API styles in public interfaces** — inconsistent parameter ordering, inconsistent naming conventions, or inconsistent use of positional vs. options-object parameters across functions that developers call frequently. Every caller must look up the specific function's signature rather than relying on convention, which slows development and invites mistakes.
- **Mixed async patterns across peer modules** — some modules using callbacks while peers use async/await for the same kind of operation, forcing callers to use adapter code or remember which style each module uses.
- **Shared utilities that exist but are not used consistently** — a utility was created to standardize an approach but adoption is incomplete, so some modules use the standard utility and others implement the same concern ad hoc. The partial adoption is worse than having no utility at all because developers cannot tell which approach is current.

### Minor

- **Minor style inconsistencies** — formatting differences (tabs vs. spaces, brace style) across files, inconsistent string quoting, or minor naming convention differences that are cosmetic rather than semantic. These are worth noting because they indicate a lack of shared formatting tooling, but they do not cause bugs.
- **Occasional pattern divergence in non-core modules** — one or two modules deviating from an otherwise consistent pattern in peripheral or rarely-touched code. The deviation is real but its impact is low because few developers encounter it.
- **Inconsistent test patterns in stable code** — test files that follow different structural conventions but all provide adequate coverage. The inconsistency makes it harder to write new tests but does not affect runtime behavior.
- **Legacy syntax in old files that are rarely modified** — files that use older language features or conventions because they predate the current standard and have not been updated. If the files are stable and rarely touched, the inconsistency is a minor nuisance rather than an active problem.
