# X1 Architecture Review

**Report:** "Using X1 Architecture Review methodology to evaluate architectural alignment."

## Purpose

Evaluate plans or code from an enterprise architecture perspective:
1. **Think big picture** - How does this component fit into the overall application?
2. **Minimize code** - Every line is a liability requiring justification
3. **Maximize reuse** - But only when abstraction is the RIGHT abstraction
4. **Foresight over hindsight** - Identify potential enterprise capabilities before duplication

**Use for:**
- Implementation plans (before coding) - architectural decisions
- Existing code (after implementation) - consolidation opportunities
- Feature areas - enterprise pattern alignment

---

## Scope Selection

Ask if unclear: **"What should I review?"**
- Implementation plan file?
- Specific files or feature area?
- Git changes since last commit?
- Entire module or subsystem?

---

## Instructions

1. **Identify scope** (ask if unclear)
2. **Map system context** - Understand WHERE this fits in the overall architecture
3. **Assess architectural fit** - Does this align with or fight existing patterns?
4. **Apply foresight** - Could this become an enterprise capability? Should it be designed that way now?
5. **Search for patterns** - Check if similar functionality exists (Rule of Three)
6. **Evaluate complexity budget** - Is the abstraction cost justified?
7. **Check for wrong abstractions** - Sometimes duplication is better
8. **Assess component boundaries** - Interface surface area, coupling, contracts
9. **Present recommendations** with trade-off analysis

---

## Phase 0: System Context (DO THIS FIRST)

**Before reviewing ANY details, understand the big picture:**

### Map the Architecture
- What are the major subsystems/modules in this application?
- What architectural style is in use? (Layered, Hexagonal, Microservices, Modular Monolith)
- Where does this new code/plan fit in the overall picture?

### Component Role Assessment
For the component being reviewed:
- **Role**: What is this component's PURPOSE in the system?
- **Consumers**: What will depend on this?
- **Dependencies**: What does this depend on?
- **Blast radius**: If this fails, what else fails?
- **Core vs Peripheral**: Is this central to the domain or auxiliary infrastructure?

### Architectural Fit Questions
- Does this follow the SAME patterns as similar components?
- Does it sit in the CORRECT layer?
- Does it use EXISTING interfaces and services where appropriate?
- Or is it fighting against the architecture?

**Red Flag**: If the new code requires workarounds or "special cases" to fit, the architecture may need adjustment—or the design is wrong.

---

## Architecture Review Philosophy

### Code as Liability
Every line of code is technical debt. Your job is to find ways to **DELETE CODE** while maintaining functionality.

### The Wrong Abstraction Problem

> "Duplication is far cheaper than the wrong abstraction." — Sandi Metz

**The wrong abstraction is WORSE than duplication because:**
- It's harder to change (everyone depends on it)
- It obscures the actual requirements
- It spreads complexity throughout the codebase
- Developers work around it instead of fixing it

**Signs of a wrong abstraction:**
- Parameters that are only used by one caller
- Branches that select completely different behavior
- "Flags" that change what the function does
- Comments explaining which mode to use
- Callers that pass nulls or empty values for unused params

### Foresight vs Hindsight

**Hindsight** (reactive): "We have 5 scheduling implementations, let's consolidate"
**Foresight** (proactive): "This looks like scheduling. Will we need more? Should we design for reuse now?"

**Foresight Questions:**
- Is this a DOMAIN-specific feature or INFRASTRUCTURE capability?
- If another feature needed this, would they find it? Or reinvent it?
- In 6 months, will we wish we'd built this more generically?

### Rule of Three (With Nuance)

| Occurrences | Recommendation |
|-------------|----------------|
| 1 | Keep specific - no abstraction |
| 2 | Watch and note - stay specific |
| 3+ | EVALUATE for consolidation (not automatic) |

