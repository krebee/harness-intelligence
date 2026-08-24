# Decision Records

This directory stores durable records of significant decisions made for Harness Intelligence.

Although the directory uses the familiar `adr` name, records here are **not limited to software architecture**. Use them for any decision that future contributors or coding agents may need to understand, including:

- architecture and runtime boundaries
- language, dependency, and tooling choices
- telemetry semantics and retention policy
- security and privacy policy
- repository and development workflow
- compatibility and versioning policy
- release and packaging policy
- product or scope decisions that materially constrain implementation

Do not create a record for every minor implementation detail. A decision record is appropriate when the decision is durable, has meaningful alternatives or tradeoffs, or is likely to be revisited later.

## Workflow

The preferred flow is:

```text
Issue / discussion
  -> decision
  -> decision record
  -> implementation PR
```

A record may be added in the same PR as implementation when the decision is already settled in the governing issue. Do not use an implementation PR to silently introduce a significant decision that has not been discussed.

## Naming

Use monotonically increasing four-digit identifiers:

```text
0001-use-rust-for-runtime.md
0002-telemetry-event-storage.md
```

The numeric identifier is stable. Do not renumber existing records.

## Status

Each record should use one of these statuses:

- `proposed` — under active discussion
- `accepted` — current decision
- `superseded` — replaced by a later decision record
- `deprecated` — intentionally no longer recommended, without a direct replacement

When a decision changes, prefer adding a new record and marking the previous record as superseded rather than rewriting history.

## Template

Start from [`0000-template.md`](0000-template.md).
