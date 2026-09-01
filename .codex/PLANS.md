# PLANS.md

## Purpose

This document defines how Codex should plan and execute non-trivial changes in `vllm-ascend`.

For complex features, the planning result must be written to:

```text
.codex-plans/<feature-name>.md
```

A plan that only exists in chat is not considered complete.

The goal of planning is to resolve architectural uncertainty before implementation and produce a concrete engineering plan that another capable agent can execute without rediscovering the design.

The most important planning question in this repository is:

```text
Does this behavior belong to upstream vLLM,
or is it genuinely Ascend-specific?
```

This question MUST be answered before deciding where to implement a feature.

---

## Planning Policy

For any non-trivial feature, significant refactor, scheduler change,
KV-cache lifecycle change, distributed-execution change, worker/model-runner
change, or performance-sensitive change:

1. Enter planning before implementation.
2. Read `.codex/PLANS.md` completely.
3. Thoroughly inspect the existing implementation.
4. Produce exactly one concrete plan document under:

   `.codex-plans/<feature-name>.md`

5. A plan that exists only in chat is not sufficient.
6. During the planning phase, do not modify production code.
7. Planning ends only after the plan passes the readiness criteria defined
   in `.codex/PLANS.md`.

---

## Planning / Implementation Separation

Planning and implementation are separate phases.

During planning:

- investigate the repository;
- inspect relevant tests;
- trace important data/control flows;
- identify invariants;
- evaluate alternatives;
- design validation;
- write or update the plan document;
- do not implement the feature.

After the plan is written, stop and allow the user to review it.

Do not begin implementation until the plan status is explicitly changed
to `APPROVED`.

---

## Implementation Policy

When implementing an approved plan:

1. Read the complete plan before modifying code.
2. Treat the approved architecture and invariants as the implementation
   contract.
3. Work milestone by milestone.
4. Run the validation specified for each milestone.
5. Do not continue past a failed validation.
6. Keep implementation scoped to the approved plan.
7. Update Progress, Discoveries, and Decision Log as work proceeds.

Implementation-level details may be adjusted locally.

Architectural decisions must not be silently changed.

If implementation reveals that an important assumption about architecture,
state ownership, distributed behavior, performance, or API semantics is
incorrect, stop that milestone and return the plan to planning state.

---

## Review Policy

Final review must compare:

- the approved plan;
- the actual git diff;
- validation results.

Pay particular attention to correctness, performance, state ownership,
distributed behavior, concurrency, backward compatibility, and missing
interaction tests.

---

# Architecture Principle

Treat `vllm-ascend` as the Ascend hardware backend for vLLM.

Prefer to reuse upstream vLLM behavior for hardware-independent functionality such as:

- frontend and serving behavior;
- request lifecycle;
- scheduler semantics;
- general engine/core behavior;
- generic configuration semantics;
- hardware-independent protocols and abstractions.

Prefer `vllm-ascend` for Ascend-specific behavior such as:

- NPU worker execution;
- NPU model runner behavior;
- Ascend attention implementations;
- NPU memory handling;
- Ascend graph execution;
- torch_npu integration;
- CANN-specific behavior;
- HCCL communication;
- Ascend custom operators;
- NPU-specific performance optimization;
- hardware-specific patches and compatibility handling.

Do not duplicate upstream vLLM logic in `vllm-ascend` merely because doing so is easier locally.

If a feature requires hardware-independent behavior that upstream vLLM cannot currently express, planning must explicitly evaluate whether an upstream vLLM change should be made first.

---

# Environment Constraints

Do not assume that the current development machine has usable Ascend NPU resources.

When NPU hardware is unavailable, distinguish between:

1. **Local Validation**
   - static checks;
   - pure Python tests;
   - mock/isolated tests;
   - structural validation;
   - tests that do not require an NPU.

2. **Deferred NPU Validation**
   - real inference;
   - NPU E2E;
   - distributed NPU tests;
   - graph-mode validation;
   - memory validation;
   - performance benchmarks.

Lack of NPU resources must not block implementation that can be validated locally.

It must also not be used to claim NPU correctness or performance without hardware evidence.

---

# When a Plan Is Required

Create an ExecPlan for changes that are:

- cross-cutting;
- architecture-sensitive;
- performance-sensitive;
- distributed or concurrency-sensitive;
- significant refactors;
- dependent on upstream vLLM behavior;
- difficult to validate locally;
- changes touching execution hot paths.

A plan is especially required for changes involving:

- worker behavior;
- model runner;
- attention backend;
- KV-cache handling;
- distributed communication;
- TP / PP / DP / EP behavior;
- speculative decoding;
- graph execution;
- NPU memory management;
- custom operators;
- sampling;
- platform integration;
- patches to upstream vLLM;
- changes requiring synchronized vLLM and vllm-ascend modifications.

Small and isolated fixes may skip an ExecPlan.

When uncertain, create one.

---

# Planning Rules

Planning and implementation are separate phases.

During planning:

1. Inspect the current `vllm-ascend` implementation.
2. Inspect the corresponding upstream vLLM implementation.
3. Inspect relevant tests in both repositories when applicable.
4. Determine which repository owns the behavior.
5. Trace relevant control flow and data flow.
6. Identify interface contracts between vLLM and vllm-ascend.
7. Identify state/resource ownership and important invariants.
8. Evaluate alternatives for non-trivial design decisions.
9. Identify concrete files and symbols expected to change.
10. Split implementation into independently verifiable milestones.
11. Define local validation for every milestone.
12. Define deferred NPU validation where required.
13. Write the plan to `.codex-plans/<feature-name>.md`.
14. Stop after the plan reaches `READY_FOR_REVIEW`.

During planning, do not modify production code.

Do not rely on remembered knowledge of vLLM or vllm-ascend when current repository evidence is available.

---

# Upstream Baseline

Every plan that depends on upstream vLLM behavior MUST record the upstream baseline used during analysis.

Include:

```text
vLLM branch/tag/commit:
vllm-ascend branch/commit:

Relevant upstream files/symbols:
...
```

The plan must verify the actual upstream implementation rather than assuming an older vLLM architecture.

This is especially important for interfaces that frequently evolve, such as:

- scheduler outputs;
- worker interfaces;
- model runner APIs;
- sampling;
- KV-cache metadata;
- distributed execution;
- compilation/graph interfaces.

---

# Change Ownership Decision

Before proposing implementation details, every plan MUST classify the change.

Use one of the following:

```text
Change Ownership:

- UPSTREAM
- ASCEND
- COORDINATED
- ASCEND_PATCH
```

## UPSTREAM

Use when the behavior is hardware-independent and belongs naturally in vLLM.

Examples may include:

- scheduler semantics;
- generic request state;
- generic engine behavior;
- hardware-independent interfaces.

The plan should describe the required upstream vLLM change.

Do not implement a permanent Ascend-only duplicate of generic behavior.

---

## ASCEND

Use when the behavior is genuinely NPU-specific.

Examples may include:

- NPU execution;
- Ascend graph handling;
- torch_npu behavior;
- CANN-specific implementation;
- HCCL communication;
- NPU kernels;
- NPU-specific memory or performance handling.

The plan should preserve upstream interfaces whenever possible.

---

## COORDINATED

Use when both repositories need changes.

The plan MUST describe:

```text
Upstream change
    ↓
new/changed contract
    ↓
vllm-ascend adaptation
```

Specify:

- which change must land first;
- compatibility requirements;
- temporary compatibility handling if required;
- how old/new combinations behave.

---

## ASCEND_PATCH

Use only when patching upstream behavior is necessary.

The plan MUST explain:

- why a normal plugin/interface solution is insufficient;
- why the behavior cannot reasonably live upstream first;
- which upstream symbol is patched;
- what assumptions the patch depends on;
- how upstream changes could break it;
- whether a future upstream interface could remove the patch.

Patches should be minimal and focused.

Do not choose patching simply because it produces a smaller immediate diff.

---

# Repository Evidence

Important conclusions must be supported by current code evidence.

For important claims, reference:

```text
repository
+
file
+
class/function/symbol
+
observed behavior
```

Example:

```text
Upstream vLLM:

`vllm/v1/.../foo.py`

- `Foo.bar()`
  - creates ...
  - passes ... to the worker.

vllm-ascend:

`vllm_ascend/worker/model_runner_v1.py`

- `NPUModelRunner.foo()`
  - consumes ...
  - converts ... for NPU execution.
```

