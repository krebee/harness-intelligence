# Harness Intelligence v0 core runtime

Status: proposed  
Governing issue: #5

## Goal

Define the smallest executable Harness Intelligence runtime that proves the control boundary before adding intelligent planning.

The v0 baseline is:

```text
Harness Adapter
    -> HarnessSnapshot
    -> HarnessBrain
    -> HarnessPlan
    -> Harness Adapter
    -> Main Agent
         |
         +-> Telemetry
```

The baseline brain is deterministic and pass-through. Intelligence is introduced only after this path is observable and testable.

## Runtime boundaries

### HarnessSnapshot

`HarnessSnapshot` is an immutable observation captured at one planning point.

It should contain data, identifiers, and descriptors only. It must not contain mutable harness services, active tool handles, database connections, or other execution capabilities.

v0 needs enough information to support the pass-through baseline and the first Skill Catalog Intelligence experiment:

- snapshot / planning-point identity
- run, task, and turn correlation identifiers when available
- main-model identity/reference needed for telemetry attribution
- task/request metadata without requiring raw request content
- available skill descriptors and revision identifiers
- context-budget metadata when observable
- adapter/runtime identity and revision metadata

The exact Rust fields may evolve during implementation, but adding unrelated future subsystems to the snapshot is out of scope.

### HarnessPlan

`HarnessPlan` is the immutable decision returned by `HarnessBrain` and consumed by the adapter.

For v0 it represents exposure/selection for the first supported capability surface. The initial plan should contain only what is needed to describe pass-through and skill filtering, including:

- selected/exposed skill identifiers and revisions
- explicit pass-through semantics
- optional structured decision metadata such as reason codes or confidence when a brain produces them
- brain identity/revision for attribution if that information is not carried separately by the runtime

A plan must not mutate the concrete harness directly.

Future policy for memory, MCP lifecycle, context GC, tool-output virtualization, or model adaptation must not be added speculatively.

### HarnessBrain

`HarnessBrain` maps one snapshot to one plan.

Conceptually:

```text
plan(snapshot) -> HarnessPlan
```

The Rust contract must be async-compatible because future implementations may call local or remote models. It must also support cancellation through the runtime abstraction selected during implementation.

The trait must not depend on:

- a concrete harness implementation
- a persistence backend
- JSONL
- a specific model provider
- mutable harness services

Every implementation must expose enough identity/version metadata for telemetry attribution.

The first implementation is `PassThroughBrain`, which exposes the v0 skill surface unchanged. The next planned implementation is a rule-based skill filter. LLM-backed planning is not part of v0 core.

### Harness Adapter

The adapter owns all concrete harness integration.

Its responsibilities are:

1. observe the concrete harness/runtime and build a `HarnessSnapshot`
2. invoke the Harness Intelligence planning path
3. validate and apply the resulting `HarnessPlan`
4. translate harness/model/capability lifecycle observations into canonical telemetry where appropriate

The core must not import concrete harness types.

The first executable implementation should use a test/reference adapter before adding a production adapter if doing so keeps the boundary simpler to validate. A concrete DeepSeek Harness adapter can follow immediately once the contract is proven.

## Workspace direction

Start with the fewest durable crate boundaries that correspond to already-decided responsibilities:

```text
crates/
  core/
  telemetry/
```

`core` owns domain types and planning contracts:

- `HarnessSnapshot`
- `HarnessPlan`
- `HarnessBrain`
- `PassThroughBrain`
- adapter-facing core contracts

`telemetry` owns the canonical telemetry model and sink/emitter abstractions defined in `docs/telemetry-v0.md`.

Do **not** create a dedicated adapter crate in the first workspace commit unless implementation demonstrates that the boundary benefits from independent packaging. A test/reference adapter may initially live in integration-test support or a small core-adjacent module. Concrete harness adapters should become separate crates when they exist.

Do not create crates for memory, MCP, model inference, analytics, policy learning, or other later phases.

## Planning orchestration

The minimal orchestration path is:

```text
capture snapshot
  -> emit observation telemetry
  -> invoke brain
  -> validate plan
  -> emit plan telemetry
  -> adapter applies plan
  -> emit application/execution telemetry
  -> emit outcome and run lifecycle telemetry
```

The orchestration layer may be a small service/function rather than a new framework abstraction. v0 should optimize for explicit control flow and inspectability.

## Plan validation

A plan must be validated before application.

