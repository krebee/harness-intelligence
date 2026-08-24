# Development workflow

This document defines the repository workflow for Harness Intelligence. The goal is to keep changes reviewable, reversible, and easy for both humans and coding agents to reason about while the v0 architecture is still evolving.

## Branching strategy

Use a GitHub Flow style workflow built around `main` and short-lived topic branches.

`main` is the integration branch and should remain in a releasable, validated state. Do not commit directly to `main`.

Use these branch prefixes where applicable:

- `feat/<topic>` — new behavior or capability
- `fix/<topic>` — bug fixes
- `docs/<topic>` — documentation-only changes
- `refactor/<topic>` — behavior-preserving restructuring
- `test/<topic>` — test-only changes
- `chore/<topic>` — repository maintenance, CI, dependencies, tooling

Examples:

```text
feat/telemetry-events
docs/v0-design
fix/event-ordering
chore/ci
```

Do not maintain a long-lived `develop` branch. Changes should flow from short-lived branches into `main` through pull requests.

## Issues

Create an issue before implementation when a change introduces or materially changes architecture, runtime behavior, public interfaces, telemetry semantics, or v0 scope.

The issue should establish the goal, scope, non-goals, acceptance criteria, and important unresolved questions before implementation begins.

A dedicated issue is optional for trivial corrections such as typos or narrowly scoped documentation fixes that introduce no architectural or behavioral decision.

## Commits

Use Conventional Commits.

Common types:

```text
feat: add telemetry event envelope
fix: preserve event ordering during flush
docs: define v0 architecture goals
refactor: separate capability state from metadata
test: add telemetry serialization coverage
chore: configure CI checks
```

Scopes are optional and should be used only when they make the change easier to identify. Do not force scopes before package or crate boundaries are stable.

Examples:

```text
feat(telemetry): add run lifecycle events
fix(adapter): propagate cancellation
docs(architecture): clarify v0 non-goals
```

Keep commits atomic. A commit should represent one logical change and should not combine unrelated feature work, fixes, refactoring, or formatting churn.

Temporary or exploratory commits are acceptable on a topic branch while work is in progress, but the pull request should be suitable for squash merge into one coherent change on `main`.

## Pull requests

All substantive changes should reach `main` through a pull request.

Open the pull request as a draft while implementation is incomplete. The pull request should reference the governing issue when one exists and should explain:

- what changed
- why it changed
- what is intentionally out of scope
- how the change was validated
- any unresolved risks or follow-up work

Avoid combining unrelated changes in one pull request. Architectural decisions should be discussed in an issue before being embedded in implementation.

Prefer squash merge. The squash commit should use a Conventional Commit style title and describe the logical change represented by the pull request.

## Required validation

Once the Rust workspace exists, Rust changes are not complete until these checks pass:

```bash
cargo fmt --check
cargo check --all-targets
cargo clippy --all-targets -- -D warnings
cargo test --all
```

These checks should eventually be enforced by CI and branch protection on `main`.

Before the Rust workspace is initialized, do not add placeholder code solely to make the commands runnable.

## Agent workflow

Coding agents should follow this sequence for substantive work:

```text
Issue
  -> short-lived branch
  -> atomic commits
  -> Draft PR
  -> required validation
  -> review / design feedback
  -> Ready for review
  -> squash merge
```

Agents should not silently broaden scope, introduce speculative abstractions, or make architectural decisions that are not supported by the active issue or repository documentation.

When implementation reveals a missing architectural decision, record it in the issue or documentation rather than hiding the decision inside code.
