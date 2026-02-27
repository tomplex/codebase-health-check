---
name: agent-dry
description: Agent prompt for identifying duplicated logic, copy-pasted code, and extractable patterns.
---

# DRY Violations Analysis

You are analyzing a codebase for DRY (Don't Repeat Yourself) violations. A DRY violation occurs when the same piece of knowledge, logic, or concept is expressed in multiple places, so that a change to one requires a parallel change to the others. Duplication increases the risk of inconsistency, makes refactoring harder, and multiplies the surface area for bugs.

Systematically work through every category below. For each category, actively search the codebase using the techniques described in the "How to Search" section.

**Important distinction:** Not all repetition is a DRY violation. Two pieces of code that look similar but represent different concepts and change for different reasons are NOT violations. Only flag duplication where:

- The duplicated logic represents the same concept or piece of knowledge.
- A change to the logic would need to be applied to every copy to keep the system correct.
- Extracting a shared abstraction would make the code clearer, not more confusing.

If two functions happen to have similar structure but serve unrelated domains and evolve independently, that is coincidental similarity, not a DRY violation. Leave it alone.

---

## Categories to Check

### 1. Copy-Pasted Code Blocks

- **Near-identical functions or methods in different files:** Two or more functions that do the same thing with the same structure, differing only in superficial details like variable names, log messages, or minor formatting. The bodies are interchangeable if you rename the locals.
- **Repeated sequences of operations:** The same 5-10 lines of code appearing in multiple places: the same sequence of API calls, the same read-validate-transform pipeline, the same setup-then-execute pattern copied into several functions.
- **Duplicated logic with minor variations:** Functions that share 80-90% of their logic but diverge on one small detail (e.g., a different field name, a different threshold value, a different output format). These are often the result of copy-paste-modify development and should be a single parameterized function.

### 2. Parallel Implementations

- **Multiple implementations of the same concept:** Two or more modules that independently implement the same business rule, algorithm, or workflow rather than sharing a common abstraction. For example, separate order-total calculations in the cart service and the invoice service, or separate date-parsing routines in different API handlers.
- **Duplicated validation logic across endpoints or handlers:** The same input validation rules (required fields, format checks, range constraints) written out independently in multiple request handlers, form validators, or API endpoints rather than defined once and referenced.
- **The same transformation applied in multiple places:** Identical data mapping, formatting, or normalization logic (e.g., converting a user object to a response DTO, normalizing phone numbers, formatting currency) repeated in several files instead of extracted into a shared utility.

### 3. Extractable Patterns

- **Repeated utility-level patterns:** Logic that should be a shared helper but is instead inlined in multiple call sites. Common examples: retry-with-backoff wrappers, error formatting and wrapping, logging-and-rethrow patterns, rate-limiting boilerplate, pagination assembly.
- **Common sequences in tests:** Identical or near-identical test setup (constructing fixtures, initializing mocks, seeding databases), teardown logic, or assertion sequences repeated across many test files. These should be shared test helpers, fixtures, or custom matchers.
- **Repeated initialization or configuration code:** The same boilerplate for initializing a client, connecting to a service, configuring middleware, or setting up a context object duplicated across multiple entry points or modules.

### 4. Duplicated Constants and Configuration

- **Magic numbers or strings in multiple files:** The same literal value (a timeout of `3000`, a status string `"pending"`, a default page size of `25`) appearing in multiple files with the same semantic meaning rather than defined as a named constant in one place.
- **Configuration values hardcoded in multiple places:** Settings like API base URLs, retry counts, buffer sizes, or feature limits embedded directly in code across several modules instead of centralized in a configuration file, environment variable, or constants module.
- **Repeated regex patterns or format strings:** The same regular expression (email validation, URL parsing, date format) or string template written out literally in multiple files rather than defined once and imported.

---

## How to Search

Follow this systematic approach. Do not skip steps.

### Step A: Map the Codebase Structure

1. List all source files within the analysis scope using glob patterns appropriate for the detected languages.
2. Identify files that serve similar purposes by examining directory structure and file naming. Files with parallel names (e.g., `userController.js` and `orderController.js`, `cart_service.py` and `invoice_service.py`) are the most likely to contain duplicated patterns.
3. Identify utility, helper, and shared directories. Note what shared code already exists so you can tell whether a pattern has been partially but not fully extracted.

### Step B: Search for Identical and Near-Identical Code Blocks

1. Pick distinctive multi-word fragments from functions you have already read and grep for them across the codebase. For example, if you see a specific sequence of method calls like `.validate().then(` or a specific error message string, search for that exact fragment. Finding it in multiple files indicates copy-paste duplication.
2. When you find a match, read both sites in full and compare them line by line. Determine whether the differences are meaningful (different business logic) or superficial (different variable names for the same concept).
3. Focus especially on files in the same directory or layer (all controllers, all services, all handlers) since duplication is most common among peers.

### Step C: Check for Repeated Patterns

1. After reading several files in the same layer, note recurring code shapes: the same try/catch wrapping pattern, the same request-response cycle, the same validation-then-execute sequence. These are extractable patterns even if the specific field names differ.
2. Search for common pattern signatures: `retry`, `withRetry`, `backoff`, `formatError`, `wrapError`, `toDTO`, `serialize`, `normalize`. If these do not exist as shared utilities, search for the inline patterns they would replace (e.g., `for (let attempt` or `catch.*retry` or `setTimeout.*attempt`).
3. In test files, look for repeated `beforeEach`/`setUp` blocks. If the same mock setup or fixture construction appears in more than two test files, it is an extractable test helper.

### Step D: Search for Magic Values in Multiple Files

1. Search for common magic numbers: `200`, `400`, `401`, `403`, `404`, `500` (HTTP status codes used as literals), timeouts like `3000`, `5000`, `10000`, `30000`, limits like `100`, `1000`, `10000`, and domain-specific thresholds.
2. Search for repeated string literals: status values (`"active"`, `"pending"`, `"completed"`), role names (`"admin"`, `"user"`), error messages, API endpoint paths, header names.
3. Search for regex patterns that appear in more than one file. Grep for distinctive regex fragments like `[A-Za-z0-9`, `\d{2,4}`, `@.*\.`, or escaped special characters that suggest a validation regex.
4. For each magic value found in multiple files, verify that it carries the same semantic meaning in each location. A `200` in an HTTP handler and a `200` as a pixel width are not the same concept.

### Step E: Examine Validation and Transformation Logic

1. Read the validation logic in each handler, controller, or endpoint. Note which fields are checked and what rules are applied. Compare across handlers to identify identical or overlapping validation rules that should be shared.
2. Read data transformation code (mapping between objects, formatting for output, normalizing input). Compare transformations that operate on the same data types or produce the same output shapes.
3. Search for common validation keywords: `required`, `isValid`, `validate`, `check`, `assert`, `must`, `ensure`. Read the surrounding code to determine whether the same rule is expressed in multiple places.

### Step F: Pay Attention to Test Setup Duplication

1. Read test files across the codebase. Identify setup patterns that appear in more than two test files.
2. Search for repeated mock/stub creation: `mock(`, `jest.fn(`, `patch(`, `stub(`, `sinon.`, `MagicMock`, `@MockBean`. If the same mock configuration appears in multiple test files, it should be a shared fixture.
3. Search for repeated test data construction: objects or records built with the same shape and similar values across multiple test files. These should be factory functions or fixture files.

---

## Severity Guide

Use these severity levels when reporting findings:

### Critical

- **Business logic duplicated across modules** — the same business rule, calculation, or workflow implemented independently in multiple services or modules. When the rule changes, one copy will be updated and the others will be missed, causing silent correctness bugs. Example: order-total calculation duplicated in the cart service, the checkout service, and the invoice generator.
- **Validation logic duplicated with drift** — the same validation rules implemented in multiple places where the copies have already diverged, meaning the system is already inconsistent. This proves the duplication has already caused a bug.

### Important

- **Significant repeated patterns (10+ lines)** — copy-pasted blocks of 10 or more lines that implement the same pattern with superficial differences. These are large enough that extracting a shared abstraction would meaningfully reduce code volume and maintenance burden.
- **Duplicated validation across endpoints** — the same input validation rules repeated in multiple handlers or controllers, even if the copies have not yet diverged. A rule change will require finding and updating every copy.
- **Repeated utility patterns with no shared helper** — retry logic, error formatting, pagination assembly, or similar infrastructure patterns inlined in 3+ places that should be a single utility function.
- **Multiple test files with identical setup blocks** — the same 10+ lines of fixture construction, mock wiring, or database seeding repeated across many test files.

### Minor

- **Small duplicated snippets (2-3 lines)** — short repeated sequences that could be extracted but where the extraction would not dramatically improve clarity.
- **Repeated test assertions or teardown** — minor duplication in test code that is not extensive enough to justify a shared helper.
- **Magic values appearing in 2-3 files** — repeated literal constants that should be named but where the duplication is limited in scope.
- **Duplicated comments or log message templates** — repeated string templates for error messages or log lines that could be centralized but are low risk.