At minimum v0 should reject:

- references to skills/capabilities not present in the snapshot
- duplicate identifiers where uniqueness is required
- structurally invalid plans
- version/revision references that cannot be reconciled with the snapshot

Validation is a core concern because an invalid plan is semantically different from an adapter application failure.

## Error model

v0 must preserve failure attribution across at least these classes:

- invalid snapshot/input
- planning/brain failure
- plan validation failure
- adapter/application failure
- telemetry emission/persistence failure
- cancellation

The implementation may use one top-level error enum with typed variants. Avoid reducing all failures to opaque strings.

Errors should carry enough structured category information for telemetry without requiring every internal error type to become part of the public API.

## Cancellation

Cancellation must propagate across one planning operation and its orchestration path.

v0 does not prescribe the exact Rust primitive in this design document. The implementation should choose the smallest abstraction that works with async planning and future adapter integration.

Cancellation must be distinguishable from ordinary failure in telemetry and tests.

## Concurrency

v0 does not require concurrent planning or scheduling.

Use one planning operation per planning point. Preserve deterministic per-run telemetry ordering. Avoid shared mutable global state until a demonstrated requirement exists.

## Telemetry integration

Telemetry is part of the baseline execution path, not an optional follow-up feature.

The core/orchestration layer depends on a narrow emitter abstraction. It does not depend directly on JSONL or another persistence backend.

The first reference sink remains append-only JSONL as defined in `docs/telemetry-v0.md`.

A baseline run should make it possible to reconstruct:

```text
run started
  -> observation captured
  -> plan produced
  -> plan applied
  -> execution/outcome observed when available
  -> run completed / failed / cancelled
```

Telemetry persistence failure must be observable. It must not silently transform a successful planning result into a different semantic plan result. The implementation should make the failure policy explicit and configurable at the orchestration boundary.

## Initial executable baseline

The first end-to-end integration test should prove:

```text
reference adapter
  -> HarnessSnapshot with N available skills
  -> PassThroughBrain
  -> HarnessPlan exposes the same N skills
  -> plan validation succeeds
  -> adapter applies the plan
  -> canonical telemetry is emitted in deterministic order
  -> JSONL sink persists the event stream
```

This becomes the control condition for later rule-based and LLM-backed experiments.

## v0 structural completion criteria

The v0 core runtime is structurally complete when:

- the Rust workspace builds with the repository validation commands
- `HarnessSnapshot` and `HarnessPlan` have explicit serialization/versioning policy where serialization is required
- `HarnessBrain` and `PassThroughBrain` exist
- plan validation exists
- a harness-independent adapter boundary exists
- an end-to-end reference/test adapter exercises Snapshot -> Brain -> Plan -> apply
- canonical telemetry is emitted across that path
- the JSONL reference sink persists it through a sink abstraction
- cancellation and the major error classes are covered by tests

After this point, Skill Catalog Intelligence can begin as an experiment rather than as infrastructure construction.

## Implementation sequence

1. initialize the Rust workspace and CI validation
2. implement canonical telemetry event types, emitter/sink contracts, and JSONL reference sink
3. implement core identity/descriptor types plus `HarnessSnapshot` and `HarnessPlan`
4. implement plan validation
5. implement `HarnessBrain` and `PassThroughBrain`
6. implement a reference/test adapter and orchestration path
7. add end-to-end telemetry integration tests
8. implement rule-based Skill Catalog Intelligence as the first non-pass-through experiment

Each substantial step should remain a narrow issue/PR.

## Explicit non-goals

The v0 core runtime does not implement:

- LLM-backed planning
- model weakness scoring or model-profile adaptation
- reasoning-budget intervention
- automatic skill creation or mutation
- project memory or automatic memory management
- tool-schema pruning
- MCP lazy activation
- context garbage collection
- tool-output virtualization
- failure-monitor / goal-drift correction
- multi-agent scheduling
- distributed execution
- training/fine-tuning pipelines
- analytics dashboard/UI

## Questions intentionally left to implementation

The following choices can be made in the first implementation PR without changing the runtime boundary, unless implementation reveals a durable tradeoff that warrants another Decision Record:

- exact Rust crate/package names
- exact async trait mechanism
- exact cancellation primitive
- concrete UUID/newtype libraries
- serialization library details
- internal module layout

If any of these choices materially constrain future adapters, compatibility, public APIs, or persistence semantics, they should be promoted to a separate durable decision.
