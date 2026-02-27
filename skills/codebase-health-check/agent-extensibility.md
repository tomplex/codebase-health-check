# Extensibility Concerns Analysis

**Your task:** Find code structures that make it difficult to add new features without modifying working code in fragile ways — tight coupling, hard-coded values, growing conditional chains, missing extension points, and monolithic functions. Focus especially on areas the user identified as actively being extended. Search systematically using the methodology below and report findings with severity ratings.

Systematically work through every category below. For each category, actively search the codebase using the techniques described in the "How to Search" section.

**Focus especially on areas the user identified as actively being extended.** Extensibility concerns in dormant code are lower priority. Extensibility concerns in code the team is actively building on are the ones causing real pain right now.

---

## Categories to Check

### 1. Tight Coupling

- **Modules that directly reference internals of other modules:** Code that reaches into another module's private fields, internal data structures, or unexported helpers rather than going through a public API or interface. For example, one service directly accessing another service's database tables, a module importing a helper that was not intended as part of the public contract, or code that depends on the internal layout of another module's data structures (accessing nested fields like `order._internal.rawItems` instead of a public method). When the internal implementation changes, every consumer breaks.
- **Classes or functions that depend on concrete implementations rather than interfaces:** Code that instantiates or type-checks against specific concrete classes instead of depending on abstractions. For example, a function that takes a `PostgresDatabase` parameter instead of a `Database` interface, or a handler that imports and calls `StripePaymentProcessor` directly instead of working with a `PaymentProcessor` abstraction. Swapping the implementation (for a different provider, for testing, for a new environment) requires rewriting consumers rather than just providing a different implementation.
- **Changes in one module requiring cascading changes in many others:** A structural indicator of tight coupling. If adding a field to a data model, changing a function signature, or modifying a return type forces updates in 5+ other files spread across different modules, those modules are too tightly coupled to the one that changed. Look for modules with very high fan-out (many things depend on them) that also expose unstable or detailed interfaces.
- **Circular dependencies between modules:** Module A imports from module B, and module B imports from module A (directly or through a chain like A -> B -> C -> A). Circular dependencies mean neither module can be understood, tested, or modified in isolation. They often signal that the module boundaries are drawn incorrectly and that shared logic should be extracted into a separate module.

### 2. Hard-Coded Values

- **Values that should be configurable but are embedded in code:** URLs, API endpoints, port numbers, timeout durations, retry counts, page sizes, rate limits, queue names, bucket names, and other operational parameters written as literals in source code rather than loaded from configuration. These force a code change and redeployment for what should be a configuration adjustment. Look for string literals containing `http://`, `https://`, `localhost`, IP addresses, or domain names in non-test source files.
- **Environment-specific values not loaded from config:** Values that differ between development, staging, and production — database connection strings, service hostnames, feature toggle states, logging levels, credential references — that are hardcoded rather than read from environment variables, config files, or a secrets manager. These make it impossible to run the same code in a different environment without editing source files.
- **Business rules embedded in code that are likely to change:** Thresholds, rates, limits, categories, tier definitions, pricing rules, eligibility criteria, or other business logic expressed as literal values or inline conditionals rather than externalized into configuration, a rules engine, or at minimum a clearly labeled constants module. Examples: a tax rate written as `0.07` inline in a calculation, a free-shipping threshold of `50` buried in an if-statement, a list of supported countries hardcoded as an array literal in business logic. When the business rule changes, a developer must find and update every occurrence in the code.

### 3. Growing Conditional Chains

- **Switch statements or if/else chains that grow with each new case:** Conditional structures where every new variant of an entity (a new user role, a new payment method, a new notification channel, a new file format) requires adding another branch to an existing switch or if/else chain. These are magnets for merge conflicts and a sign that the code should dispatch through a registry, a strategy pattern, or polymorphism instead. Look for switch/if-else blocks with 5 or more cases, especially if git history shows the number of cases growing over time.
- **Type-checking conditionals that need updates for every new variant:** Code that uses `instanceof`, `typeof`, type tags, discriminated unions checked manually, or string-based type fields to branch on an object's type, where adding a new type requires finding and updating every such check across the codebase. For example, `if (shape.type === "circle") ... else if (shape.type === "square") ... else if (shape.type === "triangle") ...` scattered across multiple files. Each new shape requires updating every location.
- **Dispatch logic mapping strings or enums to handlers inline rather than through a registry or polymorphism:** Code that maps a string, enum, or type tag to a behavior using inline conditionals rather than a lookup table, registry, or method dispatch. For example, a function that converts a string action name to a handler function using a chain of `if (action === "create") ... else if (action === "update") ... else if (action === "delete") ...` instead of a handler map like `{ create: handleCreate, update: handleUpdate, delete: handleDelete }`. The inline approach requires modifying the dispatch function every time a new action is added.

