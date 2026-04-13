---
date: 2026-04-12
topic: zenoh-localhost-transport
---

# Zenoh Localhost Transport

## Problem Frame

dimos currently uses LCM (UDP multicast) as its default pubsub transport. The team wants to adopt Zenoh for its richer feature set (QoS, reliability, security, multi-network), but needs to validate that Zenoh can match LCM's localhost performance before committing to a broader integration.

This is a scoped take-home assignment: implement Zenoh as a localhost-only pubsub transport that passes the existing transport benchmarks, using LCM encoding for message serialization.

## Requirements

**Transport Implementation**

- R1. Implement `ZenohPubSub` class conforming to the `PubSub[TopicT, MsgT]` interface (`dimos/protocol/pubsub/spec.py`)
- R2. Use LCM encoding for message serialization (compose with `LCMEncoderMixin` or equivalent pattern)
- R3. Implement `ZenohService` with singleton session management (one Zenoh session shared across all publishers/subscribers in the same process)
- R4. `ZenohTransport` wrapper class exposing `broadcast()` / `subscribe()` to the module layer, following the pattern in `dimos/core/transport.py`
- R5. Zenoh runs in peer mode on localhost only — no router, no network configuration required

**Blueprint Integration**

- R6. Add a `transport` option to `GlobalConfig` (default: `"lcm"`) that switches `_get_transport_for()` to return Zenoh transports when set to `"zenoh"`
- R7. Support both LCM-typed messages (`ZenohTransport`) and pickle-serialized messages (`pZenohTransport`) to match the existing LCM/pLCM split

**Testing and Benchmarks**

- R8. Pass the existing pubsub spec conformance tests (`dimos/protocol/pubsub/test_spec.py`)
- R9. Appear in the benchmark results with comparable performance to LCM for localhost communication
- R10. Unit tests for `ZenohPubSub` covering lifecycle (start/stop), error handling, and session management
- R11. Unit tests for `ZenohTransport` covering broadcast, subscribe, and unsubscribe

## Success Criteria

- `uv run pytest dimos/protocol/pubsub/test_spec.py` passes with Zenoh in the test matrix
- `uv run pytest -svm tool dimos/protocol/pubsub/benchmark/test_benchmark.py` produces Zenoh results without crashes
- Zenoh latency and throughput are in the same order of magnitude as LCM for localhost
- A blueprint can switch from LCM to Zenoh by setting `transport = "zenoh"` in GlobalConfig (or equivalent CLI flag)
- No "extra dumb/inefficient" patterns (Lesh's words): no unnecessary copies, no polling loops, no session-per-publish

## Scope Boundaries

- Localhost peer mode only — no router topology, no network endpoints
- Pubsub only — no RPC (scoped for trial period)
- Two encoding variants only: LCM-encoded (`ZenohTransport`) for typed DimosMsg, pickle (`pZenohTransport`) for types without `lcm_encode`. No raw bytes, JPEG, or other variants.
- No QoS configuration — default Zenoh QoS is fine for localhost benchmarking
- No security/TLS — localhost doesn't need it
- No SHM optimization — Zenoh's built-in SHM is a trial-period concern
- No discovery configuration — peer mode autodiscovery works out of the box on localhost

## Key Decisions

- **Start from `dev`, not PR #1296**: The PR is only 300 lines, 2 months stale, and missing encoder integration. Cleaner to rewrite following the same patterns.
- **LCM encoding via mixin composition**: `LCMEncoderMixin + ZenohPubSubBase` for typed messages, matching how `LCM` class composes encoding with the LCM transport.
- **GlobalConfig flag for transport switch**: Simplest path to the "one-liner blueprint switch". `--transport zenoh` on CLI.
- **Singleton session**: One `zenoh.Session` per process, shared across all publishers/subscribers. Follows the `DDSService` pattern.
- **Key expression convention**: Use `dimos/{stream_name}` pattern (e.g., `dimos/cmd_vel`, `dimos/lidar`) matching the protocol overview's namespacing recommendations. Separates cleanly from other Zenoh traffic on the same host.
- **TDD with dedicated unit tests**: Beyond spec conformance, write unit tests for lifecycle and error handling.

## Dependencies / Assumptions

- `zenoh` Python package available (add to `pyproject.toml` as optional dependency, like CycloneDDS)
- Zenoh peer mode works on localhost without any system configuration
- The existing benchmark framework handles new transports without modification (just add to `testdata.py`)
- `LCMEncoderMixin` is transport-agnostic: it calls `msg.lcm_encode()` (a DimosMsg method) and delegates via `super().publish()` through MRO. Verified compatible with non-LCM transports.
- Zenoh uses native callbacks (no message pump thread needed). Model `ZenohService` after `DDSService`, not `LCMService`.
- `global_config` singleton is already imported in `blueprints.py` — `_get_transport_for()` reads it directly.

## Outstanding Questions

### Deferred to Planning

- [Affects R2][Technical] Reuse `lcmpubsub.Topic` (which satisfies `LCMTopicProto`: has `.topic: str` and `.lcm_type: type`) or extract to a shared location? Reusing creates an odd import path; extracting is cleaner but more churn.
- [Affects R6][Technical] When `transport=zenoh`, should `_run_configurators()` skip LCM-specific configurators, or are they harmless no-ops?
- [Affects R3][Technical] Session cleanup on process exit: `DDSService` uses module-level dict with lock but no explicit shutdown hook. Verify if Zenoh session needs explicit close or handles it via `__del__`.

## Next Steps

-> `/ce:plan` for structured implementation planning
