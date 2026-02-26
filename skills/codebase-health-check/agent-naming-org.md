# Naming & Organization Analysis

You are analyzing a codebase for naming and organizational issues. Poor naming and organization means files, directories, and modules are named misleadingly, placed in the wrong location, or structured in ways that make the codebase difficult to navigate. When the project's physical structure does not match its logical structure, developers waste time searching for code, put new code in the wrong place, and build incorrect mental models of the system. Naming and organization problems compound over time — each misplaced file or misleading name makes the next developer slightly more likely to repeat the mistake.

Systematically work through every category below. For each category, actively search the codebase using the techniques described in the "How to Search" section.

---

## Categories to Check

### 1. Misplaced Files

- **Files that logically belong to one module/feature but live in another's directory:** A file implementing authentication logic sitting inside the billing directory, or a payment-related utility located in the user profile module. The file's contents have no logical relationship to the directory it lives in. This happens when developers create a file in whatever directory they have open rather than thinking about where it belongs.
- **Shared utilities buried inside a specific feature's directory:** A general-purpose helper function (date formatting, string normalization, retry logic, HTTP client wrapper) that is used by multiple features but lives inside one feature's subdirectory. Other features must reach into that feature's internals to use the utility, creating a misleading dependency relationship. The utility should live in a shared or common location.
- **Test files not co-located with or clearly mapped to their source files:** Test files that are difficult to match to the source they test. This includes test files in a completely separate directory tree with no naming convention linking them to source files, test files whose name does not correspond to any source file, and test files that test code from multiple unrelated modules combined into a single file.
- **Configuration files scattered in unexpected locations:** Config files (environment configs, build configs, service-specific configuration) placed in directories where developers would not think to look for them. For example, a database configuration file inside a UI component directory, or deployment configuration buried inside a library subdirectory rather than at the project root or in a dedicated config directory.

### 2. Unclear Module Boundaries

- **Feature A's logic importing deeply from feature B's internals:** Import paths that reach deep into another feature's internal directory structure rather than importing from that feature's public API or index file. Paths like `import { helper } from '../../featureB/internal/utils/privateHelper'` signal that module boundaries are not being respected. Each feature should expose a well-defined surface area, and other features should not need to know about its internal file organization.
- **No clear separation between modules — everything cross-references everything:** A codebase where every module imports from every other module, creating a tangled dependency graph. There is no layering, no hierarchy, and no way to change one module without potentially affecting all others. You can detect this by examining import statements across files — if feature directories routinely import from three or more other feature directories, the boundaries are unclear.
- **Shared state between modules that should be independent:** Modules that communicate through shared mutable state (global variables, singleton objects, shared caches, or module-level mutable variables) rather than through explicit interfaces. This creates invisible coupling — two modules that appear independent are actually tightly linked through state they both read and write. Changes to the shared state in one module can silently break the other.
- **"Common", "shared", or "utils" directories that have become dumping grounds:** Directories named `common`, `shared`, `utils`, `helpers`, `lib`, or `misc` that have accumulated dozens of unrelated files with no internal organization. These directories start with a reasonable purpose — holding genuinely shared code — but gradually become the default location for any file a developer does not know where to put. The result is a directory with 20, 50, or 100 files covering unrelated concerns, making it as hard to navigate as having no organization at all.

### 3. Poor File Naming

- **Generic names that tell you nothing about the file's contents:** Files named `utils.ts`, `helpers.py`, `common.js`, `misc.rb`, `functions.go`, `stuff.ts`, `data.py`, `logic.js`, `core.ts`, or `tools.py`. These names describe the syntactic category of the contents (they are utilities, they are functions) without describing the domain or purpose. A developer looking for the email validation logic has no way to know whether it is in `utils.ts`, `helpers.ts`, or `common.ts` without opening all of them.
- **Files that have grown into junk drawers of unrelated functions:** A single file that contains a date formatting function, a URL parser, an array deduplication helper, a retry wrapper, and a currency formatter. The file started with one or two related functions and grew by accretion as developers added "just one more utility." The functions in the file share no conceptual relationship — the only thing they have in common is that someone did not know where else to put them. These files are particularly harmful when they grow to hundreds of lines, because they are frequently imported but painful to read.
- **File names that do not match their primary export or purpose:** A file named `parser.ts` whose primary export is a `Validator` class. A file named `userService.py` that primarily contains order-related logic because the scope crept. A file named `config.js` that also contains application bootstrapping, server setup, and middleware registration. The file name creates an expectation that the contents violate.
- **Inconsistent naming schemes across the project:** Some files use `kebab-case` (`user-profile.ts`), others use `camelCase` (`userProfile.ts`), others use `PascalCase` (`UserProfile.ts`), and others use `snake_case` (`user_profile.ts`) — all within the same project and the same language. Similarly, inconsistent suffixes: some files use `Controller`, others use `Handler`, others use `Manager` for the same architectural role. Inconsistent naming forces developers to guess the file name every time they need to import something.

