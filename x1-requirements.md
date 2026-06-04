# X1 Requirements

**Report:** "Using X1 Requirements methodology to discover the problem and lock user stories."

This skill owns the **non-technical** front of the X1 pipeline: problem discovery, the user-story workshop, and story stress-testing. It produces a **locked requirements artifact** and nothing else — no architecture, no code, no file paths, no class names. When stories are approved, hand off to `/x1-plan`.

> **Why this is its own skill.** Requirements and architecture are different jobs with different mindsets. Keeping them physically separate stops technical detail (types, events, invariants) from bleeding *up* into the stories, and gives the architecture skill a clean, locked input to build on.

## X1 Core Principles (Apply Throughout)
- **Fail fast:** Throw on missing config. No fallbacks that hide failures.
- **Reuse before creating:** Prefer existing capability over new.
- **Full traceability:** US -> AC -> IMP -> Test. Every link visible, every AC covered.
- **User stories first:** Requirements are locked before any technical investigation begins.
- **Observable:** Every feature must answer "How will we know this is broken in production?"

## Instructions

1. **Discover the problem** — spend time here. Don't rush to solutions.
2. **Challenge and think laterally** — be a thinking partner, not a yes-machine.
3. **Run the User Story Workshop** — collaboratively refine and approve stories with acceptance criteria.
4. **Stress-test the stories** — find AC gaps by walking end-to-end scenarios.
5. **Write the requirements artifact** — stories + ACs + stress scenarios + gap analysis + rationale.
6. **Hand off** — "Stories approved — proceed to `/x1-plan`."

**Speed:** Work in parallel where possible. But do NOT investigate the codebase here — that belongs to `/x1-plan`.

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
- If something seems like a bad idea, push back respectfully, then respect the user's call — it's their codebase.

**The most valuable thing in early planning is divergent thinking. Don't converge on a solution too quickly. Explore the problem space.**

**DO NOT propose final solutions in this phase. Understand and challenge first.**

---

## Phase 2: User Story Workshop (BLOCKING GATE)

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
- ACs must not name specific classes, methods, fields, parameters, or database columns. Those details belong in the IMP checklist (authored later in `/x1-plan`).
- Litmus test: can this AC be verified by an integration/E2E test that exercises the real flow across service boundaries? If it can only be verified by inspecting a single class's internal state, it's an implementation detail, not an acceptance criterion.
- ACs feed directly into `/x1-test-plan` which builds acceptance tests from them. "Given X, when Y, then Z is observable" produces a test that catches real seam failures.
- **Why this matters:** Testing breaks down at the seams between services. An AC that names a single class drives a unit test for that class. An AC that describes an E2E outcome drives an integration test that catches the real failures: incorrect handoffs, stale state, missing wiring, broken DI.

**Bad example (system perspective, unit-level ACs, no epoch):**
```
US-1: Agent Retains Conversation Context Between Turns
As an AI agent, I need my conversation context to persist in my
in-memory message buffer across turns...

AC-1.1: After a multi-turn completes, assistant responses and tool
results from StepFinishEvent.NewMessages are added to the instance's
_messages list.
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

AC-CTX-1.3: When context exceeds the compaction threshold, the system
compacts before the next execution, and the agent continues
functioning with the compacted context.
```
ACs are technical but testable E2E. Epoch-scoped IDs are unique across all plans.

**The goal is the MINIMAL set of stories that captures the FULL problem. Not more, not fewer.**