**Before consolidating at 3+, ask:**
- Are these TRULY the same, or superficially similar?
- Will the abstraction be SIMPLER than the sum of the parts?
- Can you articulate the SINGLE purpose of the abstraction?
- Will future variations fit this abstraction naturally?

**When duplication is better:**
- Implementations are evolving in different directions
- The "commonality" is coincidental, not essential
- Abstraction would require configuration/flags that increase complexity
- The domain areas have different change rates

### Complexity Budget

Every abstraction has costs:
- **Learning curve** - New developers must understand it
- **Indirection** - Harder to trace code flow
- **Debug difficulty** - More layers to inspect
- **Maintenance burden** - One more thing to keep working
- **Coupling** - Changes affect all consumers

**The question is not "Can we abstract?" but "Should we?"**

Justify complexity spend:
- LOC reduction must be significant (not 10 lines saved for 50 lines of abstraction)
- Maintenance benefit must be real (changes actually happen in practice)
- The abstraction must be STABLE (not constantly modified)

---

## Enterprise Pattern Detection

**Proactively search** for cross-cutting concerns. Apply FORESIGHT: identify potential enterprise capabilities BEFORE they get duplicated.

### Infrastructure Patterns (Design for Reuse from First Use)
| Pattern | Red Flag | Enterprise Approach |
|---------|----------|---------------------|
| **Scheduling** | Multiple Timer/Hangfire/Quartz usages | Unified scheduler service with job registry |
| **Caching** | Scattered MemoryCache/Redis calls | Cache abstraction with consistent invalidation |
| **Messaging/Events** | Multiple event dispatch implementations | Centralized event bus or mediator |
| **Configuration** | Hardcoded values or scattered IOptions | Typed configuration classes, central registration |

### Application Patterns (Apply Rule of Three)
| Pattern | Red Flag | Enterprise Approach |
|---------|----------|---------------------|
| **Authentication** | Multiple auth checks inline | Auth middleware/attributes |
| **Authorization** | Permission checks scattered in code | Policy-based authorization |
| **Validation** | Repeated validation logic | Validation service or decorators |
| **Error Handling** | Try/catch scattered everywhere | Global exception handler + typed exceptions |
| **Logging** | Log calls with inconsistent context | Structured logging with automatic context |

### Detection Process

1. **Identify the capability** - What is this code DOING at a conceptual level?
2. **Search codebase** - Does this capability exist elsewhere?
3. **Assess consistency** - Same approach or fragmented implementations?
4. **Apply foresight** - Is this likely to be needed again?
5. **Evaluate consolidation** - Would unification SIMPLIFY or complicate?

---

## Concrete Examples

### Example 1: The Scheduling Anti-Pattern (Hindsight Lesson)

```
ARCHITECTURAL CONCERN: Scattered Enterprise Capability

OBSERVATION:
Application evolved over time with multiple scheduling needs:
- ProcessAnalysisJob.cs → Hangfire RecurringJob.AddOrUpdate()
- ReportGenerationService.cs → System.Threading.Timer
- NotificationService.cs → Task.Delay loop
- DataSyncWorker.cs → BackgroundService with manual timing
- CleanupJob.cs → Hangfire but different pattern than ProcessAnalysisJob

PROBLEM:
Each developer solved "I need to run something periodically" differently.
No one thought "scheduling is an enterprise capability."

HINDSIGHT COST:
- 5 different patterns to understand
- 5 places to configure timeouts/intervals
- 5 different monitoring approaches
- Inconsistent error handling
- No unified job dashboard

FORESIGHT QUESTION (what SHOULD have been asked):
"Is scheduling a cross-cutting concern in this application?"
→ YES - multiple features will need it
→ Therefore: Design for reuse from FIRST scheduling use

ARCHITECTURAL RECOMMENDATION:
Unified SchedulerService with:
- Consistent job registration interface
- Centralized cron/interval configuration
- Unified monitoring and health checks
- Single error handling pattern
- Job registry for discoverability

COMPLEXITY BUDGET:
- Cost: ~200 LOC for abstraction infrastructure
- Benefit: Prevents N×100 LOC for N future scheduling needs
- Maintenance: Single point to update/fix scheduling behavior
```

