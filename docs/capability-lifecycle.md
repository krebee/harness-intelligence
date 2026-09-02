# Evidence-backed capability lifecycle

- Status: proposed
- Governing issue: #9

## Purpose

Harness Intelligence should eventually manage more than a static list of skills, tools, MCP servers, memories, and other resources.

It should be able to observe how those capabilities are demanded and used, preserve the evidence behind durable decisions, and change what is exposed to the main agent without turning opaque scores into the source of truth.

This document defines the project-level design direction for that lifecycle.

The core idea is:

```text
Telemetry / Evaluation
        ↓
      Evidence
        ↓
Candidate / durable capability state
        ↓
Derived CapabilityProfile
        ↓
Exposure / recommendation policy
        ↓
Main Agent
        ↓
Outcome
        ↓
New evidence
```

The design applies to memory, skills, tools, MCP servers, and future capability kinds. It is intentionally broader than any one storage or retrieval implementation.

## Relationship to v0

This document is a forward design constraint, not a requirement that the first v0 runtime implement the full lifecycle.

The v0 runtime should remain small:

```text
HarnessSnapshot
    -> HarnessBrain
    -> HarnessPlan
    -> Adapter
```

The v0 telemetry contract should preserve the evidence needed for later lifecycle logic. The first executable baseline does not need automatic promotion, reputation scoring, skill creation, memory mutation, or lifecycle automation.

## Design principles

### 1. Evidence is canonical; profiles are derived

Harness Intelligence should preserve observable facts before it computes opinions about them.

Examples of evidence include:

- capability availability
- candidate selection
- actual exposure
- observed demand
- recognized requests
- loads/activations
- executions
- success/failure
- latency/cost
- user corrections
- evaluator annotations
- task outcomes
- contradictions
- test results

These are the inputs to later analysis.

A mutable field such as:

```text
tier = trusted
```

must not be the only retained explanation for why the capability is trusted.

A future tier, reputation, utility, trust, demand, freshness, or compatibility score should be recomputable from evidence where practical.

### 2. Durable capability changes have provenance

When Harness Intelligence creates, promotes, revises, supersedes, deprecates, or archives durable state, the decision should carry provenance.

A future contributor or runtime should be able to answer:

```text
Why does this memory exist?
Why was this skill created?
Why is this capability trusted?
Why was revision B preferred over revision A?
Why was this capability deprecated?
Which runs motivated this proposal?
```

Provenance may reference:

- telemetry event ids
- run/task ids
- evaluation annotations
- validation/test evidence
- user-confirmed decisions
- prior capability revisions
- explicit policy decisions

The exact storage representation is deferred.

### 3. Promotion and persistence are gated decisions

Observation is not the same as durable knowledge.

A model statement, one successful execution, one user request, or one inferred pattern should normally enter the system as evidence or a candidate rather than becoming trusted durable state immediately.

Conceptually:

```text
observation
    ↓
candidate
    ↓
evidence collection
    ↓
validation / promotion gate
    ↓
accepted durable state
```

A future promotion gate may use:

- explicit human approval
- deterministic rules
- repeated observations
- automated tests
- evaluator agreement
- source confidence
- contradiction checks
- age/freshness checks
- combinations of the above

The specific gate is a later policy decision.

Early implementations should prefer proposals and explicit approval over autonomous mutation.

### 4. Quality and exposure are different dimensions

A durable capability can be highly trusted and still be inappropriate to expose on a specific turn.

Examples:

```text
trusted memory + irrelevant task       -> do not expose
trusted tool + high latency/cost       -> expose only when justified
useful skill + incompatible main model -> adapt or hide
fresh low-confidence memory            -> maybe retrieve, do not assert strongly
```

Therefore:

```text
quality / trust / confidence / utility
```

must remain conceptually distinct from:

```text
expose / recommend / activate / preload / hide
```

Exposure is a runtime policy decision conditioned on the current task and environment.

### 5. Exposure policy should use multiple signals

A future exposure decision may consider:

- task relevance
- context budget
- model compatibility
- trust/confidence
- utility history
- observed demand
- freshness
- latency
- monetary/compute cost
- risk/permissions
- capability revision
- redundancy with other capabilities
- current runtime state

No single score is expected to dominate every capability kind.

This is important because a frequently used capability may only be frequently used because it was frequently exposed.

### 6. Exposure bias must remain measurable

The runtime should retain separate opportunities and lifecycle stages so future ranking systems can distinguish popularity from quality.

Conceptually:

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

These stages must not be collapsed into one mutable `usage_count` or `tier` as the canonical record.

A capability that was exposed 1,000 times and used 100 times is not directly comparable to one exposed 20 times and used successfully 18 times without considering opportunity and task mix.

### 7. Revisions are first-class

A capability is not a timeless name.

Memory contents change. Skills are edited. Tool schemas change. MCP servers are upgraded. Runtime permissions and wrappers evolve.

