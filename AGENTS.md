# AGENTS.md

## Project intent

Harness Intelligence is a runtime intelligence layer for agent harnesses. It should improve the execution environment around a user-selected main agent rather than replace the main agent itself.

## Development principles

- Use Rust for the runtime implementation unless an explicit architecture decision changes this.
- Keep changes small and scoped to the active issue.
- Do not introduce speculative abstractions or implement future phases early.
- Preserve clear boundaries between observation (`HarnessSnapshot`), decision (`HarnessBrain`), and execution plan (`HarnessPlan`).
- Treat telemetry as a first-class concern. New runtime behavior should be observable enough to evaluate later.
- Prefer explicit types and state transitions where they prevent invalid runtime states.
- Keep harness-specific integration behind adapters so the core is not coupled to a single agent harness.
- Document unresolved architectural questions instead of silently locking them into implementation details.

## Git workflow

- Do not commit directly to `main`.
- Use a short-lived branch for every substantive change.
- Use one of these branch prefixes where applicable: `feat/`, `fix/`, `docs/`, `refactor/`, `test/`, `chore/`.
- Keep commits atomic: one logical change per commit.
- Use Conventional Commits for commit messages.
- Do not mix unrelated refactoring with feature or fix work.
- Open pull requests as drafts while implementation is in progress.
- Prefer squash merge so `main` retains a concise logical history.
- Architectural changes require an issue before implementation begins.
- Small typo or trivial documentation fixes may omit a dedicated issue when no architectural or behavioral decision is involved.

See `docs/development.md` for the detailed repository workflow.

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

Follow the active issue and `docs/architecture.md` for current scope and architectural decisions.