### Example 2: When NOT to Abstract (The Wrong Abstraction)

```
OBSERVATION:
Three services have "similar" validation:
- UserService.ValidateUser() - checks email format, password strength
- ProductService.ValidateProduct() - checks SKU format, price range
- OrderService.ValidateOrder() - checks item counts, shipping address

NAIVE RESPONSE: "Let's create GenericValidator<T>!"

ARCHITECTURAL ANALYSIS:
- Are these TRULY the same? NO
  - User validation: Security concern, changes with auth requirements
  - Product validation: Business rule, changes with catalog requirements
  - Order validation: Transaction integrity, changes with checkout flow

- Different change rates: User auth changes rarely; product rules change often

- What would the abstraction look like?
  - GenericValidator<T> with dozens of configuration options
  - Or reflection-based attribute validation
  - Or complex rule engine

- Is the abstraction SIMPLER than the parts? NO
  - Sum of parts: ~150 lines, clear and direct
  - Abstraction: ~200 lines + configuration + learning curve

RECOMMENDATION: KEEP DUPLICATION
- Each validation is DOMAIN-SPECIFIC, not infrastructure
- They LOOK similar but have different purposes and change rates
- "Three things that look the same" ≠ "Three instances of the same thing"
```

### Example 3: Foresight Applied Correctly

```
PLAN REVIEW: New Feature - Export to PDF

PROPOSED IMPLEMENTATION:
- PdfExportService with hard-coded report format
- Direct HTML-to-PDF library calls
- Single export method tied to specific report

FORESIGHT QUESTIONS:
1. Is "export" a domain-specific feature or infrastructure capability?
   → Infrastructure - many features may want export

2. Will other features need export?
   → Likely: Reports, invoices, summaries all may need PDF

3. What OTHER export formats might be needed?
   → CSV, Excel, possibly Word

ARCHITECTURAL RECOMMENDATION:
Don't over-engineer, but design with expansion in mind:
- IExportService interface with format parameter
- PdfExportService as first implementation
- Keep the interface SIMPLE (not "just in case" methods)

// Simple, extensible, not over-engineered
public interface IExportService
{
    Task<byte[]> ExportAsync<T>(T data, ExportFormat format);
}

// NOT this (over-engineered):
public interface IExportService
{
    Task<byte[]> ExportAsync<T>(T data, ExportFormat format,
        ExportOptions options, ITemplateProvider template,
        CancellationToken ct, IProgress<int> progress);
}

COMPLEXITY BUDGET:
- Cost: Interface + one implementation = same LOC as no abstraction
- Benefit: Second format drops in without refactoring consumers
- Risk: Low - interface is minimal, easy to change if wrong
```

---

## Consolidation Opportunity Analysis

When patterns are found that should be unified:

### Analysis Template
```
CONSOLIDATION OPPORTUNITY: [Pattern Name]

CURRENT STATE:
- Location 1: [file:line] - [brief description]
- Location 2: [file:line] - [brief description]
- Location 3: [file:line] - [brief description]

PATTERN ANALYSIS:
- Total occurrences: X
- Consistency: [Identical/Similar/Fragmented]
- Current LOC: ~X lines across Y files

UNIFIED APPROACH:
- Single service/utility at: [proposed location]
- Estimated unified LOC: ~X lines
- Caller changes: [brief description]

IMPACT:
- LOC reduction: ~X lines (Y%)
- Maintenance: Single point of change
- Testability: Centralized testing
- Risk: [Low/Medium/High] - [reason]
```

---

## Component Boundaries & Contracts

### Interface Surface Area
- [ ] Interfaces expose minimum necessary
- [ ] No "god interfaces" with 10+ methods
- [ ] Clear single responsibility per interface
- [ ] Stable contracts - changes rare and backward-compatible