Evidence should therefore be attributable to a specific capability revision when possible.

Conceptually:

```text
CapabilityIdentity
  capability_id
  kind

CapabilityRevision
  revision
  content/schema/config identity
  created_from / supersedes
  provenance
```

A materially changed revision should not silently inherit all evidence from the previous revision as if nothing changed.

Historical revisions should remain identifiable even after they are superseded.

## Four-layer model

The lifecycle is easiest to reason about when separated into four layers.

## Layer 1: Evidence

Evidence represents observations that happened or evaluations attached to those observations.

Examples:

```text
TelemetryEvent
EvaluationAnnotation
ValidationResult
UserConfirmation
ContradictionEvidence
```

Evidence should be append-oriented where practical. Later interpretation should not rewrite the original runtime observation.

Evidence answers:

> What happened, and what supports this claim?

## Layer 2: Durable capability state

Durable state represents the current accepted project/runtime knowledge about a capability and its revision history.

Examples might later include:

```text
candidate
accepted
active
superseded
deprecated
archived
```

This document does not fix the final enum or state machine.

Durable state answers:

> What capability/revision currently exists, and what lifecycle decision has been made about it?

State transitions should reference evidence and policy decisions.

## Layer 3: Derived CapabilityProfile

A `CapabilityProfile` is an analysis/product of evidence and durable state.

Possible dimensions include:

```text
trust
confidence
utility
demand
freshness
latency
cost
risk
model compatibility
task/project affinity
```

Profiles may be cached and updated incrementally for performance, but they remain derived state.

A profile algorithm can change without invalidating historical telemetry.

Profiles answer:

> Given the evidence we currently have, what do we believe about this capability?

## Layer 4: Exposure/recommendation policy

Exposure policy decides what the main agent sees or can use now.

Possible actions include:

```text
hide
make discoverable
recommend
expose
preload
activate
```

The exact action vocabulary depends on capability kind and adapter support.

Exposure answers:

> Given this task, model, budget, and profile, what should be available now?

The same CapabilityProfile may produce different exposure decisions for different models or tasks.

## Provenance model

The project should eventually support evidence references from durable capability decisions.

A conceptual representation might look like:

```text
CapabilityRevision
  id
  capability_id
  revision
  lifecycle_state
  provenance_refs[]
  supersedes?
```

A provenance reference might point to:

```text
TelemetryEventId
EvaluationAnnotationId
RunId
ValidationArtifactId
UserDecisionId
```

The exact Rust type and storage schema are intentionally deferred.

### Provenance is not necessarily raw content

Evidence-backed does not mean storing every prompt, model output, or tool result body.

The telemetry design defaults to metadata-only capture. A provenance link can point to structured evidence and identifiers without requiring raw private content.

When content is necessary for a promotion decision, it remains subject to explicit capture/redaction policy.

## Candidate and promotion lifecycle

Candidate state is useful because it prevents premature persistence.

Examples of candidates include:

- a fact that may deserve project memory
- a recurring procedure that may deserve a skill
- an unmet need that may deserve a new tool/MCP integration
- a revised skill generated from repeated recovery evidence
- a stale capability that may deserve deprecation

Conceptually:

```text
runtime evidence
      ↓
candidate detected
      ↓
collect supporting / contradicting evidence
      ↓
validation
      ↓
proposal
      ↓
accept / reject / defer
      ↓
durable state transition
```

Not every capability kind needs every step.

### Early project policy

Until automated evaluation is proven reliable:

- proposals should be preferred over silent mutation
- destructive lifecycle changes should be conservative
- provenance should be preserved even for rejected/deferred proposals when useful
- automated promotion should not be required for v0

## Lifecycle transitions

The final lifecycle state machine is intentionally not fixed, but useful semantic transitions include:

```text
propose
accept
activate
revise
supersede
deprecate
archive
restore
reject
```

These are actions/decisions, not necessarily the final Rust enum.

Important distinction:

```text
deprecated != hidden on this turn
```

Deprecation is durable lifecycle state. Hiding is a current exposure decision.

Similarly:

```text
trusted != always exposed
fresh != trusted
popular != useful
requested != correct
successful execution != successful task
```

These distinctions should remain explicit throughout the design.

## Capability-specific examples

## Memory

A memory subsystem can use this model as:

```text
runtime observation
    ↓
memory candidate
    ↓
source / contradiction validation
    ↓
accepted memory revision
    ↓
MemoryProfile
    ↓
retrieval/exposure decision
```

Possible evidence includes:

- user-confirmed facts
- repository/tool observations
- repeated successful use
- contradiction/correction events
- age/freshness signals

A memory may be trusted but not retrieved for an unrelated task.

A memory may also remain useful but be superseded by a newer revision.

## Skill

A skill subsystem can use the lifecycle as:

```text
repeated successful procedure
    ↓
skill creation candidate
    ↓
evidence-backed proposal
    ↓
validation/tests/review
    ↓
accepted skill revision
    ↓
usage/outcome evidence
    ↓
improvement or supersession proposal
```

