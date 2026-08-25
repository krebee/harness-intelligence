# AGENTS.md

## Project intent

Harness Intelligence is a runtime intelligence layer for agent harnesses. It should improve the execution environment around a user-selected main agent rather than replace the main agent itself.

## Development principles

- Use Rust for the runtime implementation unless an explicit project decision changes this.
- Keep changes small and scoped to the active issue.
- Do not introduce speculative abstractions or implement future phases early.
- Preserve clear boundaries between observation (`HarnessSnapshot`), decision (`HarnessBrain`), and execution plan (`HarnessPlan`).
- Treat telemetry as a first-class concern. New runtime behavior should be observable enough to evaluate later.
- Prefer explicit types and state transitions where they prevent invalid runtime states.
- Keep harness-specific integration behind adapters so the core is not coupled to a single agent harness.
- Document unresolved significant decisions instead of silently locking them into implementation details.

## Git workflow

- Do not commit directly to `main`.
- Use a short-lived branch for every substantive change.
- Use one of these branch prefixes where applicable: `feat/`, `fix/`, `docs/`, `refactor/`, `test/`, `chore/`.
- Keep commits atomic: one logical change per commit.
- Use Conventional Commits for commit messages.
- Do not mix unrelated refactoring with feature or fix work.
- Open pull requests as drafts while implementation is in progress.
- Prefer squash merge so `main` retains a concise logical history.
- Significant durable project decisions require an issue or equivalent discussion before implementation begins.
- Once settled, durable decisions should be recorded under `docs/adr/` when future contributors or agents may need the rationale.
- Decision records are not limited to software architecture. They may cover runtime design, tooling, telemetry, security, process, compatibility, release policy, or other durable constraints.
- Small typo or trivial documentation fixes may omit a dedicated issue when no durable project decision or behavioral change is involved.

See `docs/development.md` for the detailed repository workflow and `docs/adr/README.md` for the decision record policy.

## Validation

Before considering Rust changes complete, run:

```bash
cargo fmt --check
cargo check --all-targets
cargo clippy --all-targets -- -D warnings
cargo test --all
```

If the workspace is not yet initialized, do not add placeholder Rust code solely to satisfy these commands.

Do not commit Rust changes while required validation is failing unless the active issue explicitly documents an intentional failing state.

## v0 scope discipline

Until the v0 design explicitly includes them, do not implement:

- local LLM inference
- automatic project memory management
- MCP lazy activation
- context garbage collection
- fine-tuning or training pipelines

Follow the active issue, accepted decision records under `docs/adr/`, and `docs/architecture.md` for current scope and project decisions.