### 4. Misleading Names

- **Directory names that do not reflect their contents:** A directory named `auth` that also contains user profile management, preference storage, and notification settings — anything tangentially related to users was filed under `auth`. A directory named `api` that also contains database models, business logic, and utility functions. A directory named `legacy` that contains actively-maintained, critical-path code. The directory name creates a scope that its actual contents have overflowed.
- **Module names that were accurate when created but no longer are (scope creep):** A module named `emailNotifications` that now also handles SMS, push notifications, and in-app messaging. A module named `csvExporter` that now supports JSON, XML, and PDF export. A module named `adminPanel` that now serves both admin and regular user dashboards. The name was correct at creation time, but the module's scope expanded without the name being updated. New developers working from the name alone will underestimate what the module does.
- **Names that imply a narrower scope than what the code actually does:** A function named `formatDate` that also performs timezone conversion, locale detection, and caching. A class named `Logger` that also handles metrics collection, error reporting, and crash analytics. A file named `constants.ts` that also contains utility functions, type definitions, and configuration logic. The name suggests a focused, single-purpose component, but the implementation is significantly broader.

### 5. Missing Structure

- **Flat directory structures with too many files at one level:** Directories containing 20 or more files with no subdirectory grouping. A `src/` directory with 40 files at the top level — models, services, controllers, utilities, types, and constants all mixed together. The developer must scan past dozens of irrelevant files to find the one they need. Flat structures also provide no visual indication of which files are related to each other.
- **No logical grouping of related functionality:** Related files scattered across the directory with no grouping convention. The user model is in one directory, the user service in another, the user controller in a third, and the user validation in a fourth. There is no way to see all the files related to "users" at a glance. Alternatively, a feature is spread across many files at the same directory level with no subdirectory to group them.
- **Missing index or barrel files where they would help organize exports:** Modules that contain multiple internal files but force importers to know the exact internal file path to import from. Every consumer must write `import { X } from './feature/internal/specific-file'` instead of `import { X } from './feature'`. The absence of an index file means the module has no defined public API — everything is equally accessible, and consumers are coupled to the internal file structure.
- **No clear convention for where new features should go:** A project where existing features are organized in mutually inconsistent ways — some features are grouped by type (all controllers together, all models together), others are grouped by feature (all user-related files together), and others are just placed at the project root. A new developer adding a feature has no template to follow and will default to whatever feels convenient, further degrading the structure.

---

## How to Search

Follow this systematic approach. Do not skip steps.

### Step A: Map the Top-Level Directory Structure

1. List the top-level directories and files in the project root and the primary source directory (e.g., `src/`, `app/`, `lib/`). Note the naming convention: is it by feature (`users/`, `billing/`, `auth/`), by type (`models/`, `controllers/`, `services/`), or inconsistent?
2. Identify any directories named `common`, `shared`, `utils`, `helpers`, `lib`, `misc`, `tools`, `core`, or `general`. These are the most likely locations for dumping-ground problems. List their contents and count their files.
3. Note whether the project follows a consistent organizational pattern. If some features are grouped by type and others by feature, record this as an inconsistency.

### Step B: Check for Directories with Too Many Files

1. For each directory, count the number of files at that level (not recursively). Flag any directory with 20 or more files at a single level.
2. For flagged directories, determine whether the files share a theme or are a grab bag. Read a sample of file names and check whether they belong to the same feature or are unrelated files that happen to share a location.
3. Check the primary source directory and any `utils`/`shared`/`common` directories first, as these are the most common locations for flat-structure problems.

### Step C: Examine Utils, Helpers, Common, and Misc Files

1. Search for files named `utils`, `helpers`, `common`, `misc`, `tools`, `functions`, `stuff`, or any other generic name across the codebase. Use glob patterns like `**/utils.*`, `**/helpers.*`, `**/common.*`, `**/misc.*`.
2. For each match, open the file and assess its coherence. Does the file have a clear theme (e.g., "string utilities" or "date helpers"), or is it a junk drawer of unrelated functions? Count the number of exported functions or definitions and note whether they belong to a single domain.
3. For junk-drawer files, check how many other files import from them. Widely-imported junk drawers are high-priority findings because they are encountered frequently.
4. Check the line count of these files. Generic-named files exceeding 200 lines are very likely to be junk drawers. Those exceeding 500 lines are almost certainly problematic.

### Step D: Analyze Import Paths for Boundary Violations

