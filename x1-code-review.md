# X1 Code Review

**Report:** "Using X1 Code Review methodology."

## X1 Core Principles (Apply Throughout)
- **Fail fast:** Throw on missing config. No fallbacks that hide failures.
- **Verify before assuming:** Read actual method signatures, field names, constructors from the codebase before writing code snippets.
- **Reuse before creating:** Search the codebase for existing services, DTOs, and patterns before proposing new ones. Extend, don't duplicate.
- **Minimal pragmatic code:** Generate the minimum code that fully satisfies all acceptance criteria. No more, no less.
- **Full traceability:** US -> AC -> IMP -> Test. Every link visible, every AC covered.
- **User stories first:** Requirements are locked before technical investigation begins.
- **Observable:** Every feature must answer "How will we know this is broken in production?"
- **Testable by design:** Structure code so orchestration-level tests with mocked boundaries can verify real workflows.

## Purpose

Review **implementation plan code snippets** before they're implemented. This is a pre-implementation review, not a git diff review. The goal is to catch issues in the plan's proposed code before any code is written.

**Primary reviewer:** AI assistants analyzing implementation plan documents.

---

## Scope

1. Find the active implementation plan (check `.claude/plans/` or ask user which plan to review)
2. Extract all code snippets from the plan
3. Review each snippet against this checklist
4. For every snippet, verify against the actual codebase (signatures, field names, patterns)

---

## Code Reuse (MANDATORY)

Before approving ANY proposed new code in the plan, the reviewer MUST:

1. **Search the codebase** for existing implementations that solve the same problem
2. **Check existing services, repositories, utilities, components** for reuse opportunities
3. **Understand existing patterns** - proposed code must follow established conventions
4. **Flag any proposed new file/class/function** that may already exist in the codebase

If unsure whether something exists: **search first, approve later.**

---

## Instructions

1. Read the implementation plan
2. For each code snippet, verify method signatures, field names, and types against the actual codebase
3. Apply Chaos Demon methodology to each snippet
4. **IDENTIFY issues, then propose fixes** (update the plan snippets directly)
5. Re-audit after fixes to ensure no new issues introduced
6. Summarize what was reviewed and fixed

---

## Pre-Review Gate

Before detailed review, verify the plan:
- [ ] Has clear IMP items with traceability to user stories/ACs
- [ ] Code snippets reference real files, methods, and types
- [ ] No snippets use assumed signatures without [VERIFIED] or [CONCEPTUAL] tags

---

## Quick Scan Order

Review in this priority: **Security → Critical bugs → Architecture → Performance → Cleanup**

---

## Chaos Demon Mindset

- **YAGNI**: Remove code not needed TODAY
- **Simplicity**: 10 lines beats 100 if it does the same thing
- **Delete more than you add**: If in doubt, delete it
- **No gold-plating**: Remove "future enhancement" markers

---

## Review Checklist (All Languages)

### Critical (Must Fix)

**Bugs**
- Null/undefined safety violations
- Race conditions in async/concurrent code
- Data integrity issues (lost updates, orphaned records)
- Off-by-one errors, boundary conditions

**Security**
- SQL injection / unparameterized queries
- XSS (unsanitized user input in rendering)
- Secrets in code, logs, or error messages
- Input validation missing at boundaries
- Vulnerable dependencies (outdated packages with known CVEs)

**Resource Management**
- Resource leaks (unclosed connections, streams, subscriptions)
- Memory leaks (unsubscribed listeners, intervals, refs)
- Missing error handling on async operations

**No Fallback Code**
- Silent fallbacks that hide failures (default values masking errors)
- Catch blocks returning empty/default instead of propagating
- "Safe" wrappers that swallow exceptions
- Fix: Fail fast. Let errors surface. Handle at appropriate boundary only.
- **This is enforced during planning too.** See x1-plan "Fail-Fast (MANDATORY)" in Code Snippet Rules.
- During code review, treat ANY `?? defaultValue` for config/required data as Critical severity.
- Fallback patterns are the #1 source of silent production bugs.

**Error Handling**
- Failing open (errors allow access/action instead of denying)
- Stack traces exposed to users
- Generic catches that hide root cause

### Warning (Should Fix)

**Architecture - Layer Separation (CRITICAL)**

