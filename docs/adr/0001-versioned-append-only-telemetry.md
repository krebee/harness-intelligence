# DR-0001: Use versioned append-only events as the canonical telemetry model

- Status: accepted
- Date: 2026-08-24
- Governing issue: #3
- Supersedes: none
- Superseded by: none

## Context

Harness Intelligence is expected to evolve from simple capability/context selection into a runtime that can evaluate and improve how models, skills, tools, MCP servers, memory, and context are used.

Future questions include:

- Did the harness expose the right capability?
- Did the model ignore an available tool or emit an invalid tool call?
- Under which model/backend/reasoning settings does a model stall before taking action?
- Which skill revision improves outcomes?
- Is a repeated recovery procedure evidence that a skill should be created or improved?
- Can a future HarnessBrain training corpus be reconstructed from actual runtime behavior?

If telemetry stores only current aggregate metrics, later analyses are limited to metrics that were anticipated at collection time. Reconstructing past runtime evidence becomes impossible.

At the same time, retaining all raw prompt/model/tool content by default creates unnecessary privacy, secret-management, and storage risk.

A durable telemetry philosophy is therefore needed before runtime implementation starts.

## Decision

Harness Intelligence will use **versioned append-only events** as the canonical telemetry model.

The following are durable parts of the decision:

1. Canonical telemetry records low-level runtime events rather than only aggregate/derived metrics.
2. Persisted events are immutable. Later corrections, classifications, and evaluations are new events/annotations that reference prior evidence.
3. Every event is explicitly schema-versioned from v0.
4. Events carry run correlation, deterministic run-local ordering, and optional causal-parent references.
5. Model, harness, and capability identities/revisions are recorded when observable so behavior can be attributed to the exact runtime configuration.
6. Derived metrics and model/skill quality scores are analysis-layer products and should be reproducible from canonical events where practical.
7. Raw prompt/model/tool content is not required for canonical telemetry. Content capture is policy-controlled and defaults to metadata-only.
8. Secrets are redacted before persistent sinks receive events.
9. Private chain-of-thought / hidden reasoning text is not a telemetry requirement. Provider/runtime-exposed reasoning counts/configuration/timing may be recorded as metadata.
10. Canonical event semantics are independent of the persistence backend.
11. The first implementation should use an append-only local JSONL sink for inspection and experimentation. Indexed/remote sinks can be added without changing canonical event meaning.

The detailed v0 event families and payload requirements are defined in `docs/telemetry-v0.md`.

## Alternatives considered

### Store only aggregate metrics

Examples would include task-success rate, token savings, tool-call failure rate, or skill usefulness scores.

This is simpler to query but permanently loses evidence needed for new metrics, failure attribution, replay, and training-corpus generation. It is rejected as the canonical representation, though aggregates may be derived and cached later.

### Store mutable per-run records

A run could be represented as one row/document that is updated as execution progresses.

This makes the current state easy to inspect but loses historical intermediate states, complicates concurrency, and allows later evaluators to overwrite original evidence. It is rejected as the canonical model.

### Store full prompts, outputs, and logs by default

This maximizes retrospective analysis but creates disproportionate privacy, credential, retention, and storage risk. It is rejected as the default.

Optional redacted content capture remains possible under explicit policy.

### Make OpenTelemetry the canonical schema

OpenTelemetry may become a useful export/integration target, but making an external observability model the canonical domain schema would couple Harness Intelligence semantics to a standard primarily designed for general tracing/metrics/logging.

It is deferred as an adapter/export concern.

### Use SQLite as the initial source of truth

SQLite is attractive for querying but would encourage storage-specific schema decisions before actual query patterns are understood.

The initial JSONL sink is intentionally simpler and corpus-friendly. SQLite can be added as a sink/index once real usage demonstrates the need.

## Consequences

### Benefits

- Future metrics can be recomputed from retained evidence.
- Runtime failures can be attributed across harness/model/capability boundaries.
- Model behavior can be compared by revision, backend, quantization, reasoning configuration, and capability surface when those fields are observable.
- Skill revisions can be correlated with exposure/use/outcome evidence.
- Future evaluation and training-data generation have a stable source corpus.
- Storage implementations can evolve independently of telemetry semantics.
- Metadata-only operation avoids requiring sensitive content retention.

### Costs

- Event schemas and compatibility need deliberate maintenance.
- Append-only streams require derived views/indexes for convenient querying.
- Run reconstruction is more complex than reading one mutable record.
- Sequence assignment and causal references require explicit runtime plumbing.
- Redaction must happen before persistence and therefore becomes part of the telemetry boundary.

### Risks

- Overly broad event payloads could still create high-volume telemetry.
- Missing identity/revision metadata can make later attribution ambiguous.
- Optional fields from heterogeneous harness/model providers can lead to sparse datasets.
- JSONL is not an efficient long-term query engine and must not become an accidental permanent storage constraint.

## Validation

This decision is validated if the v0 implementation can demonstrate that:

- Observation -> plan -> context -> model/capability execution -> outcome can be reconstructed for a run.
- Event ordering remains deterministic within a run.
- Required capability exposure can be compared with actual model requests/execution.
- Model/skill revisions survive serialization when available.
- Reasoning-stall analysis is possible from metadata without reasoning text.
- Derived metrics can be recalculated from event history.
- metadata-only capture does not persist prompt/model/tool bodies.
- secret fixtures are removed before sink output.
- a different sink can consume the same canonical event objects without changing domain semantics.

## Follow-up

- Finalize the v0 event payloads in #3.
- Create an implementation issue for the Rust telemetry envelope/emitter and JSONL sink after the design PR is accepted.
- Reconsider indexed storage only after real query/analysis patterns exist.
- Reconsider OpenTelemetry export only when integration with external observability stacks becomes a concrete requirement.
- Create new Decision Records if retention policy, remote collection, or telemetry failure semantics become durable project constraints.
