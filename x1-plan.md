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

## Instructions

1. **Discover the problem** - Spend time here. Don't rush to solutions.
2. **Challenge and think laterally** - Be a thinking partner, not a yes-machine.
3. **Elicit user stories** - At least one with acceptance criteria before proceeding.
4. **Present high-level plan** - Conceptual approach, no code, no file paths.
5. **Wait for acceptance** - Don't investigate codebase until direction approved.
6. **Investigate codebase** - Only after high-level accepted. Find patterns, reuse.
7. **Add detail iteratively** - Technical specifics, code snippets, Implementation Checklist.
8. **Iterate until approved** - Refine based on feedback.

**Goal:** Clear problem understanding, codebase-aligned plan with traceable Implementation Checklist, ready for `/x1-implement`.

**Speed:** Work in parallel wherever possible - read multiple files simultaneously, search client and server concurrently, research while investigating code.

---

## Phase 1: Problem Discovery

Spend time here. Don't rush to solutions. Understand the problem domain.

### What problem are you solving?
- What's the current state? What's wrong with it?
- What's the desired state? What does success look like?
- Who is affected? What's the impact?

### Why does this matter?
- What happens if we don't solve this?
- What's the cost of the current state?
- Are there upstream/downstream effects?

### What have you already tried or considered?
- Previous approaches and why they didn't work
- Constraints you've already identified
- Related work or prior art

### Probe deeper — ask the questions the user hasn't thought of:
- "You mentioned X — what happens when Y?"
- "Who else is affected by this besides the obvious users?"
- "What's the simplest version of this that would still be valuable?"
- "Is there a reason this hasn't been solved before?"
- "What would make this problem go away entirely vs just managing it?"

### Challenge and think laterally

**Be a critical thinking partner, not a yes-machine.**

- If an idea seems over-engineered, say so: "This could work, but have you considered [simpler approach]?"
- If the problem can be reframed, offer alternatives: "Another way to look at this is..."
- If the user is solving a symptom, probe the root cause: "Is the real problem actually X rather than Y?"
- Think out-of-the-box — propose approaches the user hasn't considered
- Present trade-offs honestly: "Approach A is simpler but limits X. Approach B is more work but gives you Y."
- If something seems like a bad idea, push back respectfully: "I'd caution against that because [reason]. Here's what I'd suggest instead."
- If the user still wants their approach after hearing alternatives, respect that — it's their codebase

**The most valuable thing in early planning is divergent thinking. Don't converge on a solution too quickly. Explore the problem space.**

**DO NOT propose final solutions in this phase. Understand and challenge first.**

### User Story Workshop (BLOCKING GATE)

**ALL user stories with acceptance criteria must be collaboratively refined and approved before proceeding.**

This is the most important phase of the entire X1 workflow. Requirements errors caught here cost 1x to fix. The same errors caught during implementation cost 15x. In production, 100x.

**Your role: Analyst and critical thinking partner.**

You are not a passive scribe waiting for the user to dictate stories. You are an analyst who:
- **Challenges every story:** "Do we actually need this? What happens if we don't build it?"
- **Questions assumptions:** "You said X. What about Y? Have you considered Z?"
- **Pushes for minimalism:** Fewer stories with precise ACs beats many stories with vague ones. Less is more, but not so little that nuance is lost.
- **Identifies hidden stories:** "You haven't mentioned [scenario]. Is that in scope or intentionally excluded?"
- **Splits bloated stories:** If a story has 7+ ACs, it's probably 2-3 stories. Challenge it.
- **Merges overlapping stories:** If two stories share most ACs, they might be one story.
- **Questions the "so that":** If the benefit is vague ("so that it works better"), push for specificity.
- **Eliminates gold-plating:** If an AC describes a nice-to-have, challenge whether it belongs in this iteration.

**Roles must be human stakeholders.**
- The role in "As [role]" must be a real human: developer, operator, business owner, end user, support agent, etc.
- "As the system" and "As an AI agent" are **never valid roles**. Reframe: who is the human that observes, benefits from, or is harmed by this behavior?
- For internal/infrastructure work, the stakeholder is typically a developer (maintainability, testability) or an operator (reliability, observability).
- Example reframe: "As the system, I need context compaction..." becomes "As an operator, I need the system to manage conversation context size, so that agents don't exceed model limits and fail silently."

**Acceptance criteria must describe testable end-to-end outcomes, not implementation details.**
- ACs can be technical, but they must test the pipeline across service boundaries, not a single unit in isolation.
- ACs must not name specific classes, methods, fields, parameters, or database columns. Those details belong in the IMP checklist.
- Litmus test: can this AC be verified by an integration/E2E test that exercises the real flow across service boundaries? If it can only be verified by inspecting a single class's internal state, it's an implementation detail, not an acceptance criterion.
- ACs feed directly into `/x1-test-plan` which builds acceptance tests from them. "Field X added to class Y" produces a trivial unit assertion. "Given X, when Y, then Z is observable" produces a test that catches real seam failures.
- **Why this matters:** Testing breaks down at the seams between services. An AC that names a single class drives a unit test for that class. An AC that describes an E2E outcome drives an integration test that catches the real failures: incorrect handoffs, stale state, missing wiring, broken DI.