### Dependency Direction
- [ ] Dependencies point inward (Infra → Core → Domain)
- [ ] No circular dependencies between modules
- [ ] Abstractions owned by consumer, not provider
- [ ] External dependencies wrapped in adapters

### Coupling Assessment
| Level | Description | Action |
|-------|-------------|--------|
| **Tight** | Direct class references, shared state | Introduce interface or mediator |
| **Data** | Shared DTOs or data structures | Acceptable if contracts stable |
| **Message** | Communication via events/messages | Preferred for cross-module |
| **None** | No direct knowledge | Ideal for independent modules |

### Contract Stability
- [ ] DTOs versioned or backward-compatible
- [ ] API contracts documented
- [ ] Breaking changes require migration path
- [ ] Internal vs external contracts distinguished

---

## Reusability Assessment

### Current Reuse Opportunities
- [ ] Existing utilities that could be leveraged
- [ ] Services that already solve part of the problem
- [ ] Patterns from other modules that apply
- [ ] Framework features not being utilized

### Abstraction Level
| Level | When Appropriate |
|-------|------------------|
| **Too Abstract** | Interface with single implementation, unused extension points |
| **Appropriate** | Clear contract, 2+ consumers or infrastructure pattern |
| **Too Concrete** | Hardcoded dependencies, repeated code, no injection |

### Extension Points
- [ ] Natural extension points identified (not speculative)
- [ ] Follows Open-Closed Principle pragmatically
- [ ] No "just in case" extensibility

---

## Architectural Principles Compliance (MANDATORY)

Before completing any architecture review, verify compliance with the project's architectural principles. Not every principle applies to every review — state which are relevant and verify those.

### Architectural Requirements

**AR-1: Single Responsibility Decomposition**
Each component owns exactly one concern. Components are named after what they do, and their constructor dependencies reveal their scope.
- [ ] Each new/modified component owns exactly one concern
- [ ] Constructor dependencies reveal scope — no hidden responsibilities
- [ ] A change to one concern doesn't require understanding another

**AR-2: Dependency Inversion**
All significant collaborators are interfaces. Components depend on abstractions, never on concrete implementations.
- [ ] All collaborators are interfaces
- [ ] No concrete dependencies between components
- [ ] Enables testing without real implementations

**AR-3: Composition Over Inheritance**
Components compose through injection, not through class hierarchies. Each layer can be replaced or wrapped independently.
- [ ] No class hierarchies where composition would suffice
- [ ] Each layer independently replaceable or wrappable
- [ ] Injection used for composition

**AR-4: Open-Closed Principle**
Open for extension, closed for modification. Adding a new capability should not require changing existing components.
- [ ] New capabilities (event sources, tool types, providers) extend, don't modify
- [ ] Existing components unchanged when new variations are added

**AR-5: Explicit State Machines**
Lifecycle states are explicit state machines with defined transitions. Invalid transitions throw. State is never implied by checking multiple booleans.
- [ ] State is managed via explicit state machine, not boolean combinations
- [ ] Invalid transitions throw (programming errors, not runtime conditions)
- [ ] Every transition is logged
- [ ] No ambiguous or "should never happen" states

**AR-6: Event-Driven Architecture**
All activity produces events through a single event pipeline. Producers don't know who consumes events; they just emit them.
- [ ] Cross-component communication via events, not direct calls
- [ ] Event pipeline is the contract between producers and consumers
- [ ] Producers don't know consumers

**AR-7: Provider Encapsulation**
Providers encapsulate their internal complexity. Retry, fallback, and internal recovery are provider responsibilities, not caller responsibilities.
- [ ] Providers own retry, fallback, and recovery internally
- [ ] Callers consume clean interfaces, not implementation details
- [ ] No retry/fallback logic leaking into callers

### Non-Functional Requirements