### 4. Missing Extension Points

- **Places where new features require modifying existing code rather than adding new code (Open/Closed Principle violation):** Functions, classes, or modules that must be edited every time a new capability is added. The canonical sign is a file that appears in the diff of every feature branch — not because it is broken, but because it is the only place where new behavior can be wired in. Look for central orchestration functions, route registrations, factory functions, or initialization code that accumulates new entries with every feature.
- **Core logic that mixes policy with mechanism:** Code where the general-purpose mechanism (how to send a notification, how to process a payment, how to serialize data) is entangled with the specific policy (which notifications to send, what payment rules to apply, what format to use). This means you cannot change the policy without risking the mechanism, and you cannot reuse the mechanism with a different policy. Look for functions where business rules and infrastructure concerns are interleaved line by line rather than separated into distinct layers or passed in as parameters.
- **Functions that handle both the general case and special cases inline:** Functions where the mainline logic is interrupted by special-case handling for specific customers, legacy formats, one-off workarounds, or edge cases — all expressed as inline conditionals rather than extracted into separate handlers, decorators, or pre/post-processing hooks. Each new special case makes the function longer and harder to understand, and the special-case code cannot be tested independently of the general case.

### 5. Monolithic Functions

- **Large functions that would need splitting to add a new feature:** Functions spanning 50+ lines of logic that handle an entire workflow end-to-end. Adding a new step, a new branch, or a new output format to this workflow means modifying the monolith rather than composing a new piece. The function is too large to safely modify without understanding all of it, and too intertwined to extract a piece without refactoring the whole.
- **Functions mixing multiple responsibilities that would need partial reuse:** Functions that combine validation, business logic, data access, formatting, and side effects in a single body, where a new feature needs only one of those pieces but cannot call it in isolation. For example, a function that validates an order, calculates the total, saves to the database, and sends an email — and a new feature needs the calculation logic but not the save or the email. The developer must either duplicate the calculation or call the monolith and work around its side effects.
- **Entry-point functions that do everything instead of delegating to composable pieces:** Controller actions, CLI command handlers, API route handlers, or event listeners that contain all the logic for their operation inline rather than delegating to service functions, middleware, or composable steps. These functions cannot be reused, recombined, or extended without modification because their logic is not accessible outside the entry-point context.

---

## How to Search

Follow this systematic approach. Do not skip steps.

### Step A: Focus on User-Identified Active Areas

1. Start with the directories and files the user identified as areas they are actively extending or working in. These are where extensibility concerns cause the most pain.
2. Read the key files in these areas thoroughly. Understand the current architecture: how modules relate to each other, how new features are added, and where the boundaries are between components.
3. If the user identified pain points related to difficulty adding features, rigidity, or "having to touch too many files," prioritize investigating those specific areas.

### Step B: Examine Import and Dependency Patterns

1. For each major module or directory within the analysis scope, examine the import statements at the top of each file. Map which modules depend on which others.
2. Look for modules that are imported by a very large number of other files (high fan-in). These are central dependencies — if they are tightly coupled or expose unstable interfaces, changes to them cascade widely.
3. Search for circular imports. In JavaScript/TypeScript, look for pairs of files that import from each other. In Python, look for circular import workarounds (imports inside functions, lazy imports, `TYPE_CHECKING` guards). In other languages, check for bidirectional dependencies between packages or modules.
4. Search for imports that reach into internal or private paths: paths containing `internal`, `_private`, `impl`, or deeply nested submodule paths that bypass a module's public API (e.g., importing from `../../other-module/src/utils/internal-helper` instead of `other-module`).

### Step C: Search for Switch/If-Else Chains and Dispatch Logic

1. Search for `switch` statements, `match` expressions, and long `if/else if` chains (3+ branches). Read each one and determine whether it dispatches based on a type, category, role, action, or other discriminator that is likely to grow.
2. For each chain found, ask: "When a new variant is added (a new user type, a new file format, a new action), does this chain need a new branch?" If yes, it is a growing conditional chain.
3. Search for string comparisons that look like dispatch logic: patterns like `=== "create"`, `== "admin"`, `case "json"`, `case "csv"`. These often indicate inline dispatch that should be a registry.
4. If git is available, check git history on files with long conditional chains. Search the git log for commits that added new cases to existing switch/if-else blocks. A pattern of repeated additions confirms the chain is actively growing.

### Step D: Search for Hard-Coded Values

