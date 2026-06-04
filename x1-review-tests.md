# X1 Review / Generate Tests

**Report:** "Using X1 Test Plan methodology."

## Mode Selection

Ask: **"Review existing tests OR generate new test plan?"**

---

## Mode 1: Review Existing Tests

### Instructions
1. Identify test files related to the feature/code
2. Evaluate test coverage and quality
3. Identify missing test scenarios
4. **FIX or ADD tests** as needed
5. Re-audit after changes

### Test Quality Checklist
- Tests verify CORRECT behavior (not just current behavior)
- Tests have clear Arrange-Act-Assert structure
- Test names describe what is being tested
- Edge cases covered (null, empty, boundary values)
- No flaky tests (random failures)
- Tests trace to user stories via `[Trait("UserStory", "US-N")]` attributes (AC definitions are the stable source in `specs/…`, not the plan)
- Every AC has at least one test, or a documented justification for why not
- **Invariant coverage**: every invariant the feature touches (the plan's Invariant Impact Matrix) has a test realizing its *Observed by* signal; cross-slice/seam integration ACs are tested, not just per-AC behavioral cases
- **Mock scope is correct**: orchestrators are only mocked in controller tests; internal seams use real implementations
- **Scope invariants tested**: if the feature involves scoped IDs (instanceId, conversationId, tenantId), isolation tests exist
- **False confidence check**: at least one test was verified to fail when given wrong inputs/assertions (no tests pass trivially because mocks return expected values)

---

## Mode 2: Generate Test Plan

### Instructions
1. Identify code to test (from git changes, specific files, or implementation plan)
2. Generate comprehensive test scenarios using Chaos Demon methodology
3. Create actual test files with test cases
4. Run tests and verify they pass

---

## Chaos Demon Test Categories

### Null & Empty Chaos
- Null objects, empty strings, whitespace-only, missing properties
- Expected: 400 Bad Request or ArgumentException

### Boundary & Extreme Values
- Zero, negative, max int, empty pagination
- Dates in far past/future, string max length + 1
- Unicode, special characters, null bytes

### Concurrency & Race Conditions
- Parallel create/update/delete operations
- Shared state in parallel code
- Transaction isolation violations

### Data Integrity & Validation
- Invalid enum values, SQL injection, XSS
- Foreign key violations, duplicate unique keys
- Invalid state transitions

### Permission & Security
- No auth token, expired token, wrong role
- Cross-tenant data access (CRITICAL)
- IDOR (Insecure Direct Object Reference)

### Performance & Resource Exhaustion
- Large datasets without pagination
- N+1 query problems
- Connection pool exhaustion

### Error Handling & Recovery
- Database failures, API timeouts
- Partial batch failures
- Transaction rollbacks

### Scope Invariants (MANDATORY for features with scoped identifiers)
Apply when the feature involves instanceId, conversationId, tenantId, contactId, or any other scoping identifier passed between services.

- **Isolation**: Two entities with same parent scope (same agent, different conversations) cannot see each other's data. Test by storing under A, reading via B, asserting not found.
- **Null scope fail-safe**: When scope ID lookup fails (instance evicted, race condition, wrong ID), system returns an error — NOT all data for that scope level. A silent null coalesce is a privacy leak.
- **Cross-tenant**: Different tenantIds cannot access each other's data even when other IDs match.
- **Stale reference**: Scope ID exists in one service (e.g., in-memory manager) but not another (DB). Behavior must be consistent — fail or degrade, never silently return wrong data.

**False confidence gate**: For each scope invariant test, first write the assertion wrong (assert isolation is broken), run it, confirm it fails. Then fix the assertion. If you cannot make it fail, the test is not detecting real bugs.

## Mock Scope Rules (MANDATORY)

Mock ONLY at true system boundaries. Never mock internal seams when testing business logic.

| Boundary | Mock? |
|----------|-------|
| LLM providers, email providers, external HTTP | YES |
| `IClock` / time | YES |
| `AgentInstanceManager` | NO — mocking it hides scope bugs |
| Repositories | NO — use in-memory DB to exercise EF queries |
| Orchestrators (in **controller** tests) | YES — controller tests verify HTTP contract only |
| Orchestrators (in **orchestrator** tests) | NO — these tests verify business logic |

Controller tests and orchestrator tests coexist as separate test classes. Both are required.

## Test Layers Required

| Layer | Test approach | When required |
|-------|--------------|---------------|
| Controller | HTTP test, mock orchestrator | Always |
| Orchestrator | Real orchestrator + in-memory DB, called directly | Any AC with business logic |
| Tool + InstanceManager + Repository | Real services wired together, in-memory DB | Any AC involving agent tools, memory, or scheduling |
| Background services | Direct invocation of evaluation method (requires testable seam) | Any AC involving scheduled events or queued processing |

**Background service prerequisite**: Hosted services must expose a testable seam (`EvaluateAsync(DateTimeOffset now)` or injected `IClock`) before tests can be written. If the seam doesn't exist, add it as a production code change first.

## Test Case Template

```csharp
[Fact]
public async Task MethodName_Scenario_ExpectedBehavior()
{
    // Arrange
    var mockRepo = new Mock<IRepository>();
    // Setup...

    // Act
    var result = await service.MethodAsync(input);

    // Assert
    Assert.NotNull(result);
    mockRepo.Verify(...);
}
```

## Important Principles

- If implementation has a bug, **FIX THE IMPLEMENTATION** first
- Never write tests that accommodate bugs
- Tests should EXPOSE bugs, not work around them
- Aim for 80%+ coverage on critical paths
- If every test passes on first run without any real logic exercised, that is a warning — verify tests can fail by testing wrong behavior

## Final Report

- Test scenarios identified
- Tests created/modified
- Test results (pass/fail)
- Coverage summary
