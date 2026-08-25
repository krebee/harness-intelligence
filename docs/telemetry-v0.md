# Telemetry v0 design

- Status: proposed
- Governing issue: #3

## Purpose

Telemetry v0 defines the canonical evidence Harness Intelligence must preserve before higher-level intelligence is added.

The goal is not to predict every future metric. The goal is to retain enough low-level, attributable runtime evidence that later components can analyze:

- harness decision quality
- context efficiency
- model-specific strengths and weaknesses
- tool-call reliability
- reasoning stalls or failure to reach an externally observable action
- skill/capability usefulness
- repeated procedural discovery and potential skill-creation opportunities
- future training/evaluation data

Telemetry is therefore a runtime contract, not a logging convenience.

## Scope

v0 covers:

- a versioned event envelope
- run-local event ordering and causality
- required event families
- identity/revision metadata for models, harness components, capabilities, and skills
- context/token accounting
- model/tool execution metadata
- outcome and later evaluation annotations
- capture/redaction policy
- schema evolution rules
- a backend-independent sink contract
- a simple JSONL reference sink for the first implementation

v0 does not implement model profiling, skill self-improvement, automatic failure attribution, analytics dashboards, remote ingestion, or training pipelines.

## Design principles

### 1. Canonical events are append-only

Persisted telemetry events are immutable. Corrections, evaluations, and later interpretations are represented by new events that reference prior events or runs.

This preserves the original observation and avoids silently rewriting training/evaluation history.

### 2. Raw evidence precedes derived metrics

Metrics such as `skill_miss_rate`, `reasoning_stall_rate`, `tool_call_reliability`, or a future `ModelProfile` are derived views.

The canonical telemetry stream should retain the lower-level observations required to recompute them.

### 3. Causality is explicit

A run must be reconstructable as:

```text
observation
  -> harness decision
  -> context assembly
  -> model execution
  -> capability requests/execution
  -> outcome
  -> later evaluation/annotation
```

Each event belongs to a run and may reference its causal parent.

### 4. Attribution must separate harness, model, and capability behavior

Telemetry must allow later analysis to distinguish cases such as:

```text
required tool was never exposed
  !=
tool was exposed but model never requested it
  !=
model emitted an invalid tool call
  !=
tool call was valid but execution failed
```

The same principle applies to skills, MCP servers, memory, and future capability kinds.

### 5. Content retention is optional

Metadata needed for attribution and measurement must not depend on storing raw prompts, model outputs, or tool outputs.

Content may be retained only under an explicit policy and only after redaction.

### 6. Hidden reasoning content is not telemetry

Telemetry may record provider/runtime-exposed reasoning metadata such as reasoning-token counts, thinking-mode configuration, timing, or stop reasons.

It must not require capture or persistence of private chain-of-thought / hidden reasoning text.

### 7. Schema evolution starts at v0

The envelope and event payloads are versioned from the beginning. Existing field meanings must not be silently reinterpreted.

### 8. Storage is a sink concern

The canonical event model is independent of JSONL, SQLite, OpenTelemetry, or a future remote store.

## Canonical event envelope

Conceptually, every event has this shape:

```rust
struct TelemetryEvent<P> {
    schema_version: u16,
    event_version: u16,
    event_id: EventId,
    event_type: EventType,
    occurred_at: DateTime<Utc>,
    sequence: u64,

    run_id: RunId,
    session_id: Option<SessionId>,
    task_id: Option<TaskId>,
    turn_id: Option<TurnId>,

    parent_event_id: Option<EventId>,
    source: EventSource,
    payload: P,
}
```

The exact Rust API is an implementation detail for a follow-up issue, but these semantics are part of the v0 telemetry contract.

### Envelope fields

#### `schema_version`

Version of the shared envelope semantics.

Breaking envelope changes increment this value.

v0 begins at `1`.

#### `event_version`

Version of the payload schema for the specific `event_type`.

Breaking changes to one event family can therefore evolve without changing the entire envelope.

#### `event_id`

Opaque globally unique event identifier.

v0 should use UUIDv7 unless implementation constraints reveal a concrete reason to choose another identifier. Event consumers must treat the identifier as opaque.