1. Search for URL patterns in source files: `http://`, `https://`, `localhost`, IP address patterns like `\d+\.\d+\.\d+\.\d+`, and port numbers like `:3000`, `:8080`, `:5432`, `:6379`.
2. Search for common configuration-like values that appear as literals: timeout numbers (`3000`, `5000`, `10000`, `30000`, `60000`), retry counts, page sizes, rate limits, and similar operational parameters.
3. Search for business-rule values embedded in logic: tax rates, discount thresholds, tier boundaries, category lists, and domain-specific magic numbers. Check whether these are defined as named constants in a configuration module or scattered as literals in business logic.
4. Look for environment-specific values: database hostnames, service URLs, API keys or key placeholders, region names, bucket names. Verify these are loaded from environment variables or config files rather than hardcoded.

### Step E: Look at Recent Feature Additions

1. If git is available, examine the last several feature-addition commits or pull requests. Look at the list of changed files. If adding a single feature required modifying 8+ files across multiple directories, that is a sign of tight coupling or missing extension points.
2. Look for files that appear in the diff of many different feature branches. These "always-modified" files are extensibility bottlenecks — they are the places where every new feature must be wired in.
3. Identify the typical pattern for adding a new feature to the codebase. Is there a clear, additive path (add a new file that implements an interface and register it)? Or does it require modifying existing code in multiple places (add a new case to the switch, update the factory, add a new route, update the type union)?

### Step F: Check for Missing Abstractions and Extension Points

1. Search for factory functions, builder functions, or initialization code that contains long lists of registrations or case-by-case construction. These are often the places where new features are wired in and where an extension point (plugin registry, auto-discovery, convention-based loading) would help.
2. Look for functions that mix infrastructure mechanism with business policy. Signs include: database queries interspersed with business rule checks, HTTP response formatting mixed with domain logic, notification sending mixed with eligibility determination. These functions cannot be extended with new policies without modifying the mechanism.
3. Look for functions that handle special cases inline. Search for comments like `// special case`, `// workaround`, `// hack`, `// temporary`, `// for [specific customer]`, `// legacy`. These indicate ad-hoc extensions that should be factored into a proper extension mechanism.
4. Identify monolithic entry-point functions (controller actions, CLI handlers, main processing functions). Check whether they delegate to composable sub-functions or contain all logic inline. If inline, they are extensibility bottlenecks.

---

## Severity Guide

Use these severity levels when reporting findings:

### Critical

- **Tight coupling in areas being actively extended** — modules in the user-identified active areas that directly reference each other's internals, share concrete dependencies without interfaces, or require cascading changes across many files when modified. This is critical because it is actively impeding the team's current work — every feature they try to add forces them to understand and modify tightly coupled code, increasing the risk of regressions.
- **Monolithic functions in areas being actively extended** — large functions spanning 50+ lines of logic in the active areas that handle entire workflows inline. When the team needs to add a new step or new branch to the workflow, they must modify the monolith, which is risky and hard to test. If the function mixes multiple responsibilities, partial reuse is impossible without duplication.
- **Missing extension points in frequently modified code** — central functions or modules that appear in the diff of many feature branches because they are the only place new behavior can be wired in. Every developer working on a new feature must modify the same file, causing merge conflicts and requiring everyone to understand everyone else's changes.

### Important

- **Hard-coded values that will need to change** — operational parameters (URLs, timeouts, limits) or business rules (rates, thresholds, tier definitions) embedded as literals in source code rather than in configuration. These are important because they are certain to change — business rules change with business needs, and operational parameters change with scale and environment — and when they do, someone must find every occurrence in the code.
- **Growing conditional chains with 5+ cases** — switch statements or if/else chains with 5 or more branches that dispatch based on a discriminator (type, role, action, format) that is likely to keep growing. Each new case added to the chain increases the risk of bugs in existing cases and makes the function harder to understand. If git history shows the chain has grown over time, this confirms it is actively compounding.
- **Circular dependencies between modules** — modules that import from each other directly or through a chain. Circular dependencies prevent independent testing, complicate refactoring, and often cause subtle initialization-order bugs. They indicate that module boundaries need to be redrawn.
- **Type-checking conditionals scattered across multiple files** — `instanceof`, `typeof`, or type-tag checks for the same set of types appearing in multiple locations. Adding a new type requires finding and updating every location, which is error-prone and scales poorly.

### Minor

- **Theoretical extensibility concerns in code that is not actively changing** — tight coupling, hardcoded values, or missing extension points in modules that the team does not currently work in and has no near-term plans to extend. These are real design issues but are not causing active pain. They should be noted but prioritized below concerns in active areas.
- **Small conditional chains (3-4 cases) in stable code** — short switch or if/else chains in code that is not actively gaining new cases. These are technically extensibility concerns but are not yet a burden.
- **Mildly monolithic functions in peripheral code** — functions in non-core areas that are longer than ideal and would benefit from decomposition but are not currently being extended or modified.
- **Hard-coded values that are unlikely to change soon** — literals that should theoretically be configurable but represent values that are stable and unlikely to change in the near term (e.g., mathematical constants, well-established protocol defaults).