**Bad example (system perspective, unit-level ACs, no epoch):**
```
US-1: Agent Retains Conversation Context Between Turns
As an AI agent, I need my conversation context to persist in my
in-memory message buffer across turns...

AC-1.1: After a multi-turn completes, assistant responses and tool
results from StepFinishEvent.NewMessages are added to the instance's
_messages list.

AC-1.2: AgentInstanceRecord.LastContextSizeTokens is updated after
each multi-turn with the last step's InputTokens.
```
Problems: "As an AI agent" is not a human. ACs name internal classes/fields. Tests would only verify a single unit's state, missing seam failures. IDs will collide with every other plan's US-1.

**Good example (human perspective, E2E ACs, epoch-scoped):**
```
Epoch: CTX

US-CTX-1: Agents Remember Prior Conversation
As an operator, I need agents to retain conversation context across
turns, so that they don't repeat questions or lose track of prior work.

AC-CTX-1.1: Given a multi-turn conversation, when the agent receives
a follow-up referencing earlier context, the agent responds using that
context without re-fetching data.

AC-CTX-1.2: Context size is tracked per instance so compaction
decisions can be made without loading full message history.

AC-CTX-1.3: When context exceeds the compaction threshold, the system
compacts before the next execution, and the agent continues
functioning with the compacted context.
```
ACs are technical but testable E2E. They exercise hydration + execution + persistence together, catching seam failures. Epoch-scoped IDs are unique across all plans.

**The goal is the MINIMAL set of stories that captures the FULL problem. Not more, not fewer.**

**Process:**
1. Listen to the problem discovery conversation
2. Draft an initial set of user stories. Be opinionated. Propose what you think is right.
3. For each story, state WHY you included it and what you'd lose by cutting it
4. Challenge the user: "I think US-3 could be merged into US-1. Here's why..."
5. Ask probing questions about assumptions: "You said contacts should be resolved. What happens with anonymous callers? Is that a separate story or an AC on the existing one?"
6. Iterate. Add, remove, merge, split based on discussion.
7. When both sides agree the stories are tight, present the final Gate Document
8. Ask: "Are these user stories and acceptance criteria complete and accurate? I will not investigate the codebase until you confirm."
9. Repeat from step 6 if the user has changes

**Analyst Questions to Ask (use judgment, not all apply to every feature):**
- "What's the simplest version of this that would still solve the core problem?"
- "If we shipped only US-1, would users get value? Or is US-2 required for it to make sense?"
- "This AC seems like an implementation detail, not a user-facing behavior. Can we reframe it?"
- "Are there scenarios you haven't mentioned that would break this?"
- "Who else is affected besides the primary user?"
- "What does the user do TODAY without this feature? What's the actual pain?"
- "Can this AC be verified by a test, or is it too vague to test?"

**Output format (Gate Document):**

```
## User Stories (APPROVED)

**Epoch:** {PREFIX} (short code derived from feature name, e.g. CTX, AUTH, VC)

### US-{PREFIX}-1: [Title]
**As** [human stakeholder role], **I want** [capability], **so that** [benefit].

**Acceptance Criteria:**
- AC-{PREFIX}-1.1: [Testable end-to-end outcome]
- AC-{PREFIX}-1.2: [Testable end-to-end outcome]

### US-{PREFIX}-2: [Title]
...

**User confirmed: [date/time or "Yes, approved"]**
```

**Rules:**
- Define the **epoch prefix** before writing any stories. Derive it from the feature name (2-4 uppercase letters). All IDs use this prefix throughout the plan.
- Do NOT investigate the codebase until user stories are approved
- Do NOT reference file paths, class names, or technical implementation in stories or ACs
- Do NOT write code snippets or propose architecture
- Do NOT discuss "how" until the "what" is locked
- Roles must be **human stakeholders**, never "the system" or "an AI agent"
- ACs must describe **testable E2E outcomes** that exercise the real flow across service boundaries. No class names, method names, field names, or database columns in ACs.
- Each AC drives a [TEST]+[CODE] IMP pair (TDD: test written first)
- Push back on stories that are too broad, too narrow, or unnecessary
- Every story must justify its existence: what do we lose if we cut it?

User stories are the foundation of the entire X1 workflow:
- Plan items trace back to user stories
- Implementation Checklist items map to acceptance criteria
- Tests verify acceptance criteria are met
- Audit verifies traceability end-to-end

