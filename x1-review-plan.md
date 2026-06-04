# X1 Review Implementation Plan

**Report:** "Using X1 Implementation Plan QA methodology to validate the plan."

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

1. Identify the implementation plan file (ask if unclear)
2. **Understand the codebase first** - read CLAUDE.md, explore relevant source files, understand existing patterns and architecture before reviewing the plan
3. **Review in parallel** - read multiple plan sections simultaneously, check codebase patterns across files concurrently
4. Validate plan against actual codebase: Does the plan match how this application is built? Are the proposed files/patterns consistent with existing code?
5. Check for reuse of existing services, components, utilities - search the codebase to verify nothing is being duplicated
6. Think like a production incident - find every failure mode
7. **UPDATE THE PLAN** to correct any issues found
8. Re-audit the corrected plan for completeness
9. If anything is unclear, ask before proceeding

**Goal:** Penultimate quality - catch problems BEFORE coding begins. Work fast by parallelizing reads and searches.

---

## Chaos Demon Review Philosophy

**CRITICAL MINDSET**: Your job is NOT to validate the plan. Your job is to **FIND EVERY POSSIBLE WAY THIS PLAN WILL FAIL IN PRODUCTION**.

### The "What If" Checklist

For EVERY architectural component, run through these categories:

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

**For Controller → Service → External API:**
```
CHAOS DEMON ASKS:
- Does service throw typed exception with StatusCode? (not generic HttpRequestException)
- Does controller catch and return StatusCode(ex.StatusCode, ...)? (not letting it bubble as 500)
- Is error message from API response extracted? (not discarded by EnsureSuccessStatusCode)
- Are API errors logged with status code, reason phrase, and response body?
```

### Seam Stress Testing (MANDATORY)

**The reviewer MUST verify that the plan's architecture has been stress tested at the seams, where services hand off to each other.**

Individual services work well in isolation. The system breaks at integration points. If the plan does not show evidence of architecture stress testing, this is a BLOCKER.