Important claims requiring evidence include:

- ownership;
- lifecycle;
- scheduler behavior;
- worker contracts;
- model runner contracts;
- KV-cache metadata;
- distributed behavior;
- graph execution;
- NPU synchronization;
- performance-sensitive paths.

Do not infer architecture from file names alone.

If something cannot be verified statically, mark it as:

```text
Requires NPU runtime verification
```

---

# Required Plan Structure

Every ExecPlan should contain:

```text
# <Feature Name>

Status: DRAFT

## Goal
## Non-Goals
## Upstream Baseline
## Change Ownership
## Current Architecture & Evidence
## vLLM ↔ Ascend Boundary
## Invariants
## Proposed Design
## Alternatives Considered
## File / Symbol Changes
## Milestones
## Local Validation
## Deferred NPU Validation
## Risks & System Impact
## Open Questions
## Progress
## Discoveries
## Decision Log
```

Keep sections concise.

Do not omit important reasoning merely to shorten the document.

---

# Goal

Describe the desired end behavior rather than a specific implementation.

Bad:

```text
Add a new field to NPUModelRunner.
```

Better:

```text
Allow the Ascend model runner to consume the new upstream scheduler metadata
while preserving existing NPU execution behavior.
```

The Goal should make clear what must become true after the feature is implemented.

---

# Non-Goals

Explicitly define what is outside the scope.

Typical examples:

- redesigning the upstream scheduler;
- refactoring unrelated Ascend components;
- adding support for unrelated models;
- optimizing unrelated NPU paths;
- changing public APIs unnecessarily.

Avoid scope expansion during implementation.

---

# Current Architecture & Evidence

Describe only the architecture relevant to the feature.

A typical execution path may resemble:

```text
vLLM frontend
      ↓
vLLM Engine / EngineCore
      ↓
vLLM Scheduler
      ↓
scheduler output / worker contract
      ↓
vllm-ascend Worker
      ↓
vllm-ascend ModelRunner
      ↓
Ascend attention / ops / graph / communication
      ↓
NPU
```

Do not assume this exact path applies to every feature.

Verify the actual path from current code.

Document:

- who creates important state;
- who owns it;
- who transforms it;
- which data crosses the plugin boundary;
- which component is hardware-neutral;
- which component is Ascend-specific.

---

# vLLM ↔ Ascend Boundary

This section is mandatory for changes involving upstream interfaces.

Explicitly describe the contract between the two repositories.

For example:

```text
Upstream produces:
- SchedulerOutput.xxx
- metadata Y

Ascend consumes:
- Worker.execute_model(...)
- NPUModelRunner.xxx(...)

Ascend-specific transformation:
- ...

Returned to upstream:
- ...
```

Answer:

1. What behavior is owned by upstream?
2. What behavior is owned by vllm-ascend?
3. What data crosses the boundary?
4. Does the proposed change alter that contract?
5. Can the feature be implemented without duplicating upstream logic?
6. Does the design remain compatible with expected upstream evolution?

The preferred design keeps hardware-neutral decisions upstream and hardware-specific execution in the Ascend plugin.

---

# Invariants

List the properties that must remain true.

Relevant invariants may include:

## Upstream Compatibility

- Ascend consumes upstream interfaces according to their intended contract.
- Generic behavior is not reimplemented unnecessarily.
- Existing upstream behavior remains unchanged when the Ascend backend is not involved.

## State Ownership

- State has one clear authoritative owner.
- Ascend does not create a second source of truth for scheduler/core state.

## Worker / Model Runner

- Worker and model runner responsibilities remain clearly separated.
- Scheduler metadata is interpreted consistently.
- Request state remains synchronized with execution state.

## KV Cache / Memory

- allocation and release ownership remain correct;
- no stale NPU references remain;
- memory lifetime matches request lifetime.

## Distributed Execution

- ranks observe consistent metadata;
- collective ordering remains valid;
- no unintended synchronization is introduced.

## Performance

- no unnecessary CPU↔NPU transfer;
- no unnecessary device synchronization;
- no new expensive work in hot paths without justification.

Only include invariants relevant to the feature.

---

# Proposed Design

Describe the selected architecture clearly enough that implementation does not need to redesign the feature.

