# X1 Merged Plan + Review

**Report:** "Using X1 merged Plan + Review methodology."

## X1 Core Principles (Apply Throughout)
- **Fail fast:** Throw on missing config. No fallbacks that hide failures.
- **Verify before assuming:** Read actual method signatures, field names, constructors from the codebase before writing code snippets.
- **Reuse before creating:** Search the codebase for existing services, DTOs, and patterns before proposing new ones. Extend, don't duplicate.
- **Minimal pragmatic code:** Generate the minimum code that fully satisfies all acceptance criteria. No more, no less.
- **Full traceability:** US -> AC -> IMP -> Test. Every link visible, every AC covered.
- **User stories first:** Requirements are locked before technical investigation begins.
- **Observable:** Every feature must answer "How will we know this is broken in production?"
- **Testable by design:** Structure code so orchestration-level tests with mocked boundaries can verify real workflows.

## Overview

This is the merged **architecture + review** skill: it reads the approved requirements from `/x1-requirements`, runs the technical planning half of `/x1-plan` (architecture, invariants, detailed checklist), then transitions to an adversarial Chaos Demon review — all in a single context window so the review has full memory of every assumption made during planning. **User stories are NOT elicited here** — they are locked upstream by `/x1-requirements`.

### When to Use This vs Separate Skills

**Use this merged skill when:**
- Single-implementation / medium features where a single context window suffices
- Faster turnaround needed, no information loss between planning and review

