# DR-0004: Durable capabilities should be evidence-backed

- Status: proposed
- Date: 2026-09-02
- Governing issue: #9
- Supersedes: none
- Superseded by: none

## Context

Harness Intelligence is expected to evolve from a small runtime that selects what the main agent can see into a system that can improve how memory, skills, tools, MCP servers, and future capabilities are managed over time.

Earlier telemetry decisions establish two important principles:

1. canonical telemetry preserves low-level append-only evidence rather than only aggregate scores; and
2. capability demand, exposure, request, execution, and outcome remain distinct so future ranking can account for exposure bias.

Those principles are necessary but not sufficient for durable capability management.

Without a project-level lifecycle rule, later implementations could introduce opaque mutable state such as:

```text
tier = 4
trust = 0.91
promoted = true
```

without preserving why those values exist, which runtime observations support them, or which capability revision they describe.

The same problem appears in several forms:

- a memory may be persisted after one weak observation
- a skill may be generated without retaining the runs that motivated it
- a capability may gain reputation simply because it was exposed more often
- a tool integration may remain active even after its environment changes
- a revision may silently inherit evidence collected for a materially different revision
- a later policy change may be unable to explain or recompute previous rankings

A durable rule is needed before automated capability evolution is implemented.

## Decision

Harness Intelligence will treat durable capability state as **evidence-backed and revision-aware**, while keeping capability profiles and exposure/recommendation decisions separate from the canonical evidence that supports them.

### 1. Durable capability changes require provenance

When Harness Intelligence creates, accepts, promotes, revises, supersedes, deprecates, or archives a durable capability or capability revision, the decision should be traceable to explicit provenance.

Provenance may reference:

- telemetry event ids
- run/task ids
- evaluation annotations
- test/validation results
- user-confirmed decisions
- prior capability revisions
- explicit policy decisions

The exact storage type is not fixed by this Decision Record.

The durable requirement is that a future contributor or runtime can answer why a capability/revision exists and what evidence supported a lifecycle decision.

### 2. Observation does not automatically become durable state

A single model statement, inferred need, successful execution, or user interaction does not automatically become trusted durable capability state.

Potential durable changes should conceptually pass through a candidate/proposal and validation or promotion gate.

The gate may later use human approval, deterministic policy, repeated evidence, tests, evaluator consensus, contradiction checks, or other mechanisms.

Early implementations should prefer explicit proposals over silent autonomous mutation.

### 3. Capability quality/profile is separate from exposure policy

Long-term beliefs about a capability and a per-task decision to expose it are different concerns.

A capability may be trusted yet irrelevant, expensive, risky, stale for the current environment, or incompatible with the current main model.

Therefore derived dimensions such as:

```text
trust
confidence
utility
demand
freshness
cost
risk
model compatibility
```

must not semantically imply:

```text
always expose
always recommend
always activate
```

Exposure/recommendation is a runtime policy decision conditioned on the current task, model, context budget, runtime state, and capability profile.

### 4. CapabilityProfile is derived state

A future `CapabilityProfile`, tier, reputation score, trust score, or recommendation priority may be stored as a cache or operational state, but it is not the canonical explanation for capability quality.

Where practical, those values should be reproducible from retained evidence and durable revision state.

Algorithms may change without rewriting historical runtime evidence.

### 5. Capability revisions are first-class

Evidence and lifecycle decisions should be attributable to the capability revision that produced or received them.

Material changes to memory content, skill instructions, tool contracts, MCP integrations, or other capability semantics should not silently rewrite historical identity.

Prefer explicit revision/supersession relationships so evidence collected for one revision is not incorrectly treated as evidence for another.

### 6. Durable lifecycle and runtime exposure remain separate

Durable lifecycle state may later include concepts such as:

```text
candidate
accepted
active
superseded
deprecated
archived
```

The exact state machine is deferred.

A durable state such as `deprecated` is not equivalent to a per-turn exposure decision such as `hidden`.

Likewise:

```text
trusted != always exposed
fresh != trusted
popular != useful
requested != correct
successful capability execution != successful task
```

These distinctions are part of the design contract.

### 7. The principle applies across capability kinds

The evidence-backed lifecycle is not memory-specific.

It applies to:

- memory
- skills
- tools and their harness integrations
- MCP servers/integrations
- future capability kinds
- proposals for capabilities that do not yet exist

Capability-specific storage and activation mechanics may differ while preserving the same evidence/provenance principles.

## Alternatives considered

### Store only mutable tier/reputation fields

This is operationally simple but loses the explanation behind those values, makes policy changes difficult to recompute, and encourages exposure-driven feedback loops.

Mutable derived profiles may still exist for efficiency, but they are not sufficient as the canonical record.

### Automatically persist every useful observation

This minimizes gating complexity but promotes noisy, stale, contradictory, or accidental observations into durable state too easily.

It is rejected as the default lifecycle principle.

### Treat exposure as a consequence of trust

For example, every `trusted` memory or high-tier skill could always be exposed.

This conflates long-term capability quality with current task relevance, model compatibility, context cost, risk, and latency.

It is rejected.

### Make the lifecycle memory-specific

Memory is likely to be an early use case, but the same provenance/promotion problem appears for skills, tools, MCP integrations, and future capability kinds.

A memory-only durable rule would duplicate concepts later and is rejected.

### Require raw content for provenance

Provenance can often be represented through structured telemetry/evaluation references and metadata.

Requiring raw prompts, model outputs, or tool outputs would conflict with the metadata-only telemetry default and create unnecessary privacy/storage pressure.

It is rejected.

## Consequences

### Benefits

- Durable memories and capability changes remain explainable.
- Skill creation/improvement proposals can reference the runs that motivated them.
- Capability reputation/profile algorithms can evolve without rewriting history.
- Exposure policy can account for task/model/context/cost separately from long-term trust.
- Historical revisions remain analyzable.
- Incorrect promotions or deprecations can be audited against the evidence available at the time.
- The same lifecycle model can support memory, skills, tools, MCP servers, and future capability kinds.

### Costs

- Lifecycle operations require provenance plumbing.
- Candidate/proposal/gate states add implementation complexity compared with direct mutation.
- Derived profile computation requires an analysis/state layer in addition to raw telemetry.
- Revision identity must be maintained deliberately.

### Risks

- Overly strict gates could prevent useful capability evolution.
- Weak evaluators could attach misleading evidence to lifecycle decisions.
- Provenance graphs could become large or expensive to query.
- A future ranking policy can still create feedback loops if it ignores opportunity/exposure bias.
- Different capability kinds may eventually require lifecycle states that do not map cleanly to one universal state machine.

These risks should be addressed in later implementation-specific decisions rather than by weakening the evidence/provenance requirement.

## Validation

This decision is validated if future implementations can demonstrate that:

- a durable capability/revision can report the evidence supporting its creation or promotion
- a skill improvement proposal can identify the runs/evaluations that motivated it
- a memory correction/supersession does not erase the evidence behind the prior revision
- capability profile values can be recomputed or reinterpreted from retained evidence
- high trust does not force unconditional exposure
- exposure policy can differ by task, model, context budget, cost, or risk
- materially different revisions retain separate identity and attribution
- the lifecycle can support memory, skills, tools, and MCP integrations without making one capability kind the universal storage model

## Follow-up

- Document the broader lifecycle and four-layer model in `docs/capability-lifecycle.md` under #9.
- Keep the first v0 runtime focused on deterministic execution and telemetry collection rather than implementing the full lifecycle.
- Add capability identity/revision plumbing before relying on cross-run reputation or profile analysis.
- Prototype derived capability analysis offline before defining permanent tier/reputation formulas.
- Define exact lifecycle events, state machine, storage representation, and promotion policies in separate implementation/design issues when concrete requirements exist.
- Define automatic skill creation, memory promotion, and adaptive exposure policy only after evidence quality can be evaluated against real runtime data.
