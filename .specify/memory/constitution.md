<!--
Sync Impact Report
- Version change: (unversioned template) → 1.0.0
- Bump rationale: Initial ratification — placeholder template populated with concrete
  principles for the RMQTT broker. First adopted version, hence MAJOR 1.0.0.
- Modified principles: none (initial definition)
  - [PRINCIPLE_1_NAME] → I. MQTT Protocol Conformance (NON-NEGOTIABLE)
  - [PRINCIPLE_2_NAME] → II. Memory-Safe, Idiomatic Async Rust
  - [PRINCIPLE_3_NAME] → III. Modular Plugin Architecture
  - [PRINCIPLE_4_NAME] → IV. Test & Interoperability Discipline
  - [PRINCIPLE_5_NAME] → V. Performance & Horizontal Scalability
- Added sections: Technology & Quality Standards; Development Workflow
- Removed sections: none
- Templates requiring updates:
  - .specify/templates/plan-template.md ✅ aligned (Constitution Check references this file generically)
  - .specify/templates/spec-template.md ✅ aligned (no constitution-specific gates needed)
  - .specify/templates/tasks-template.md ✅ aligned (task categories cover protocol/test/perf work)
- Follow-up TODOs: none
-->

# RMQTT Constitution

## Core Principles

### I. MQTT Protocol Conformance (NON-NEGOTIABLE)

RMQTT MUST implement the MQTT v3.1, v3.1.1, and v5.0 protocols correctly and completely
for every supported feature (QoS 0/1/2, retained messages, last-will, sessions, shared
subscriptions, topic aliases, flow control). Any change touching packet encoding/decoding,
session state, or message delivery MUST keep the Paho interoperability suites
(`client_test.py`, `client_test5.py`) passing. Spec-violating behavior is a defect, not a
feature — broker-specific extensions (`$share`, `$exclusive`, `$limit`, `$delayed`) MUST be
additive and never break standard-conformant clients.

Rationale: A broker that silently diverges from the spec breaks heterogeneous client
fleets in ways that are expensive to diagnose. Conformance is the product's core contract.

### II. Memory-Safe, Idiomatic Async Rust

Code MUST be 100% safe Rust. `unsafe` is forbidden unless accompanied by a comment proving
the invariants it upholds and a reviewer sign-off. All I/O and concurrency MUST use the
async/tokio model already in the workspace. Fallible operations MUST return `Result` with
`thiserror`/`anyhow`; panics MUST NOT be reachable from connection or message-handling paths
(the release profile uses `panic = "abort"`). `cargo fmt`, `cargo clippy`, and `cargo build`
MUST be clean before merge.

Rationale: The broker handles millions of untrusted concurrent connections; memory-safety
bugs and panics translate directly into crashes and security incidents.

### III. Modular Plugin Architecture

New functionality SHOULD be delivered as a plugin crate under `rmqtt-plugins/` or a focused
workspace crate, not by growing the core. The core (`rmqtt`, `rmqtt-codec`, `rmqtt-net`,
`rmqtt-conf`) MUST stay protocol- and transport-focused and MUST NOT depend on individual
plugins. Each plugin MUST be self-contained, independently buildable, documented with a
README, and integrate only through the published hook/extension interfaces.

Rationale: A 26-plugin ecosystem stays maintainable only if features are isolated and the
core remains small and stable.

### IV. Test & Interoperability Discipline

Every behavioral change MUST ship with tests at the appropriate level: unit tests in the
owning crate, integration/functional/stress/chaos coverage via the `rmqtt-test` harness for
end-to-end behavior, and Paho interoperability validation for protocol-affecting changes.
Bug fixes MUST add a regression test that fails before the fix. A change is not "done" until
its tests are green and the reasoning behind them is recorded in the PR.

Rationale: Concurrency, clustering, and protocol edge cases are not reliably caught by
inspection; executable tests are the only durable guard against regressions.

### V. Performance & Horizontal Scalability

RMQTT MUST sustain high concurrency (target: ~1M clients per node) and high throughput
without per-connection unbounded resource growth. Changes on hot paths (connect handshake,
publish/subscribe routing, codec) MUST NOT introduce measurable throughput or latency
regressions; when in doubt, benchmark before and after. Features MUST work in both single-node
and Raft cluster modes, and MUST honor configured back-pressure controls (rate limits,
inflight windows, queues).

Rationale: Scalability is a headline guarantee of the project; a regression here breaks
production deployments that depend on the documented benchmarks.

## Technology & Quality Standards

- Language: Rust, edition 2021, minimum `rust-version` as declared in the workspace
  `Cargo.toml` (currently 1.89.0). Bumping the MSRV is a deliberate, documented decision.
- Runtime/stack: tokio async runtime; rustls/tokio-rustls for TLS; quinn for MQTT-over-QUIC;
  serde for serialization. Prefer existing workspace dependencies over adding new ones.
- Workspace layout: changes MUST respect crate boundaries and the `[patch.crates-io]` path
  mappings; cross-crate coupling that violates the core/plugin separation is rejected.
- Licensing: all source MUST remain compatible with the project's dual `MIT OR Apache-2.0`
  license; new dependencies MUST be license-compatible.
- Configuration & observability: user-facing behavior MUST be configurable through the
  established `rmqtt-conf`/TOML mechanism with sane defaults, and significant operations MUST
  emit structured `tracing` logs and expose metrics/stats where applicable.

## Development Workflow

- Branching & review: work happens on feature branches; merges go through pull requests.
  Every PR MUST pass CI (fmt, clippy, build, tests) and receive maintainer review.
- Compliance gate: reviewers MUST verify that a change upholds the Core Principles. A PR that
  violates a principle is blocked until corrected or until the violation is justified in the
  plan's Complexity Tracking with a rejected simpler alternative.
- Documentation: changes affecting configuration, APIs, protocol behavior, or plugins MUST
  update the relevant README/`docs/` pages and the CHANGELOG in the same PR.
- Versioning: workspace crates follow semantic versioning; protocol- or API-breaking changes
  require a major-version bump and migration notes.

## Governance

This constitution supersedes other development practices where they conflict. Amendments MUST
be proposed via pull request, describe the rationale and migration impact, and be approved by
project maintainers before taking effect.

Versioning of this document follows semantic versioning:
- MAJOR: backward-incompatible governance changes or removal/redefinition of a principle.
- MINOR: a new principle/section or materially expanded guidance.
- PATCH: clarifications and wording fixes with no semantic change.

Compliance is reviewed on every pull request (see Development Workflow). Complexity that
appears to violate a principle MUST be justified in the implementation plan's Complexity
Tracking table or removed. For day-to-day runtime and contributor guidance, defer to
`CONTRIBUTING.md` and the `docs/en_US/development/` guides, which MUST stay consistent with
this constitution.

**Version**: 1.0.0 | **Ratified**: 2026-06-30 | **Last Amended**: 2026-06-30
