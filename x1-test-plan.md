# Strategic Test Plan Template: Breaking Code to Build Perfection

**Project**: Any
**Purpose**: Design strategic tests to systematically break code, expose vulnerabilities, and hunt bugs before they reach production.

**When to Use**: After implementation plan and code generation, BEFORE deployment. Use this template to design a comprehensive test strategy that pushes code to its breaking point.

---

## The Chaos Demon Philosophy

You are not here to validate that code works. You are here to **prove it doesn't**. Your job is to:

- **Break assumptions** - What did the developer assume would never happen?
- **Push boundaries** - Test at extremes: zero, negative, max values, null, empty
- **Race for races** - Force bad timing in concurrent operations
- **Inject chaos** - Failures, randomness, delays at the worst possible moments
- **Never mask bugs** - If something feels wrong, write a test that exposes it

**Success Metric**: The more bugs you find NOW, the fewer disasters in PRODUCTION.

---

## Test Plan Metadata

| Field | Value |
|-------|-------|
| **Feature** | [Feature name from implementation plan] |
| **Test Designer** | |
| **Design Date** | |
| **Implementation Plan Version** | |
| **Source Code Version** | |
| **Estimated Test Duration** | |
| **Priority** | [ ] Critical [ ] High [ ] Medium [ ] Low |

---

## 1. Requirements Analysis & Test Scope 🎯

### 1.1 Business Requirements Coverage