Specify:

- which repository owns each change;
- which component owns new state;
- how upstream information reaches the Ascend backend;
- where NPU-specific transformation occurs;
- whether inheritance, composition, plugin hooks, custom ops, or patching are used;
- how existing interfaces and invariants are preserved.

Prefer designs that reuse upstream abstractions.

Avoid copying large upstream implementations into vllm-ascend unless NPU-specific divergence genuinely requires it.

---

# Alternatives Considered

For meaningful design decisions, briefly evaluate reasonable alternatives.

Typical alternatives may include:

```text
upstream interface extension
vs
Ascend patch

inherit upstream implementation
vs
NPU-specific implementation

extend existing metadata
vs
introduce new backend state
```

Record:

```text
Option A:
...

Option B:
...

Chosen:
...

Reason:
...
```

Do not invent alternatives for trivial implementation details.

---

# File / Symbol Changes

Identify expected changes at:

```text
repository
→ file
→ symbol
→ intended change
→ reason
```

Example:

```text
vLLM:

`vllm/.../scheduler_output.py`

- `SchedulerOutput`
  - expose ...

vllm-ascend:

`vllm_ascend/worker/model_runner_v1.py`

- `NPUModelRunner.execute_model()`
  - consume ...
  - preserve ...
```

If only vllm-ascend changes are required, state explicitly why no upstream modification is needed.

---

# Milestones

Break implementation into coherent, independently verifiable milestones.

Every milestone must contain:

```text
### Milestone N — <Name>

Objective:
...

Repository:
vLLM / vllm-ascend / both

Files / Symbols:
...

Changes:
...

Acceptance Criteria:
- ...
- ...

Local Validation:
...

Deferred NPU Validation:
...

Escalation Conditions:
...
```

A good milestone allows:

```text
implement
→ local validate
→ fix
→ complete
→ proceed
```

without reopening the entire architecture.

Avoid:

```text
1. Modify vLLM
2. Modify Ascend
3. Test
```

Milestones should express behavior, not merely repositories.

---

# Local Validation

Every milestone must define the strongest locally executable validation.

Prefer repository-standard commands.

Possible validation includes:

## Static Checks

```bash
ruff check ...
mypy ...
python -m compileall ...
```

## Unit Tests

Use tests that do not require NPU runtime where possible.

Prefer existing tests under the repository's normal test structure.

## Mock / Isolated Tests

Useful for:

- metadata transformation;
- worker/model runner control flow;
- configuration;
- state transitions;
- protocol adaptation;
- error handling.

Mocks should test real logic rather than reproduce the implementation.

## Structural Validation

When NPU execution is unavailable, verify:

- upstream and plugin signatures;
- call sites;
- scheduler output consumption;
- metadata consistency;
- inheritance/override assumptions;
- patch targets;
- configuration wiring;
- distributed message structure.

## Upstream Compatibility

When an upstream interface changes, verify as much as possible that:

```text
upstream producer
      ↓
defined contract
      ↓
Ascend consumer
```

remain consistent.

Do not claim NPU correctness based solely on local validation.

---

# Deferred NPU Validation

NPU-dependent behavior must be documented separately when it cannot be tested locally.

For each required validation define:

```text
Purpose:
...

Environment:
- Ascend hardware type
- NPU count
- relevant CANN / torch_npu / vLLM configuration

Command / Test:
...

Configuration:
...

Acceptance Criteria:
...
```

Possible validation dimensions include:

- basic NPU inference;
- accuracy;
- worker/model runner execution;
- eager mode;
- graph mode;
- KV-cache behavior;
- TP;
- PP;
- DP;
- EP;
- speculative decoding;
- distributed communication;
- multi-node behavior;
- NPU memory usage;
- throughput;
- TTFT / ITL;
- long-running stability.

Select only dimensions that can realistically interact with the feature.

Do not mechanically require every mode.

---

# NPU Performance Analysis

For changes touching NPU execution hot paths, inspect the design for:

- CPU↔NPU transfers;
- device synchronization;
- `.item()` or equivalent host synchronization;
- per-token Python work;
- per-request allocations;
- tensor copies;
- NPU memory lifetime;
- graph breaks;
- graph capture/replay compatibility;
- kernel/operator count;
- HCCL communication;
- collective frequency;
- rank synchronization;
- metadata movement.

