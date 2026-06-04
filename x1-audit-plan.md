# X1 Audit Plan

**Report:** "Using X1 Audit methodology to verify implementation completeness."

## Purpose

Audit implementation against a plan to identify gaps. Use this **after implementation** to ensure nothing was missed before considering work complete.

**Inputs (read all that apply):** the **requirements artifact** (`specs/…`, the ACs), the **plan** (the IMP checklist), and — when the feature fanned out — the **spine** (its coverage ledger). The ACs are defined in `specs/…`, not the plan; do not report an AC as a gap just because it isn't restated in the plan.

**Two-pass audit:**
1. **Requirements coverage** — via the spine's coverage ledger (N>1) or the plan's Traceability Matrix (N=1): every `AC-{EPOCH}-N.M` is owned by exactly one slice/plan and implemented; every touched invariant from the Impact Matrix has a test. *Forward:* no unowned AC. *Backward:* no `IMP-{EPOCH}.{SLICE}` that fails to trace to an AC.
2. **Plan-item completion** — every IMP item in the (slice) plan is done/justified.

**When to Use:**
- After implementing from an `/x1-plan` implementation plan
- After implementing tests from an `/x1-test-plan` test plan
- Before marking a feature as "done"
- When you suspect implementation may be incomplete

---

## Instructions

1. **Identify the plan** - Ask for plan file path if not provided
2. **Read the plan** - Extract all planned items (files, features, tests, etc.)
3. **Verify against codebase** - Check each item exists and is correctly implemented
4. **Report status** - Categorize each item
5. **Generate gap-filling tasks** - Create actionable list for missing items
6. **Fill gaps** - Implement missing items (or ask user preference)

**Goal:** 100% plan coverage with no missing items.

---

## Phase 1: Plan Extraction

Read the plan file and extract ALL items into categories:

### For Implementation Plans
- [ ] Files to create (with paths)
- [ ] Files to modify (with paths)
- [ ] Database changes (entities, migrations, configurations)
- [ ] API endpoints (with methods and routes)
- [ ] Services/orchestrators to implement
- [ ] DI registrations needed
- [ ] Business logic rules
- [ ] Error handling requirements
- [ ] Edge cases to handle

### For Test Plans
- [ ] Test files to create
- [ ] Test classes by category (Unit, Integration, E2E)
- [ ] Individual test cases (by Test ID if specified)
- [ ] Test fixtures/builders needed
- [ ] Mock implementations needed
- [ ] Test data setup requirements

---

## Phase 2: Verification

For each extracted item, verify implementation:

### File Verification
```
For each planned file:
1. Does file exist at specified path?
2. Does it contain the planned functionality?
3. Does it follow specified patterns?
```

### Code Verification
```
For each planned feature/function:
1. Is it implemented?
2. Does it match the plan's specification?
3. Are edge cases handled as planned?
```

### Test Verification
```
For each planned test:
1. Does test file exist?
2. Does test method exist with correct name?
3. Does test cover the specified scenario?
4. Is test passing?
```

---

## Phase 3: Status Report

Categorize each item:

| Status | Symbol | Meaning |
|--------|--------|---------|
| Implemented | ✅ | Fully implemented as planned |
| Partial | ⚠️ | Exists but incomplete or differs from plan |
| Missing | ❌ | Not implemented at all |
| Skipped | ⏭️ | Intentionally skipped (with documented reason) |
| N/A | ➖ | Not applicable (plan changed, requirement removed) |

---

## Phase 4: Gap Report

### Output Format

```markdown
# Audit Report: [Plan Name]

**Plan File:** [path]
**Audit Date:** [date]
**Overall Status:** [X of Y items complete] ([percentage]%)

## Summary

| Category | ✅ Done | ⚠️ Partial | ❌ Missing | Total |
|----------|---------|------------|------------|-------|
| Files    |    X    |     X      |     X      |   X   |
| Features |    X    |     X      |     X      |   X   |
| Tests    |    X    |     X      |     X      |   X   |
| **Total**|  **X**  |   **X**    |   **X**    | **X** |

## Detailed Status

### Files
- ✅ `src/path/to/file.cs` - Implemented
- ⚠️ `src/path/to/other.cs` - Missing error handling
- ❌ `src/path/to/missing.cs` - Not created

### Features
- ✅ Feature A - Working as specified
- ❌ Feature B - Not implemented

### Tests
- ✅ TEST-001: Test description
- ❌ TEST-002: Test description - Missing

## Gaps to Fill

### High Priority (Blocking)
1. [ ] Create missing file: `src/path/to/missing.cs`
2. [ ] Implement Feature B

### Medium Priority (Incomplete)
1. [ ] Add error handling to `src/path/to/other.cs`

### Low Priority (Polish)
1. [ ] Add XML documentation
```

