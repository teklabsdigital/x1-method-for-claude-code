# X1 Problem Solve

**Report:** "Using X1 Problem Solving methodology to diagnose and fix."

## Core Principle

**Observe, hypothesize, validate, then fix.** No code changes until the hypothesis is confirmed with measured data. Never jump from observation to fix.

## Scope

Primarily for **Expo React Native client application** bugs where the developer cannot directly observe the user's device. Server-side issues rarely need this rigour; console logging and stack traces are usually sufficient.

---

## The Diagnostic Pipeline

### Phase 1: Observe

1. **Get the user's observation.** What do they see? When does it happen? What's the expected behavior?
2. **Request visual evidence** before theorizing. Be prescriptive:
   - "Screenshot the problem state"
   - "Tap the problem area with Element Inspector (RN dev menu > Show Element Inspector) and screenshot it"
   - "Screen record the transition"
   - Device and OS version if not already known
3. **Find the last working state.** `git log` + `git show` to find what changed. The diff between working and broken is the search space.
4. **Check platform matrix.** Which platforms/devices are affected? Which are verified working?

### Phase 2: Hypothesize

5. **Classify the problem type:**
   - **Layout/sizing** - container dimensions, padding, margins not adding up
   - **Timing/race** - events firing in wrong order or competing
   - **State management** - wrong state transitions or stale state
   - **Platform-specific** - OS version quirks, API behavior differences
   - **Data flow** - wrong props, missing updates, broken bindings
6. **Form multiple hypotheses in parallel.** Don't settle on one theory. Generate 2-4 plausible explanations based on the code, the symptoms, and the classification. Each hypothesis should be a concrete, falsifiable statement, e.g.:
   - H1: "KAV is not receiving keyboard events on Android 15 edge-to-edge"
   - H2: "KAV receives events but toolbar height change during transition confuses its retraction calculation"
   - H3: "adjustResize is double-compensating with KAV padding"
7. **For each hypothesis, define what the data would show if true.** This is critical. Before instrumenting, write down what values you'd expect to see for each hypothesis. e.g.:
   - H1 true: keyboard show/hide events never fire
   - H2 true: open delta != close delta, difference matches toolbar height change
   - H3 true: viewport shrinks by more than keyboard height

### Phase 3: Validate

8. **Design ONE instrumentation pass that discriminates between ALL hypotheses.** The user's time is expensive. Every "test on device and share logs" is a round trip. Design the logging so that one pass confirms or rejects every hypothesis simultaneously.
9. **Instrumentation rules:**
   - Log **state transitions**, not renders. Only when something changes.
   - Use a **filterable prefix** (e.g. `[KB]`, `[NAV]`, `[AUTH]`)
   - Include the **phase/event name** in each log (e.g. `KB_OPENED`, `KB_CLOSED`, not just values)
   - **5-10 log points maximum.** If you need more, your hypotheses aren't specific enough.
   - No per-frame, per-scroll, or per-render logging. Ever.
   - Deploy instrumentation only. No other code changes.
10. **User reproduces and shares logs.** One round trip.
11. **Evaluate each hypothesis against the data:**
    - Write down what the numbers say before drawing conclusions
    - Check symmetry (open delta vs close delta), totals (do heights add up?), sequences (correct order?)
    - If data says "working correctly" but user says otherwise: your hypotheses are all wrong. Your mental model is wrong, not the user's eyes. Go back to step 6 with new hypotheses.
12. **Confirm the root cause in one sentence** tied to measured data. e.g. "Toolbar height changes by 46px during KAV transition, causing KAV to miscalculate retraction by 76px." If you can't state it in one sentence, you don't understand it yet.

### Phase 4: Design the Fix

Once user confirms the root cause is identified:

13. **Write a failing test first** that reproduces the bug (TDD).
    - If testable in the component test harness: write a unit/integration test that fails now, will pass after fix.
    - If purely visual/device-specific: document **manual verification steps** as acceptance criteria. This is acceptable.
