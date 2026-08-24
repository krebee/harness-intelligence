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

### Harness Adapter

Bridges a concrete agent harness to the Harness Intelligence core. It is responsible for collecting the information needed to build a snapshot and applying the resulting plan back to the harness.

### HarnessSnapshot

Represents the observation available to Harness Intelligence at a planning point. Its exact schema is not finalized for v0.

Likely inputs include task context, available capabilities, runtime state, model identity, and context-budget information.

### HarnessBrain

Consumes a snapshot and produces a structured decision. v0 should begin with the smallest possible implementation, such as pass-through and rule-based behavior, before introducing a local or remote LLM planner.

### HarnessPlan

Represents the structured output of the brain. The exact schema is not finalized for v0. It is expected to describe what capabilities and context should be exposed to the main agent.

### Telemetry

Telemetry is a foundational concern rather than a later add-on. The runtime should make it possible to reconstruct the relationship between observation, harness decision, execution, and outcome so future evaluation and training-data generation remain possible.

The proposed v0 telemetry contract is documented in [`telemetry-v0.md`](telemetry-v0.md). Its durable event-model philosophy is recorded in [`DR-0001`](adr/0001-versioned-append-only-telemetry.md).

The proposal uses versioned append-only events, separates raw evidence from derived metrics, makes model/capability revision attribution first-class, and keeps raw content capture policy-controlled rather than mandatory.

## Architectural constraints

- The main agent is replaceable and remains outside the Harness Intelligence core.
- The core should not depend on a single harness implementation.
- Skills, tools, MCP servers, and memory providers may eventually be represented through a common capability model, but v0 should not over-generalize before requirements are validated.
- Project-specific knowledge should remain external to model weights.
- Derived metrics should be reproducible from lower-level telemetry where practical.
- Telemetry semantics should remain independent of the persistence backend.

## Explicitly unresolved for v0 design

The following should be decided in dedicated v0 design work rather than in repository bootstrap:

- Rust workspace and crate boundaries
- exact `HarnessSnapshot` schema
- exact `HarnessPlan` schema
- `HarnessBrain` trait contract
- adapter boundary and lifecycle hooks
- initial skill catalog filtering interface
- error and cancellation model
- synchronization and concurrency strategy

For telemetry, the domain semantics are proposed in `docs/telemetry-v0.md`; exact Rust type layout, sink async strategy, redactor implementation, and adapter instrumentation hooks remain implementation decisions.

## Later phases, not v0 bootstrap scope

- project memory intelligence
- tool-schema pruning
- MCP capability discovery and lazy activation
- context budgeting and context garbage collection
- tool-output virtualization
- assumption and decision tracking
- failure monitoring and goal-drift detection
- Harness LLM fine-tuning