**Service Layer Hierarchy:**
```
Controllers / Hubs          -> HTTP endpoints, WebSocket/SignalR handlers. Lightweight. No business logic.
  | calls
Orchestration Services       -> Coordinate multi-step workflows. Named *Orchestrator.
  | calls
Business Services            -> Single-responsibility domain logic.
  | calls
Technical Services           -> Infrastructure concerns (email, SMS, scheduling, caching).
  | calls
Repositories                 -> Data access abstraction. Never called from controllers/hubs.
  | calls
DbContext                    -> EF Core. Never injected above the repository layer.
```

- **Controllers/Hubs calling repositories directly** - Controllers and SignalR hubs must NEVER inject or call repositories
- **Business logic in controllers/hubs** - Move ALL logic beyond request/response handling to services
- **DbContext/data access in services** - Services must use repositories, not DbContext directly
- **Data transformation in controllers** - Mapping, filtering, aggregation belongs in services
- **Skipping service tiers** - Controllers should call orchestrators, not technical services directly (exceptions must be minimal and justified)
- **External services without provider abstraction** - If an external service (LLM, voice, payment, etc.) could have multiple implementations, it MUST go through a provider interface. Check if one exists before approving direct integration.

**Design Elegance**
- **Constructor dependency count >5** - Service is doing too much. Should be split into focused services that compose. Count the parameters; if the test setup needs >5 mocks, the design is wrong.
- **Mixing decisions with side effects** - A method that both evaluates a condition (should we suspend? is the threshold breached?) AND executes the result (update database, call another service) should be split. Decisions should be pure or near-pure; execution should be separate. This makes the decision logic trivially testable.
- **Policy logic embedded in orchestration** - Threshold checks, eligibility rules, scheduling decisions embedded inside lifecycle methods. Extract to dedicated policy/guard services or pure functions.
- **Seam assumptions undocumented** - Service A calls Service B and assumes B's state. If the assumption is not validated (check return value, verify precondition), flag it. Silent assumptions at service boundaries become production bugs.

**Tenant & DI**
- Missing tenant isolation checks
- DI registration issues (wrong lifetime, missing registration)

**API Contracts**
- Breaking changes to DTOs (removed/renamed fields)
- Inconsistent naming conventions
- Hardcoded secrets/config (API keys, connection strings)

**Performance**
- N+1 queries
- Loading entire tables without pagination
- Unbounded collections
- Missing indexes on filtered/sorted columns

**Error Handling**
- Swallowing exceptions silently
- Generic error messages (not actionable for debugging)
- No logging context (correlation IDs, user context)

**Type Safety**
- Using `any` or equivalent loose typing
- Missing nullability annotations
- Unvalidated external input
- Non-exhaustive switch/match statements

### Cleanup (Nice to Fix)

**Over-Engineering**
- Interfaces with single implementation
- Premature optimization (caching without measurements)
- Configuration for values that never change
- File proliferation (merge small related files)
- Wrapper classes that add no behavior

**Code Quality**
- Names don't reveal intent
- Magic numbers/strings (extract constants)
- Dead code / commented-out code
- Inconsistent terminology
- Comments restating what code does

---

## .NET Specific

### Critical
- `async void` methods (except event handlers) - unhandled exceptions
- `Task.Result` or `.Wait()` causing deadlocks
- Missing `CancellationToken` propagation in async chains
- Raw SQL without parameterization (even in EF Core)

### Warning
- `IDisposable` not implemented when holding unmanaged resources
- `DateTime.Now` instead of `DateTime.UtcNow` for storage/comparison
- String concatenation in loops (use `StringBuilder`)
- LINQ `.ToList()` mid-chain (materializes early)
- `FirstOrDefault()` + null check vs pattern matching
- EF tracking queries where no-tracking would suffice (use `.AsNoTracking()`)
- Missing `.ConfigureAwait(false)` in library/non-UI code
- Enum serialized as int (breaks if enum values change order)

### Cleanup
- Missing `sealed` on classes not designed for inheritance
- Missing `readonly` on fields that don't change
- `var` vs explicit type inconsistency
- Expression-bodied members not used where appropriate
- Missing response caching headers for GET endpoints

---

## React/TypeScript Specific