**NFR-1: Testability**
Max 5 mocked dependencies per component. Pure decision logic extracted as pure functions.
- [ ] No component requires more than 5 mocked dependencies in test setup
- [ ] Pure decision logic extracted as pure functions testable without mocks
- [ ] Every layer boundary has integration tests
- [ ] Every state machine transition tested under concurrent input
- [ ] Every persist-then-reload path tested for round-trip fidelity

**NFR-2: Observability**
Every state transition, decision point, and error is logged with structured context.
- [ ] Structured logging with instance ID, tenant ID, step number
- [ ] Events provide real-time visibility
- [ ] Any production incident diagnosable from logs alone without a debugger
- [ ] Every multi-turn can be fully reconstructed from events

**NFR-3: Resilience**
Transient failures retried. Permanent failures surfaced as events. No silent failures.
- [ ] Transient failures retried with backoff
- [ ] Permanent failures surfaced as events and logged
- [ ] Partial state handled explicitly, never silently dropped
- [ ] No silent failure paths — every catch block retries, raises an event, or throws
- [ ] No swallowed exceptions

**NFR-4: Resource Efficiency**
Idle components unloaded. In-memory resource counts bounded. Context growth managed.
- [ ] Idle resources unloaded from memory
- [ ] In-memory counts bounded
- [ ] Memory proportional to active entities, not total entities

**NFR-5: Concurrency Safety**
Mutual exclusion via state machine design, not ad-hoc locking. Thread-safe collections for queuing.
- [ ] Mutual exclusion enforced by state machine design, not ad-hoc locks
- [ ] Thread-safe collections (Channel<T> or ConcurrentQueue) for queuing
- [ ] State transitions are atomic
- [ ] No race conditions at boundaries or injection points

**NFR-6: Persistence and Recovery**
Persisted state is source of truth. Incremental persistence. Lazy reload on demand.
- [ ] Persisted state is source of truth
- [ ] State persisted incrementally
- [ ] Server restart loses only in-progress operation
- [ ] Lazy reload on demand, not eager

**NFR-7: Real-Time Responsiveness**
Events stream with minimal latency. No buffering or batching.
- [ ] Events stream to clients within milliseconds of generation
- [ ] No buffering or batching of events
- [ ] Reconnection provides complete sync from appropriate sync point
- [ ] No stale event delivery or phantom buildup

**NFR-8: Deterministic Lifecycle**
No ambiguous states. Invalid transitions throw. Every transition logged.
- [ ] States, boundaries, and transitions are deterministic and predictable
- [ ] No ambiguous states or "unknown state" code paths
- [ ] Invalid transitions are programming errors that throw
- [ ] State machine transitions can be formally verified

---

## Architectural Decision Checklist

### SOLID (Applied Pragmatically)

**Single Responsibility**
- [ ] Each class/module has one reason to change
- [ ] Not split too fine (over-engineering)
- [ ] Not bundled too much (god objects)

**Open-Closed**
- [ ] Extensible without modification where needed
- [ ] Not prematurely abstracted
- [ ] Extension points based on actual requirements

**Liskov Substitution**
- [ ] Subtypes substitutable for base types
- [ ] No surprising behavior in derived classes
- [ ] Interface segregation prevents violations

**Interface Segregation**
- [ ] Clients depend only on methods they use
- [ ] No "fat interfaces" forcing empty implementations
- [ ] Role-based interfaces where appropriate

**Dependency Inversion**
- [ ] High-level modules independent of low-level details
- [ ] Abstractions owned by consumers
- [ ] Infrastructure adapters at boundaries

### Layer Separation (Context-Specific)

```
API/Controllers → Services → Repositories → DbContext
    (HTTP)        (Logic)     (Data)        (EF)
```

- [ ] Controllers thin - HTTP only
- [ ] Business logic in services only
- [ ] Data access in repositories only
- [ ] No layer skipping

---

## Trade-off Analysis Framework

Architecture is about trade-offs. Document them explicitly.