#### `event_type`

Stable namespaced string/enum identifier, for example:

```text
run.started
observation.captured
plan.created
context.assembled
model.call.finished
capability.requested
capability.execution.finished
outcome.recorded
evaluation.annotation
```

#### `occurred_at`

UTC wall-clock timestamp used for human inspection and external correlation.

Ordering inside one run must not rely on this timestamp alone.

#### `sequence`

Monotonically increasing sequence number assigned per run.

The telemetry emitter owns sequence assignment. This is the authoritative ordering for events accepted into a run's telemetry stream.

#### `run_id`

Required correlation id for one Harness Intelligence execution run.

#### `session_id`, `task_id`, `turn_id`

Optional higher-level identities supplied by adapters when available. The core telemetry design must not require every harness to expose all three.

#### `parent_event_id`

Optional causal parent. This is not merely nesting for display; it allows reconstruction of which observation/decision/call caused subsequent work.

#### `source`

Identifies the producer of the event, including component kind and revision where available.

Recommended fields:

```text
component_kind
component_id
component_version
```

Examples include `core`, `adapter`, `brain`, `model_runtime`, `capability_runtime`, `evaluator`.

## Identity and revision requirements

Telemetry is useful only when behavior can be attributed to the exact thing that produced it.

### Harness identity

Runs should record when available:

- Harness Intelligence version
- source revision / git SHA
- adapter name and version
- HarnessBrain name and version
- relevant policy/config revision or hash

### Model identity

Model execution telemetry should preserve enough identity to avoid incorrectly grouping materially different runtimes.

Recommended fields:

```text
role                    # main / harness / sub-agent / verifier
provider
model_id
model_revision
inference_backend
inference_backend_version
quantization
chat_template_revision
tool_parser_revision
context_limit
```

Fields are optional when not observable, but absence must not be replaced with guessed values.

For local models, backend, quantization, template, and tool parser can materially change behavior and are therefore valuable analysis dimensions.

### Model request configuration

Record behavior-affecting request configuration when known:

- temperature
- top_p
- top_k
- seed
- max_output_tokens
- reasoning/thinking mode
- reasoning effort/budget when supported
- tool-choice mode
- response/schema mode

Provider-specific configuration may be carried in a sanitized extension map rather than forcing every provider option into the core schema.

### Capability identity

Every capability reference should carry:

- `capability_id`
- `kind` (`skill`, `tool`, `mcp`, `memory`, or future kinds)
- revision/version when available

Skills specifically need a revision identifier. A content hash is acceptable when an explicit semantic version does not exist.

This allows later comparison of outcomes across skill revisions rather than treating a changing skill as one timeless object.

## Event families

## 1. Run lifecycle

### `run.started`

Marks creation of a run.

Payload should include:

- harness identity
- adapter identity
- initial telemetry policy
- optional sanitized execution-environment metadata

### `run.finished`

Marks normal terminal completion.

Payload should include terminal status and aggregate measurements when available.

### `run.failed`

Marks terminal failure.

### `run.cancelled`

Marks explicit cancellation.

A terminal lifecycle event does not replace `outcome.recorded`; lifecycle state and evaluated task outcome are separate concepts.

## 2. Observation

### `observation.captured`

Records what Harness Intelligence had available when planning.

Payload should include metadata for:

- task/request identity
- task classification if one exists
- attachment metadata
- repository/project signals available to the adapter
- available capability ids/revisions
- main model identity
- current context budget
- runtime state relevant to planning

Raw user/task text is governed by the content-capture policy and is not required in default telemetry.

The observation event is the baseline for evaluating later pruning or selection decisions.

## 3. Harness planning

### `plan.created`

Records a `HarnessBrain` decision.

Payload should include when available:

- brain identity/version
- candidate capability ids
- selected capability ids
- excluded capability ids
- selected memory/context ids
- context policy
- tool-output policy
- decision confidence
- structured reason codes
- planner latency

Free-form hidden reasoning is not required and should not be persisted as telemetry.

The important data is the structured decision and the evidence available to it.

## 4. Context assembly

### `context.assembled`

Records the actual context surface constructed for a model call.