Do not write:

```text
Performance impact should be small.
```

Instead explain:

```text
what additional work is introduced
+
where it occurs
+
how often it occurs
+
why it should be acceptable
```

Runtime performance remains unverified until measured on actual NPU hardware.

---

# Risks & System Impact

Analyze only risks relevant to the feature.

## Upstream Drift

Could a future or current vLLM change invalidate:

- inheritance assumptions;
- patched symbols;
- metadata layouts;
- worker contracts;
- model runner interfaces?

If yes, describe the coupling explicitly.

## Correctness

Consider:

- stale scheduler metadata;
- duplicated state;
- mismatched upstream/backend lifecycle;
- incorrect KV-cache ownership;
- cleanup errors;
- request state divergence.

## NPU Runtime

Consider:

- wrong device context;
- unsupported operator behavior;
- graph-mode differences;
- memory lifetime;
- NPU synchronization.

## Distributed

Consider:

- rank consistency;
- HCCL collective ordering;
- TP/DP/EP interaction;
- multi-node ordering;
- cancellation and failure propagation.

## Performance

Consider:

- CPU↔NPU synchronization;
- graph breaks;
- additional collectives;
- metadata copies;
- memory growth;
- hot-path overhead.

---

# Open Questions

Questions that can materially change the architecture must be resolved before the plan becomes ready.

Examples:

```text
Does this behavior belong upstream or in vllm-ascend?

Who owns this state?

Does upstream already expose the required hook?

Would this require patching an unstable private symbol?

Does NPU execution require different semantics rather than only a different implementation?
```

Questions that require only NPU runtime measurement may remain deferred.

Example:

```text
Deferred question:
Does this change cause a measurable decode latency regression?

Resolution:
Benchmark during Deferred NPU Validation.
```

Such questions do not block implementation if the architecture does not depend on their answer.

---

# Plan Readiness Gate

A plan may be marked `READY_FOR_REVIEW` only when all applicable items are true:

- [ ] Goal and non-goals are clear.
- [ ] Current vllm-ascend implementation has been inspected.
- [ ] Relevant upstream vLLM implementation has been inspected.
- [ ] Upstream baseline is recorded when applicable.
- [ ] Change ownership is explicitly classified.
- [ ] Important architectural conclusions have repository evidence.
- [ ] vLLM ↔ Ascend boundary is understood.
- [ ] State/resource ownership is understood.
- [ ] Important invariants are documented.
- [ ] Proposed design is concrete.
- [ ] Important alternatives were considered.
- [ ] Patching is justified if used.
- [ ] Expected files and symbols are identified.
- [ ] Work is split into executable milestones.
- [ ] Every milestone has acceptance criteria.
- [ ] Every milestone has local validation.
- [ ] Required NPU validation is explicitly deferred.
- [ ] Upstream compatibility risk was analyzed.
- [ ] NPU performance impact was analyzed where relevant.
- [ ] Distributed/concurrency impact was analyzed where relevant.
- [ ] No unresolved question can materially change the architecture.
- [ ] Another capable agent could implement the feature using only the repositories and this plan.

If required items are missing:

```text
Status: DRAFT
```

When complete:

```text
Status: READY_FOR_REVIEW
```

Stop before implementation.

---

# Approval

Implementation may begin only after the plan is reviewed and marked:

```text
Status: APPROVED
```

Normal flow:

```text
Sol planning
    ↓
inspect vLLM + vllm-ascend
    ↓
ownership decision
    ↓
READY_FOR_REVIEW
    ↓
human review
    ↓
APPROVED
    ↓
implementation
```

---

# Implementation Rules

When implementing an approved plan:

1. Read the complete plan.
2. Verify the upstream baseline still matches.
3. Verify important symbols/interfaces still exist.
4. Work milestone by milestone.
5. Run all locally executable validation.
6. Fix local failures before proceeding.
7. Keep changes scoped to the approved design.
8. Update Progress, Discoveries, and Decision Log.
9. Do not attempt unavailable NPU tests repeatedly.

Implementation agents may make local decisions such as:

- naming;
- helper extraction;
- fixture design;
- small control-flow changes.

They must not silently change:

- upstream/backend ownership;
- state ownership;
- worker/model runner responsibilities;
- patch strategy;
- public behavior;
- distributed protocol;
- important performance assumptions.

---

# Replanning / Escalation

If implementation discovers that a material plan assumption is incorrect, stop the affected milestone.

Typical escalation conditions include:

- the behavior actually belongs upstream;
- upstream interface differs from the plan;
- a required plugin hook does not exist;
- patching would be broader than planned;
- state ownership differs from the plan;
- worker/model runner responsibilities differ;
- NPU semantics require an architectural change;
- a new distributed synchronization mechanism is required.

Then:

1. record repository evidence;
2. explain which assumption is invalid;
3. set:

```text
Status: NEEDS_REPLAN
```

4. return to a planning-capable model.

Do not hide an architectural redesign inside implementation.

Missing NPU hardware alone is not a reason for replanning.

---

# Progress

Maintain lightweight progress:

```text
## Progress

- [x] Milestone 1
- [ ] Milestone 2

Current:
...

Last Local Validation:
...

Next:
...
```

A milestone is locally complete only after its local validation passes.

---

# Discoveries

Record unexpected repository facts:

```text
### Discovery

Observed:
...

Evidence:
...

Impact:
...
```

Especially record discoveries about:

- upstream interface behavior;
- hidden patches;
- backend divergence;
- state ownership;
- NPU-specific behavior.

If a discovery invalidates architecture, trigger replanning.

---

# Decision Log

Record significant decisions:

```text
### Decision — <Title>

Context:
...

Decision:
...

Reason:
...
```

Important decisions should include ownership choices such as:

```text
Why this belongs in vllm-ascend instead of upstream vLLM.
```

or:

```text
Why an upstream interface change is preferred over an Ascend patch.
```

---

# Local Completion

When implementation and all locally executable validation are complete:

```text
Status: LOCAL_VALIDATED
```

If NPU validation remains:

```text
Status: NPU_VALIDATION_PENDING
```

This means:

```text
implementation complete
+
local/static/unit validation complete
+
NPU runtime validation still required
```

Do not interpret it as full production validation.

---

# DONE Status

Use:

```text
Status: DONE
```

only when all required validation has completed, including deferred NPU validation where applicable.

If NPU results are unavailable, leaving:

```text
Status: NPU_VALIDATION_PENDING
```

is correct.

---

# Final Review

Final review should compare:

```text
approved ExecPlan
+
upstream baseline
+
actual diff
+
local validation
+
NPU validation status
```

Review should focus on:

1. Did the change land in the correct repository?
2. Was hardware-neutral behavior duplicated unnecessarily?
3. Is the vLLM ↔ Ascend contract preserved?
4. Are patches minimal and justified?
5. Are worker/model runner responsibilities correct?
6. Are state and resource ownership preserved?
7. Are NPU synchronization and data transfers reasonable?
8. Are distributed semantics preserved?
9. Are performance-sensitive paths safe?
10. Is deferred NPU coverage sufficient?
11. Is the implementation consistent with the approved plan?

Passing local tests alone does not prove NPU correctness.

---

# Core Principle

For vllm-ascend, planning must solve two problems in order:

```text
1. Where should the behavior live?
2. How should that behavior be implemented?
```

The preferred flow is:

```text
Feature
   ↓
inspect upstream vLLM
   +
inspect vllm-ascend
   ↓
determine ownership
   ↓
┌───────────────┬─────────────────┐
│               │                 │
Upstream     Ascend-only      Coordinated
│               │                 │
vLLM logic    NPU backend      vLLM contract
                                  ↓
                             Ascend adaptation
└───────────────┴─────────────────┘
                 ↓
            ExecPlan
                 ↓
          human approval
                 ↓
          implementation
                 ↓
        local validation
                 ↓
       NPU validation later
```

The core rules are:

```text
Keep hardware-neutral behavior upstream.

Keep NPU-specific execution in vllm-ascend.

Reuse upstream abstractions instead of duplicating them.

Treat patches as exceptions that require justification.

Verify both sides of the vLLM ↔ Ascend interface.

Planning resolves architecture before implementation.

Local validation proves only what can be proved without NPU hardware.

NPU-dependent claims remain unverified until tested on real Ascend hardware.
```