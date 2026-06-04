# X1 Implementation Plan

**Report:** "Using X1 Planning methodology to design implementation."

## X1 Core Principles (Apply Throughout)
- **Fail fast:** Throw on missing config. No fallbacks that hide failures.
- **Verify before assuming:** Read actual method signatures, field names, constructors from the codebase before writing code snippets.
- **Reuse before creating:** Search the codebase for existing services, DTOs, and patterns before proposing new ones. Extend, don't duplicate.
- **Minimal pragmatic code:** Generate the minimum code that fully satisfies all acceptance criteria. No more, no less.
- **Full traceability:** US -> AC -> IMP -> Test. Every link visible, every AC covered.
- **User stories first:** Requirements are locked before technical investigation begins.
- **Observable:** Every feature must answer "How will we know this is broken in production?"
- **Testable by design:** Structure code so orchestration-level tests with mocked boundaries can verify real workflows.

This skill owns the **technical** back of the X1 pipeline. It reads the **approved requirements artifact** produced by `/x1-requirements` and authors the architecture and implementation plan(s). It does NOT elicit or re-open user stories — those are locked upstream.

## Instructions

1. **Load approved requirements** - Read the requirements artifact from `specs/…` (see Phase 0). If none exists, STOP and tell the user to run `/x1-requirements` first.
2. **Present high-level plan** - Conceptual approach, no code, no file paths. Wait for acceptance.
3. **Investigate codebase** - Only after high-level accepted. Find patterns, reuse.
4. **Identify invariants** - Run the Invariant Impact Matrix (reuse known) and the Invariant Generator (derive new) against the catalog at `docs/architecture/architectural-principles.md`.
5. **Decide cardinality** - Single implementation → one plan inline; multiple → emit an architecture spine + slice manifest + coverage ledger, then plan each slice.
6. **Add detail iteratively** - Technical specifics, code snippets, Implementation Checklist.
7. **Iterate until approved** - Refine based on feedback.

**Goal:** Codebase-aligned plan with traceable Implementation Checklist and an honoured invariant set, ready for `/x1-implement`.

**Speed:** Work in parallel wherever possible - read multiple files simultaneously, search client and server concurrently, research while investigating code.

---

## Phase 0: Load Approved Requirements (BLOCKING)

This skill begins from locked requirements. Do not re-elicit or edit stories here.

1. **Read the requirements artifact** — `specs/user-story.md` (or `specs/<feature>-requirements.md`): the epoch prefix, `US-{EPOCH}-N`, `AC-{EPOCH}-N.M`, the stress scenarios, gap analysis, and rationale. This is the single source of truth for ACs; the plan never invents an AC.
2. **If it is missing or stories are not approved** → STOP: "No approved requirements found. Run `/x1-requirements` first."
3. **If a gap surfaces** mid-plan → the new AC goes BACK to `/x1-requirements` for re-approval, then returns here. Slices never mint ACs locally.

Carry the epoch prefix forward — it is the key that joins requirements → plan(s) → tests.

---

## Phase 1: High-Level Plan

**This phase begins ONLY after Phase 0 — the approved requirements artifact is loaded.**

Present the approach at a conceptual level. No code. No file paths. Just the shape of the solution.