At minimum, record token counts by category:

```text
system
task
history
skills
tool_schemas
memory
repository
retrieved
tool_results
other
```

The event should support both before-policy and after-policy totals when a planning/filtering step changes the context surface.

Recommended per-item metadata:

```text
kind
source_id
source_revision
included
estimated_tokens
selection_source
reason_code
```

Full item content is not required.

This event enables direct measurement of context reduction without losing attribution to what was removed or retained.

## 5. Capability exposure and selection

### `capability.exposed`

Records the actual capability surface made visible/available to a model at a specific call or planning point.

Payload should include capability ids and revisions.

This event is intentionally distinct from `plan.created`: a plan may request exposure but an adapter may fail or alter application of that plan.

### `capability.requested`

Records that a model requested a capability action or load.

For tools/MCP this corresponds to a parsed, semantically recognized tool request before execution.

For skills this may represent an explicit skill load/request where the harness supports such a lifecycle.

Recommended payload fields:

- model call id / parent event
- capability id/kind/revision
- action name when applicable
- sanitized argument metadata
- argument hash when enabled
- parse status

Invalid or unparsable tool syntax must not be represented as a successful `capability.requested`; it belongs in model-call parsing metadata or an error event.

### `capability.loaded`

Optional event for capability kinds with an explicit load/activation lifecycle, especially skills and MCP servers.

### `capability.rerequested`

Optional explicit event when the model requests a capability that had been filtered or previously unavailable. This is particularly valuable for measuring pruning misses.

## 6. Model execution

### `model.call.started`

Records request dispatch to a model runtime.

Payload should contain:

- model identity
- request configuration
- context token count when already known
- exposed capability ids or a reference to `capability.exposed`

### `model.call.finished`

Records normal completion, timeout, parser failure, or other terminal model-call state.

Recommended fields:

```text
status
latency_ms
input_tokens
output_tokens
reasoning_tokens
reasoning_tokens_source
stop_reason
final_output_reached
tool_calls_emitted
invalid_tool_calls
first_action_latency_ms
```

All token/timing fields are optional when not observable.

Measurements should preserve provenance when their interpretation matters:

```text
provider_reported
runtime_measured
estimated
```

#### Reasoning-stall support

A later analyzer should be able to identify patterns such as:

```text
high reasoning-token use
+ no valid capability request
+ no final output
+ timeout/max-token stop
```

without access to reasoning text.

`first_action_latency_ms` is the duration until the first externally observable action, such as a valid tool request or user-visible final output, when the backend makes that measurement possible.

## 7. Capability execution

### `capability.execution.started`

Records dispatch of an executable capability.

### `capability.execution.finished`

Recommended payload fields:

- capability id/kind/revision
- action name
- status
- latency
- sanitized argument metadata/hash
- error category
- stable error signature when available
- raw result byte/token size when known
- result byte/token size exposed to the model when known

Keeping raw-result size separate from exposed-result size makes future tool-output compression/virtualization measurable.

### `capability.result.exposed`

Optional event for future runtimes where capability execution and later selective result exposure are separate lifecycle stages.

v0 does not need Tool Output Virtualization to exist, but its telemetry model should not make that feature impossible to observe later.

## 8. Runtime errors

### `runtime.error`

Used for errors that do not fit naturally into a terminal event such as `model.call.finished` or `capability.execution.finished`.

Payload should prefer stable error categories and signatures over unrestricted raw exception/log bodies.

Raw stack traces/log text are content and follow capture/redaction policy.

## 9. Outcome

### `outcome.recorded`

Outcome is an evaluation of task/run result, not merely a lifecycle state.

Required status enum:

```text
success
partial
failure
unknown
```

`unknown` is important. Telemetry must not invent correctness when no evaluator exists.

Recommended optional evidence:

- task completed signal
- test/validation result references
- retry count
- human intervention occurred
- user correction occurred
- total latency
- aggregate token counts
- estimated/reported cost

Multiple outcome/evaluation events may exist over time if a later evaluator adds stronger evidence.

## 10. Evaluation and annotation

### `evaluation.annotation`

Append-only interpretation attached to an event, call, capability, or entire run.