**Use separate `/x1-plan` + `/x1-review-plan` when:**
- Large or multi-slice features (the spine + N slice plans won't fit one context)
- You want a different AI perspective on each pass (fresh eyes catch different things)

Either way, run `/x1-requirements` first — this skill begins from locked requirements.

## Instructions

1. **Load approved requirements** - Read the `/x1-requirements` artifact from `specs/…` (Phase 0). If none, STOP and tell the user to run `/x1-requirements`.
2. **Present high-level plan** - Conceptual approach, no code, no file paths. Wait for acceptance.
3. **Investigate codebase** - Only after high-level accepted. Find patterns, reuse.
4. **Identify invariants** - Invariant Impact Matrix (reuse) + Generator (derive) against `docs/architecture/architectural-principles.md`.
5. **Add detail iteratively** - Technical specifics, code snippets, Implementation Checklist.
6. **TRANSITION to adversarial mode** - Switch from architect to Chaos Demon.
7. **Break the plan** - Find every failure mode, gap, and assumption.
8. **Output corrected plan** - With review findings appended.

**Goal:** Codebase-aligned plan with traceable Implementation Checklist and honoured invariants, adversarially reviewed and corrected, ready for `/x1-implement`.

**Speed:** Work in parallel wherever possible. Read multiple files simultaneously, search client and server concurrently, research while investigating code.

---

## Phase 0: Load Approved Requirements (BLOCKING)

Begin from locked requirements; do not re-elicit or edit stories.
1. Read the requirements artifact — `specs/user-story.md` (or `specs/<feature>-requirements.md`): epoch, `US-{EPOCH}-N`, `AC-{EPOCH}-N.M`, stress scenarios, gap analysis, rationale. Single source of truth for ACs.
2. If missing or not approved → STOP: "Run `/x1-requirements` first."

This merged skill is for **single-implementation** features. If the architecture turns out to fan out to multiple slices, stop and use separate `/x1-plan` (which emits a spine + slice plans) + `/x1-review-plan`.

**Invariants:** in Phase 2 (Codebase Investigation), run the **Invariant Impact Matrix** and **Invariant Generator** exactly as defined in `/x1-plan` — reuse the catalog at `docs/architecture/architectural-principles.md`, derive new invariants from the feature's deltas, and turn each touched invariant's *Observed by* into a test. Cross-slice/coverage-ledger steps do not apply here (single implementation).

---

## Phase 1: High-Level Plan

**This phase begins ONLY after Phase 0 — the approved requirements artifact is loaded.**

Present the approach at a conceptual level. No code. No file paths. Just the shape of the solution.

### Problem Statement
[1-2 sentences: what we're solving and why]

### User Stories
[List with acceptance criteria — loaded from the `/x1-requirements` artifact in `specs/…`, not re-authored here]

### Proposed Approach
[Conceptual design. How will we solve this? What's the strategy?]

### Scope
- **In scope:** [list]
- **Out of scope:** [list]

### Key Decisions
[Architectural choices, trade-offs, alternatives considered]

**Ask:** "Does this direction feel right? Any concerns before I investigate the codebase and add detail?"

**Wait for acceptance before proceeding to Phase 4.**

---

## Phase 2: Codebase Investigation

Only after the high-level approach is accepted. Investigate **in parallel**:

1. **Read CLAUDE.md** - Understand project conventions
2. **Explore relevant code** - Client and server simultaneously
3. **Identify patterns** - How similar features are implemented
4. **Find reuse opportunities** - Existing services, components, utilities
5. **Research if needed** - Web search for libraries, APIs, best practices

**Parallelize:** Read multiple files at once. Search client while searching server. Research online while exploring code.

**Do NOT generate code snippets until codebase is understood.**

### Architecture Verification (BLOCKING)

Before designing any new code, verify WHERE it belongs in the application's layer hierarchy. Placing code in the wrong layer during planning means every downstream step (review, implement, test) builds on the wrong foundation.

**Service Layer Hierarchy:**
```
Controllers / Hubs          -> HTTP endpoints, WebSocket handlers. Lightweight. No business logic.
  | calls
Orchestration Services       -> Coordinate multi-step workflows. Named *Orchestrator.
  | calls
Business Services            -> Single-responsibility domain logic.
  | calls
Technical Services           -> Infrastructure concerns (email, SMS, scheduling, caching).
  | calls
Repositories                 -> Data access abstraction. Never called directly from controllers.
  | calls
DbContext                    -> EF Core. Never injected above the repository layer.
```

**Communication Layers:**
- **REST Controllers:** External HTTP API endpoints consumed by client applications
- **SignalR Hubs:** Real-time bidirectional communication (transcripts, audio, notifications). Treat as equivalent to controllers for layer discipline: lightweight, delegate to services.
- **WebSocket Handlers:** Raw WebSocket connections (e.g. Twilio media streams). Same layer discipline as controllers.
- **Background Services:** Long-running processes. Call orchestrators, not repositories directly.

**Provider Pattern (for external service abstraction):**
When the plan involves external services (LLMs, voice, security, payment, etc.):
1. Check if a provider abstraction already exists (e.g. `ILlmProvider`, `IVoiceProvider`)
2. If yes, new work MUST go through the existing provider interface, not bypass it
3. If no provider exists and the service could have multiple implementations, propose one
4. Provider pattern enables switching service providers without rewriting consuming code
5. Providers are adapters: they implement a common interface and encapsulate provider-specific details

**Verification during planning:**
For every new class, method, or modification proposed in the plan:
- [ ] Which layer does it belong to? State it explicitly.
- [ ] Does it follow the hierarchy? (Controllers don't call repositories. Services don't inject DbContext.)
- [ ] If it involves external services, does it use the provider pattern?
- [ ] If it's a hub/WebSocket handler, is business logic delegated to a service?

### Observability by Design (MANDATORY)

Plan for observability during architecture, not after implementation. The question "How will we know this is broken in production?" must be answered for every significant code path BEFORE writing code.

**For every service or workflow in the plan, document:**
1. **Key state transitions** that need debug logging (e.g. "Contact resolved", "Voice session created", "Provider switched")
2. **Failure points** that need structured error logging with context (correlation IDs, tenant, entity IDs)
3. **Metrics or health signals** for production monitoring (e.g. "Call duration histogram", "Provider error rate")

**Logging principles:**
- Log at decision points, not at every method entry/exit
- Include context: what entity, which tenant, what triggered the action
- Structured logging (Serilog properties), not string interpolation
- Log the WHY, not just the WHAT: "Contact created because no match found for phone +61..." not just "Contact created"
- Error logs must include enough context to reproduce the issue without a debugger

**Ask during planning:** "If this feature breaks silently in production, how do we detect it? What log entry or metric would alert us?"

### Testability by Design (MANDATORY)

Structure code so it CAN be tested effectively. Unit tests that pass trivially without catching real bugs are a false signal. The real value is in orchestration-level and integration-level testing that validates service interactions.

**The testing pyramid for this architecture:**
```
Unit Tests            -> Pure logic, calculations, validation rules. Fast, isolated.
                        Useful but limited: a unit test passing doesn't prove the service works.

Service/Orchestrator  -> Test orchestrators with mocked repositories and mocked providers.
Integration Tests       This is the HIGHEST VALUE layer. Verifies that services coordinate correctly,
                        call the right dependencies, handle errors, and produce correct side effects.

API Integration Tests -> Full HTTP pipeline with in-memory DB and mocked external services.
                        Verifies controllers, DI wiring, auth, and end-to-end request flow.
```

**Design rules for testability:**
1. **Extract logic from untestable contexts.** If business logic lives inside a WebSocket handler, SignalR hub, or background service callback, extract it to a service method that can be tested independently. The handler becomes a thin wrapper that calls the service.
2. **External services are always behind interfaces.** Never call an external API directly. Provider interfaces enable mocking at the orchestration test level.
3. **Repositories are always behind interfaces.** Enables mocking data access in service tests without needing an in-memory database.
4. **Orchestrator tests are the priority.** An orchestrator test with mocked repos and providers verifies the entire internal workflow. If the orchestrator test passes with correct mock setups, we have high confidence the feature works.
5. **Mock at boundaries, not internally.** Mock external services (providers) and data access (repositories). Do NOT mock internal services. Let orchestrator tests exercise the real service chain.

**During planning, for each feature ask:**
- Can every orchestrator method be tested with mocked repos and providers?
- Is there business logic trapped in a handler/hub that should be extracted?
- What mocks are needed, and do the interfaces already exist?
- What side effects should the test verify? (e.g. "monologue was appended", "ChannelMessage was created")

### API Verification (BLOCKING)

Before writing ANY code snippet, verify the actual API you plan to use. Every blocker from past execution cycles traces back to code written against assumed rather than verified APIs.

**For every code snippet in the plan, verify BEFORE writing:**
1. Every method you CALL: actual name, parameter list (types and order), return type. Cite `file:line`.
2. Every type you CONSTRUCT or REFERENCE: actual property/field names and types. Cite `file:line`.
3. Every constructor you MODIFY: full current parameter list. Cite `file:line`.
4. Every local variable in scope at the code injection point. Cite `file:line`.
5. Every non-obvious assumption: sync vs async, nullable vs non-nullable, interface vs concrete.

**Verification format:**
```
[Verified] CallOrchestrator.HandleInboundCallAsync(string phoneNumber, CancellationToken ct) -> Task<Conversation> @ src/WorkyBot.Services/Orchestrators/CallOrchestrator.cs:98
[Verified] TranscriptDto { Speaker, Text, Timestamp } @ src/WorkyBot.Api/Hubs/VoiceHub.cs:881
```

**If you cannot find the actual signature after searching, STOP and note it as unverified. Do not guess.**

### Code Reuse Verification (BLOCKING)

Before proposing ANY new class, service, DTO, hub method, controller endpoint, or utility, search the codebase for existing implementations. The default is REUSE. Creation requires justification.

**Search before creating:**
1. **Services/Orchestrators:** Search for existing services that handle the same domain. Can this be a new method on an existing service rather than a new class?
2. **DTOs:** Search for existing DTOs with similar shapes. Can an existing DTO be extended or reused?
3. **Hub methods:** Search existing SignalR hub methods. Is there already a method that sends similar data to clients?
4. **Controller endpoints:** Search existing controllers in the same domain. Should this be a new action on an existing controller?
5. **Repository methods:** Search existing repositories. Does a query method already exist that returns the data needed?
6. **Utilities/Helpers:** Search for existing helper methods. String formatting, date handling, validation patterns likely already exist.

**Verification format:**
```
[Reuse] Adding HandleInboundCallAsync to existing CallOrchestrator (src/WorkyBot.Services/Orchestrators/CallOrchestrator.cs)
[Reuse] Using existing TranscriptDto (src/WorkyBot.Api/Hubs/VoiceHub.cs:45) - already has Speaker, Text, Timestamp
[New - Justified] Creating VoiceToolContext - no existing context class handles voice-specific tool state
```

**Rules:**
- Every new file proposed in the plan must state why an existing file cannot serve the purpose
- "It's cleaner as a separate class" is NOT sufficient justification. Single-responsibility does not mean single-method-per-file.
- If two existing services overlap with what you need, propose consolidation rather than a third service
- Check DependencyInjection.cs files to understand what's already registered before proposing new registrations

### Repository Performance Verification (When Plan Touches Data Access)

When the plan involves new repository methods, new queries, schema changes, or new entity configurations, verify performance characteristics BEFORE writing code. Performance defects in the repository layer are invisible during development (small datasets) and catastrophic in production.

**For every new repository method, specify during planning:**
1. **Access pattern:** Read-only, read-for-update, insert, update, delete, upsert
2. **Result cardinality:** Single entity, bounded list (specify max), or unbounded (requires pagination)
3. **Required fields:** List exact columns the caller needs (drives projection vs full entity)
4. **Filter predicates:** List WHERE conditions (drives index design)
5. **Related data:** Navigation properties needed (drives Include vs projection decision)
6. **Expected frequency:** Calls per minute (drives compiled query decision for hot paths)

**Mandatory checks:**

| # | Check | If YES | If NO |
|---|-------|--------|-------|
| 1 | Does the caller need the full entity? | Return entity | Return projected DTO with only needed fields |
| 2 | Does the caller modify the returned data? | Use change tracking | Use AsNoTracking |
| 3 | Can the result set exceed 100 rows? | MUST have pagination (keyset for deep paging) | Bounded, OK |
| 4 | Does the query join multiple collections? | Use AsSplitQuery to avoid cartesian explosion | Single query OK |
| 5 | Is this a hot path (>100 calls/sec)? | Consider compiled query | Standard query OK |
| 6 | Does this table have FK columns? | Verify FK columns are indexed | N/A |
| 7 | Is this multi-tenant? | Verify TenantId leads every composite index | N/A |
| 8 | Does this write touch >10 rows? | Use ExecuteUpdate/ExecuteDelete, not load-modify-save | Standard SaveChanges OK |
| 9 | Could a query end up inside a loop? | Batch lookups before the loop | OK |
| 10 | Does the query filter strings case-insensitively? | Use collation or EF.Functions.Like, not ToLower() | OK |

**Index Design (for new entities or schema changes):**
- Every FK column MUST have an index. EF Core does not always create these automatically.
- Composite index column order must match query patterns (most selective filter first, after TenantId).
- For soft-delete patterns, use filtered indexes: `.HasFilter("[IsDeleted] = 0")`
- For time-ordered data on PostgreSQL, consider BRIN indexes instead of B-tree.
- Covering indexes (with INCLUDE columns) for hot queries that would otherwise require key lookups.
- TenantId MUST be the leading column on every composite index in multi-tenant tables.

**Anti-patterns to reject:**
- `ToList()` followed by LINQ-to-Objects filtering (push to database)
- `Count() > 0` instead of `Any()` for existence checks
- Loading entities in a loop (batch before the loop)
- `SingleOrDefault` when `FirstOrDefault` suffices (extra scan for uniqueness check)
- Unbounded `ToListAsync()` without pagination on tables that grow
- `nvarchar(max)` columns loaded in list queries when not needed by caller

---

## Phase 3: Detailed Plan

Add technical specifics that align with the codebase:
- Technical design details
- Code snippets (aligned with codebase patterns)
- Edge cases and error handling
- Testing approach
- Implementation Checklist (IMP-NNN items)

### UI/UX Design Clarity (For Frontend Work)

**Before detailing any UI implementation, verify:**

1. **Visual Unity** - If multiple modes/states exist (create/edit, view/edit, loading/error):
   - Are they visually identical except for data?
   - Or intentionally different with documented reasons?
   - **Ask:** "Should [mode A] and [mode B] share the same layout?"

2. **Layout as Contract** - If an ASCII/visual diagram is provided:
   - Treat it as the **literal specification**
   - Match structure exactly, not just conceptually
   - **State:** "The layout diagram is the visual contract. Implementation will match it exactly."

3. **Mode-Specific Differences** - Document explicitly:
   - What's identical across all modes (layout, components, styling)
   - What differs per mode (data state, button labels, enabled/disabled)
   - **Ask:** "What should be different between [modes]? Everything else will be identical."

4. **Interaction Patterns** - Clarify before implementing:
   - Tap-to-edit vs always-editable inputs
   - Save-on-blur vs explicit save button
   - Inline validation vs form-level validation

**If any of these are unclear, ask before proceeding.**

### Code Snippet Rules

Apply X1 Code Review principles:

**Critical**
- Null safety - handle null/undefined
- Async safety - no race conditions, proper cancellation
- Resource safety - no leaks (dispose, unsubscribe)
- Input validation - at boundaries
- Security - no secrets, no injection vulnerabilities

**Architecture**
- Correct layer - business logic in services, not controllers
- Reuse existing - don't duplicate what exists
- Follow conventions - match project patterns
- Proper DI - correct lifetimes, registrations

**Quality**
- YAGNI - only what's needed now
- Simplicity - 10 lines beats 100
- No premature abstraction - interfaces need 2+ implementations
- Clear names - reveal intent

**Fail-Fast (MANDATORY)**
- No fallback values for missing configuration: throw `InvalidOperationException`, don't default
- No `?? defaultValue` for config that MUST exist at runtime
- No `try/catch` that returns a default value or empty collection: catch, log, rethrow
- No silent error swallowing: if you catch, propagate or throw a typed exception
- Missing configuration is a development defect, not a runtime condition to handle gracefully

**Code Reuse (MANDATORY)**
- Search the codebase BEFORE writing any snippet that creates a new class, interface, or method
- If similar functionality exists, adapt the snippet to extend the existing code

### Code Snippet Confidence
Mark each snippet:
- **[VERIFIED]** Written against actual API read from codebase. Cite file:line for all referenced types/methods.
- **[CONCEPTUAL]** Pseudocode showing intent. Must be verified during implementation.

Never mark a snippet [VERIFIED] if you haven't read the actual source. Phase 4 will challenge [CONCEPTUAL] snippets and may reject the plan if too many are unverified.

### Code Generation Discipline (MANDATORY)

Generate the MINIMUM code that FULLY satisfies every acceptance criterion. This is a pragmatic balance, not minimalism for its own sake.

**Too much code (wasteful):**
- Helper classes for one-time operations
- Abstractions before the second concrete use case
- Configuration options nobody asked for
- Error handling for scenarios that cannot occur given the architecture
- Wrapper classes that add no behavior
- DTOs that duplicate existing ones with minor field differences

**Too little code (incomplete):**
- Missing error handling at system boundaries
- Skipping logging at critical decision points
- Omitting null checks where null is a valid runtime state
- Not handling edge cases specified in acceptance criteria
- Leaving implicit behavior that should be explicit

**The test:** For every code snippet or IMP item, ask:
1. Which AC does this satisfy? (If none, remove it)
2. If I deleted this, would an AC fail? (If no, question whether it's needed)
3. Is there a simpler way to satisfy this AC? (If yes, use the simpler way)

### Plan Output Format

Structure plan sections to align with what Phase 4 (Chaos Demon Review) will validate:

#### Problem Statement
[What we're solving, 1-2 sentences]

#### Requirements Coverage
- **User Stories:** [List with acceptance criteria]
- **Success Criteria:** [How we know it's done, measurable]

#### Scope
- **In scope:** [list]
- **Out of scope:** [list]

#### Architecture Approach
- **Design:** [High-level approach]
- **Layer placement:** [Where code goes]
- **Reuse:** [Existing components to leverage]

#### Technical Design (if applicable)
- **Database:** [Schema changes, migrations]
- **API:** [Endpoints, request/response]
- **Error Handling:** [Strategy for failures]

#### UI/UX Design (if applicable)
- **Layout:** [ASCII diagram or reference, THIS IS THE VISUAL CONTRACT]
- **Visual Unity:** [All modes share same layout? If not, why?]
- **Mode Differences:** [Only: data state, button labels, visibility, NOT layout]
- **Interaction Pattern:** [Tap-to-edit, save-on-blur, inline validation, etc.]

#### Security and Authorization (if applicable)
- **Auth requirements:** [Who can access]
- **Multi-tenancy:** [Tenant isolation approach]

#### Files to Create/Modify
| File | Action | Purpose |
|------|--------|---------|
| path/to/file | Create/Modify | Brief description |

#### Implementation Steps
1. [Step with detail]
2. [Step with detail]

#### Code Snippets
[Only after codebase understood, aligned with existing patterns]

#### Edge Cases and Error Handling
[Failure modes considered, what happens when X fails?]

#### Testing Approach (if applicable)
[How to verify]

### Implementation Checklist

Every deliverable item numbered for traceability. This is the **contract**; `/x1-implement` must satisfy every item.

#### Files
- IMP-001: Create `path/to/file.cs` - [purpose] -> US-1
- IMP-002: Modify `path/to/existing.cs` - [what changes] -> US-1

#### Features / Business Logic
- IMP-003: [Feature/rule description] -> US-1 AC-1
- IMP-004: [Feature/rule description] -> US-1 AC-2

#### Infrastructure (DI, config, migrations)
- IMP-005: Register `ServiceName` in DI container -> US-1
- IMP-006: Add migration for [schema change] -> US-1

#### Error Handling and Edge Cases
- IMP-007: Handle [failure scenario] in [location] -> US-1 AC-3
- IMP-008: Validate [input] at [boundary] -> US-2 AC-1

#### Tests
- IMP-009: Unit test - [scenario] -> US-1 AC-1
- IMP-010: Integration test - [scenario] -> US-2 AC-1

Each IMP item traces to a User Story (US-N) and optionally an Acceptance Criterion (AC-N).

**Total: N items. Implementation is complete when all items are done, skipped (justified), or not applicable.**

**Traceability check:** Every AC must have at least one IMP item. If an AC has no IMP items, the plan is incomplete.

### Test Traceability Convention
Tests MUST reference the user story they verify using xUnit Trait attributes:

```csharp
[Trait("UserStory", "US-1")]
[Fact]
public async Task HandleVoiceWebhook_InboundCall_RoutesViaLineRental()
```

- One `[Trait("UserStory", "US-N")]` per test (required)
- Test class organization: group by component, tag by US
- IMP items for tests should specify the US/AC they trace to

### Traceability Matrix (MANDATORY)

Every plan MUST end with these two tables. They make gaps visible before review begins.

#### Table 1: Requirements to Architecture to Implementation

This traces every AC through its architectural placement to the code that implements it.

| US | AC | Layer / Service | Provider? | Observability | IMP (Code) |
|----|-----|----------------|-----------|---------------|------------|
| US-1 | AC-1 | CallOrchestrator (Orchestration) | - | Log: "Inbound call routed via LineRental" | IMP-001, IMP-002 |
| US-1 | AC-2 | CallOrchestrator (Orchestration) | - | Log: "Contact resolved: {contactId}" | IMP-003 |
| US-1 | AC-3 | TwilioMediaHandler -> extracted setup service (Technical) | IVoiceProvider | Log: "WorkPlan loaded for agent {id}" | IMP-004, IMP-005 |

**Verification checks:**
- Every AC has at least one row. If an AC has no row, the plan is incomplete.
- Every IMP (Code) item appears in at least one row. If an IMP item doesn't trace to an AC, question whether it's needed.
- The "Layer / Service" column names the actual service and its tier (Orchestration, Business, Technical, Repository). This is NOT optional.
- The "Provider?" column identifies external service dependencies. If populated, the provider interface must exist or be created.
- The "Observability" column states the key log/metric for this AC. If blank, ask: "How will we know this AC is working in production?"

#### Table 2: Implementation to Testing

This traces every testable AC to its test, including WHAT the test verifies architecturally.

| US | AC | IMP (Code) | IMP (Test) | Test Level | What Test Verifies | Gap? |
|----|-----|------------|------------|------------|-------------------|------|
| US-1 | AC-1 | IMP-001, IMP-002 | IMP-050 | Orchestrator | Mock LineRentalRepo returns bound agent; verify orchestrator passes correct agentDefinitionId downstream | No |
| US-1 | AC-2 | IMP-003 | IMP-051 | Orchestrator | Mock ContactResolver returns existing contact; verify contact ID flows to conversation creation | No |

**The "What Test Verifies" column is critical.** It forces the plan to specify what the test actually checks, not just that a test exists. This prevents vacuous tests that pass without catching real bugs.

Good: "Mock LineRentalRepo returns bound agent; verify orchestrator passes correct agentDefinitionId downstream"
Bad: "Test that inbound call works"

**Test Level must be one of:**
- **Unit**: Pure logic in isolation. Validates calculations, validation rules, data transformations.
- **Orchestrator**: Service with mocked repos and providers. Validates coordination, side effects, error handling. HIGHEST VALUE.
- **API Integration**: Full HTTP pipeline with in-memory DB and mocked external services. Validates DI wiring, auth, end-to-end flow.
- **Manual**: Cannot be automated. Must justify why.

**Coverage: X of Y ACs have automated tests (Z%)**

**For each GAP (AC without automated test):**
1. Why it cannot be tested (specific technical constraint, e.g. "WebSocket event handler is a sync delegate, cannot mock SignalR group in unit test")
2. Alternative verification method (manual test steps, structured logging, integration test)
3. Whether the untestable logic CAN be extracted to a testable helper (if yes, include extraction as an IMP item)

A plan with >30% untested ACs should justify the gap or restructure code to improve testability.

#### Orphan Check

After completing both tables:
- **Orphan IMP items:** Any IMP item that doesn't appear in Table 1 or Table 2 is orphan code. Either trace it to an AC or remove it.
- **Orphan code snippets:** Any code snippet in the plan that doesn't map to an IMP item is untracked work. Either create an IMP item or remove the snippet.
- **Orphan tests:** Any test IMP item that doesn't appear in Table 2 is testing something not traced to requirements. Either trace it or question whether it's needed.

---

## --- TRANSITION: Creative Mode -> Adversarial Mode ---

You have now completed the plan. Switch mindset completely. You are no longer the architect. You are the Chaos Demon whose job is to BREAK this plan.

**CRITICAL MINDSET**: Your job is NOT to validate the plan. Your job is to **FIND EVERY POSSIBLE WAY THIS PLAN WILL FAIL IN PRODUCTION**.

You have full memory of every assumption, every "should work" moment, every area where you verified vs guessed. Use that knowledge ruthlessly. The plan author's blind spots are now visible to you because you ARE the plan author.

---

## Phase 4: Chaos Demon Review

### The "What If" Checklist

For EVERY architectural component in the plan, run through these categories:

**Network Failures:**
- What if network is slow? (10 second latency)
- What if network drops mid-request?
- What if SSL certificate is invalid?

**Data Validation:**
- What if input is null or empty?
- What if input is 10MB of data?
- What if input contains special characters? (`<script>`, SQL injection, path traversal)

**Concurrency:**
- What if two requests happen simultaneously?
- What if user clicks submit twice?
- What if background job runs while user is modifying data?

**Dependencies:**
- What if external API is down?
- What if external API changes response format?
- What if database is at max connections?

**Resource Limits:**
- What if response is 1MB? 10MB? 100MB?
- What if 1000 users hit this endpoint simultaneously?
- What if memory usage grows unbounded?

**Observability:**
- How do we know this is broken in production?
- What logs will help debug this?
- How do we reproduce production issues locally?

### Concrete Failure Scenario Examples

**For Streaming/SSE Architecture:**
```
CHAOS DEMON ASKS:
- What if provider streams 1000 events/sec and consumer processes 10/sec? (backpressure)
- What if event #47 is malformed JSON? (skip? crash? log?)
- What if stream is interrupted midway? (partial response? retry?)
```

**For Database Operations:**
```
CHAOS DEMON ASKS:
- What transaction boundary? (before operation? after? mid-stream?)
- What if DB write fails after user saw success response?
- What if two requests create the same record simultaneously?
```

**For External API Calls:**
```
CHAOS DEMON ASKS:
- What if service is down? (return fallback? fail request? timeout?)
- What if service takes 30 seconds to respond?
- What if service rate-limits us? (circuit breaker? queue?)
- What if service returns 403/401? (does client see actual error or generic 500?)
- What if error message is in response body? (is it extracted and propagated?)
```

**For Controller -> Service -> External API:**
```
CHAOS DEMON ASKS:
- Does service throw typed exception with StatusCode? (not generic HttpRequestException)
- Does controller catch and return StatusCode(ex.StatusCode, ...)? (not letting it bubble as 500)
- Is error message from API response extracted? (not discarded by EnsureSuccessStatusCode)
- Are API errors logged with status code, reason phrase, and response body?
```

### Review Output Style

When you find issues, be SPECIFIC and CONCRETE:

**BAD:** "Error handling is missing"

**GOOD:**
```
CRITICAL BLOCKER: NO EXCEPTION HANDLING FOR SSE PARSING

PRODUCTION FAILURE SCENARIO:
1. Azure returns malformed JSON
2. JsonException thrown during parsing
3. Entire request fails with 500 error
4. User sees error after waiting 30 seconds

MISSING: try/catch, fallback behavior, logging, retry policy
```

### Red Flags That Demand Deep Scrutiny

- **"Will tackle separately"** -> Translation: "Not going to do it"
- **"Should work"** -> Translation: "Haven't thought through edge cases"
- **"Simple refactoring"** -> Translation: "30 files changed, 100 bugs"
- **"No breaking changes"** -> Translation: "Don't understand dependencies"
- **No exception handling** -> Translation: "First error crashes system"
- **No logging strategy** -> Translation: "Can't debug production"
- **No rollback plan** -> Translation: "Deploy and pray"
- **"Testing is out of scope"** -> Translation: "We're shipping bugs"

### Review Checklist

#### Plan Quality Gates
- [ ] Requirements loaded: approved `/x1-requirements` artifact present in `specs/…` before codebase investigation
- [ ] Invariant Impact Matrix present; new invariants from the Generator recorded
- [ ] API Verification: All code snippets cite actual method signatures with file:line references
- [ ] Snippet confidence: All snippets marked [VERIFIED] or [CONCEPTUAL]
- [ ] Fail-fast: No fallback patterns in code snippets (no `?? default`, no silent catch)
- [ ] Traceability matrix present: Every AC has IMP items, test coverage visible
- [ ] GAP justifications: Each untested AC has a specific technical reason and alternative verification
- [ ] Code reuse: Every new class/file justifies why existing code cannot serve the purpose
- [ ] Repository performance: New repo methods specify access pattern, cardinality, projection needs

#### [CONCEPTUAL] Snippet Verification
For every snippet marked [CONCEPTUAL] in the plan:
1. Read the actual codebase to verify assumptions
2. Promote to [VERIFIED] if correct, or flag as BLOCKER if wrong
3. If >50% of snippets are [CONCEPTUAL], the plan needs more investigation before proceeding

#### Requirements Coverage
- All user stories addressed
- All acceptance criteria have technical implementations
- Success criteria are measurable

#### Architecture Compliance
- Follows established patterns
- Correct layer placement
- Reuses existing components (DON'T duplicate)

#### Technical Design Quality
- Database schema is complete
- API design follows conventions
- Error handling is comprehensive

#### Security and Authorization
- Auth requirements specified
- Role-based access configured
- Multi-tenancy isolation addressed

#### UI/UX Design Clarity (if plan involves frontend)
- **Visual Unity Verified**: Multiple modes (create/edit, view/edit) explicitly share same layout, or differences are intentional with documented reasons
- **Layout is Contract**: ASCII diagram or visual spec treated as literal specification, not suggestion
- **Mode Differences Explicit**: Document states what's identical (layout, components) vs what differs (data state, button labels, visibility)
- **Interaction Pattern Clear**: Tap-to-edit vs always-editable, save-on-blur vs save button, inline vs form-level validation

**UI/UX Red Flags to Catch:**
- Plan says "create mode" and "edit mode" without specifying if layout differs -> ASK
- Layout diagram present but implementation describes different structures -> REJECT
- Assumes traditional form for create, display-only for view without requirement -> CLARIFY
- "Unified component" without specifying what's actually unified -> CLARIFY

### Conditional Sections (Include If Applicable)

**Data Access Patterns (if plan involves database work):**
- ORM/data access layer used consistently
- Entities in appropriate layer
- Migrations strategy documented
- Audit fields included (`CreatedBy`, `CreatedAt`, `UpdatedBy`, `UpdatedAt`)
- Transaction boundaries clearly defined

**Performance Considerations (if plan involves high-throughput or complex queries):**
- Database query optimization planned (indexes, pagination)
- Caching strategy defined
- Parallel processing for bulk operations
- Connection pooling and resource management
- Performance requirements measurable

**File Organization (if plan creates new files):**
- All new files listed with full paths
- Modified files listed separately
- File naming follows project conventions
- Folder structure follows established patterns
- No files in incorrect layers

**Domain-Specific Concerns (if applicable):**
- Multi-Tenancy: Data access filtered by tenant, cross-tenant access prevented, tenant context properly set
- AI/ML Integration: Error handling for AI service failures, rate limiting and cost management
- Compliance: Sensitive data (PII, PHI) handling, audit trail requirements, timezone handling

---

## Phase 5: Output

After the Chaos Demon review, produce the final corrected plan with review findings appended.

### Corrections Applied
List every change made to the plan as a result of the review. Be specific:
- What was wrong
- What was changed
- Why

### Blockers (Must Fix Before Coding)
List critical issues with specific failure scenarios. If all blockers were resolved during the review, state that.

### Major Concerns (Should Fix)
List significant issues that could cause problems.

### Minor Issues (Can Address During Development)
List minor improvements needed.

### Approval Decision
- **Approved** - Plan is solid, proceed to `/x1-implement`
- **Approved with Changes** - Plan corrected during review, changes documented above, proceed to `/x1-implement`
- **Rejected** - Unresolvable blockers found, must return to earlier phase

---

## Red Flags to Avoid in Plans

- "Will tackle separately" - Include it or explicitly scope out
- "Should work" - Verify with codebase investigation
- "Simple change" - Still needs proper error handling
- Code snippets without reading codebase - Will miss patterns
- Skipping user acceptance - Leads to rework
- No error handling strategy - Review will reject
- Missing success criteria - Can't measure done
- Missing user stories - Can't trace requirements to implementation
- No Implementation Checklist - Can't verify completeness

**UI/UX Red Flags:**
- "Create mode" vs "Edit mode" without specifying if layout differs - Ask first
- Layout diagram treated as "suggestion" - It's the contract
- Different structures for different modes without explicit requirement - Default to unified
- Assuming traditional form for create, display-only for view - Clarify interaction pattern

---

## When Complete

Report:
- Problem clearly articulated
- User stories captured with acceptance criteria
- Architecture approach approved by user
- Files to create/modify identified
- Code snippets aligned with codebase (if detailed plan)
- Implementation Checklist with numbered IMP-NNN items
- Chaos Demon review completed with findings
- All blockers resolved or documented
- Approval decision rendered

**Next step:** `/x1-implement`