**Verification checklist:**
1. Does the plan include end-to-end scenarios that cross at least two service boundaries?
2. For each service interaction, is the state assumption at the handoff documented?
3. Has the plan considered: temporal collisions (two services acting simultaneously), ordering assumptions (A before B), state assumptions (stale reads)?
4. If the plan has >2 services that interact, are there scenarios showing what happens when they have conflicting intents (one suspending while another is starting work)?
5. Has the service decomposition been revised based on stress test findings? (If the architecture is identical to the "obvious first pass", it likely hasn't been stress tested.)

**If stress testing evidence is missing:**
```
BLOCKER: ARCHITECTURE NOT STRESS TESTED AT SEAMS

The plan proposes [N] interacting services but includes no scenarios that test
cross-service state handoffs. Individual services may work in isolation, but
the system will fail at integration points.

REQUIRED: Add 5-8 end-to-end scenarios crossing service boundaries. For each,
document: the state at each handoff, what assumptions are being made, and what
happens if those assumptions are violated.
```

### Service Design Quality (MANDATORY)

**The reviewer MUST evaluate service decomposition for testability and elegance.**

1. **Constructor dependency count.** Any service with >5 injected dependencies is a design smell. Flag it and ask whether the service should be split.
2. **Decision/side-effect separation.** Services that both decide ("should this happen?") and execute ("make it happen") are harder to test and harder to reason about. Flag services that mix policy decisions with state mutations.
3. **Mock count in tests.** If a service's test setup requires >5 mocks, the service is doing too much. This is a signal to split, not a signal to write complex test setup.
4. **Pure function extraction.** Policy evaluations, validation rules, and threshold checks should be pure functions where possible. Flag any policy logic embedded inside a service method with side effects.

### Review Output Style

When you find issues, be SPECIFIC and CONCRETE:

**❌ BAD:** "Error handling is missing"

**✅ GOOD:**
```
CRITICAL BLOCKER: NO EXCEPTION HANDLING FOR SSE PARSING

PRODUCTION FAILURE SCENARIO:
1. Azure returns malformed JSON
2. JsonException thrown during parsing
3. Entire request fails with 500 error
4. User sees error after waiting 30 seconds

MISSING: try/catch, fallback behavior, logging, retry policy
```

---

## Red Flags That Demand Deep Scrutiny

🚩 **"Will tackle separately"** → Translation: "Not going to do it"
🚩 **"Should work"** → Translation: "Haven't thought through edge cases"
🚩 **"Simple refactoring"** → Translation: "30 files changed, 100 bugs"
🚩 **"No breaking changes"** → Translation: "Don't understand dependencies"
🚩 **No exception handling** → Translation: "First error crashes system"
🚩 **No logging strategy** → Translation: "Can't debug production"
🚩 **No rollback plan** → Translation: "Deploy and pray"
🚩 **"Testing is out of scope"** → Translation: "We're shipping bugs"

---

## Review Checklist

### Requirements Coverage
- All user stories addressed
- All acceptance criteria have technical implementations
- Success criteria are measurable

### Architecture Compliance
- Follows established patterns
- Correct layer placement
- Reuses existing components (DON'T duplicate)

### Architectural Principles & Invariants Compliance
- Plan references `docs/architecture/architectural-principles.md`
- Relevant AR principles identified and verified (AR-1 through AR-7)
- Relevant NFR principles identified and verified (NFR-1 through NFR-8)
- State machines are explicit where lifecycle exists (AR-5)
- No silent failure paths (NFR-3)
- Event pipeline used for cross-component communication (AR-6)
- **Invariant Impact Matrix present**: every catalog invariant the feature touches is listed with *how it is honoured*; no "touched but not honoured" left open.
- **Invariant Generator ran**: new invariants derived from the feature's deltas are recorded as design constraints (and flagged for the ratchet).
- **Coverage ledger passes (N > 1)**: *forward* — every AC owned by exactly one slice (0 = gap, >1 = duplication); *backward* — every `IMP-{EPOCH}.{SLICE}` traces to an AC.

### Technical Design Quality
- Database schema is complete
- API design follows conventions
- Error handling is comprehensive

### Security & Authorization
- Auth requirements specified
- Role-based access configured
- Multi-tenancy isolation addressed

### Plan Quality Gates (NEW)
- [ ] User Story Workshop gate passed: ALL stories approved before codebase investigation
- [ ] Stress Test gate passed: User stories stress tested with end-to-end scenarios, gaps fed back into ACs
- [ ] Architecture Stress Test: Service interactions stress tested at seams with cross-boundary scenarios
- [ ] Service Design Elegance: No service with >5 dependencies, decisions separated from side effects
- [ ] API Verification: All code snippets cite actual method signatures with file:line references
- [ ] Snippet confidence: All snippets marked [VERIFIED] or [CONCEPTUAL]
- [ ] Fail-fast: No fallback patterns in code snippets (no `?? default`, no silent catch)
- [ ] Traceability matrix present: Every AC has IMP items, test coverage visible
- [ ] GAP justifications: Each untested AC has a specific technical reason and alternative verification
- [ ] Code reuse: Every new class/file justifies why existing code cannot serve the purpose
- [ ] Repository performance: New repo methods specify access pattern, cardinality, projection needs
- [ ] Architectural Principles: Relevant AR/NFR principles identified and plan verified against them

### [CONCEPTUAL] Snippet Verification
For every snippet marked [CONCEPTUAL] in the plan:
1. Read the actual codebase to verify assumptions
2. Promote to [VERIFIED] if correct, or flag as BLOCKER if wrong
3. If >50% of snippets are [CONCEPTUAL], the plan needs more investigation before review can proceed

### UI/UX Design Clarity (if plan involves frontend)
- **Visual Unity Verified**: Multiple modes (create/edit, view/edit) explicitly share same layout, or differences are intentional with documented reasons
- **Layout is Contract**: ASCII diagram or visual spec treated as literal specification, not suggestion
- **Mode Differences Explicit**: Document states what's identical (layout, components) vs what differs (data state, button labels, visibility)
- **Interaction Pattern Clear**: Tap-to-edit vs always-editable, save-on-blur vs save button, inline vs form-level validation

**UI/UX Red Flags to Catch:**
- 🚩 Plan says "create mode" and "edit mode" without specifying if layout differs → ASK
- 🚩 Layout diagram present but implementation describes different structures → REJECT
- 🚩 Assumes traditional form for create, display-only for view without requirement → CLARIFY
- 🚩 "Unified component" without specifying what's actually unified → CLARIFY

---

## Conditional Sections (Include If Applicable)

### Data Access Patterns (if plan involves database work)
- ORM/data access layer used consistently
- Entities in appropriate layer
- Migrations strategy documented
- Audit fields included (`CreatedBy`, `CreatedAt`, `UpdatedBy`, `UpdatedAt`)
- Transaction boundaries clearly defined

### Performance Considerations (if plan involves high-throughput or complex queries)
- Database query optimization planned (indexes, pagination)
- Caching strategy defined
- Parallel processing for bulk operations
- Connection pooling and resource management
- Performance requirements measurable

### File Organization (if plan creates new files)
- All new files listed with full paths
- Modified files listed separately
- File naming follows project conventions
- Folder structure follows established patterns
- No files in incorrect layers

### Domain-Specific Concerns (if applicable)

**Compliance (healthcare, finance, etc.):**
- Sensitive data (PII, PHI) handling compliant
- Audit trail requirements met
- Timezone handling for multi-region

**Multi-Tenancy:**
- Data access filtered by tenant
- Cross-tenant access prevented
- Tenant context properly set

**AI/ML Integration:**
- Error handling for AI service failures
- Rate limiting and cost management
- Fallback behavior when service unavailable

---

## Output Format

### Blockers (Must Fix Before Coding)
List critical issues with specific failure scenarios

### Major Concerns (Should Fix)
List significant issues that could cause problems

### Minor Issues (Can Address During Development)
List minor improvements needed

### Approval Decision
- Approved
- Approved with Changes
- Rejected - Must address blockers first

---

## Final Report

- Issues found and fixed in plan
- Updated plan sections
- Approval status