Recommended fields:

```text
target_kind
target_id
evaluator_kind       # human / test / rule / model / offline-analysis
evaluator_id
evaluator_version
label
confidence
structured_details
```

Potential future labels include:

```text
capability_miss
model_ignored_capability
invalid_tool_call
reasoning_stall
harness_pruning_error
stale_memory
incorrect_memory
skill_useful
skill_ineffective
skill_creation_opportunity
skill_improvement_opportunity
```

These labels are not required to be produced automatically in v0. The event exists so future evaluators can add interpretation without rewriting raw history.

## Content capture and redaction

## Capture modes

v0 defines two policy modes:

### `metadata_only`

Default.

Prompt bodies, model response bodies, tool arguments, and tool-result bodies are omitted from persisted telemetry.

Metadata such as ids, revisions, sizes, token counts, timings, status, and sanitized structured descriptors may still be persisted.

### `redacted_content`

Explicit opt-in.

Content may be persisted only after the configured redaction pipeline has completed.

There is intentionally no `unredacted_full` v0 mode.

## Redaction boundary

Redaction occurs before an event reaches a persistent sink.

Persistent sinks must never be responsible for discovering secrets after data is already written.

At minimum, the redaction layer must be designed to prevent persistence of:

- authentication headers
- API keys/tokens
- environment secret values
- connection strings/credentials
- explicitly marked secret tool arguments

The redaction subsystem itself is a later implementation concern, but the boundary is part of this contract.

## Hidden reasoning

Private chain-of-thought / hidden reasoning text is never required by this telemetry design and should not be captured merely because a model backend exposes an internal representation.

Provider-reported reasoning token counts, thinking configuration, and timing are ordinary metadata and may be recorded.

## Omitted content descriptors

When a body is omitted, events may retain non-content metadata such as:

- byte length
- token count
- MIME/content kind
- optional fingerprint/hash when policy allows

A fingerprint is not mandatory and should itself be considered potentially sensitive correlation metadata.

## Retention

Metadata and optional retained content must be independently configurable in future sinks.

v0 semantics assume:

- metadata can be retained without content
- content capture defaults to off
- content may have a shorter retention period than metadata
- deleting optional content must not make the remaining event stream structurally invalid

The first JSONL sink does not need automated TTL deletion. Retention automation is a later storage feature.

## Schema evolution

### Envelope changes

Breaking changes to envelope meaning increment `schema_version`.

### Payload changes

Breaking changes to one `event_type` payload increment `event_version`.

### Compatibility rules

Consumers should:

- ignore unknown optional fields
- tolerate unknown event types when replaying/storing a stream
- never assume all adapters populate every optional field
- preserve event ids and ordering when transforming/copying events

Producers must not:

- change the meaning of an existing field without a version change
- reuse an event type for a different semantic action
- replace unknown/missing measurements with guessed zero values

`null`/absence means unknown or unavailable; zero means an observed zero.

## Sink contract

The core emits canonical telemetry events to a sink abstraction.

Conceptually:

```rust
trait TelemetrySink {
    async fn emit(&self, event: TelemetryEvent) -> Result<()>;
    async fn flush(&self) -> Result<()>;
}
```

The exact Rust trait and async strategy are not fixed by this design.

Required sink semantics:

- accept canonical already-redacted events
- preserve event contents without reinterpretation
- preserve run-local sequence values
- surface persistence failure to the runtime according to configured policy

Telemetry persistence should normally be best-effort with visible failure reporting rather than silently breaking the main-agent task, but exact failure policy belongs in implementation design.

## v0 reference sink: JSONL

The first implementation should provide a local append-only JSONL sink.

Rationale:

- easy to inspect during early development
- trivial to export into Python/Rust analysis
- suitable as future training-corpus input
- minimal schema/storage coupling
- easy to diff and archive during experiments

Each line is one canonical serialized event.

SQLite/indexed storage is deferred until query patterns are demonstrated by real telemetry usage.

The JSONL representation is a sink format, not the source of truth for domain semantics.

## Reconstructing a run

A minimal successful run may look like:

```text
run.started
  observation.captured
    plan.created
      capability.exposed
      context.assembled
        model.call.started
          model.call.finished
            capability.requested
              capability.execution.started
                capability.execution.finished
            model.call.started
              model.call.finished
  outcome.recorded
run.finished
```

Not every run has every event. The important property is that present events retain identity, ordering, and causal references.

## Future analyses enabled by v0 telemetry

## Model-specific behavior

### Tool-call reliability

Inputs:

- model identity/revision/runtime configuration
- capability exposure
- model call completion/parsing metadata
- capability requests
- capability execution outcomes

Possible derived views:

```text
valid_tool_requests / tool_intent_opportunities
invalid_tool_calls / model_calls_with_tools
tool_execution_success / valid_tool_requests
```

The precise metric definitions are intentionally deferred.

### Reasoning stalls / overthinking

Inputs:

- model/backend/revision
- thinking/reasoning configuration
- reasoning token count when available
- model-call duration
- first action latency
- tool-call count
- final output reached
- stop reason

A future rule/model can classify stalls without requiring reasoning-text retention.

### Runtime-policy adaptation

Telemetry can later compare outcome and efficiency across dimensions such as:

- reasoning effort/budget
- number of exposed tools
- context size/composition
- tool parser revision
- quantization/backend

The result may eventually feed a `ModelProfile`, but v0 records evidence only.

## Skill and capability evolution

### Skill effectiveness

Inputs:

- skill id/revision
- availability/candidate/exposure/load events
- model/capability behavior
- outcome/evaluation annotations

This permits later analysis of skill revisions rather than assigning permanent quality to a skill name.

### Skill miss detection

A future evaluator can identify cases where:

```text
skill was available
+ harness did not expose it
+ model later re-requested equivalent capability or failed
```

This is primarily a harness-selection signal.

### Skill improvement proposals

A future analyzer can correlate a skill revision with:

- repeated retries
- user corrections
- post-skill failures
- successful recovery sequences
- evaluation annotations

It can then propose a patch without v0 having to mutate skills automatically.

### Skill creation opportunities

Across runs, repeated successful procedural sequences where no appropriate skill was available can become candidates for a new skill.

v0 does not need to detect the pattern; it needs consistent task/capability/action/outcome evidence so the pattern can be discovered later.

## Context and capability-surface metrics

v0 should enable later derivation of:

- context token reduction
- token composition by category
- irrelevant capability exposure
- capability miss rate
- model re-request rate
- tool/MCP unnecessary activation
- tool-output compression ratio
- planner latency overhead
- total task latency and retry count

Again, metric formulas are analysis-layer concerns rather than canonical event fields.

## Implementation boundaries for the first telemetry PR after this design

The first implementation should remain small:

1. define core ids and the versioned event envelope
2. define the minimum typed v0 event payloads
3. implement a telemetry emitter that assigns per-run sequence numbers
4. implement capture-policy/redaction interfaces with `metadata_only` as the safe default
5. implement JSONL sink
6. add serialization and ordering tests
7. instrument only the first v0 pipeline events that already exist

Do not implement analytics, model profiling, automatic skill evaluation, SQLite, remote ingestion, or training-data generation in the same change.

## Validation requirements

The implementation of this design should eventually prove:

- events serialize deterministically enough for the declared compatibility policy
- sequence numbers are monotonic within a run under concurrent emission
- causal references survive serialization/deserialization
- unknown optional fields can be ignored
- unknown event types do not corrupt stream processing
- `metadata_only` does not persist configured content bodies
- redaction occurs before sink invocation
- secret fixtures never appear in sink output
- absent measurements remain absent rather than becoming zero
- skill/model revision metadata round-trips when present

## Open implementation questions

These are intentionally deferred from the durable v0 semantics:

- exact Rust enum/trait layout
- concrete UUIDv7 crate
- synchronous vs asynchronous sink API
- batching and flush/fsync policy
- telemetry failure/backpressure policy
- exact redactor interface and secret classifiers
- configuration file/environment-variable format
- which adapter lifecycle hooks emit each event
- provider-specific model metadata extensions

These questions should be resolved in implementation issues or subsequent Decision Records when they become durable constraints.