---

## Phase 5: Gap Filling

After reporting gaps, ask user:

**"Found [X] gaps. Options:**
1. **Fill all gaps now** - I'll implement missing items
2. **Fill specific gaps** - Tell me which ones to prioritize
3. **Report only** - Just document gaps for later
4. **Review gaps first** - Let's discuss before proceeding"

If filling gaps:
- Use TodoWrite to track each gap as a task
- Implement systematically
- Re-verify after each fix
- Run tests to confirm

---

## Verification Techniques

### For Files
```bash
# Check file exists
ls path/to/file.cs

# Search for specific implementation
grep -n "MethodName" path/to/file.cs
```

### For Tests
```bash
# Run specific test to verify it exists and passes
dotnet test --filter "TestClassName.TestMethodName"

# List all test methods in a file
grep -n "\[Fact\]" path/to/tests.cs
```

### For Database
```bash
# Check migration exists
ls src/*/Migrations/*MigrationName*

# Check entity configuration
grep -n "EntityName" src/*/Data/*Configuration.cs
```

### For API Endpoints
```bash
# Check controller has endpoint
grep -n "HttpGet\|HttpPost\|HttpPut\|HttpDelete" src/*/Controllers/*Controller.cs
```

---

## Common Gap Patterns

### Implementation Plans
- **DI Registration** - Service created but not registered
- **Error Handling** - Happy path done, error cases missing
- **Edge Cases** - Core logic done, boundaries not handled
- **Validation** - Business logic done, input validation missing
- **Logging** - Functionality works, no observability

### Test Plans
- **E2E Tests** - Unit tests done, integration tests missing
- **Error Scenarios** - Success cases tested, failure cases not
- **Boundary Tests** - Normal values tested, edge values not
- **Security Tests** - Functionality tested, auth/authz not
- **Concurrency Tests** - Single-threaded works, parallel untested

---

## Phase 6: Re-Verification Loop

After gap-filling, re-run Phases 2-4 to confirm:
1. All filled gaps are actually implemented (not just planned)
2. No new gaps were introduced during gap-filling
3. Final count matches: 0 ❌, 0 ⚠️

**Loop until:** Zero gaps remain or user explicitly accepts remaining items.

If audit was preceded by `/x1-implement`, report:
- "Implementation gate reported N items complete. Audit confirms: [match/mismatch]"

---

## Integration with X1 Workflow

```
/x1-plan          → Create plan with Implementation Checklist
/x1-review-plan   → Validate plan before coding
/x1-implement     → Implement with completeness tracking
/x1-audit-plan    → Verify completeness (safety net) ← YOU ARE HERE
/x1-code-review   → Review code quality
/x1-git-commit    → Commit changes
```

---

## Red Flags During Audit

- **"I thought I did that"** - If uncertain, verify in code
- **Plan section with no corresponding code** - Likely missed
- **Code that doesn't match plan** - Either plan changed or implementation drifted
- **Tests passing but not covering plan scenarios** - False confidence
- **Skipped items without documentation** - Ask why skipped

---

## Final Checklist

Before marking audit complete:

- [ ] Every plan item has a status (✅⚠️❌⏭️➖)
- [ ] All ❌ items either filled or documented why skipped
- [ ] All ⚠️ items either completed or documented as acceptable
- [ ] Tests run and pass after gap-filling
- [ ] User confirms audit is satisfactory

---

## Example Usage

**User:** `/x1-audit-plan docs/CRM_ENHANCEMENT_SPEC.md`

**Claude:**
1. Reads the implementation plan
2. Extracts all planned items (files, features, tests)
3. Verifies each against codebase
4. Reports: "Found 47 items: 42 ✅, 3 ⚠️, 2 ❌"
5. Lists gaps with priorities
6. Asks: "Fill gaps now or report only?"

---

**Template Version:** 1.0
**Created:** 2026-01-24
**Purpose:** Ensure implementation completeness against plans