### Trade-off Template
```
DECISION: [What was decided]

TRADE-OFF:
- Gained: [What we get]
- Lost: [What we give up]

ALTERNATIVES CONSIDERED:
1. [Alternative A] - Rejected because [reason]
2. [Alternative B] - Rejected because [reason]

WHEN TO REVISIT:
- If [condition], reconsider this decision
```

### Common Architectural Trade-offs

| Trade-off | Favor Simplicity When... | Favor Abstraction When... |
|-----------|--------------------------|---------------------------|
| **Duplication vs DRY** | Code changes independently, <3 occurrences | Identical logic, shared change rate |
| **Direct vs Indirect** | Flow is straightforward, performance critical | Multiple callers, need flexibility |
| **Concrete vs Abstract** | Single implementation likely permanent | Multiple implementations exist/expected |
| **Coupled vs Decoupled** | Components change together, simple system | Different change rates, team boundaries |
| **Inline vs Extracted** | One-off logic, context helps understanding | Reused logic, testable in isolation |

---

## Severity Guide

| Level | Criteria | Examples |
|-------|----------|----------|
| **Critical** | Architectural debt that will compound | Scattered enterprise capabilities, god classes, circular dependencies, wrong abstraction locking in bad design |
| **Warning** | Missed opportunity or suboptimal choice | Existing utility ignored, pattern could be consolidated, architectural fit issues |
| **Suggestion** | Improvement without urgency | Natural extension point, cleaner boundaries, documentation of decisions |

---

## Output Format

### System Context Summary
```
ARCHITECTURE CONTEXT:
- Style: [Layered/Hexagonal/Modular Monolith/etc.]
- Component reviewed: [Name]
- Role in system: [Purpose]
- Consumers: [Who depends on this]
- Dependencies: [What this depends on]
- Fit assessment: [Aligns/Conflicts with existing architecture]
```

### Foresight Assessment
```
ENTERPRISE CAPABILITY ANALYSIS: [Capability Name]
  Type: Infrastructure / Application
  Current occurrences: X
  Likelihood of reuse: [High/Medium/Low]
  Recommendation: [Design for reuse now / Wait for Rule of Three]
  Rationale: [Why]
```

### Consolidation Opportunities
```
[Critical/Warning] CONSOLIDATION: [Pattern Name]
  Occurrences: X locations
  Current LOC: ~X lines across Y files
  Unified LOC: ~X lines
  Net reduction: ~X lines (Y%)
  Complexity cost: [Learning curve, indirection added]
  Recommendation: [Consolidate / Keep separate]
  Rationale: [Why this trade-off]
```

### Architecture Violations
```
[Critical/Warning/Suggestion] [file:line or component] - Brief description
  Problem: What's wrong architecturally
  Impact: Why this matters (maintenance, scalability, etc.)
  Trade-off: What would be lost by fixing
  Fix: Specific recommendation
```

### Reuse Recommendations
```
[Warning] REUSE: [Existing component]
  Currently: [What's being duplicated/reinvented]
  Existing: [Where it already exists]
  Action: [How to leverage]
  LOC saved: ~X lines
```

### Trade-off Decisions (Document for Future)
```
ARCHITECTURAL DECISION: [Title]
  Context: [Situation that prompted decision]
  Decision: [What was decided]
  Trade-off: Gained [X], Lost [Y]
  Alternatives: [What else was considered]
  Revisit if: [Conditions that would change this]
```

---

## Final Report

### System Context
- Architecture style: [Style identified]
- Component fit: [How well the reviewed code fits]

### Summary
- Foresight flags: X (potential enterprise capabilities)
- Consolidation opportunities: X (est. LOC reduction: Y)
- Architecture violations: Critical X, Warning X, Suggestion X
- Reuse opportunities: X (est. LOC saved: Y)

### Key Findings
1. [Most significant architectural concern]
2. [Second most significant]
3. [Third]

