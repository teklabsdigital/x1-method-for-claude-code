# X1 Implement

**Report:** "Using X1 Implementation methodology with completeness tracking."

## Instructions

1. Identify the approved plan (ask if unclear)
2. Extract the Implementation Checklist (all IMP-NNN items)
3. Display the checklist with count: "Found N items to implement"
4. Implement systematically — update TodoWrite with each item
5. After implementing, self-verify EVERY item against the codebase
6. Report completion status — cannot declare done with unresolved items

**Goal:** 100% implementation of the approved plan. No gaps. No silent skips.

**Speed:** Work in parallel wherever possible — read multiple files simultaneously, implement independent items concurrently.

---

## Phase 1: Checklist Extraction

Read the plan and extract ALL IMP-NNN items into a tracking table:

| ID | Category | Description | Status |
|----|----------|-------------|--------|
| IMP-001 | File | Create `path/to/file` | ⬜ Pending |
| IMP-002 | Feature | Implement X | ⬜ Pending |
| ... | ... | ... | ... |

**Count:** "Extracted N items. Beginning implementation."

If no Implementation Checklist exists in the plan, STOP and tell the user: "This plan has no Implementation Checklist. Run `/x1-plan` to create a plan with numbered IMP items first."

---

## Phase 2: Systematic Implementation

For each IMP item:
1. Mark as 🔄 In Progress (update TodoWrite)
2. Implement the item
3. Verify it works (build, basic test)
4. Mark as ✅ Done

**Rules:**
- Implement in **TDD order**:
  1. Infrastructure first (DI, config, migrations, schema) — prerequisites
  2. For each AC (in dependency order):
     a. Implement the [TEST] item — run it, confirm **RED** (fails because code doesn't exist yet)
     b. Implement the [CODE] item — run it, confirm **GREEN** (test passes)
     c. Refactor if needed, confirm still GREEN
  3. If the plan has no [TEST]/[CODE] pairs, fall back to dependency order
- Do NOT skip items silently. If an item cannot be implemented, mark it ⏭️ Skipped with a justification and **inform the user immediately**
- Do NOT batch — complete each item before starting the next
- Update progress after EVERY item: "Completed IMP-CTX-003 (5/47)"
- When implementing [TEST] IMP items, ensure each test has `[Trait("UserStory", "US-{PREFIX}-N")]` as specified in the plan

---

## Phase 3: Self-Verification (MANDATORY — DO NOT SKIP)

After implementing all items, verify EVERY item against the codebase:

For each IMP item:
1. Does the file/function/test actually exist?
2. Does it match the plan's specification?
3. Is it complete (not a stub, not partial)?

Update status:
- ✅ Verified — exists and matches plan
- ⚠️ Partial — exists but incomplete
- ❌ Missing — not found in codebase

**TDD Verification (for plans with [TEST]/[CODE] pairs):**
For each [TEST] IMP item:
- Was it implemented before its corresponding [CODE] item?
- Did it fail (RED) before the code was written?
- Does it pass (GREEN) after the code was written?

If TDD order was violated (code written before test), note it in the completion report. This is not a blocker but should be flagged.
- ⏭️ Skipped — intentionally deferred (justification required)
- ➖ N/A — requirement changed during implementation (justification required)

---

## Phase 4: Completion Gate (BLOCKING)

**You CANNOT declare implementation complete until this gate passes.**

### Gate Criteria
1. Every IMP item has a status (no ⬜ Pending items remain)
2. Zero ❌ Missing items
3. Zero ⚠️ Partial items
4. All ⏭️ Skipped items have documented justification
5. All ➖ N/A items have documented justification
6. Tests pass (if tests were in the checklist)

### Completion Report (Gate PASSED)

```
## Implementation Complete

**Plan:** [plan name/path]
**Total Items:** N
**Status:**
- ✅ Implemented: X
- ⏭️ Skipped (justified): Y
- ➖ N/A: Z

### Skipped Items (if any)
- IMP-XXX: [reason for skipping]

### Verification: All implemented items verified against codebase.
**Gate Status: PASSED**
```

### If Gate FAILS

```
## Implementation Incomplete — Gate FAILED

**Remaining:**
- ❌ IMP-XXX: [description] — Not implemented
- ⚠️ IMP-YYY: [description] — Partial: [what's missing]

**Action Required:** Implement remaining items before declaring done.
```

Then continue implementing the remaining items. Re-run the gate after each pass.

---

## Red Flags During Implementation

- 🚩 Skipping an item "because it's simple" — implement it anyway
- 🚩 "Will do this after" — do it now, it's in the checklist
- 🚩 Implementing a different approach than planned without discussing — flag to user
- 🚩 Tests not implemented because "implementation works" — tests are in the checklist
- 🚩 Declaring done without running self-verification — Phase 3 is mandatory
- 🚩 Partial implementation claimed as complete — verify against the spec, not against "it compiles"

---

## When Complete

Report completion status with gate result.

**Next step:** `/x1-audit-plan` for independent verification (should find 0 gaps)