**Process:**
1. Listen to the problem discovery conversation
2. Draft an initial set of user stories. Be opinionated. Propose what you think is right.
3. For each story, state WHY you included it and what you'd lose by cutting it
4. Challenge the user: "I think US-3 could be merged into US-1. Here's why..."
5. Ask probing questions about assumptions
6. Iterate. Add, remove, merge, split based on discussion.
7. When both sides agree the stories are tight, present the final Gate Document
8. Ask: "Are these user stories and acceptance criteria complete and accurate?"
9. Repeat from step 6 if the user has changes

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
- Define the **epoch prefix** before writing any stories. Derive it from the feature name (2-4 uppercase letters). All IDs use this prefix throughout the pipeline. This is the key that joins requirements → plan(s) → tests.
- Do NOT investigate the codebase. Do NOT reference file paths, class names, or technical implementation. Do NOT write code snippets or propose architecture. Do NOT discuss "how" until the "what" is locked.
- Roles must be **human stakeholders**, never "the system" or "an AI agent".
- ACs must describe **testable E2E outcomes** across service boundaries. No class/method/field/column names in ACs.
- Each AC will drive a [TEST]+[CODE] IMP pair downstream (TDD: test first).
- Every story must justify its existence: what do we lose if we cut it?

**The user story gate is the foundation of the entire X1 workflow.** Plan items trace to stories; IMP items map to ACs; tests verify ACs; audit verifies traceability end-to-end.

### Stress Test User Stories (BLOCKING GATE)

**After user stories are approved but BEFORE handing off, stress test the acceptance criteria with end-to-end scenarios.**

Requirements that look complete in isolation break when composed. The goal is to find gaps in the ACs by running realistic scenarios that cross multiple stories and multiple concerns simultaneously.

**Process:**
1. For each user story, generate 3-5 stress scenarios that combine multiple ACs, concurrent operations, or edge conditions
2. Walk each scenario through the acceptance criteria step by step
3. At each step ask: "Is there an AC that covers what happens here? If not, we have a gap."
4. Add missing ACs or refine existing ones based on gaps found
5. Re-present updated stories for approval

**Stress scenario categories:**
- **Temporal collisions:** Two things happening at the same time
- **State transitions under load:** What happens at boundaries (last item completes, budget hits exactly 100%)
- **Failure during multi-step operations:** Operation fails halfway through
- **Ordering assumptions:** Does the system assume A happens before B? What if B happens first?
- **Cascading effects:** Action on entity A triggers changes to B, C, D. Are all downstream effects covered?

**This step typically finds 3-8 missing or imprecise ACs.** These are the ACs that would otherwise become bugs discovered during implementation or, worse, in production.

**Output:** Updated user stories with stress-test-derived ACs marked (e.g. "AC-X.30: Idempotent operations [stress-tested]"). Present for re-approval before proceeding.

---

## Phase 3: Write the Requirements Artifact

When the stories are approved and stress-tested, write the **locked requirements artifact**. This is the hand-off contract to `/x1-plan` — and the anti-handoff-loss mitigation: it carries not just the bare US/AC list but the *reasoning*, so the architect inherits context, not just requirements.

**Location:** `specs/user-story.md` (or `specs/<feature>-requirements.md` for a scoped feature). `specs/` holds requirements only; everything technical lives under `docs/`.

**The artifact contains:**
1. **Epoch + metadata** — the `{PREFIX}`, date, and any prototypes/sources.
2. **User stories + acceptance criteria** — the approved Gate Document (`US-{EPOCH}-N`, `AC-{EPOCH}-N.M`).
3. **Stress scenarios** — the end-to-end scenarios walked in the stress test, and the gaps they exposed (so the architect can see what shaped each AC).
4. **Gap analysis** — what was considered and explicitly left out of scope, and why.
5. **Decisions / rationale** — for any contentious or non-obvious AC, a one-line *why* and what alternative was rejected. This is what the architect would otherwise lose across the skill boundary.

**Do NOT include:** architecture, layer placement, invariants, file paths, class names, code, or IMP items. Those are authored by `/x1-plan`, which reads this artifact.

---

## When Complete

Report:
- Problem clearly articulated
- Epoch prefix defined
- User stories captured with acceptance criteria, approved by the user
- Stories stress-tested; gap-derived ACs added
- Requirements artifact written to `specs/…` with scenarios + gap analysis + rationale

**Next step:** `/x1-plan` — it loads this artifact and authors the technical architecture and implementation plan(s). Do not investigate the codebase or design architecture here.