### Critical
- Direct state mutation (modifying arrays/objects without spread)
- Missing `key` prop or using index as key for dynamic lists
- `useEffect` with missing dependencies causing stale closures
- Infinite loops from effect dependencies
- Missing accessibility (aria-labels, keyboard nav, focus management)
- Sensitive data in localStorage/sessionStorage

### Warning
- Missing `useCallback`/`useMemo` for expensive operations
- State that should be derived being stored separately
- Props drilling >3 levels (use context or composition)
- Uncontrolled to controlled component switches
- Missing cleanup in `useEffect` return
- Missing error boundaries
- Missing loading/error/empty states
- Form validation not using schema (Zod/Yup)
- API errors not surfaced to user
- Files >400-500 lines (split into smaller)

### Cleanup
- Inline functions in JSX causing re-renders
- Components >300 lines (split into smaller)
- Business logic in components (extract to hooks/utils)
- Hardcoded strings that should be constants
- Console.log statements left in code

---

## Date/Time Handling

**Context matters** - check database field names/comments for intent (e.g., `CreatedAtUtc`, `DueDate`, `AppointmentTime`).

**Standards:**
- Display format: `DD-MM-YYYY` for dates, `HH:MM` (24hr) for times
- API/DTO format: ISO 8601 with UTC suffix when applicable (`2025-12-28T14:30:00Z`)

**Date/Time Categories:**
| Type | Examples | Storage | API Format | Display |
|------|----------|---------|------------|---------|
| Timestamp (UTC) | `CreatedAt`, `UpdatedAt`, `LastLoginAt` | UTC | `2025-12-28T14:30:00Z` | Convert to user's timezone |
| Date-only | `DueDate`, `BirthDate`, `ExpiryDate` | Noon UTC | `2025-12-28T12:00:00Z` | `DD-MM-YYYY` only |
| Local datetime | `AppointmentTime`, `ShiftStart` | As entered | `2025-12-28T14:30:00` (no Z) | As stored |

### Critical
- **Midnight timezone shift**: `startOfDay()` or `setHours(0,0,0,0)` creates midnight local → shifts day when serialized to UTC
  - Fix: Use noon `setHours(12,0,0,0)` for date-only values
- **Date range queries wrong day**: User's "today" query misses records because UTC boundaries differ
  - Fix: Convert user's local day boundaries to UTC range before querying
- **UTC timestamps missing Z suffix**: DTOs returning UTC datetime without `Z` causes ambiguous parsing
  - Fix: Ensure UTC timestamps serialize as `2025-12-28T14:30:00Z`

### Warning
- **Timezone intent unclear**: Database field or DTO property doesn't indicate UTC vs local
  - Fix: Name fields explicitly (`CreatedAtUtc`) or document in schema
- **Display not converted**: UTC timestamps displayed without converting to user's timezone
  - Check: API returns UTC → client must convert for display
- **DST arithmetic bugs**: `date.getTime() + 86400000` breaks across DST transitions
  - Fix: Use `addDays()`, `addWeeks()`, `addMonths()` from date-fns
- **Month/year edge cases**: Manual month increment overflows (Jan 31 + 1 month)
  - Fix: Use `addMonths()` which handles Feb 28/Mar 3 correctly

### Cleanup
- **Range boundary semantics unclear**: "Due before 28-12-2025" - inclusive or exclusive?
  - Document and test boundary conditions
- **Relative labels across timezones**: "Today" for AU user shows as "Yesterday" for UK user
  - Consider showing explicit date alongside relative labels

---

## Severity Guide

| Level | Action | Examples |
|-------|--------|----------|
| Critical | Block merge, fix now | Security holes, data corruption, crashes, fallback code |
| Warning | Fix before merge | Architecture violations, performance issues |
| Cleanup | Fix if touching that code | Style, naming, minor improvements |

---

## Stop Criteria

**Request rewrite instead of patching if:**
- Fundamental architecture is wrong
- >50% of code needs changes
- Security model is broken
- Would need extensive workarounds

---

## Output Format

For each issue:
```
[Critical/Warning/Cleanup] file:line - Brief description
  Problem: What's wrong
  Fix: What to do (then do it)
```

---

## Final Report

- Scope reviewed
- Issues by severity (Critical x, Warning x, Cleanup x)
- Files modified
- Ready status or remaining blockers

---

## Quick Reference: Common Patterns