**Sources (read BOTH — test design is driven by *what must stay true*, not only *what the feature does*):**
1. **Requirements** — the `/x1-requirements` artifact in `specs/…` (`US-{EPOCH}-N`, `AC-{EPOCH}-N.M`) → behavioral coverage: every AC → ≥1 test.
2. **Architecture + invariants** — the plan/spine + the invariant catalog `docs/architecture/architectural-principles.md`. Turn each invariant the feature **touches** (from the plan's Invariant Impact Matrix) into an assertion test via that invariant's ***Observed by*** signal, and turn the spine's **cross-slice integration ACs** into seam/integration tests. Without this input the plan covers only happy-path ACs and misses the invariant/seam failures the harness otherwise finds late.

**Critical User Stories to Test**:
```
[List critical user stories with acceptance criteria]

Example:
- US-001: As a nurse, I can create an order for a customer
  - AC1: Order saves with correct timestamp
  - AC2: Order associates with correct tenant
  - AC3: Audit trail created for creation event
```

**Success Criteria Tests**:
```
[For each success criterion, define how you'll verify it]
```

### Traceability (MANDATORY — two chains)
Every test case MUST trace to **either** a user story + acceptance criterion (the behavioral chain) **or** a catalog invariant the feature touches (the invariant chain: `Impact-Matrix invariant → Observed by → Test`). Every AC and every touched invariant has ≥1 test, or a documented justification.

Test IDs should include the US reference: `US1-AC1-001` or document the mapping in the test metadata section.

```csharp
[Trait("UserStory", "US-1")]
[Fact]
public async Task MethodName_Scenario_ExpectedBehavior()
```

If a test doesn't trace to any AC, question whether it's needed. If an AC has no test, justify why.

**Out of Scope** (won't test):
```
[Explicitly list what you're NOT testing and why]
```

---

### 1.2 Test Coverage Matrix

| Component | Unit Tests | Integration Tests | E2E Tests | Performance Tests | Security Tests |
|-----------|------------|-------------------|-----------|-------------------|----------------|
| API Endpoints | | | | | |
| Service Layer | | | | | |
| Data Access | | | | | |
| Business Logic | | | | | |
| UI Components | | | | | |
| ETL/Classification | | | | | |

**MANDATORY rows — required when applicable:**

| Component | Mandatory When | Test Approach |
|-----------|----------------|---------------|
| Orchestrator (business logic) | Any AC involving business rules | Real orchestrator + real in-memory DB, called directly (not via HTTP). No mock on the orchestrator itself. |
| Tool + InstanceManager + Repository | Any AC involving agent tools, memory, or scheduling | Full internal workflow: real tool, real AgentInstanceManager, real repository, in-memory DB |
| Background services | Any AC involving scheduled events or queued processing | Direct invocation of evaluation logic (requires testable seam — see Mock Scope Rules below) |

**Coverage Gaps** (identified areas with insufficient testing):
```
[List any components or scenarios that lack adequate test coverage]
```

---

## 2. Bug Hunting Strategy: The Chaos Arsenal ☠️

### 2.1 Null & Empty Chaos Tests

**Principle**: Every input can be null, empty, or missing. Assume all external data is hostile.

**Test Scenarios**:

| Test ID | Scenario | Input | Expected Behavior | Bug Risk |
|---------|----------|-------|-------------------|----------|
| NULL-001 | Null object passed to API endpoint | `null` | 400 Bad Request | NullReferenceException |
| NULL-002 | Empty string in required field | `""` | 400 Bad Request with validation error | Validation bypass |
| NULL-003 | Null collection in batch operation | `null` | 400 Bad Request | NullReferenceException in loop |
| NULL-004 | Empty collection in batch operation | `[]` | Empty result set, no errors | Incorrect "success" message |
| NULL-005 | Null nested object property | `{ customer: null }` | 400 Bad Request | NullReferenceException on property access |
| NULL-006 | Whitespace-only string | `"   "` | 400 Bad Request | Passes validation as "not empty" |
| NULL-007 | Missing required property in JSON | `{ }` | 400 Bad Request | Default value assigned incorrectly |

**Specific Null Chaos Tests**:
```
[Based on implementation, list specific methods/endpoints and null scenarios]

Example:
- CreateOrderAsync(null) → Should throw ArgumentNullException
- GetCustomersByTenantId(tenantId: 0) → Should return empty list, not exception
- UpdateOrder with null DTO properties → Should validate and reject
```

---

### 2.2 Boundary & Extreme Value Tests

**Principle**: Test at the edges: zero, negative, max values, overflow, underflow.

**Test Scenarios**:

| Test ID | Scenario | Input | Expected Behavior | Bug Risk |
|---------|----------|-------|-------------------|----------|
| BOUND-001 | Zero ID in GET request | `id=0` | 404 Not Found | Queries database with 0, unexpected results |
| BOUND-002 | Negative ID in GET request | `id=-1` | 400 Bad Request | Database error or wrong record |
| BOUND-003 | Maximum integer value | `id=2147483647` | 404 Not Found (likely) | Integer overflow in calculations |
| BOUND-004 | Minimum integer value | `id=-2147483648` | 400 Bad Request | Overflow in arithmetic |
| BOUND-005 | Empty pagination (page=0, size=0) | `page=0&size=0` | 400 Bad Request | Division by zero, empty result |
| BOUND-006 | Negative page size | `page=1&size=-10` | 400 Bad Request | SQL error or all records returned |
| BOUND-007 | Excessive page size | `page=1&size=1000000` | 400 Bad Request (too large) | Memory exhaustion, timeout |
| BOUND-008 | Date in far past | `1900-01-01` | Accept or reject based on rules | Arithmetic underflow, invalid date range |
| BOUND-009 | Date in far future | `2100-01-01` | Accept or reject based on rules | Validation bypass |
| BOUND-010 | String max length + 1 | `"A" * 256` (if max 255) | 400 Bad Request | Truncation, database error |
| BOUND-011 | Unicode/special chars | `"日本語\u0000<script>"` | Properly escaped/handled | XSS, encoding issues, null byte injection |

**Specific Boundary Tests**:
```
[List specific boundaries for your domain]

Example:
- Order score range: 0-100
  - Test: -1, 0, 1, 99, 100, 101 → Validate rejection of invalid scores
- Tenant capacity: 1-500
  - Test: 0, 1, 500, 501 → Validate bounds
- Date range for orders: Today - 90 days to Today
  - Test: 91 days ago, 90 days ago, today, tomorrow → Validate constraints
```

---

### 2.3 Concurrency & Race Condition Tests

**Principle**: Thread safety bugs hide in timing. Force bad timing with parallel operations.

**Test Scenarios**:

| Test ID | Scenario | Setup | Expected Behavior | Bug Risk |
|---------|----------|-------|-------------------|----------|
| RACE-001 | Parallel create operations (same data) | 10 threads create same order simultaneously | Either: 1 succeeds + 9 fail with conflict, OR unique constraint violation | Duplicate records, race condition |
| RACE-002 | Parallel update operations (same record) | 10 threads update same order simultaneously | Last write wins, OR optimistic concurrency error | Lost updates, data corruption |
| RACE-003 | Parallel delete + read operations | Thread A deletes, Thread B reads simultaneously | Thread B gets 404 or stale data | NullReferenceException, inconsistent state |
| RACE-004 | ETL batch processing (shared state) | Process 1000 orders in parallel with shared error list | All errors captured correctly | Lost errors due to non-thread-safe collection |
| RACE-005 | Cache invalidation race | Thread A updates DB, Thread B reads cache simultaneously | Thread B sees stale data (acceptable) or exception (bad) | Cache inconsistency, stale reads |
| RACE-006 | Transaction isolation | Thread A in transaction, Thread B reads uncommitted data | Thread B sees committed data only (isolation) | Dirty reads, phantom reads |
| RACE-007 | Deadlock scenario | Thread A locks Resource 1 then 2, Thread B locks Resource 2 then 1 | Deadlock detection or timeout | Application hang, unresponsive |

**Specific Concurrency Tests**:
```
[Based on implementation, identify concurrent operations]

Example:
- Classification ETL process: 100 orders processed in parallel
  - Test: Shared classification error collection (ConcurrentBag?)
  - Test: Database connection pool exhaustion
  - Test: AI API rate limiting and retry logic
```

---

### 2.4 Data Integrity & Validation Chaos

**Principle**: Users will send garbage. Hackers will send malicious input. Test every validation rule.

**Test Scenarios**:

| Test ID | Scenario | Input | Expected Behavior | Bug Risk |
|---------|----------|-------|-------------------|----------|
| VAL-001 | Invalid enum value | `orderType=999` | 400 Bad Request | Deserialization error, default value assigned |
| VAL-002 | SQL injection attempt | `name="'; DROP TABLE Customers; --"` | Input escaped/rejected | SQL injection |
| VAL-003 | XSS injection attempt | `notes="<script>alert('xss')</script>"` | Escaped on display | XSS vulnerability |
| VAL-004 | Invalid date format | `orderDate="not-a-date"` | 400 Bad Request | Deserialization error, default date |
| VAL-005 | Invalid JSON structure | Malformed JSON payload | 400 Bad Request | 500 Internal Server Error |
| VAL-006 | Mismatched types | `customerId="abc"` (string instead of int) | 400 Bad Request | Deserialization error |
| VAL-007 | Foreign key violation | `tenantId=99999` (non-existent) | 400 Bad Request with clear error | Database foreign key exception |
| VAL-008 | Duplicate unique constraint | Create two orders with same unique key | 409 Conflict | Database constraint violation |
| VAL-009 | Invalid state transition | Move order from "Completed" to "Draft" | 400 Bad Request (if rule exists) | Business rule violation |
| VAL-010 | Circular reference in JSON | `{ a: { b: <ref to a> } }` | 400 Bad Request or handle gracefully | Stack overflow, infinite loop |

**Specific Validation Tests**:
```
[List business-specific validation rules to test]

Example:
- Order must have at least one section completed
  - Test: Submit order with no sections → Reject
- Customer must be assigned to tenant before order
  - Test: Create order for unassigned customer → Reject
```

---

### 2.5 Permission & Security Chaos

**Principle**: Users will try to access data they shouldn't. Test every authorization boundary.

**Test Scenarios**:

| Test ID | Scenario | Setup | Expected Behavior | Bug Risk |
|---------|----------|-------|-------------------|----------|
| SEC-001 | No authentication token | Call API without JWT | 401 Unauthorized | Endpoint accessible without auth |
| SEC-002 | Expired token | Use expired JWT | 401 Unauthorized | Token not validated |
| SEC-003 | Invalid token signature | Tampered JWT | 401 Unauthorized | Token validation bypass |
| SEC-004 | Wrong role for operation | User role tries admin endpoint | 403 Forbidden | Authorization bypass |
| SEC-005 | Cross-tenant data access | User from Tenant A requests Tenant B data | 403 Forbidden or empty result | Data leak, tenant isolation failure |
| SEC-006 | Missing tenant context | User not associated with any tenant | 403 Forbidden or empty result | NullReferenceException |
| SEC-007 | SQL injection in filter | `filter="1=1 OR 1=1"` | Escaped/rejected | SQL injection |
| SEC-008 | Path traversal attempt | `file=../../etc/passwd` | 400 Bad Request | File system access |
| SEC-009 | IDOR (Insecure Direct Object Reference) | User requests order ID from another tenant | 403 Forbidden or 404 | Unauthorized data access |
| SEC-010 | Mass assignment vulnerability | Send extra fields in DTO (e.g., `isAdmin=true`) | Extra fields ignored | Privilege escalation |

**Specific Security Tests**:
```
[Based on implementation, list specific auth/authz scenarios]

Example:
- Nurse role can create orders, but NOT delete
  - Test: Nurse calls DELETE /api/orders/{id} → 403 Forbidden
- User can only view orders from their assigned facilities
  - Test: User with Tenant A tries GET /api/orders?tenantId=B → Empty or 403
```

---

### 2.6 Performance & Resource Exhaustion Tests

**Principle**: Systems fail under load. Test with realistic (and unrealistic) data volumes.

**Test Scenarios**:

| Test ID | Scenario | Setup | Expected Behavior | Bug Risk |
|---------|----------|-------|-------------------|----------|
| PERF-001 | Large dataset query (no pagination) | GET 10,000 orders without pagination | 400 Bad Request or enforced pagination | Memory exhaustion, timeout |
| PERF-002 | N+1 query problem | Load 100 orders with customers (each triggers query) | Response time < 2s (reasonable) | N+1 queries, slow response |
| PERF-003 | Expensive operation in loop | Process 1,000 orders with AI classification in loop | Completes within reasonable time or uses batch | Timeout, high latency |
| PERF-004 | Database connection pool exhaustion | Make 100 concurrent API calls | All succeed or gracefully queue | Connection pool exhaustion, timeouts |
| PERF-005 | Memory leak test | Run batch ETL process repeatedly | Memory usage stabilizes after GC | Memory leak, OutOfMemoryException |
| PERF-006 | Large payload (upload) | Upload 10 MB JSON payload | Accept or reject with size limit | Timeout, memory exhaustion |
| PERF-007 | Slow external API | AI classification API takes 30s to respond | Timeout with retry or fallback | Application hang, no timeout |
| PERF-008 | Unbounded parallelism | ETL process spawns 10,000 parallel tasks | Throttled to reasonable limit (e.g., 10 concurrent) | Thread pool exhaustion, CPU spike |

**Specific Performance Tests**:
```
[Identify performance-critical operations]

Example:
- Dashboard statistics query (all facilities, all customers)
  - Test: Query with 50 facilities, 5,000 customers → Response < 3s
- Classification ETL batch (1,000 orders)
  - Test: Complete within 10 minutes with error handling
```

---

### 2.7 Error Handling & Recovery Tests

**Principle**: Everything that can fail, will fail. Test every failure mode.

**Test Scenarios**:

| Test ID | Scenario | Setup | Expected Behavior | Bug Risk |
|---------|----------|-------|-------------------|----------|
| ERR-001 | Database connection failure | Simulate DB offline | 503 Service Unavailable with retry | 500 error, unhandled exception |
| ERR-002 | Database timeout | Simulate slow query (> 30s) | Timeout with clear error | Application hang |
| ERR-003 | External API failure (AI classification) | Simulate 500 error from API | Retry logic or graceful degradation | Unhandled exception, no classification |
| ERR-004 | External API timeout | Simulate API not responding | Timeout after 10s (or configured) | Hang, no error |
| ERR-005 | Disk space exhaustion | Simulate no disk space for logs | Log warning, continue or fail gracefully | Crash, no error message |
| ERR-006 | Out of memory | Simulate memory exhaustion | Handle gracefully or fail with clear error | Crash, corrupted state |
| ERR-007 | Network partition | Simulate network loss mid-operation | Retry or fail with timeout | Hung connection, no error |
| ERR-008 | Partial failure in batch | 10% of batch items fail validation | Continue processing, report failures | Entire batch fails, no results |
| ERR-009 | Transaction rollback | Simulate error mid-transaction | Rollback all changes, no partial state | Partial commit, data corruption |
| ERR-010 | Deserialization error | Send invalid JSON to API | 400 Bad Request | 500 error, stack trace exposed |

**Specific Error Handling Tests**:
```
[Based on implementation, identify failure points]

Example:
- AI classification API returns malformed response
  - Test: Handle gracefully, log error, mark classification as failed
- Database migration fails mid-execution
  - Test: Rollback to previous state, no partial schema
```

---

### 2.8 Scope Invariant Tests (MANDATORY for features with scoped identifiers)

**Principle**: Any feature that passes scope identifiers (instanceId, conversationId, tenantId, contactId) between services must prove isolation holds at every seam. Scoping bugs are privacy violations and data leaks — they pass all happy-path tests while silently corrupting data.

**When to apply**: Any feature involving agent instances, conversations, tenant-filtered queries, or per-user/per-contact data.

**Required test categories:**

| Test ID | Invariant | Scenario | Expected Behavior | Failure Mode |
|---------|-----------|----------|-------------------|--------------|
| SCOPE-001 | Conversation isolation | Two instances of same agent, each with own conversation — store data under A, read via B | B cannot see A's data | Scope ID silently becomes null, returns all data |
| SCOPE-002 | Null scope fail-safe | Scope ID lookup returns null (instance evicted, race condition, wrong ID) | Fail with error, NOT return all data | Silent null coalesce exposes all records |
| SCOPE-003 | Cross-tenant isolation | Same entity IDs, different tenantIds | Tenant B cannot see Tenant A's data even if IDs match | Missing WHERE tenantId clause |
| SCOPE-004 | Stale reference | Scope ID exists in one service (e.g., in-memory manager) but not another (DB) — or vice versa | Consistent behavior: fail or degrade gracefully, no partial data | In-memory state diverges from DB, one service sees data the other doesn't |
| SCOPE-005 | Scope propagation | Scope ID passed through chain: Controller → Orchestrator → Tool → Repository | Each layer receives and forwards the same scope ID unchanged | ID lost or overwritten at any seam |

**Test pattern for scope isolation:**
```csharp
// RED test first (MANDATORY): write this assertion, run it, confirm it FAILS
// This proves the test can detect a real scoping bug
instanceB_result.Should().Contain(instanceA_data);  // This MUST fail

// Then write the correct assertion:
instanceB_result.Should().NotContain(instanceA_data);  // This MUST pass
```

**Specific Scope Invariants** (from implementation):
```
[For each scoped feature, list the scope identifiers and what data they protect]

Example (agent memory):
- Scope: AgentConversationId (from AgentInstance.AgentConversationId)
- Protected: AgentMemoryItem records
- SCOPE-001: Instance A stores memory → Instance B (same agent, diff conversation) cannot retrieve it
- SCOPE-002: AgentInstanceManager.GetInstance() returns null → AgentMemoryTool returns error, not all memories
- SCOPE-003: tenantId=1 memories invisible to tenantId=2, even if AgentDefinitionId matches
```

---

## 3. Mock Scope Rules (MANDATORY — read before writing any test)

**Rule**: Mock ONLY at true system boundaries. Never mock internal seams when the goal is to test business logic.

| Boundary type | Mock? | Reason |
|--------------|-------|--------|
| LLM providers (OpenAI, Anthropic) | YES | External, non-deterministic, costly |
| Email providers, IMAP/SMTP | YES | External I/O |
| External HTTP APIs | YES | External, unreliable in tests |
| `IClock` / time | YES | Makes tests deterministic |
| `AgentInstanceManager` | **NO** | Core business logic; mocking it hides scope bugs |
| Repositories | **NO** (use in-memory DB) | EF queries must be exercised |
| Orchestrators (in controller tests) | YES | Controller tests verify HTTP contract only |
| Orchestrators (in orchestrator tests) | **NO** | Orchestrator tests verify business logic |

**Controller tests vs Orchestrator tests — both are required, both are correct:**
- `SomeFeatureControllerTests`: Mocks orchestrator. Proves HTTP routing, auth, response shape.
- `SomeFeatureOrchestratorTests`: No HTTP, real orchestrator, real in-memory DB. Proves business logic.

**False confidence gate (MANDATORY):**
Before finalizing the test plan, for at least one scope-sensitive test:
1. Write the assertion wrong (e.g., assert instance B CAN see instance A's memory)
2. Run it — it MUST fail
3. Fix the assertion
4. If you cannot make a test fail by asserting wrong behavior, the test is not detecting real bugs — rethink the test

---

## 3a. Test Case Design Examples: Unit Tests 🧪

All tests and scenarios here are examples.

### 3.1 Service Layer Unit Tests

**Test Pattern**: Arrange-Act-Assert (AAA)

**Template**:
```csharp
[Fact]
public async Task MethodName_Scenario_ExpectedBehavior()
{
    // Arrange
    var mockRepo = new Mock<IRepository>();
    mockRepo.Setup(r => r.GetAsync(It.IsAny<int>())).ReturnsAsync((Entity)null);
    var service = new Service(mockRepo.Object);

    // Act
    var result = await service.MethodAsync(input);

    // Assert
    Assert.Null(result);
    mockRepo.Verify(r => r.GetAsync(It.IsAny<int>()), Times.Once);
}
```

**Test Case Examples**:

| Test ID | Method Under Test | Scenario | Input | Expected Output | Mocks/Setup |
|---------|-------------------|----------|-------|-----------------|-------------|
| SVC-001 | CreateOrderAsync | Valid input | Valid DTO | Success result | Mock repo returns saved entity |
| SVC-002 | CreateOrderAsync | Null input | null | ArgumentNullException | N/A |
| SVC-003 | CreateOrderAsync | Invalid tenant ID | tenantId=0 | Validation error | Mock tenant repo returns null |
| SVC-004 | GetOrderAsync | Non-existent ID | id=999 | Null result | Mock repo returns null |
| SVC-005 | UpdateOrderAsync | Concurrent update | Same ID, different updates | Optimistic concurrency exception | Mock throws DbUpdateConcurrencyException |
| SVC-006 | DeleteOrderAsync | Locked order | Locked status | Business rule error | Mock returns locked order |

**Specific Service Tests** (from implementation):
```
[List methods from implementation with specific test scenarios]

Example:
- OrderTrackerService.GetOrdersByTenantAsync
  - Test: Returns only orders for specified tenant
  - Test: Returns empty list if tenant has no orders
  - Test: Throws exception if tenantId is 0 or negative
```

---

### 3.2 Business Logic Unit Tests

**Focus**: Test business rules in isolation (no database, no external APIs).

**Test Cases**:

| Test ID | Business Rule | Scenario | Input | Expected Result |
|---------|---------------|----------|-------|-----------------|
| BL-001 | Order score calculation | All sections completed | Sections: [10, 20, 30] | Total: 60 |
| BL-002 | Order score calculation | Some sections null | Sections: [10, null, 30] | Total: 40 (ignore null) |
| BL-003 | Classification determination | Score 85 | Score: 85 | Classification: "High" |
| BL-004 | Order status transition | Draft → In Progress → Completed | Status changes | All transitions valid |
| BL-005 | Order status transition | Completed → Draft | Invalid transition | Validation error |
| BL-006 | Date range validation | Order date within 90 days | Today - 30 days | Valid |
| BL-007 | Date range validation | Order date beyond 90 days | Today - 100 days | Invalid |

**Specific Business Logic Tests**:
```
[Extract business rules from implementation and design tests]

Example:
- Domain-specific classification rules
  - Test: Customer with mobility score X → Classification Y
  - Test: Edge cases for classification boundaries
```

---

### 3.3 Data Access Unit Tests

**Focus**: Test repository/DbContext operations with in-memory database or mocks.

**Test Cases**:

| Test ID | Operation | Scenario | Setup | Expected Result |
|---------|-----------|----------|-------|-----------------|
| DA-001 | GetByIdAsync | Entity exists | Seed entity with ID=1 | Returns entity |
| DA-002 | GetByIdAsync | Entity not found | Empty database | Returns null |
| DA-003 | AddAsync | Valid entity | New entity | Entity saved with ID assigned |
| DA-004 | AddAsync | Duplicate unique key | Entity with existing key | DbUpdateException |
| DA-005 | UpdateAsync | Valid update | Update existing entity | Changes saved |
| DA-006 | UpdateAsync | Concurrent update | Two updates simultaneously | DbUpdateConcurrencyException |
| DA-007 | DeleteAsync | Valid delete | Delete existing entity | Entity removed |
| DA-008 | DeleteAsync | Foreign key constraint | Delete entity with children | DbUpdateException (or cascade) |
| DA-009 | Query with includes | Eager load navigation | Include related entities | Single query, no N+1 |
| DA-010 | Query with filtering | Tenant-based filter | Filter by tenantId | Only matching entities returned |

**Specific Data Access Tests**:
```
[Based on repositories/DbContext usage]

Example:
- OrderRepository.GetByCustomerIdAsync
  - Test: Returns all orders for customer
  - Test: Returns empty list if no orders
  - Test: Includes related entities (Customer, Tenant)
```

---

## 4. Test Case Design: Integration Tests 🔗

### 4.1 API Endpoint Integration Tests

**Setup**: In-memory or test database, mock external dependencies (AI APIs).

**Test Cases**:

| Test ID | Endpoint | Method | Scenario | Request | Expected Response | Status Code |
|---------|----------|--------|----------|---------|-------------------|-------------|
| API-001 | /api/orders | POST | Create valid order | Valid JSON body | Order created | 201 |
| API-002 | /api/orders | POST | Create with null body | null | Validation error | 400 |
| API-003 | /api/orders/{id} | GET | Get existing order | Valid ID | Order data | 200 |
| API-004 | /api/orders/{id} | GET | Get non-existent order | Invalid ID | Not found | 404 |
| API-005 | /api/orders/{id} | PUT | Update valid order | Valid ID + body | Updated order | 200 |
| API-006 | /api/orders/{id} | PUT | Update with invalid data | Invalid body | Validation error | 400 |
| API-007 | /api/orders/{id} | DELETE | Delete order | Valid ID | Success message | 204 |
| API-008 | /api/orders/{id} | DELETE | Delete non-existent | Invalid ID | Not found | 404 |
| API-009 | /api/orders | GET | List with pagination | page=1&size=10 | Paginated results | 200 |
| API-010 | /api/orders | GET | List with invalid pagination | page=-1&size=0 | Validation error | 400 |
| API-011 | /api/orders | GET | Unauthorized access | No auth token | Unauthorized | 401 |
| API-012 | /api/orders/{id} | GET | Cross-tenant access | User Tenant A, Order Tenant B | Forbidden or 404 | 403/404 |

**Specific API Integration Tests**:
```
[List all API endpoints from implementation with integration scenarios]

Example:
- POST /api/orders/classify
  - Test: Trigger AI classification for order
  - Test: Handle AI API failure gracefully
  - Test: Validate classification result saved to database
```

---

### 4.2 Database Integration Tests

**Setup**: Test database with migrations applied, seed data.

**Test Cases**:

| Test ID | Scenario | Setup | Operation | Expected Result |
|---------|----------|-------|-----------|-----------------|
| DB-001 | Migration applies cleanly | Empty database | Run migrations | All tables created |
| DB-002 | Rollback migration | Migrated database | Rollback last migration | Schema reverted |
| DB-003 | Seed data loads | Empty database | Run seed scripts | Reference data populated |
| DB-004 | Foreign key cascade | Order with children | Delete order | Children deleted (if cascade) |
| DB-005 | Unique constraint | Duplicate entry | Insert duplicate | Constraint violation |
| DB-006 | Index usage | Large dataset query | Query with WHERE on indexed column | Query plan uses index |
| DB-007 | Transaction rollback | Error mid-transaction | Insert + error + rollback | No data committed |
| DB-008 | Concurrent updates | Two updates simultaneously | Both update same row | One succeeds, one fails with concurrency exception |

**Specific Database Tests**:
```
[Based on schema and migrations]

Example:
- Test: Classification audit trail writes to ClassificationAudit and ClassificationAuditDetails
- Test: Deleting a Tenant cascades to Orders (or prevents delete if has orders)
```

---

### 4.3 ETL & Background Job Integration Tests

**Setup**: Test environment with mock AI APIs, test data.

**Test Cases**:

| Test ID | Scenario | Setup | Operation | Expected Result |
|---------|----------|-------|-----------|-----------------|
| ETL-001 | Process single order | One order needing classification | Run ETL | Classification saved, audit trail created |
| ETL-002 | Process batch of orders | 100 orders | Run ETL | All processed, errors logged |
| ETL-003 | Handle AI API failure | AI API returns 500 | Run ETL | Error logged, order marked as failed |
| ETL-004 | Handle AI API timeout | AI API times out | Run ETL | Timeout handled, retry or fail gracefully |
| ETL-005 | Handle malformed AI response | AI API returns invalid JSON | Run ETL | Error logged, classification fails |
| ETL-006 | Parallel processing | 1000 orders processed in parallel | Run ETL | All processed, no race conditions |
| ETL-007 | Partial batch failure | 10% of batch fails validation | Run ETL | Successful items processed, failures reported |
| ETL-008 | Idempotency | Run ETL twice on same data | Run ETL 2x | Second run detects already processed, no duplicates |

**Specific ETL Tests**:
```
[Based on ETL implementation]

Example:
- ExtractService: Extracts order data from database
  - Test: Handles large datasets with pagination
- TransformService: Applies business rules and AI classification
  - Test: Correctly maps order data to AI API format
- LoadService: Saves classification results
  - Test: Saves to Classification, ClassificationAudit, and ClassificationAuditDetails tables
```

---

### 4.4 Internal Workflow Integration Tests (MANDATORY for agent/tool/instance features)

**Purpose**: Test complete internal workflows that span multiple services with scoped identifiers. These tests do NOT go through HTTP. They instantiate real services directly, wire them together, and assert observable side-effects (DB state, tool results).

**Setup**: In-memory `WorkyBotDbContext` (via `CustomWebApplicationFactory` service scope or direct `DbContextOptionsBuilder`), real `AgentInstanceManager` (singleton in-process), real tool implementations.

**PREREQUISITE for background service tests**: Hosted services (e.g., `ScheduleEvaluationService`) must expose a testable seam before tests can be written:
- Option A: Extract evaluation logic into `EvaluateAsync(DateTimeOffset now)` that tests call directly
- Option B: Inject `IClock` and advance it in tests
- If neither exists: add the seam as a production code change BEFORE writing the test

**Required test cases for agent instance features:**

| Test ID | Workflow | Services Involved | Observable Assertion |
|---------|----------|-------------------|---------------------|
| IWF-001 | Memory isolation between conversations | AgentMemoryTool + AgentInstanceManager + AgentMemoryItemRepository | Instance B cannot retrieve memory stored by Instance A (same agent, different AgentConversationId) |
| IWF-002 | Null scope fail-safe for memory | AgentMemoryTool + AgentInstanceManager | GetInstance() returning null results in error ToolResult, not empty-scope query returning all data |
| IWF-003 | Scheduler creates with correct conversation binding | AgentSchedulerTool + AgentInstanceManager + ScheduleRepository | AgentSchedule.AgentConversationId matches caller instance's AgentConversationId |
| IWF-004 | Scheduler list isolation between conversations | AgentSchedulerTool + AgentInstanceManager + ScheduleRepository | Instance B's ListSchedules does not include Instance A's conversation-bound schedules |
| IWF-005 | Schedule fire resolves to correct instance | ScheduleEvaluationService (direct invocation) + ScheduleRepository + AgentInstanceManager | Due schedule fires, LastFiredAt set, correct AgentDefinitionId+AgentConversationId targeted |
| IWF-006 | Cross-tenant isolation | Any repository + WorkyBotDbContext | tenantId=1 data invisible to tenantId=2 even when other IDs match |

**Specific Internal Workflow Tests**:
```
[For each feature involving agent instances, tools, or background services, list the internal workflows to test]

Example (agent memory feature):
- IWF-001: Store memory via Instance A → recall via Instance B → assert not found
- IWF-002: Remove Instance A from AgentInstanceManager → recall via Instance A's instanceId → assert ToolResult.IsError = true
- IWF-006: Seed memories for tenantId=1 → query with tenantId=2 → assert empty result
```

---

## 5. Test Case Design: End-to-End Tests 🌐

### 5.1 User Workflow E2E Tests

**Setup**: Full application stack (API + Database + Frontend), test user accounts.

**Test Cases**:

| Test ID | Workflow | Steps | Expected Outcome |
|---------|----------|-------|------------------|
| E2E-001 | Create order workflow | 1. Login as nurse<br>2. Navigate to orders<br>3. Click "Create"<br>4. Fill form<br>5. Submit | Order created, visible in list |
| E2E-002 | View order workflow | 1. Login as nurse<br>2. Navigate to orders<br>3. Click order | Order details displayed |
| E2E-003 | Update order workflow | 1. Login as nurse<br>2. Open order<br>3. Edit fields<br>4. Save | Order updated, audit trail created |
| E2E-004 | Delete order workflow | 1. Login as admin<br>2. Open order<br>3. Click delete<br>4. Confirm | Order deleted, no longer in list |
| E2E-005 | Classification workflow | 1. Create order<br>2. Submit for classification<br>3. Wait for result | Classification completed, result displayed |
| E2E-006 | Cross-tenant access prevention | 1. Login as Tenant A user<br>2. Try to access Tenant B order (direct URL) | Forbidden or 404 error |
| E2E-007 | Role-based access | 1. Login as nurse<br>2. Try to access admin-only endpoint | Forbidden error |

**Specific E2E Tests**:
```
[Map critical user journeys from business spec]

Example:
- Complete order lifecycle: Create → In Progress → Classify → Review → Approve
- Dashboard analytics: View statistics, filter by tenant, export report
```

---

### 5.2 Data Flow E2E Tests

**Focus**: Verify data flows correctly through entire system.

**Test Cases**:

| Test ID | Data Flow | Steps | Verification |
|---------|-----------|-------|--------------|
| FLOW-001 | Order creation → Database → API → UI | Create via UI | Data in database matches UI input |
| FLOW-002 | Order → ETL → Classification → Audit | Create → Classify | Classification and audit records created |
| FLOW-003 | Customer assignment → Order → Report | Assign customer → Create order → View report | Customer data appears in order |
| FLOW-004 | Tenant filter → API → Database | Filter by tenant | Only tenant's orders returned |

---

## 6. Test Case Design: Performance Tests ⚡

### 6.1 Load Testing

**Goal**: Verify system handles expected load.

**Test Cases**:

| Test ID | Scenario | Load | Duration | Success Criteria |
|---------|----------|------|----------|------------------|
| LOAD-001 | Normal load | 10 concurrent users | 5 minutes | Response time < 2s, 0 errors |
| LOAD-002 | Peak load | 50 concurrent users | 5 minutes | Response time < 5s, < 1% errors |
| LOAD-003 | Sustained load | 20 concurrent users | 1 hour | No memory leaks, stable performance |
| LOAD-004 | Spike test | 10 → 100 users in 1 min | 10 minutes | System recovers, no crashes |

**Specific Load Tests**:
```
[Based on expected usage patterns]

Example:
- Dashboard statistics query under load: 50 users simultaneously
- ETL batch processing under load: 10 concurrent batch jobs
```

---

### 6.2 Stress Testing

**Goal**: Find breaking point.

**Test Cases**:

| Test ID | Scenario | Load | Success Criteria |
|---------|----------|------|------------------|
| STRESS-001 | Increase load until failure | 10 → 500 users | Identify max capacity, graceful degradation |
| STRESS-002 | Database connection pool exhaustion | 1000 concurrent queries | Connection pooling handles load or queues requests |
| STRESS-003 | Memory exhaustion | Process 1M orders | Memory usage stabilizes, no OutOfMemoryException |

---

## 7. Test Execution Plan 🚀

### 7.1 Test Execution Order

1. **Unit Tests** (run first, fastest feedback)
2. **Integration Tests** (API, Database, ETL)
3. **End-to-End Tests** (complete workflows)
4. **Performance Tests** (load, stress)
5. **Security Tests** (penetration testing, vulnerability scanning)

### 7.2 Test Environment Setup

**Requirements**:
```
[List environment requirements]

Example:
- .NET 9 SDK installed
- SQL Server test database
- Mock AI API service (or test API key)
- Test user accounts (Admin, Nurse, etc.)
- Test data seed scripts
```

**Configuration**:
```
[List test-specific configuration]

Example:
- appsettings.Test.json with test database connection
- AI API mock endpoints
- Reduced timeouts for faster test execution
```

---

### 7.3 Test Automation Strategy

**Unit Tests**: Run on every commit (CI/CD pipeline)
**Integration Tests**: Run on every PR
**E2E Tests**: Run nightly or before release
**Performance Tests**: Run weekly or before major releases

**CI/CD Integration**:
```bash
# Example: Azure DevOps pipeline
- dotnet test ServiceTests --configuration Release --logger "trx" --collect:"XPlat Code Coverage"
- dotnet test IntegrationTests --configuration Release
```

---

## 8. Chaos Engineering: Advanced Destruction Techniques 💣

### 8.1 Chaos Injection Tests

**Principle**: Inject failures at random to test resilience.

**Chaos Scenarios**:

| Chaos ID | Chaos Type | Injection Point | Expected Behavior |
|----------|------------|-----------------|-------------------|
| CHAOS-001 | Random database timeouts | Random queries time out (10% chance) | Retry logic handles, no user errors |
| CHAOS-002 | Random AI API failures | AI API fails randomly (20% chance) | Graceful degradation, fallback classification |
| CHAOS-003 | Network latency | Add 1-5s random delay to all requests | Timeouts configured, no hangs |
| CHAOS-004 | CPU throttling | Limit CPU to 50% | Performance degrades gracefully, no crashes |
| CHAOS-005 | Memory pressure | Limit available memory | Garbage collection aggressive, no OutOfMemory |
| CHAOS-006 | Disk I/O errors | Simulate disk read/write failures | Error handling, retry logic |

**Implementation**:
```csharp
// Example: Chaos middleware for ASP.NET Core
public class ChaosMiddleware
{
    private readonly RequestDelegate _next;
    private readonly Random _random = new Random();

    public async Task InvokeAsync(HttpContext context)
    {
        // 10% chance of random delay (1-5 seconds)
        if (_random.Next(100) < 10)
        {
            await Task.Delay(_random.Next(1000, 5000));
        }

        // 5% chance of random failure
        if (_random.Next(100) < 5)
        {
            context.Response.StatusCode = 500;
            return;
        }

        await _next(context);
    }
}
```

---

### 8.2 Fuzz Testing

**Principle**: Throw random garbage at the system and see what breaks.

**Fuzz Test Strategy**:

| Fuzz ID | Target | Fuzzing Strategy | Expected Behavior |
|---------|--------|------------------|-------------------|
| FUZZ-001 | API endpoints | Random JSON payloads | 400 Bad Request, no crashes |
| FUZZ-002 | API endpoints | Extremely large payloads (100 MB) | 413 Payload Too Large |
| FUZZ-003 | String fields | Random Unicode, emojis, null bytes | Handled gracefully, escaped |
| FUZZ-004 | Numeric fields | Random values (negative, zero, max int) | Validation rejects invalid |
| FUZZ-005 | Date fields | Random strings, timestamps | Deserialization error, 400 Bad Request |

**Tools**:
- **REST API Fuzzer**: [RESTler](https://github.com/microsoft/restler-fuzzer)
- **Generic Fuzzer**: [AFL (American Fuzzy Lop)](https://lcamtuf.coredump.cx/afl/)

---

## 9. Bug Tracking & Reporting 🐛

### 9.1 Bug Report Template

**For each bug found**:

```markdown
## Bug ID: [BUG-XXX]

**Severity**: [ ] Critical [ ] High [ ] Medium [ ] Low

**Test ID**: [Test case that discovered the bug]

**Summary**: [One-line description]

**Steps to Reproduce**:
1. [Step 1]
2. [Step 2]
3. [Step 3]

**Expected Behavior**:
[What should happen]

**Actual Behavior**:
[What actually happened]

**Evidence**:
- Error message: [Full error text or screenshot]
- Stack trace: [If applicable]
- Logs: [Relevant log entries]
- Screenshot: [If UI bug]

**Impact**:
- Data loss: [ ] Yes [ ] No
- Security risk: [ ] Yes [ ] No
- Availability: [ ] Yes [ ] No
- User experience: [ ] Yes [ ] No

**Suggested Fix**:
[If known, suggest how to fix]
```

---

### 9.2 Bug Severity Matrix

| Severity | Criteria | Example |
|----------|----------|---------|
| **Critical** | Data loss, security breach, system crash, blocks all users | SQL injection vulnerability, NullReferenceException crashes API |
| **High** | Major functionality broken, blocks some users, significant security risk | Cross-tenant data leak, authentication bypass |
| **Medium** | Feature partially broken, workaround exists, minor security risk | Validation error shows stack trace, N+1 query slows page load |
| **Low** | Cosmetic issue, minor inconvenience, no security risk | Typo in error message, minor UI misalignment |

---

## 10. Test Results & Quality Metrics 📊

### 10.1 Test Execution Summary

| Test Suite | Total Tests | Passed | Failed | Skipped | Pass Rate |
|------------|-------------|--------|--------|---------|-----------|
| Unit Tests | | | | | |
| Integration Tests | | | | | |
| E2E Tests | | | | | |
| Performance Tests | | | | | |
| **Total** | | | | | |

**Bugs Found**:

| Severity | Count | Fixed | Open |
|----------|-------|-------|------|
| Critical | | | |
| High | | | |
| Medium | | | |
| Low | | | |
| **Total** | | | |

---

### 10.2 Code Coverage Metrics

**Target Coverage**: 80% (pragmatic, not dogmatic)

| Component | Line Coverage | Branch Coverage | Notes |
|-----------|---------------|-----------------|-------|
| Services | | | |
| Repositories | | | |
| Controllers | | | |
| Business Logic | | | |
| **Overall** | | | |

**Coverage Gaps** (untested code):
```
[List critical code paths with insufficient coverage]
```

---

### 10.3 Quality Gates

**Criteria for Passing**:
- [ ] All Critical and High severity bugs fixed
- [ ] Code coverage ≥ 80% (or project-specific target)
- [ ] All unit tests pass
- [ ] All integration tests pass
- [ ] Performance tests meet SLA (e.g., response time < 2s)
- [ ] Security tests pass (no critical vulnerabilities)
- [ ] Zero data integrity issues found
- [ ] Zero compliance violations found (adapt to your domain requirements)

**Go/No-Go Decision**:
- [ ] **GO** - Ready for deployment
- [ ] **NO-GO** - Issues must be resolved before deployment

---

## 11. Test Retrospective & Lessons Learned 🎓

### 11.1 What Went Well

```
[What testing strategies were effective?]

Example:
- Boundary value testing caught 5 validation bugs early
- Concurrency tests revealed a race condition in ETL batch processing
```

### 11.2 What Went Wrong

```
[What testing strategies were ineffective or missed bugs?]

Example:
- E2E tests were too slow, need to optimize test data setup
- Performance tests didn't catch N+1 query issue until production
```

### 11.3 Improvements for Next Time

```
[Actionable improvements for future test planning]

Example:
- Add database query profiling to integration tests
- Implement chaos engineering in staging environment
- Increase code coverage target to 85%
```

---

## 12. Appendix: Test Data & Test Utilities 🛠️

Example data strategies

### 12.1 Test Data Strategy

**Seed Data**:
```
[Define test data fixtures]

Example:
- 3 test facilities (Tenant A, B, C)
- 10 test customers per tenant
- 5 test users (Admin, Nurse1, Nurse2 for Tenant A, Nurse3 for Tenant B)
- 20 test orders (various states: Draft, In Progress, Completed)
```

**Data Builders**:
```csharp
// Example: Test data builder
public class OrderBuilder
{
    private Order _order = new Order();

    public OrderBuilder WithId(int id)
    {
        _order.Id = id;
        return this;
    }

    public OrderBuilder WithCustomerId(int customerId)
    {
        _order.CustomerId = customerId;
        return this;
    }

    public OrderBuilder WithStatus(OrderStatus status)
    {
        _order.Status = status;
        return this;
    }

    public Order Build() => _order;
}

// Usage
var order = new OrderBuilder()
    .WithId(1)
    .WithCustomerId(100)
    .WithStatus(OrderStatus.Completed)
    .Build();
```

---

### 12.2 Test Utilities

**Mock Helpers**:
```csharp
// Example: Mock repository setup
public static class MockHelpers
{
    public static Mock<IOrderRepository> CreateOrderRepositoryMock()
    {
        var mock = new Mock<IOrderRepository>();
        mock.Setup(r => r.GetByIdAsync(It.IsAny<int>()))
            .ReturnsAsync((int id) => new Order { Id = id });
        return mock;
    }
}
```

**Test Database Helpers**:
```csharp
// Example: In-memory database setup
public static class TestDbContext
{
    public static AppDbContext CreateInMemoryContext()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase(Guid.NewGuid().ToString())
            .Options;

        var context = new AppDbContext(options);
        context.Database.EnsureCreated();
        return context;
    }
}
```

---

## Final Checklist: The Chaos Demon's Seal of Approval ☠️

- [ ] **Null Chaos**: Every input tested with null, empty, whitespace
- [ ] **Boundary Chaos**: Tested at zero, negative, max values, overflow
- [ ] **Concurrency Chaos**: Parallel operations, race conditions, deadlocks tested
- [ ] **Validation Chaos**: SQL injection, XSS, invalid enums, malformed JSON tested
- [ ] **Security Chaos**: Auth bypass, cross-tenant access, privilege escalation tested
- [ ] **Performance Chaos**: N+1 queries, memory leaks, connection pool exhaustion tested
- [ ] **Error Chaos**: Database failures, API timeouts, network partitions tested
- [ ] **Data Integrity Chaos**: Foreign key violations, transaction rollbacks, duplicate keys tested
- [ ] **Compliance Chaos**: PII leaks, missing audit trails, unauthorized access tested

**IF ALL CHAOS TESTS PASS**:
**THEN** your code has been forged in fire and is ready for production.
**ELSE** back to the drawing board, developer. The chaos demon is not satisfied.

---

**Template Version**: 1.0
**Last Updated**: 2025-11-24
**Applicable to**: Any System
**Philosophy**: "Break it before production does. Test with malice, deploy with confidence."
