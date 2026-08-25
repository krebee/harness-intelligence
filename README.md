# Harness Intelligence

Harness Intelligence is an experimental runtime intelligence layer for agent harnesses.

The project explores how a harness can improve a main agent without replacing it by deciding, at runtime, what context and capabilities should be exposed. The long-term scope includes context planning, skill and tool selection, MCP activation, project memory, telemetry, failure monitoring, and other runtime support.

The main agent remains user-selectable. Harness Intelligence is responsible for shaping the execution environment around that agent.

## Current status

The repository is in the v0 design phase. The first target is a minimal pipeline around `HarnessSnapshot`, `HarnessBrain`, `HarnessPlan`, telemetry, and a harness adapter. Detailed APIs and crate boundaries are intentionally not finalized yet.

See [`docs/architecture.md`](docs/architecture.md) for the current architecture notes.
