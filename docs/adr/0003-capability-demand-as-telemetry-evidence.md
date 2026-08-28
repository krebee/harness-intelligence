# DR-0003: Treat capability demand as first-class telemetry evidence

- Status: proposed
- Date: 2026-08-28
- Governing issue: #7
- Supersedes: none
- Superseded by: none

## Context

Harness Intelligence is intended to shape the capability surface around a replaceable main agent. Future capability kinds include skills, tools, MCP servers, memory, and other runtime-provided resources.

The existing v0 telemetry design records capability availability, planning/exposure, recognized requests, execution, and outcomes. That is sufficient to answer questions such as whether a tool was exposed and whether a valid call succeeded.

It is not sufficient to represent an important class of runtime evidence:

> The main agent wanted or needed a capability/function, but the harness did not expose it or no matching capability existed.

This signal matters for several later features:

- detecting harness pruning/selection misses
- understanding model/capability compatibility
- distinguishing ignored capabilities from unavailable ones
- identifying repeated unmet needs that may justify a new skill/tool/MCP integration
- building future capability utility/reputation models
- adjusting recommendation/exposure policy by model, task, project, and context budget

A naive solution would add a mutable tier or popularity score directly to each capability. That would lose the observations behind the score and could create a self-reinforcing exposure loop: capabilities that are exposed more often are used more often, which then makes them appear more valuable and causes still more exposure.

A durable decision is needed before telemetry implementation begins.

## Decision

Harness Intelligence will treat **capability demand as first-class canonical telemetry evidence**, while keeping capability tiers, reputation, utility, ranking, and recommendation policy as derived state.

### 1. Demand is distinct from a recognized request

The canonical telemetry model may record `capability.demand.observed` independently from `capability.requested`.

`capability.requested` means the runtime recognized a concrete capability action/load request.

`capability.demand.observed` means an observable signal indicates that an actor needs a capability or capability-like function, regardless of whether a valid executable request was possible.

This distinction makes it possible to represent demand for:

- a capability that was exposed
- a capability that existed but was not exposed
- a capability that was unavailable
- functionality for which no capability currently exists

### 2. Demand must have provenance

Demand must not be silently inferred and stored as fact.

When demand is inferred rather than directly represented by a structured runtime signal, telemetry must record evidence/provenance and confidence where applicable.

v0 does not require natural-language intent extraction and never requires private chain-of-thought content.

### 3. Capability lifecycle stages remain separate

The canonical event stream must preserve distinct evidence for:

```text
available
candidate
exposed
demand observed
requested
loaded / invoked
success / failure
outcome / evaluation
```

Not every capability uses every stage, but canonical telemetry must not replace these observations with one mutable `used`, `quality`, or `tier` field.

### 4. Exposure bias must remain measurable

Future analysis must be able to distinguish a capability that is popular because it is frequently exposed from one that performs well when given comparable opportunities.

Availability, candidacy, exposure, demand, request, execution, and outcome counts therefore remain independently derivable from raw events.

### 5. Reputation and tier are derived state

The following are explicitly not canonical v0 telemetry fields:

- tier / level
- reputation
- trust
- utility
- demand score
- recommendation priority
- exploration/exploitation weight
- automatic promotion/demotion state

A future `CapabilityProfile` may derive these values from canonical events and may condition them on model, task, environment, capability revision, cost, latency, and time window.

### 6. Missing capability demand is representable

A demand event does not require an existing `capability_id`.

When no matching capability exists, telemetry may carry a normalized non-content need/category identifier and an `unavailable` or `unresolved` resolution state.

Repeated unresolved/unavailable demand may later become evidence for proposing a new skill, tool, MCP integration, memory provider, or other capability.

Automatic creation or mutation is not part of this decision.

## Alternatives considered

### Use only `capability.requested`

This is insufficient because a main agent may be unable to issue a valid request for a capability that was hidden, unavailable, unknown to the current tool parser, or nonexistent.

It also conflates semantic need with executable request syntax.

### Infer unmet demand only from failures

Failure is weak evidence. A task can fail for many reasons unrelated to a missing capability, and a main agent can express a need without the overall task failing.

Demand should be independently observable when the runtime has a reliable signal.

### Store a tier/reputation counter directly on each capability

This is simple operationally but makes the score's provenance unclear, loses historical evidence, is difficult to recompute after policy changes, and amplifies exposure bias.

Mutable profiles may exist later as caches/derived state, but they are not canonical evidence.

### Automatically parse all model text for capability intent

This could produce useful signals, but making it mandatory in v0 would introduce an inference subsystem before the runtime baseline exists and could incorrectly treat model text as ground truth.

Natural-language demand extraction is deferred. Later evaluators can add append-only inferred demand with explicit provenance.

## Consequences

### Benefits

- Harness selection misses can be distinguished from missing-capability gaps.
- Skill/tool/MCP/memory demand can share one evidence model.
- Future capability ranking can account for opportunity/exposure bias.
- Capability revisions can be compared using demand/request/use/outcome evidence.
- Repeated unmet demand can support future skill/tool creation proposals.
- Derived tiers/reputation algorithms can change without rewriting historical telemetry.

### Costs

- Capability lifecycle telemetry becomes more detailed.
- Demand events require careful provenance semantics.
- Analysis must guard against double-counting demand that is also represented by a request.
- Unresolved needs require a normalized representation that does not depend on retaining raw content.

### Risks

- Weak intent classifiers could generate noisy inferred demand.
- Different adapters may observe demand at different fidelity.
- A future recommendation system could still create feedback loops if it ignores exposure/opportunity data.
- Treating demand as equivalent to correctness would be a semantic error; demand only records need, not whether satisfying it was the right decision.

## Validation

This decision is validated if future telemetry/runtime code can demonstrate that:

- demand for an available-but-unexposed capability is distinguishable from demand for an unavailable capability
- a recognized executable request remains distinguishable from demand
- demand can be represented without storing hidden reasoning or raw model/user text
- inferred demand records provenance/confidence rather than masquerading as direct observation
- future analyses can compute separate available/candidate/exposed/demand/request/execution/outcome rates
- a future tier/reputation model can be recomputed from canonical events
- capability demand can be analyzed by capability revision and model/task/runtime dimensions

## Follow-up

- Extend `docs/telemetry-v0.md` with `capability.demand.observed` and exposure-bias requirements in #7.
- Include direct demand representation in the first Rust telemetry payload design where the adapter/runtime can observe it reliably.
- Do not block the first executable v0 baseline on free-form intent extraction.
- Define any future `CapabilityProfile`, ranking algorithm, tier system, or exploration policy in separate issues/Decision Records after real telemetry exists.
- Define automatic capability/skill creation and mutation separately from demand observation.
