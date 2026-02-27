---
name: agent-complexity
description: Agent prompt for finding deeply nested logic, long functions, high cyclomatic complexity, and over-engineering.
---

# Complexity Analysis

You are analyzing a codebase for unnecessary complexity. Unnecessary complexity is any code structure that is harder to understand, modify, or debug than it needs to be for the problem it solves. Complexity increases cognitive load, slows development, hides bugs, and discourages contributors from making changes.

Systematically work through every category below. For each category, actively search the codebase using the techniques described in the "How to Search" section.

---

## Categories to Check

### 1. Overly Long Functions

- **Functions exceeding ~30 lines of logic:** Count the lines of actual logic in each function or method, excluding blank lines, comments, and pure-formatting lines (closing braces on their own line, etc.). Functions that exceed roughly 30 lines of logic are candidates. Use judgment — a 35-line function that does one clear thing is less concerning than a 25-line function that does three unrelated things.
- **Functions that do more than one conceptual thing:** A single function that validates input, queries a database, transforms the result, sends a notification, and logs the outcome is doing too many things regardless of line count. Look for functions where you can draw clear boundaries between distinct responsibilities.
- **God functions/methods:** Functions that orchestrate too many concerns or that most of the codebase flows through. These are often named generically (`process`, `handle`, `run`, `execute`, `main`, `update`) and contain the core logic for an entire feature or subsystem in one place. They are the hardest to test, the most fragile to change, and the most likely to contain bugs.

### 2. Deep Nesting

- **3+ levels of nesting:** Code where control structures are nested three or more levels deep. For example: an `if` inside a `for` inside a `try` inside another `if`. Each level of nesting multiplies the number of mental states a reader must track. Look for indentation that pushes code far to the right.
- **Deeply nested callbacks or promise chains:** Callback hell patterns where asynchronous operations are nested inside each other rather than composed sequentially. Similarly, `.then().then().then()` chains that nest further callbacks inside each `.then`. These are common in JavaScript/TypeScript but can appear in any language with callback-based APIs.
- **Complex nested ternary expressions:** Ternary operators nested inside other ternary operators (`a ? (b ? c : d) : (e ? f : g)`). A single ternary is fine. Nested ternaries force the reader to mentally parse a tree of conditions that would be clearer as an `if/else` block or a lookup table.

### 3. High Cyclomatic Complexity

- **Functions with many branches:** Functions containing long `if/else if/else` chains or `switch`/`match` statements with many cases. If a function has more than 5-6 distinct branches, it is becoming hard to reason about all the possible paths through it.
- **Functions requiring path tracing:** Functions where understanding the output requires tracing many interacting paths — for instance, multiple early returns combined with nested conditionals and fallthrough logic. If you cannot quickly answer "what does this function return when X?" for common inputs, the cyclomatic complexity is too high.
- **Complex boolean expressions:** Conditions that combine multiple `&&` and `||` operators, especially when they mix both in a single expression without clear grouping. Expressions like `if (a && b || c && !d || e)` force the reader to mentally evaluate operator precedence and multiple states simultaneously.

### 4. Over-Engineering

- **Interfaces/abstractions with only one implementation:** An interface, abstract class, trait, or protocol that has exactly one concrete implementation, with no clear reason to expect additional implementations. The abstraction adds a layer of indirection without providing any polymorphic value. Common examples: `IUserService` with only `UserService`, `AbstractRepository` with only `PostgresRepository` when no other database is planned.
- **Unnecessary indirection layers:** Wrapper functions, facade classes, or delegation methods that add no logic, validation, transformation, or error handling — they simply forward calls to another function or object. If removing the layer and calling the underlying code directly would be equally clear and correct, the layer is unnecessary.
- **Abstract factory patterns where a simple constructor would suffice:** Factory classes, builder patterns, or provider abstractions for objects that only have one concrete variant and no configuration complexity. If the factory always produces the same type with the same setup, a direct constructor call is simpler.
- **Generic solutions for single concrete cases:** Parameterized or generic code (generic types, strategy patterns, plugin architectures) that is only ever instantiated or used with a single concrete type or strategy. The generic machinery adds complexity without delivering its intended flexibility benefit.

### 5. Premature Optimization

- **Complex caching where simple recomputation would be fast enough:** Memoization layers, cache invalidation logic, or custom caching data structures applied to computations that are cheap to re-run. Look for caching of simple lookups, string formatting, small list operations, or other sub-millisecond work. The cache management code is often more complex and bug-prone than just doing the computation each time.
- **Manual memory management patterns in garbage-collected languages:** Object pools, explicit free/dispose calls, or manual reference counting in languages like JavaScript, Python, Ruby, Go, or Java where the runtime garbage collector handles memory. Unless profiling shows a specific GC pressure problem, these patterns add complexity for no measurable gain.
- **Bit manipulation or micro-optimizations in non-performance-critical code:** Using bit shifts instead of multiplication, manual loop unrolling, hand-inlined functions, or other micro-optimizations in code that is not on a measured hot path. These patterns sacrifice readability for performance gains that are invisible in practice.
- **Complex data structures where a simple array/list would work fine:** Custom trees, hash maps, skip lists, or other advanced data structures used for collections that are small enough that a simple array with linear search would be equally fast (or faster due to cache locality). If the collection typically contains fewer than ~100 items and is not accessed in a tight loop, a simpler structure is usually better.

---

