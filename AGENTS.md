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

## Validation

Before considering Rust changes complete, run:

```bash
cargo fmt --check
cargo check --all-targets
cargo clippy --all-targets -- -D warnings
cargo test --all
```

If the workspace is not yet initialized, do not add placeholder Rust code solely to satisfy these commands.

## v0 scope discipline

Until the v0 design explicitly includes them, do not implement:

- local LLM inference
- automatic project memory management
- MCP lazy activation
- context garbage collection
- fine-tuning or training pipelines

Follow the active issue and `docs/architecture.md` for current scope and architectural decisions.
