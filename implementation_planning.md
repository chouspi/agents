# Implementation Planning

How to produce an implementation plan that a different agent, with no memory of this conversation, can execute stage by stage and get a verified, correct result.

A plan is **not** a to-do list. It is a body of documents that (1) fixes what must be true when the work is done, (2) fixes the architecture and the rules that must never be broken, (3) splits the work into stages that each end in proof, and (4) leaves an auditable trail. Quality comes from precision, traceability, and refusing to accept "done" without evidence.

---

## 1. Principles

1. **Plan for a cold-start executor.** Assume the reader has never seen this conversation. Every stage brief must be understandable on its own plus the documents it references. No "as discussed".
2. **Evidence over claims.** A statement like "done", "works", "matches the old behavior" is worthless without a command that was run, a test that passed, or an artifact that can be opened. Plans must demand evidence and executors must record it.
3. **Verify the world, don't trust the docs.** Previous handoffs, READMEs, and status files may be stale or wrong. Every stage begins by checking the actual files, schema, and results.
4. **Decisions are made once, in writing.** Resolve ambiguities during planning and record the decision, the date, and who made it. Executors must not re-ask settled questions, and must not silently reverse them.
5. **Newer explicit user decisions beat older text.** When documents contradict each other, follow the newest explicit decision and record the discrepancy. When a decision is reversed, keep the old wording visible (struck through) with the date, so nobody resurrects it.
6. **No fake completeness.** Mocks, stubs, placeholders, and simulated success must never be presented as a finished feature. If something is not implemented yet, the product says so or the thing is absent.
7. **Failures are fixed in the stage that caused them.** "Later hardening" is not a stage. A stage is done only when its own acceptance is met.
8. **Honest reporting.** Skipped checks, failing tests, known defects, and unverified assumptions are listed explicitly. Never label a system "production-ready" while known unresolved defects exist.

---

## 2. Process

### 2.1 Discover before designing
- Read the existing code, schema, data, tests, deployed configuration, and any previous documentation. Prefer the real thing over descriptions of it.
- Write down the **current state** as facts with their source (file, line, command output), and separately list **limits of the evidence** (what you could not verify and why).
- Identify what exists and must be preserved, what is wrong or fragile, and what is entirely new.

### 2.2 Extract requirements
- Turn the user's words into numbered, testable requirements (`R01`, `R02`, …). One requirement = one verifiable statement. Keep the user's intent; do not invent scope.
- Mark each as **confirmed** (the user said it) or **proposal** (your design choice, changeable on review). Never present a proposal as a confirmed requirement.
- Record explicit later clarifications from the user with their dates, as their own section, instead of silently editing earlier requirements.
- For every requirement write the **acceptance**: how a stranger could check it.

### 2.3 Ask only what is blocking
- Resolve everything you can from the code, the request, and sensible defaults. Ask the user only about decisions that are genuinely theirs and that change the plan (product behavior, risk tolerance, irreversible choices).
- Ask precise, answerable questions with a recommended option. Do not ask what you could verify yourself. Do not re-ask confirmed preferences.

### 2.4 Design the architecture
- State the decision, the alternatives considered, and **why** the decision wins. A plan without reasons cannot be reviewed.
- Define module boundaries, ownership of data, dependency direction, and integration contracts. Name what is deliberately **not** done (e.g. "no microservice per feature", "no second implementation of the calculation in the UI").
- Define where the single source of truth for each calculation, state machine, or piece of data lives. Duplicate logic is a defect by design.
- Describe security and authorization model, failure and recovery behavior, deployment and rollback.

### 2.5 Model data, states, and invariants
- List entities and their relationships (a logical model, not necessarily SQL).
- For anything with a lifecycle, write the **state machine**: states, allowed transitions, who or what triggers them, what is persisted at each step.
- Write the **invariants** as explicit statements that must always hold (sums balance, no double application, no orphan references, immutable after commit, etc.). Each invariant must map to at least one test.
- Specify the algorithms that matter precisely enough that two implementers would produce the same numbers, including rounding, ordering, tie-breaking, and edge cases.

### 2.6 Plan the unhappy paths
For every external interaction and every multi-step write, answer:
- What if it times out? Returns an ambiguous result? Is called twice? Is called concurrently from two instances?
- What if the process crashes **between** any two steps? What is persisted before the side effect so it can be recovered?
- What if the external state changed in the meantime? What if time itself is the problem (clock, time zones, DST, month ends, leap days)?
- What is the idempotency key, and where is it stored? Is an uncertain result ever retried automatically with a new key? (Usually the answer must be no.)