## How to Search

Follow this systematic approach. Do not skip steps.

### Step A: Scan for Long Functions

1. List all source files within the analysis scope using glob patterns appropriate for the detected languages.
2. Open files and scan for function/method definitions. Identify functions that appear to span many lines by looking for the distance between the function signature and the closing delimiter.
3. For any function that looks long, count its lines of logic (excluding blanks and comments). Flag those exceeding ~30 lines.
4. Pay extra attention to files with few functions — a file with only one or two very long functions often contains a god function.

### Step B: Look for Deep Indentation

1. Search for lines with high indentation levels. In languages that use braces, look for code indented 4+ levels (typically 16+ spaces or 4+ tabs). In Python, look for code at 4+ indentation levels.
2. When you find deeply indented code, trace back through the nesting levels to understand the structure. Determine whether the nesting is genuinely needed or whether early returns, guard clauses, or extraction into helper functions could flatten it.
3. Search for nested ternary patterns using language-appropriate regex (e.g., `? .* \? .* :` patterns).
4. In JavaScript/TypeScript codebases, search for callback nesting patterns: function definitions inside function arguments inside function arguments.

### Step C: Check for Abstraction Layers with Single Implementations

1. Search for interface and abstract class definitions using language-appropriate patterns (e.g., `interface I`, `abstract class`, `trait`, `protocol`).
2. For each interface or abstract class found, search the codebase for classes that implement or extend it. Count the implementations.
3. If an abstraction has exactly one implementation, check whether there is a clear architectural reason for the abstraction (dependency injection for testing, published library API, clear plan for multiple implementations). If none is evident, flag it.
4. Search for factory classes and builder patterns. Check whether the factory ever produces more than one type of object.

### Step D: Look for Over-Engineering Patterns

1. Search for class names containing common over-engineering signals: `AbstractFactory`, `Builder`, `Strategy`, `Provider`, `Manager`, `Handler`, `Processor`, `Orchestrator`, `Coordinator`, `Mediator`, `Adapter` (when wrapping a single concrete type), `Wrapper`, `Proxy` (when not providing access control or lazy loading), `Delegate`.
2. For each match, examine whether the pattern provides genuine value or whether a simpler approach would work. Not all uses of these patterns are over-engineering — only flag those where the pattern adds complexity without delivering its intended benefit.
3. Search for wrapper functions that contain only a single forwarding call: functions whose body is just `return otherFunction(args)` or `this.delegate.method(args)` with no additional logic.
4. Search for generic type parameters or template types that are only ever instantiated with a single concrete type across the entire codebase.

### Step E: Check for Premature Optimization

1. Look for caching-related patterns: memoization decorators, `cache` variables, `Map`/`WeakMap`/`dict` used as caches, LRU cache implementations. For each, assess whether the cached computation is expensive enough to warrant the caching overhead.
2. In garbage-collected languages, search for object pool patterns, explicit memory management, or manual reference counting.
3. Search for bit manipulation operators (`<<`, `>>`, `&`, `|`, `^`) in code that is not in a clearly performance-critical path (not a codec, not a crypto library, not a hot inner loop).
4. Look for custom data structure implementations where standard library collections would suffice for the actual data sizes involved.

### Step F: Evaluate Cyclomatic Complexity

1. Search for long `if/else if` chains (3+ `else if` branches in sequence).
2. Search for `switch`/`match` statements and count their cases. Flag those with more than 7-8 cases, especially if the cases contain non-trivial logic.
3. Look for functions that combine multiple control flow mechanisms: early returns mixed with nested conditionals, loops with complex break/continue conditions, exception handling interleaved with branching logic.
4. Search for boolean expressions that combine 3+ conditions with mixed `&&` and `||` operators.

---

## Severity Guide

Use these severity levels when reporting findings:

### Critical

- **God objects/functions central to the codebase** — functions or classes that are core to the system's operation and are excessively long, deeply nested, or extremely high in cyclomatic complexity. These are the highest risk because they are both hard to understand AND frequently touched or depended upon.
- **Deeply nested critical-path logic** — nesting of 4+ levels in code that runs on every request, transaction, or core operation. Bugs hiding in deeply nested critical-path code have the widest blast radius.

### Important

- **Over-engineered abstractions adding cognitive load** — interfaces with single implementations, unnecessary factory/builder patterns, or generic solutions for single concrete cases that force every developer to navigate extra layers of indirection to understand the code.
- **Deeply nested logic (3+ levels)** — functions with 3+ levels of nesting that are not on the critical path but are still actively maintained or extended. The nesting makes changes risky and code review difficult.
- **High cyclomatic complexity** — functions with many branches that require careful path tracing to understand. These are magnets for bugs because it is easy to miss a path during changes or reviews.
- **Premature optimization patterns** — caching, custom data structures, or micro-optimizations that add significant complexity without measurable performance benefit.

### Minor

- **Slightly long functions (30-50 lines of logic)** — functions that are longer than ideal but still do roughly one thing and are readable. They would benefit from extraction but are not urgent.
- **Minor over-abstractions** — a single unnecessary wrapper or a lone interface-with-one-implementation in a non-core part of the codebase. Low cognitive cost in isolation.
- **Mildly complex boolean expressions** — conditions with 3 terms that could be clearer but are not genuinely confusing.
- **Nested ternaries in non-critical code** — nested ternary expressions in code that is rarely read or modified.