14. **Auto-trigger lightweight `/x1-plan`:**
    - Link to user story (which US/AC does this bug violate? If none, draft a bug-fix AC)
    - Root cause summary (one sentence with measured data)
    - Minimal IMP checklist: what changes, why (tied to root cause), verification (test or manual step)
    - Platform impact: confirm fix does not regress other platforms
    - Test traceability: Bug symptom -> Root cause -> IMP items -> Tests
    - This is lightweight. No architectural deep-dive unless the fix requires structural changes.
15. **Auto-trigger `/x1-code-review`** on the plan before implementing.
16. **Get user approval.**

### Phase 5: Implement and Verify

17. **Implement the fix** per the approved plan. Single change addressing the root cause.
18. **Run regression test.** It should now pass.
19. **User verifies on device.** Original symptom gone, no new symptoms.
20. **Remove all instrumentation** from Phase 3.
21. **Commit** via `/x1-git-commit` with root cause in the message.

---

## Available Diagnostic Channels

Use the right channel for the problem. Don't rely solely on console logs when visual tools would be faster.

| Channel | What it gives you | When to use |
|---------|------------------|-------------|
| **Screenshots/recordings** | Exact visual state | **Always ask first for UI problems** |
| **Element Inspector** (RN dev menu) | Dimensions, padding, margins as overlay | Layout/spacing; shows exactly which View has extra space |
| **Console logs** | Dynamic state, event sequences, measurements | State transitions, timing, data flow |
| **React DevTools** (`npx react-devtools`) | Component tree, props, state | Wrong props, unexpected re-renders, state bugs |
| **Chrome DevTools for Hermes** (`j` in Metro) | Breakpoints, network, console | Complex logic bugs, network issues |
| **Git history** (`git log`, `git show`, `git diff`) | Last working state, what changed | Regressions, "this used to work" |
| **Platform docs/issues** | Known bugs, API behavior | Platform-specific behavior, OS version quirks |
| **Test harness** | Isolated component behavior | Narrowing bug to library vs host app |

---

## Anti-Patterns

1. **Don't jump from observation to fix.** Observe -> Hypothesize -> Validate -> Fix. Never skip steps.
2. **Don't settle on one hypothesis.** Form multiple, validate in parallel with one instrumentation pass.
3. **Don't combine multiple speculative fixes.** One change at a time. If you layer three changes, you learn nothing.
4. **Don't try the opposite of a failed fix** without new data. That's guess-and-check, not diagnosis.
5. **Don't conclude "working correctly" from logs when the user reports a visual problem.** You can't see their screen.
6. **Don't flood logs.** Per-render logging buries the signal in noise.
7. **Don't skip reading the old working code.** The diff between working and broken is almost always smaller than you think.
8. **Don't change code you haven't measured.** "This might be the problem" is not a hypothesis; "this will show X in the data if true" is.

---

## Workflow Summary

```
Observe -> Hypothesize (parallel) -> Validate (single pass) -> Confirm root cause
    -> Regression test (TDD) -> /x1-plan (lightweight) -> /x1-code-review
    -> Implement -> Verify on device -> Clean up -> /x1-git-commit
```

---

## Phase Transition Checklists

**Before instrumenting (Phase 2 -> 3):**
- [ ] Visual evidence requested
- [ ] Last working version identified (or confirmed new code)
- [ ] Platform matrix noted
- [ ] 2-4 hypotheses formed, each with expected data signature
- [ ] Single instrumentation pass designed to discriminate all hypotheses (5-10 log points max)

**Before designing fix (Phase 3 -> 4):**
- [ ] Root cause confirmed in one sentence with measured data
- [ ] User agrees with the diagnosis

**Before implementing (Phase 4 -> 5):**
- [ ] Regression test written (or manual verification documented)
- [ ] Lightweight plan reviewed via `/x1-code-review`
- [ ] User approved the plan
- [ ] No platform regression risk identified

**Before committing (Phase 5 done):**
- [ ] Regression test passes
- [ ] User confirms symptom resolved on device
- [ ] All instrumentation removed
- [ ] No new symptoms introduced