1. Search for import statements that reach into other features' internal directories. Look for import paths that traverse upward and then descend into a sibling feature's subdirectory (paths containing patterns like `../../featureX/internal` or `../../featureX/lib/private`).
2. Count how many cross-feature imports exist. If most features import from multiple other features' internal paths, the module boundaries are effectively nonexistent.
3. Check whether features expose index or barrel files. If a feature directory has an `index` file, check whether other features actually import from it or whether they bypass it and import from internal files directly.
4. Look for circular import patterns where feature A imports from feature B and feature B imports from feature A. These are a strong signal that the boundary between the two features is unclear or that shared code has not been extracted.

### Step E: Check File Naming Consistency

1. List all source files and examine their naming conventions. Categorize each file as `kebab-case`, `camelCase`, `PascalCase`, `snake_case`, or other. Count the prevalence of each style.
2. If the project uses multiple naming conventions, check whether there is a clear rule (e.g., `PascalCase` for classes, `kebab-case` for utilities). If there is no discernible rule and the same type of file uses different conventions, flag this as an inconsistency.
3. Look for inconsistent suffix conventions: files in the same architectural layer using different suffixes (`UserController` vs. `OrderHandler` vs. `ProductManager` for the same role).
4. Check whether file names match their primary export. For files that export a single class, function, or constant, compare the file name to the export name. Flag mismatches.

### Step F: Check for Misleading Directory and Module Names

1. For each feature or module directory, read a sample of the files inside. Determine whether all files relate to the directory's name or whether the directory has accumulated unrelated code.
2. Search for directories whose contents span multiple unrelated concerns. A directory named `auth` should contain authentication and authorization code — if it also contains user settings, notification preferences, or billing logic, the name is misleading.
3. Look for modules that have outgrown their name. Search for modules whose exported symbols or public API suggest a broader scope than the module name implies. For example, a module named `email` that exports `sendSMS`, `sendPushNotification`, and `sendEmail` has outgrown its name.

---

## Severity Guide

Use these severity levels when reporting findings:

### Critical

- **Module boundaries so unclear that developers regularly put code in the wrong place:** The codebase has no discernible organizational convention, features cross-reference each other's internals extensively, and there is no way for a new developer to determine where a new file should go. This causes ongoing structural decay — every new file placement is a coin flip.
- **Junk-drawer files exceeding 500 lines of unrelated functions:** A single file (often named `utils`, `helpers`, or `common`) containing 500+ lines of unrelated exported functions that is widely imported across the codebase. The file is impossible to navigate, impossible to name accurately, and accumulates more unrelated code over time because developers default to adding to it.
- **Deeply misleading directory names on critical modules:** A directory name that actively misleads developers about its contents on a core module — for example, a directory named `auth` that is actually the primary location for all user-related logic including profiles, settings, and billing. Developers will fail to find code they need and will place new code incorrectly.

### Important

- **Misplaced files causing confusion:** Files that clearly belong in a different module or directory based on their contents, especially when this forces other modules to import across boundaries to use them. Shared utilities buried inside a specific feature's directory fall into this category.
- **Directories with 20+ files at one level:** Flat directory structures where a single level contains 20 or more files with no subdirectory grouping. The lack of grouping forces developers to scan past many irrelevant files and provides no visual signal of which files are related.
- **Misleading directory or module names:** Directory names that were accurate when created but no longer reflect the full scope of their contents due to scope creep. The name is not wildly wrong but is incomplete — it describes the original scope, not the current scope.
- **Junk-drawer files of 200-500 lines:** Generic-named files (utils, helpers, common, misc) that contain many unrelated functions and are growing. These are on their way to becoming critical findings if not addressed.
- **Missing index or barrel files causing tight coupling to internal structure:** Features that force consumers to import from specific internal file paths because there is no public API surface defined via an index file. Any internal reorganization breaks all consumers.

### Minor

- **Inconsistent file naming conventions:** A project that mixes `kebab-case`, `camelCase`, `PascalCase`, and `snake_case` without a clear rule. This is a friction point for developers who must guess the naming convention for each new file but does not cause logical errors.
- **Minor organizational improvements:** Files that would be slightly better in a different location but are not actively causing confusion. The current placement is not wrong, just not ideal.
- **Slightly misleading names:** A module or file name that is not inaccurate but could be more precise. The name describes the primary purpose but omits secondary responsibilities — the misleading effect is mild and unlikely to cause developers to place code incorrectly.
- **Small junk-drawer files under 200 lines:** Generic-named files with a few unrelated functions that are not yet large enough to be a significant navigation burden. These should be monitored to prevent growth.
- **Test file organization that is slightly unclear:** Test files that are findable but not ideally co-located or named. The mapping between test and source is discoverable with minor effort.