**The user story gate is the foundation of the entire X1 workflow.**

### Stress Test User Stories (BLOCKING GATE)

**After user stories are approved but BEFORE architecture begins, stress test the acceptance criteria with end-to-end scenarios.**

Requirements that look complete in isolation break when composed. The goal is to find gaps in the ACs by running realistic scenarios that cross multiple stories and multiple concerns simultaneously.

**Process:**
1. For each user story, generate 3-5 stress scenarios that combine multiple ACs, concurrent operations, or edge conditions
2. Walk each scenario through the acceptance criteria step by step
3. At each step ask: "Is there an AC that covers what happens here? If not, we have a gap."
4. Add missing ACs or refine existing ones based on gaps found
5. Re-present updated stories for approval

**Stress scenario categories:**
- **Temporal collisions:** Two things happening at the same time (budget hit while agent is mid-call, campaign paused while contact is being added)
- **State transitions under load:** What happens at boundaries (last contact completes, budget hits exactly 100%, daily limit reached mid-batch)
- **Failure during multi-step operations:** Operation fails halfway through (instance creation succeeds for 5 of 10 contacts, then fails)
- **Ordering assumptions:** Does the system assume A happens before B? What if B happens first?
- **Cascading effects:** Action on entity A triggers changes to entities B, C, D. Are all downstream effects covered by ACs?

**Example:**
```
Scenario: "Budget exhausted mid-batch-operation"
1. System is processing 10 items with a $50 budget
2. Item 7 completes at $45 total (80% threshold hit)
   -> AC gap found: what happens at 80%? No AC covers warning behaviour.
   -> Added AC for threshold warning.
3. Item 8 costs $8, pushing total to $53 (over budget)
   -> AC gap found: guard runs post-task, not mid-task. Budget can overshoot.
   -> Added AC clarifying post-task check semantics and overshoot tolerance.
4. Guard suspends processing. But item 9 was already in-flight via the scheduler.
   -> AC gap found: what about in-flight items when processing is suspended?
   -> Added AC for idempotent suspension (in-flight items complete, no new items start).
```

**This step typically finds 3-8 missing or imprecise ACs.** These are the ACs that would otherwise become bugs discovered during implementation or, worse, in production.

**Output:** Updated user stories with stress-test-derived ACs marked (e.g. "AC-X.30: Idempotent operations [stress-tested]"). Present for re-approval before proceeding.

---

## Phase 2: High-Level Plan

**This phase begins ONLY after the User Story Workshop gate AND Stress Test gate have passed.**

Present the approach at a conceptual level. No code. No file paths. Just the shape of the solution.

### Problem Statement
[1-2 sentences: what we're solving and why]

### User Stories
[List with acceptance criteria — approved in Phase 1]

### Proposed Approach
[Conceptual design — how will we solve this? What's the strategy?]

### Scope
- **In scope:** [list]
- **Out of scope:** [list]

### Key Decisions
[Architectural choices, trade-offs, alternatives considered]

**Ask:** "Does this direction feel right? Any concerns before I investigate the codebase and add detail?"

**Wait for acceptance before proceeding to Phase 3.**

---

## Phase 3: Codebase Investigation

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

## Phase 4: Detailed Plan (On Acceptance)

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

## Phase 5: Iterate

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
- Test class organization: group by component, tag by US
- IMP items for tests should specify the US/AC they trace to

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

#### Orphan Check

After completing both tables:
- **Orphan IMP items:** Any IMP item that doesn't appear in Table 1 or Table 2 is orphan code. Either trace it to an AC or remove it.
- **Orphan code snippets:** Any code snippet in the plan that doesn't map to an IMP item is untracked work. Either create an IMP item or remove the snippet.
- **Orphan tests:** Any test IMP item that doesn't appear in Table 2 is testing something not traced to requirements. Either trace it or question whether it's needed.

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

**Next step:** `/x1-review-plan` to validate before implementation

---

## Congruence with X1 Workflow

This plan output is structured so downstream skills can validate:

| Plan Section | Downstream Skill |
|--------------|------------------|
| User Stories + Acceptance Criteria | `/x1-review-plan` Requirements Coverage |
| Architecture Approach + Layer Placement | `/x1-review-plan` Architecture Compliance |
| Database + API + Error Handling | `/x1-review-plan` Technical Design Quality |
| UI/UX Design (layout, unity, modes) | `/x1-review-plan` UI/UX Design Clarity |
| Auth + Multi-tenancy | `/x1-review-plan` Security & Authorization |
| Files to Create/Modify | `/x1-review-plan` File Organization |
| Edge Cases | `/x1-review-plan` Chaos Demon "What If" |
| Implementation Checklist (IMP items) | `/x1-implement` Completion Gate |
| Implementation Checklist (IMP items) | `/x1-audit-plan` Gap Analysis |