### Trade-off Decisions Made
[List any architectural trade-offs that should be documented]

### Priority Actions
1. [Highest impact - with effort/benefit ratio]
2. [Second priority]
3. [Third priority]

### Approval Decision
- [ ] **Approved** - Architecture is sound, aligns with system
- [ ] **Approved with Changes** - Address warnings, document trade-offs
- [ ] **Needs Discussion** - Significant trade-offs require stakeholder input
- [ ] **Rejected** - Fundamental architectural issues, needs redesign

---

## Quick Reference: Architecture Anti-Patterns

| Anti-Pattern | Symptom | Fix |
|--------------|---------|-----|
| **Scattered Infrastructure** | Same pattern (caching, scheduling) in 5+ places | Centralize to enterprise service |
| **Copy-Paste Inheritance** | Similar code in multiple files | Extract shared component |
| **God Service** | Service with 20+ methods, 1000+ lines | Split by responsibility |
| **Leaky Abstraction** | Implementation details in interface | Push details to implementation |
| **Circular Dependency** | A→B→C→A | Introduce mediator or event |
| **Feature Envy** | Class uses another class's data more than its own | Move method to data owner |
| **Primitive Obsession** | String/int passed everywhere instead of type | Create value object |
| **Speculative Generality** | Interface with one implementation "for future" | Delete until needed |
| **Wrong Abstraction** | Flags/modes, unused params, workarounds | Inline and re-extract correctly |
| **Architectural Drift** | New code doesn't fit existing patterns | Align or intentionally evolve |

---

## When to Use This Skill vs Others

| Skill | Use When | Focus |
|-------|----------|-------|
| **x1-plan** | Creating implementation plan | Requirements, approach, steps |
| **x1-review-plan** | Validating plan completeness | Chaos demon, failure modes, edge cases |
| **x1-architecture-review** | Evaluating architectural decisions | Big picture, reuse, consolidation, trade-offs |
| **x1-code-review** | Reviewing implemented code | Quality, bugs, security, patterns |

### Recommended Workflow
```
1. x1-plan              → Create implementation plan
2. x1-review-plan       → Validate for completeness (chaos demon)
3. x1-architecture-review → Evaluate architectural fit and decisions
4. [Implement]
5. x1-code-review       → Review code quality
```

**x1-architecture-review answers:** "Is this the RIGHT design for this system?"
**x1-review-plan answers:** "Will this design WORK without failures?"
**x1-code-review answers:** "Is this code CORRECT and safe?"

---

## Quick Architecture Checklist (Rapid Review)

For quick reviews, answer these 10 questions:

### Big Picture
1. [ ] Does this FIT the existing architecture? (Or does it fight it?)
2. [ ] What is this component's ROLE in the system?

### Foresight
3. [ ] Is this an INFRASTRUCTURE capability? (If yes, design for reuse)
4. [ ] Will this be needed ELSEWHERE? (If yes, consider abstraction)

### Reuse
5. [ ] Does similar functionality EXIST? (Search before building)
6. [ ] If Rule of Three applies, should we CONSOLIDATE? (Not automatic)

### Complexity
7. [ ] Is the abstraction SIMPLER than the sum of parts?
8. [ ] Is the complexity spend JUSTIFIED?

### Trade-offs
9. [ ] What are we GAINING and LOSING?
10. [ ] Should we DOCUMENT this decision?

---

## The Architecture Demon's Mantra

Before approving any architectural decision:

1. **ZOOM OUT** - How does this fit the whole system?
2. **SEARCH FIRST** - Does this already exist?
3. **THINK FORWARD** - Will we need this again?
4. **QUESTION ABSTRACTION** - Is this the RIGHT abstraction?
5. **COUNT THE COST** - Is the complexity justified?
6. **DOCUMENT TRADE-OFFS** - Why did we choose this?
7. **DELETE IF POSSIBLE** - Can we solve this with LESS code?