### 2.7 Define verification up front
- Build an **acceptance scenario table** (`A01`, `A02`, …): scenario, expected result, evidence. Scenarios must be concrete (with numbers and dates), include edge cases and failure injection, and trace back to requirements.
- Maintain a **traceability matrix**: every `R` and `A` maps to specific tests or artifacts. At the end of the project no row may remain uncovered.
- Plan the test levels: unit/property tests for calculations, integration tests with deterministic fakes for external systems, fault-injection tests for crash windows, end-to-end simulations, UI checks on desktop and mobile where there is a UI.
- Plan regression fixtures captured from the existing system (inputs and outputs), so the rewrite is proven to match instead of assumed to.

### 2.8 Split into stages
See section 4.

### 2.9 Review the plan itself
Before presenting it, re-read as the executor: Can I start stage N knowing only the documents? Is any term undefined? Is any requirement without acceptance or without a stage? Is any stage without an observable end state? Is any decision missing its reason? Fix before delivering.

---

## 3. Document set

Produce a small set of focused documents rather than one giant file. Typical layout:

```
docs/analysis/
  README.md                  # summary, status, conclusion, index, date and basis of the document
  01-current-state.md        # facts about what exists + limits of evidence
  02-requirements.md         # R01..Rnn with acceptance; confirmed vs proposal; dated clarifications
  03-architecture.md         # decisions, alternatives, structure, authorization, integrations, operations
  04-data-and-automation.md  # entities, algorithms, state machines, invariants, uncertainty handling
  05-migration-and-verification.md  # migration/cutover, rollback, acceptance scenarios A01..Ann
  06-implementation-goals.md # shared rules + the ordered stage briefs
docs/implementation/
  status.md                  # live status per stage + R/A coverage table
  NN-handoff.md              # one per finished stage
  reference/                 # frozen reference of the old system, parity matrix, fixtures
```

Rules for the documents:
- State the **status and date** at the top. Say clearly that analysis is a description of the target, **not** proof that it is implemented.
- Keep a conclusion/summary up front, details below.
- Tables for requirements and scenarios; prose for reasoning.
- No secrets, tokens, credentials, or personal data in any document.
- Link between documents instead of copying text (except the shared rules that must be seen in every stage brief).

---

## 4. Stages ("goals")

### 4.1 How to cut stages
- Order stages by dependency: foundation → identity/security → read-only and data modules → domain writes → engines/calculations → integrations → automation → UI completion → settings/notifications → migration/import → integration hardening → rehearsal on real-data copy → production cutover.
- One stage = one coherent deliverable that can be **verified on its own**. "Finish the stage" means implemented, tested, documented, and handed off, not "files created".
- Do not start stage N+1 on an unverified assumption that stage N is done. Say so in the plan.
- Where a later module is not ready, define an **explicit contract** and allow test doubles **only in tests**. Production screens must never show mock data or fake success. State in which stage the real connection is made, at the latest.
- Order risky/irreversible things last and rehearse them first: import on synthetic data → rehearsal on an isolated copy of real data → cutover with a rollback path.
- Default execution is sequential. Parallel agents only if the user explicitly allows it, with shared contracts and separated file ownership. Never parallelize financial integrations or migrations.

### 4.2 Shared rules block
Put one **shared rules** section at the top of the stages document, applying to every stage. It must contain at least:
- Which documents to read first, and the instruction to verify actual state rather than trust "done" text.
- The conflict rule (newest explicit user decision wins; record the discrepancy).
- Technology and architecture constraints (languages, frameworks, no duplicate calculators, no direct access around the integration contract).
- Safety rules for dangerous operations (what may never be touched, what may never be called live, default of no real side effects in dev/test/staging, secrets never printed or copied into tests).
- Rules for mutations: authorization, transaction boundary, audit, concurrency protection, durable idempotency, no external call inside a DB transaction, never auto-retry an uncertain result with a new key.
- UI verification requirements (states: normal, empty, loading, error, dialog, permissions; keyboard/focus/Escape/Tab; locale-specific input; mobile and desktop).
- The handoff and status requirement (see 4.4).
- The definition of done: a stage is finished only when its own acceptance is met; defects are not deferred.
- Policy for new product contradictions: report them concretely instead of guessing; do not re-ask what is already confirmed.

### 4.3 Anatomy of one stage brief
Each stage brief is a dense paragraph or short list that states, in this order:
1. **Goal** in one sentence, and which earlier stages/contracts it builds on.
2. **Scope**: the concrete things to build, named precisely (entities, flows, screens, commands, edge cases). List what must be preserved from the old system.
3. **Constraints specific to this stage**: what must *not* be done (e.g. "no live payments", "do not turn a transfer into a purchase automatically", "preview must truly write nothing").
4. **Contracts** it exposes to later stages, and contracts it must honor from earlier ones.
5. **Verification required**: the specific tests and scenarios (reference `A..` and `R..` IDs), including concurrency, failure injection, and UI states.
6. **Completion statement**: "Done means …" — the observable end state, plus the handoff number.
7. **Honesty clause** where relevant: what must not be claimed (e.g. do not present placeholders as features; state pending operational checks explicitly rather than calling the goal finished).