A future skill proposal should be able to answer:

> Which runs showed that this procedure was repeatedly needed or successful?

Likewise, an improvement proposal should reference evidence such as retries, failures, recoveries, or user corrections associated with the current revision.

## Tool

A tool may already exist outside Harness Intelligence, so the durable state may describe the harness-facing integration rather than tool ownership.

Evidence may include:

- repeated demand
- valid/invalid call rates
- execution reliability
- latency/cost
- model-specific argument-generation failures
- permission/risk observations

A reliable but expensive tool may be highly trusted while remaining conditionally exposed.

## MCP server

An MCP integration may have additional lifecycle dimensions such as discovery, connection, activation, and permission boundaries.

The same principles still apply:

- demand evidence is distinct from activation
- activation frequency is not quality
- server/version identity matters
- cost/risk may influence exposure independently from trust

## Missing capabilities

The lifecycle should also support the absence of a current capability.

Repeated demand with no matching capability can accumulate evidence for a future proposal:

```text
unresolved demand
    ↓
repeated pattern across runs
    ↓
capability-gap candidate
    ↓
proposal: create/integrate capability
```

The proposal itself should reference the motivating evidence.

## Feedback loops and exploration

A future recommendation system can create self-reinforcing loops:

```text
high score
  -> more exposure
  -> more use
  -> higher score
```

The evidence model reduces this risk by preserving opportunity denominators, but policy must still account for exploration.

Possible future approaches include:

- explicit exploration budget
- confidence intervals
- time-decayed evidence
- model/task stratification
- controlled experiments
- contextual bandits

None of these is selected by this design.

The durable requirement is only that canonical evidence remains rich enough to evaluate such policies later.

## Relationship to HarnessBrain

`HarnessBrain` should eventually consume a snapshot that may include relevant capability state/profile information and return an exposure plan.

Conceptually:

```text
HarnessSnapshot
  task/runtime/model state
  capability candidates
  capability profiles
  context budget
      ↓
HarnessBrain
      ↓
HarnessPlan
  expose / hide / recommend / activate
```

The core `HarnessBrain` contract should not itself become the persistent capability database.

Lifecycle storage/profile computation and runtime planning are separate concerns even if one implementation package hosts both initially.

## Relationship to telemetry

The v0 telemetry design remains the primary evidence source for runtime behavior.

Important telemetry concepts include:

- capability identity/revision
- availability/candidacy/exposure
- observed demand
- recognized request
- execution outcome
- task outcome/evaluation
- model/runtime identity

Later lifecycle events may need their own canonical event families, for example:

```text
capability.candidate.created
capability.revision.proposed
capability.revision.accepted
capability.revision.superseded
capability.deprecated
```

Those event names are illustrative only. A later implementation/design issue should decide exact schemas.

## Evaluation questions this design enables

Future analysis should be able to answer questions such as:

- Which capabilities are repeatedly demanded but missing?
- Which capabilities are exposed often but rarely demanded?
- Which revisions improve task outcomes?
- Which memories are repeatedly contradicted or corrected?
- Which skills reduce retries or recovery steps?
- Which tools are reliable for one model but error-prone for another?
- Which expensive capabilities are rarely necessary?
- Which capability creation proposals are supported by repeated evidence?
- Which promotion decisions later proved wrong?
- Does a new exposure policy improve task success without growing context cost?

## Non-goals

This design intentionally does not choose:

- exact tier names or number of levels
- reputation formula
- trust/utility/demand weights
- promotion/demotion thresholds
- vector database
- retrieval algorithm
- embedding model
- memory storage backend
- automatic skill-generation model
- automatic mutation policy
- contextual-bandit algorithm
- exact `CapabilityProfile` Rust schema
- exact lifecycle event payloads
- exact human-approval UI

These should be decided after runtime evidence and concrete implementation needs exist.

## Implementation sequencing

The full lifecycle should not block the first v0 runtime.

Recommended order:

```text
1. v0 telemetry + deterministic baseline
2. capability identity/revision plumbing
3. real usage/demand evidence collection
4. offline derived analyses
5. explicit CapabilityProfile prototype
6. proposal-only lifecycle actions
7. gated promotion/revision flows
8. adaptive exposure policy
9. carefully evaluated automation
```

The system should earn automation by demonstrating that its evidence and evaluation are reliable.

## Validation criteria

This design is successful if future implementations can demonstrate that:

- durable capability changes reference explicit evidence/provenance
- historical revisions remain attributable
- profile values can be recomputed from retained evidence
- profile/quality can change without rewriting historical observations
- high trust does not imply unconditional exposure
- exposure policy can differ by task/model/context budget
- capability creation/improvement proposals can explain their motivating evidence
- rejected/superseded decisions do not erase prior evidence
- the same lifecycle concepts can support memory, skills, tools, MCP, and future capability kinds
