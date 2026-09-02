# Architecture

## Purpose

Harness Intelligence is intended to add an intelligence layer to an agent harness without replacing the main agent. The harness observes the current task and runtime environment, decides what the agent should see or be able to use, and exposes an optimized execution context.

## Current v0 direction

The initial pipeline is intentionally small:

```text
Harness Adapter
    -> HarnessSnapshot
    -> HarnessBrain
    -> HarnessPlan
    -> Harness Adapter
    -> Main Agent
```

The v0 core runtime contract is documented in [`core-v0.md`](core-v0.md). The canonical v0 telemetry contract is documented in [`telemetry-v0.md`](telemetry-v0.md).

A forward design for evidence-backed capability evolution is proposed in [`capability-lifecycle.md`](capability-lifecycle.md). It does not expand the first v0 implementation scope; it defines how later memory, skill, tool, MCP, and other durable capability systems should relate evidence, lifecycle state, derived profiles, and runtime exposure policy.

### Harness Adapter

Bridges a concrete agent harness to the Harness Intelligence core. It collects the information needed to build a snapshot and applies the resulting plan back to the harness. Harness-specific types and lifecycle hooks remain outside the core.

### HarnessSnapshot

Represents the immutable observation available to Harness Intelligence at one planning point. v0 focuses on the metadata and skill descriptors required for the pass-through baseline and the first Skill Catalog Intelligence experiment.

### HarnessBrain

Consumes a snapshot and produces a structured plan without mutating the concrete harness. v0 begins with deterministic `PassThroughBrain` behavior before rule-based or LLM-backed planners.

### HarnessPlan

Represents the structured output of the brain. A plan is validated before a concrete adapter applies it so planning, validation, and application failures remain distinguishable.

### Telemetry

Telemetry is a foundational concern rather than a later add-on. The runtime must make it possible to reconstruct the relationship between observation, harness decision, execution, capability demand/use, and outcome so future evaluation and training-data generation remain possible.

The v0 telemetry event model, content policy, schema evolution rules, and sink abstraction are defined in [`telemetry-v0.md`](telemetry-v0.md) and DR-0001.

Capability lifecycle evidence deliberately distinguishes availability, candidacy, exposure, observed demand, recognized request, execution, and outcome. This allows later systems to build capability utility/reputation/tier models without making those derived scores canonical telemetry or confusing frequent exposure with capability quality. The durable extension is proposed in DR-0003 and issue #7.

### Capability lifecycle

Later capability evolution should preserve an explicit separation between:

```text
canonical evidence
    -> durable capability state + revisions
    -> derived CapabilityProfile
    -> exposure / recommendation policy
```

Durable capability creation, promotion, revision, supersession, deprecation, or archival should be traceable to provenance. Quality/trust/profile state must not imply unconditional exposure; exposure remains a task/model/context-budget/runtime policy decision.

The detailed forward design is proposed in [`capability-lifecycle.md`](capability-lifecycle.md), DR-0004, and issue #9.

## Architectural constraints

- The main agent is replaceable and remains outside the Harness Intelligence core.
- The core should not depend on a single harness implementation.
- Skills, tools, MCP servers, and memory providers may eventually be represented through a common capability model, but v0 should not over-generalize before requirements are validated.
- Project-specific knowledge should remain external to model weights.
- Derived metrics and profiles should be reproducible from lower-level telemetry where practical.
- Capability tiers, reputation, utility, and recommendation priority are derived state rather than canonical runtime evidence.
- Durable capability lifecycle changes should remain traceable to explicit provenance/evidence.
- Capability quality/profile and per-task exposure policy are separate concerns.
- Capability revisions should remain attributable rather than silently rewriting materially changed historical state.
- The first executable path must be deterministic, observable, and suitable as a baseline for later intelligent planning.

## v0 design status

Telemetry semantics are defined by [`telemetry-v0.md`](telemetry-v0.md) and DR-0001. Capability-demand semantics are being extended through issue #7 and DR-0003 without introducing a ranking/tier implementation into v0.

The core runtime boundary is defined in [`core-v0.md`](core-v0.md) and DR-0002. Implementation-level choices such as exact async/cancellation primitives and internal module layout remain intentionally deferred until the first Rust implementation PR.

The evidence-backed capability lifecycle in [`capability-lifecycle.md`](capability-lifecycle.md), issue #9, and DR-0004 is a later-phase design constraint. The full lifecycle, promotion gates, profiles, and adaptive exposure policies are not requirements for the first v0 executable baseline.

## Later phases, not v0 core scope

- local or remote LLM-backed planning
- model weakness profiling and runtime adaptation
- capability reputation/tier/ranking policies
- evidence-backed capability promotion/supersession automation
- automatic skill creation or mutation
- project memory intelligence
- tool-schema pruning
- MCP capability discovery and lazy activation
- context budgeting and context garbage collection
- tool-output virtualization
- assumption and decision tracking
- failure monitoring and goal-drift detection
- multi-agent scheduling
- Harness LLM fine-tuning