### Problem Statement
[1-2 sentences: what we're solving and why]

### User Stories
[List with acceptance criteria — loaded from the `/x1-requirements` artifact in `specs/…`, not re-authored here]

### Proposed Approach
[Conceptual design — how will we solve this? What's the strategy?]

### Scope
- **In scope:** [list]
- **Out of scope:** [list]

### Key Decisions
[Architectural choices, trade-offs, alternatives considered]

**Ask:** "Does this direction feel right? Any concerns before I investigate the codebase and add detail?"

**Wait for acceptance before proceeding to Phase 2 (Codebase Investigation).**

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
4. Provider pattern enables switching service providers (e.g. OpenAI to Anthropic, Twilio to another voice provider) without rewriting consuming code
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

### Test-Driven Development (MANDATORY)

This project uses TDD. The plan must produce IMP items that enforce the red-green cycle.

**What TDD means for planning:**
- Each AC is a test specification. The plan's job is to make ACs precise enough that a failing test can be written directly from the AC, before any implementation code exists.
- The IMP checklist pairs [TEST] and [CODE] items per AC. The test comes first.
- "Testability by Design" (above) ensures the architecture supports TDD. This section ensures the plan enforces it.

**ACs that resist TDD are a design smell:**
- If you can't write a failing test from an AC, the AC is too vague. Tighten it.
- If the test requires complex setup across many services, the architecture has too much coupling. Revisit service boundaries.
- If the only way to test is to inspect internal state, the behavior isn't observable enough. Add an observable output (return value, event, log, persisted state).

**Exempt from TDD:** Pure infrastructure (DI wiring, config, migrations). These are tested implicitly when the first [TEST] item runs.

### Invariant Impact Matrix (MANDATORY — run BEFORE stress testing)

Reuse the project's invariant catalog so **known** invariants are never re-derived. Load `docs/architecture/architectural-principles.md` (§1 runtime invariants, §2 boundary rules). Produce one table for this feature — each catalog invariant × *does this feature touch it, and how is it honoured?*

| Invariant | Touched? | How honoured |
|-----------|----------|--------------|
| INV-DET-1 byte-identical replay | yes/no | ... |
| INV-CC-1 single writer per stream | yes/no | ... |
| ... walk every §1/§2 entry ... | | |

- Most features touch 5–8 invariants. **"Touched but not honoured" is a BLOCKER** — fix the design before proceeding.
- Each touched invariant's ***Observed by*** signal becomes a test case in `/x1-test-plan` (the invariant chain, alongside the AC chain).

### Invariant Generator (MANDATORY — derive NEW invariants at design time)

The Matrix reuses KNOWN invariants; the Generator derives **new** ones *before build*, so the harness confirms invariants rather than discovering them late. Enumerate the feature's architectural **deltas** and run each past the generator questions:

| When the design introduces… | Ask… | → candidate invariant family |
|---|---|---|
| new persisted state / fold field | how is it derived? settled or transient? what must it never contradict? | ES / GR |
| a new writer or shared mutable point | who else can touch this concurrently? what serializes it? | CC |
| a read of anything that differs run-to-run (time, random, network order, external id) | **does this enter the replayed request?** how is it frozen/injected? | DET |
| a new event / stream / bracket | what's the single terminal? can it double? what's transient? | GR |
| a new input source or external sender | is it trusted? where is provenance stamped? what side effects can it reach? | TR |
| a new side effect / external action | what if it runs twice, or crashes mid-way? what's the idempotency key? | CR |
| a new status / lifecycle | what are the illegal transitions — internal (throw) or external (ignore)? | SM |
| a new loop / fan-out / recursion / cost driver | what bounds it? what happens at the bound? | EXE |
| a new mandatory dependency or default | what if missing/wrong — does it fail loud at startup? | CFG |
| a new tenant / scope / partition | can data or reach cross this boundary? | ISO |

Plus two reflexes: **negate every Key Decision** ("we chose X; X is only safe if ___" → the blank is a candidate invariant) and **harvest the stress test below** (each seam failure → the invariant the redesign must preserve).

New invariants become **design constraints now** and **harness assertions later**. Durable ones are promoted into `architectural-principles.md` via the **ratchet** (triggered at `/x1-implement` completion, formalized by `/x1-architecture-review`).

### Architecture Stress Testing (MANDATORY)

**After the first-pass architecture is designed, stress test it with end-to-end scenarios before writing code snippets.**

Individual services work well internally. The system breaks at the seams, where services hand off to each other. Stress testing targets these integration points by running scenarios that cross multiple service boundaries simultaneously.

**Process:**
1. Design the initial service decomposition (first pass)
2. Generate 5-8 scenarios that cross at least two service boundaries
3. Walk each scenario through the architecture step by step, tracking state at every handoff point
4. At each handoff ask: "What is the state assumption here? What if it's wrong?"
5. Identify seam failures: temporal assumptions, ordering assumptions, state assumptions
6. Redesign service boundaries based on failures found
7. Re-run scenarios against the revised architecture

**What to look for at the seams:**
- **Temporal assumptions:** Service A assumes Service B has finished before it acts. What if B is still in progress?
- **Ordering assumptions:** The design assumes A fires before B. What if the scheduler triggers B first?
- **State assumptions:** Service A reads data, then Service B modifies it, then Service A acts on stale data.
- **Conflicting intents:** Two services are both active with opposing goals (one is suspending, another is starting new work).
- **Partial completion:** A multi-step operation fails midway. What state are the downstream services left in?

**Example:**
```
Architecture: OrderService (lifecycle) + BillingGuard (budget) + Scheduler (dispatch)

Scenario: "Billing threshold hit while scheduler is dispatching next item"
1. Scheduler evaluates: 3 items due, picks item A, begins dispatch
2. Meanwhile, item B completes and BillingGuard runs post-completion check
3. BillingGuard determines budget is exhausted, calls OrderService.Suspend()
4. OrderService deactivates all schedules
5. But Scheduler already picked item A in step 1 and is mid-dispatch
   -> SEAM FAILURE: Scheduler holds a reference to a schedule that was just deactivated
   -> FIX: Scheduler must check schedule.IsActive before executing, or suspension
      must be idempotent for in-flight items
```

**The first-pass architecture is rarely optimal.** Expect to revise service boundaries, split or merge services, and move responsibilities based on what the stress scenarios reveal. This is the cheapest place to make these changes.

### Service Design Elegance (MANDATORY)

The best architecture emerges when testability drives the design. If a service is hard to test, the design is wrong. Do not work around bad design with complex test setup; redesign the service.

**The elegance test: Can someone read the service constructor and immediately understand what it does?**

A service with 8 injected dependencies is a service trying to do too much. Before accepting a service design, challenge it:

1. **Single-axis responsibility.** Each service should have ONE reason to change. If a service handles both lifecycle orchestration AND policy evaluation, split it. Each focused service becomes independently testable with 3-4 mocks.

2. **Dependencies reveal design quality.** Count the constructor parameters. If a service needs >5 dependencies, it is likely doing too much. Challenge: can this be split into focused services that compose?

3. **Decisions as pure functions where possible.** Policy decisions (should this fire? is a threshold exceeded?) are pure logic that takes data in and returns a decision. Extract them from services that have side effects. Pure decision functions are trivially testable without mocks.

4. **Side effects at the edges.** Services that make decisions should not also execute those decisions. A guard service decides "suspend this operation" but delegates to a lifecycle service to do it. This separation means guard logic tests never need to verify database state.

5. **Challenge the initial design.** The first service decomposition is rarely optimal. During planning, propose the initial design, then immediately ask: "If I were writing tests for this, what would be painful?" Painful test setup signals design problems. Redesign before coding.

**During planning, for each new service ask:**
- How many mocks does the test need? (Target: 3-4. If >5, redesign.)
- Can the core logic be tested without mocking anything? (Extract pure functions.)
- Does this service mix decisions with side effects? (Split them.)
- Would a new developer understand this service's purpose from the constructor alone?

### Architectural Principles Compliance (MANDATORY)

The plan MUST comply with the project's architectural principles defined in `docs/architecture/architectural-principles.md`. During architecture design, verify each principle:

**Architectural Requirements:**
- **AR-1 Single Responsibility:** Each new component owns exactly one concern. Constructor dependencies reveal scope.
- **AR-2 Dependency Inversion:** All collaborators are interfaces. No concrete dependencies.
- **AR-3 Composition Over Inheritance:** Compose through injection, not class hierarchies.
- **AR-4 Open-Closed Principle:** New capabilities via extension, not modification of existing components.
- **AR-5 Explicit State Machines:** Lifecycle states are explicit with defined transitions. No implied state from booleans.
- **AR-6 Event-Driven Architecture:** Activity produces events through a pipeline. Producers don't know consumers.
- **AR-7 Provider Encapsulation:** Providers own retry, fallback, and recovery internally.

**Non-Functional Requirements:**
- **NFR-1 Testability:** Max 5 mocked dependencies per component. Pure decision logic extracted as pure functions.
- **NFR-2 Observability:** Every state transition and decision logged with structured context.
- **NFR-3 Resilience:** Transient failures retried. Permanent failures surfaced as events. No silent failures.
- **NFR-4 Resource Efficiency:** Idle resources unloaded. Bounded in-memory counts.
- **NFR-5 Concurrency Safety:** Mutual exclusion via state machines. Thread-safe collections for queuing.
- **NFR-6 Persistence and Recovery:** Persisted state is source of truth. Incremental persistence. Lazy reload.
- **NFR-7 Real-Time Responsiveness:** Events stream with minimal latency. No buffering.
- **NFR-8 Deterministic Lifecycle:** No ambiguous states. Invalid transitions throw.

Not every principle applies to every plan. State which are relevant and verify compliance for those.

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

## Phase 2.5: Cardinality & Decomposition

Once the high-level architecture is sound and the invariant set is settled, decide whether this is **one** implementation or **many**. One requirements artifact can fan out to multiple implementations, each with its own checklist.

**N = 1 (single implementation):** continue inline. This skill produces one plan doc (`docs/plans/<feature>-x1-plan.md`) with the full IMP checklist. No spine, no ledger. Stop reading this section.

**N > 1 (multiple implementations):** the high-level architecture **is the spine**. Author `docs/plans/<feature>-spine.md` and STOP the detailed checklist here; then re-invoke `/x1-plan --slice <TAG>` once per slice. The spine owns everything shared, so slices never duplicate it:

1. **Shared components** — the building blocks ≥2 slices need (a delivery path, a binding table, a port). Identified here, owned by a foundation slice, consumed by reference — never re-implemented per slice.
2. **Invariants** — the Impact-Matrix + Generator output. Cross-cutting, so owned by the spine/catalog; slices reference them, never re-derive.
3. **Slice manifest** — each slice = a **vertical** capability (delivers ACs end-to-end), never a horizontal layer. Declare the slice dependency DAG; implement foundation/shared slices first.
4. **Cross-slice integration ACs** — the seam scenarios that cross slice boundaries. **Owned by the spine**, tested as integration tests, owned by no single slice. (Gaps live in the seams; give the seams an owner.)
5. **Coverage ledger** — see below.

### Coverage Ledger (MANDATORY when N > 1)

The cross-document join that makes traceability, dedup, and completeness *computable* across the fan-out. Lives in the spine doc.

```
## Coverage ledger
| AC | Owning slice | Consuming slices | Status |
|----|--------------|------------------|--------|
| AC-PLAT-3.1 | PLAT.CHAN | PLAT.SUB, PLAT.MON | planned |
| AC-PLAT-4.2 | PLAT.SCHED | — | planned |
```

- **Forward check (completeness):** every requirements AC appears with **exactly one** owning slice. **0 owners = gap (BLOCK). >1 owner = duplication** → resolve to one owner + listed consumers.
- **Backward check (no orphans):** every `IMP-{EPOCH}.{SLICE}-NNN` traces to an AC; a slice's owned-AC set ⊆ requirements ACs (slices never invent ACs).
- Each slice plan declares, up top: the ACs it **owns** (from the manifest), the spine components it **consumes**, and the invariants **in scope** (from the spine).

## Phase 3: Detailed Plan (On Acceptance)

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
- Cross-reference: see x1-code-review "No Fallback Code" for enforcement during review

**Code Reuse (MANDATORY)**
- Search the codebase BEFORE writing any snippet that creates a new class, interface, or method
- If similar functionality exists, adapt the snippet to extend the existing code
- Cross-reference: see x1-code-review "Code Reuse (MANDATORY)" for enforcement during review

### Code Snippet Confidence
Mark each snippet:
- **[VERIFIED]** Written against actual API read from codebase. Cite file:line for all referenced types/methods.
- **[CONCEPTUAL]** Pseudocode showing intent. Must be verified during implementation.

Never mark a snippet [VERIFIED] if you haven't read the actual source. x1-review-plan will challenge [CONCEPTUAL] snippets and may reject the plan if too many are unverified.

---

## Phase 4: Iterate

- Refine based on feedback
- Update plan sections as needed
- Continue until user says "ready to implement"

---

## Plan Output Format

Structure plan sections to align with what `/x1-review-plan` will validate:

### Problem Statement
[What we're solving - 1-2 sentences]

### Requirements Coverage
- **User Stories:** [List with acceptance criteria]
- **Success Criteria:** [How we know it's done - measurable]

### Scope
- **In scope:** [list]
- **Out of scope:** [list]

### Architecture Approach
- **Design:** [High-level approach]
- **Layer placement:** [Where code goes]
- **Reuse:** [Existing components to leverage]

### Technical Design (if applicable)
- **Database:** [Schema changes, migrations]
- **API:** [Endpoints, request/response]
- **Error Handling:** [Strategy for failures]

### UI/UX Design (if applicable)
- **Layout:** [ASCII diagram or reference - THIS IS THE VISUAL CONTRACT]
- **Visual Unity:** [All modes share same layout? If not, why?]
- **Mode Differences:** [Only: data state, button labels, visibility - NOT layout]
- **Interaction Pattern:** [Tap-to-edit, save-on-blur, inline validation, etc.]

### Security & Authorization (if applicable)
- **Auth requirements:** [Who can access]
- **Multi-tenancy:** [Tenant isolation approach]

### Files to Create/Modify
| File | Action | Purpose |
|------|--------|---------|
| path/to/file | Create/Modify | Brief description |

### Implementation Steps
1. [Step with detail]
2. [Step with detail]

### Code Snippets
[Only after codebase understood - aligned with existing patterns]

### Edge Cases & Error Handling
[Failure modes considered - what happens when X fails?]

### Testing Approach (if applicable)
[How to verify]

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

**Pragmatic balance:**
- Write enough code to satisfy ACs completely
- Include logging at decision points (observability)
- Include error handling at boundaries (fail-fast)
- Do NOT include speculative features, future-proofing, or "nice to have" code
- Prefer modifying 3 lines in an existing service over creating a new 50-line class

### Implementation Checklist

Every deliverable item numbered for traceability. This is the **contract** — `/x1-implement` must satisfy every item.

#### Infrastructure (DI, config, migrations) — implement first, prerequisites for TDD cycle
- IMP-{PREFIX}-001: Register `ServiceName` in DI container → US-{PREFIX}-1
- IMP-{PREFIX}-002: Add migration for [schema change] → US-{PREFIX}-1

#### AC-{PREFIX}-1.1: [AC description]
- IMP-{PREFIX}-003: [TEST] Integration test — [what it verifies E2E] → AC-{PREFIX}-1.1 (write FIRST, expect RED)
- IMP-{PREFIX}-004: [CODE] [Implementation description] → AC-{PREFIX}-1.1 (write SECOND, expect GREEN)

#### AC-{PREFIX}-1.2: [AC description]
- IMP-{PREFIX}-005: [TEST] Integration test — [what it verifies E2E] → AC-{PREFIX}-1.2 (write FIRST, expect RED)
- IMP-{PREFIX}-006: [CODE] [Implementation description] → AC-{PREFIX}-1.2 (write SECOND, expect GREEN)

Each IMP item traces to a User Story and Acceptance Criterion via the epoch-scoped ID.

**Slice-tagged IDs (when N > 1).** In a multi-slice feature, IMP ids embed the slice tag so the source plan is visible by inspection and ids never collide across slices: `IMP-{EPOCH}.{SLICE}-NNN` (e.g. `IMP-PLAT.SCHED-014`). For a single-implementation feature, the plain `IMP-{PREFIX}-NNN` form is used. The owning slice for each AC comes from the spine's coverage ledger — a slice's checklist only contains ACs it **owns**.

**TDD Rules:**
- Every AC produces at least one `[TEST]` + `[CODE]` pair
- `[TEST]` items are always listed before their corresponding `[CODE]` items
- Infrastructure IMP items come before all test-code pairs (they're prerequisites)
- The `[TEST]` and `[CODE]` tags make TDD order explicit and auditable

**Implementation order:**
1. Infrastructure (DI, config, migrations, schema) — prerequisites
2. For each AC, in TDD cycle:
   a. Write the [TEST] item (expect RED — test fails because code doesn't exist yet)
   b. Write the [CODE] item (expect GREEN — test passes)
   c. Refactor if needed
3. Repeat for next AC

**Total: N items. Implementation is complete when all items are ✅, ⏭️ (justified), or ➖.**

**Traceability check:** Every AC must have at least one [TEST]+[CODE] IMP pair. If an AC has no IMP items, the plan is incomplete.

### Test Traceability Convention
Tests MUST reference the user story they verify using xUnit Trait attributes:

```csharp
[Trait("UserStory", "US-CTX-1")]
[Fact]
public async Task ProcessLoop_MultiTurnConversation_RetainsContextAcrossTurns()
```

- One `[Trait("UserStory", "US-{PREFIX}-N")]` per test (required, epoch-scoped)
- In a multi-slice feature, add `[Trait("Slice", "{EPOCH}.{SLICE}")]` so tests group by slice
- Test class organization: group by component, tag by US (and slice)
- IMP items for tests should specify the US/AC they trace to
- Invariant tests: each invariant the feature touches (Impact Matrix) gets a test realizing its *Observed by*; `/x1-test-plan` authors these alongside the AC tests

### Traceability Matrix (MANDATORY)

Every plan MUST end with these two tables. They make gaps visible before review begins.

#### Table 1: Requirements to Architecture to Implementation

This traces every AC through its architectural placement to the code that implements it.

| US | AC | Layer / Service | Provider? | Observability | IMP (Code) |
|----|-----|----------------|-----------|---------------|------------|
| US-VC-1 | AC-VC-1.1 | CallOrchestrator (Orchestration) | - | Log: "Inbound call routed via LineRental" | IMP-VC-001, IMP-VC-002 |
| US-VC-1 | AC-VC-1.2 | CallOrchestrator (Orchestration) | - | Log: "Contact resolved: {contactId}" | IMP-VC-003 |
| US-VC-1 | AC-VC-1.3 | TwilioMediaHandler -> extracted setup service (Technical) | IVoiceProvider | Log: "WorkPlan loaded for agent {id}" | IMP-VC-004, IMP-VC-005 |

**Verification checks:**
- Every AC has at least one row. If an AC has no row, the plan is incomplete.
- Every IMP (Code) item appears in at least one row. If an IMP item doesn't trace to an AC, question whether it's needed.
- The "Layer / Service" column names the actual service and its tier (Orchestration, Business, Technical, Repository). This is NOT optional.
- The "Provider?" column identifies external service dependencies. If populated, the provider interface must exist or be created.
- The "Observability" column states the key log/metric for this AC. If blank, ask: "How will we know this AC is working in production?"
- **Multi-slice (N > 1):** add a **Slice** column; an AC's IMP items all belong to its one **owning** slice (per the coverage ledger). An AC owned by >1 slice is duplication — resolve it.

#### Table 2: Implementation to Testing

This traces every testable AC to its test, including WHAT the test verifies architecturally.

| US | AC | IMP [CODE] | IMP [TEST] | Test Level | What Test Verifies | Gap? |
|----|-----|------------|------------|------------|-------------------|------|
| US-VC-1 | AC-VC-1.1 | IMP-VC-004 | IMP-VC-003 | Orchestrator | Mock LineRentalRepo returns bound agent; verify orchestrator passes correct agentDefinitionId downstream | No |
| US-VC-1 | AC-VC-1.2 | IMP-VC-006 | IMP-VC-005 | Orchestrator | Mock ContactResolver returns existing contact; verify contact ID flows to conversation creation | No |

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

**Invariant coverage:** every invariant marked *Touched* in the Invariant Impact Matrix has at least one test realizing its *Observed by* signal. An untested touched-invariant is a gap, justified or closed like any AC gap.

#### Orphan Check

After completing both tables:
- **Orphan IMP items:** Any IMP item that doesn't appear in Table 1 or Table 2 is orphan code. Either trace it to an AC or remove it.
- **Orphan code snippets:** Any code snippet in the plan that doesn't map to an IMP item is untracked work. Either create an IMP item or remove the snippet.
- **Orphan tests:** Any test IMP item that doesn't appear in Table 2 is testing something not traced to requirements. Either trace it or question whether it's needed.
- **Cross-document checks (N > 1, against the spine's coverage ledger):** *forward* — every requirements AC has exactly one owning slice (0 = gap → BLOCK; >1 = duplication → resolve); *backward* — every `IMP-{EPOCH}.{SLICE}` traces to an AC the slice owns; no slice invents an AC.

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
- Approved requirements loaded from `/x1-requirements` (epoch + ACs)
- Architecture approach approved by user
- Invariant Impact Matrix completed; new invariants from the Generator recorded
- Cardinality decided (single plan, or spine + slice manifest + coverage ledger)
- Files to create/modify identified
- Code snippets aligned with codebase (if detailed plan)
- Implementation Checklist with numbered IMP-NNN items

**Inputs:** the approved requirements artifact (`specs/…`) + the invariant catalog (`docs/architecture/architectural-principles.md`).
**Next step:** `/x1-review-plan` to validate before implementation (or `/x1-plan-to-review` to plan+review in one pass).

---

## Congruence with X1 Workflow

This plan reads the `/x1-requirements` artifact and is structured so downstream skills can validate:

| Plan Section | Downstream Skill |
|--------------|------------------|
| User Stories + Acceptance Criteria (loaded from `/x1-requirements`) | `/x1-review-plan` Requirements Coverage |
| Invariant Impact Matrix + Generator | `/x1-review-plan` Invariant/Catalog gate · `/x1-test-plan` invariant tests |
| Coverage ledger (N > 1) | `/x1-review-plan` + `/x1-audit-plan` forward/backward coverage |
| Architecture Approach + Layer Placement | `/x1-review-plan` Architecture Compliance |
| Database + API + Error Handling | `/x1-review-plan` Technical Design Quality |
| UI/UX Design (layout, unity, modes) | `/x1-review-plan` UI/UX Design Clarity |
| Auth + Multi-tenancy | `/x1-review-plan` Security & Authorization |
| Files to Create/Modify | `/x1-review-plan` File Organization |
| Edge Cases | `/x1-review-plan` Chaos Demon "What If" |
| Implementation Checklist (IMP items) | `/x1-implement` Completion Gate |
| Implementation Checklist (IMP items) | `/x1-audit-plan` Gap Analysis |