### Critical Bugs
| Pattern | Risk | Fix |
|---------|------|-----|
| `.First()` without check | Exception if empty | `.FirstOrDefault()` + null check |
| `.Result` or `.Wait()` | Deadlock | Use `await` |
| `List<T>` in parallel code | Race condition | `ConcurrentBag<T>` |
| Missing `using` on `IDisposable` | Resource leak | Add `using` statement |
| `catch` without logging | Silent failure | Log exception, rethrow or handle |
| Missing TenantId check | Security breach | Validate tenant context |
| Fallback returning default | Hidden failure | Fail fast, propagate error |

### Architecture Violations
| Pattern | Risk | Fix |
|---------|------|-----|
| Controller injects repository | Bypasses business logic | Controller → Service → Repository |
| Business logic in controller | Untestable, not reusable | Move to service class |
| DbContext in service | Service doing repository's job | Create repository for data access |
| Data transformation in controller | Violates single responsibility | Move to service |
| Controller calls technical service | Skips orchestration layer | Route through orchestrator |
| Direct external API call | No provider abstraction | Create or use existing provider interface |
| Business logic in SignalR hub | Hub doing service's job | Delegate to orchestrator/service |

### Design Elegance
| Pattern | Risk | Fix |
|---------|------|-----|
| Service constructor >5 dependencies | Doing too much, hard to test | Split into focused services that compose |
| Method decides AND executes | Untestable decision logic | Separate decision (pure) from execution (side effects) |
| Policy logic inside orchestrator | Guards/rules buried in workflow | Extract to dedicated guard/policy service or pure function |
| Unchecked cross-service assumption | Silent seam failure in production | Validate state at handoff point (check return, verify precondition) |
| Test needs >5 mocks | Design smell, not test smell | Redesign service, don't write complex test setup |

### External API Error Handling
| Pattern | Risk | Fix |
|---------|------|-----|
| `EnsureSuccessStatusCode()` only | Loses API error message, returns generic 500 | Extract error from response body before throwing |
| `HttpRequestException` bubbles up | Client gets 500 instead of actual status code | Create typed exception with StatusCode property |
| Controller missing API exception catch | API errors (403, 429, etc.) return as 500 | Add catch block returning `StatusCode(ex.StatusCode, ...)` |
| Error message not logged | Can't debug API failures | Log status code, reason phrase, and response body |

### Performance
| Pattern | Impact | Fix |
|---------|--------|-----|
| Query in loop | N+1 queries | Batch query with `GetByIdsAsync()` |
| `.ToList()` on large dataset | Memory: loads all records | Pagination `.Take(n)` or streaming |
| Multiple enumerations | CPU: re-iterates collection | Cache with `.ToList()` first |
| String concat in loop | O(n²) allocations | `StringBuilder` or `string.Join()` |
| Loading entire table | Memory + I/O | Filter in DB, select needed columns |

**Repository Performance (WARNING -> CRITICAL based on data volume)**

| Code Pattern | Problem | Fix |
|---|---|---|
| `ToListAsync()` without pagination | Unbounded result set | Add pagination or verify bounded by other means |
| `.ToList().Where(...)` | Client-side filtering | Push Where into IQueryable before materialization |
| Query inside foreach/for loop | N+1 queries (O(N) round-trips) | Batch lookups before the loop |
| `Count() > 0` | Full count when only existence needed | Use `AnyAsync()` |
| Missing AsNoTracking on read path | 30-40% unnecessary overhead | Add `.AsNoTracking()` |
| Include with multiple collections | Cartesian explosion | Use `.AsSplitQuery()` or projection |
| Missing FK index in configuration | Full table scan on JOIN | Add `.HasIndex()` in entity configuration |
| Loading full entity for 2-3 fields | Excessive data transfer | Use projection (`.Select(x => new { ... })`) |
| Sync database call (e.g. `ToList()`) | Blocks thread pool thread | Use async variant (`ToListAsync()`) |

### React/TypeScript
| Pattern | Risk | Fix |
|---------|------|-----|
| State mutation | Stale UI, bugs | Spread operator `{...obj}` or `[...arr]` |
| Index as key | Incorrect updates | Use stable unique ID |
| Missing effect deps | Stale closures | Add all dependencies |
| Missing effect cleanup | Memory leak | Return cleanup function |
| `any` type | Type safety lost | Define proper types |
