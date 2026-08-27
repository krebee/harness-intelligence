# DR-0002: Keep the v0 runtime boundary small and harness-independent

- Status: proposed
- Date: 2026-08-26
- Governing issue: #5
- Supersedes: none
- Superseded by: none

## Context

Harness Intelligence needs a concrete runtime boundary before Rust implementation begins. The project already defines telemetry semantics, but the core contracts among observation, planning, application, and concrete harness integration are still intentionally unresolved.

The main risk at this stage is premature abstraction: modeling future memory, MCP, tool, context-GC, model-adaptation, and multi-agent systems before the first executable baseline exists. A second risk is coupling the core to the first harness integration and making later adapters expensive.

The project needs a baseline that is simple enough to implement and test, but durable enough that later intelligent planners can be compared against it.

## Decision

Harness Intelligence v0 will use a small, harness-independent control boundary built around four responsibilities:

1. `HarnessSnapshot` — immutable observation at one planning point.
2. `HarnessBrain` — pure planning boundary that consumes a snapshot and returns a plan.
3. `HarnessPlan` — structured, validated decision describing the v0 exposure/selection result.
4. Harness adapter — concrete integration that captures snapshots and applies plans.

The first `HarnessBrain` implementation will be deterministic `PassThroughBrain` behavior. It establishes the control condition before rule-based or LLM-backed intelligence is introduced.

The initial Rust workspace will start with only durable boundaries already justified by current requirements:

```text
crates/
  core/
  telemetry/
```

A separate adapter crate is not required until a concrete adapter exists and independent packaging is useful. Test/reference adapter code may live with integration-test support initially.

Telemetry is part of the executable baseline. Planning/orchestration depends on an emitter abstraction rather than a concrete sink, while append-only JSONL remains the first reference sink under DR-0001.

Plans are validated before adapter application so planning/validation failures remain distinguishable from adapter/application failures.

v0 will not introduce speculative domain types or crates for later-phase memory, MCP, model adaptation, context GC, tool-output virtualization, multi-agent scheduling, or training systems.

## Alternatives considered

### One crate for the entire v0 implementation

This minimizes initial packaging work, but telemetry already has a durable backend-independent contract and is expected to be reused across core and adapters. Keeping telemetry separate provides a justified boundary without requiring broad decomposition.

### Create separate crates for core, telemetry, adapters, capabilities, memory, MCP, and inference immediately

Rejected because most of those boundaries are not yet supported by executable requirements. They would turn future hypotheses into present architecture and increase refactoring surface before evidence exists.

### Couple the core directly to DeepSeek Harness first

This may shorten the very first integration path, but conflicts with the project's goal of adding an intelligence layer to arbitrary harnesses. Harness-specific state and hooks therefore stay behind an adapter boundary.

### Begin with a rule-based or LLM-backed brain

Rejected as the baseline. Without `PassThroughBrain`, later changes would lack a deterministic control path for correctness, telemetry, context reduction, latency, and success comparisons.

### Let brains mutate the concrete harness directly

Rejected because it mixes observation, decision, and execution, makes replay/evaluation harder, and weakens failure attribution. Brains return plans; adapters apply them.

## Consequences

### Benefits

- Rust implementation can begin with a small, explicit architecture.
- The first executable path is deterministic and suitable as an experimental control.
- Concrete harness dependencies remain outside the core.
- Telemetry exists from the first runnable path.
- Plan validation makes failure attribution clearer.
- Later features can earn new abstractions based on observed requirements rather than speculation.

### Costs and tradeoffs

- Early adapter code may move once a production adapter deserves its own crate.
- The first capability surface is intentionally narrow and may require schema evolution for tools/MCP/memory later.
- Some implementation choices such as async/cancellation primitives remain deferred until Rust code exists.
- A pass-through baseline adds an implementation step before intelligent selection, but this cost is intentional for evaluation quality.

## Validation

This decision is validated when the first implementation can demonstrate an end-to-end reference path:

```text
Adapter -> HarnessSnapshot -> PassThroughBrain -> HarnessPlan -> validation -> apply
```

with canonical telemetry emitted and persisted through the JSONL reference sink.

Validation must include:

- repository Rust validation commands passing
- deterministic pass-through behavior
- rejection of invalid plan references
- distinguishable planning, validation, adapter, telemetry, and cancellation outcomes
- no concrete harness dependency in the core crate
- end-to-end telemetry ordering sufficient to reconstruct the run

The decision should be reconsidered if a concrete adapter cannot be implemented without leaking harness-specific types into the core, or if the two-crate workspace creates a demonstrated circular dependency or ownership problem.

## Follow-up

After acceptance:

1. create a Rust workspace bootstrap issue/PR
2. implement the telemetry crate baseline from the accepted v0 telemetry design
3. implement `HarnessSnapshot` and `HarnessPlan`
4. implement plan validation
5. implement `HarnessBrain` and `PassThroughBrain`
6. implement a reference/test adapter and end-to-end integration test
7. begin Rule-Based Skill Catalog Intelligence experiments

Any implementation choice that creates a durable compatibility or public-API constraint should be recorded as a separate Decision Record rather than expanding this record retroactively.