Write briefs so that a stage can be launched with a single instruction such as "Execute stage 7 per `06-implementation-goals.md` and its shared rules".

### 4.4 Handoff and status (mandatory per stage)
At the end of every stage the executor must:
- Update `docs/implementation/status.md`.
- Create `docs/implementation/NN-handoff.md` containing: what changed; which checks were **actually run** and their **actual results** (not intended ones); paths to evidence; migrations; public contracts introduced; remaining integration points; open issues and known limitations. No secrets, no personal data.
- Update the R/A traceability table with links to concrete tests. Failed or missing coverage is reported, not hidden.

### 4.5 Reference stage (stage 1 for rewrites)
When replacing or rewriting an existing system, the first stage freezes a **verified reference**: inventory of every page/endpoint/service/query/test, a **parity matrix** (every screen and action), a manifest of exact sources (including uncommitted changes and deployed versions, with hashes), synthetic deterministic data, screenshots, and **regression fixtures** of calculation inputs/outputs. Later stages compare against this reference, not against memory. Production access in this stage is strictly read-only.

---

## 5. Quality bar checklist

Before delivering a plan, confirm every item:

**Requirements**
- [ ] Every requirement is numbered, testable, and marked confirmed vs proposal.
- [ ] Later user clarifications are recorded with dates; reversed decisions keep the old text struck through.
- [ ] Each requirement maps to at least one stage and at least one acceptance scenario.

**Architecture**
- [ ] Each decision lists alternatives and the reason for the choice.
- [ ] Single source of truth is named for every calculation and state machine.
- [ ] Module boundaries, dependency direction, and integration contracts are defined.
- [ ] Authorization is enforced server-side and defined for every actor, including default/anonymous ones.
- [ ] Deployment, monitoring, backup/restore, and rollback are described.

**Correctness**
- [ ] Algorithms are specified exactly (rounding, ordering, remainders, edge cases).
- [ ] State machines and invariants are written out, each invariant has a test.
- [ ] Every external call has timeout, ambiguity, duplicate, concurrency, and crash-window behavior defined.
- [ ] Time handling (time zone, DST, month ends, restarts, injected clock) is specified.
- [ ] Money/precision types are specified; no floating point for money.

**Verification**
- [ ] Acceptance scenarios contain concrete numbers/dates and failure injection.
- [ ] Traceability matrix exists and must be fully covered at the end.
- [ ] Regression fixtures from the existing system are planned.
- [ ] UI states and accessibility checks are listed for desktop and mobile.

**Safety**
- [ ] Dev/test/staging cannot cause real side effects (payments, messages, orders); this is enforced technically, not by convention.
- [ ] Production and legacy systems are protected from damage during development.
- [ ] Irreversible steps are rehearsed first and have a rollback path with its limits stated (e.g. rollback is different after the first external side effect).
- [ ] Secrets are never printed, committed, or put in tests.

**Execution**
- [ ] Stages are ordered by dependency, each independently verifiable, each with a "done means" statement.
- [ ] Shared rules block exists and is referenced by every stage.
- [ ] Handoff/status duty is defined.
- [ ] The plan states honestly what is not yet verified and which operational checks remain open.

---

## 6. Anti-patterns (do not do these)

- Vague requirements ("should be fast", "support payments") with no acceptance.
- A stage that ends with "files created" or "compiles".
- Mock data or stubbed success shown in production UI.
- Two implementations of the same calculation (e.g. backend and UI).
- Retrying an ambiguous external result with a new idempotency key.
- Matching bank/exchange/external records by guessing among identical amounts.
- Treating an "accepted by the external system" event as "completed"; distinguish accepted, pending user action, booked, rejected, and unknown.
- Starting the next stage on top of an unverified previous one.
- Copying "done" claims from old documents without checking.
- Hiding unresolved defects behind labels like "production-ready" or "later hardening".
- Re-asking the user about decisions they already made, or silently overriding them.
- Presenting an estimate or a timeline as a commitment when no spike or profiling supports it.
- Putting secrets, tokens, or personal data into plans, fixtures, tests, or handoffs.

---

## 7. Delivering the plan

- Present a short summary: the conclusion, the main decisions, the stage list, and what remains open or unverified.
- Point to the documents rather than pasting them.
- Record open technical questions as a list of **things to verify during implementation** (not product questions for the user), stating that they are not resolved yet.
- State clearly that the plan is a plan and not an implemented system.
