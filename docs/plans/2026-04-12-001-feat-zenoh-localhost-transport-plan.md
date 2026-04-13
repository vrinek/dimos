---
title: "feat: Add Zenoh localhost pubsub transport"
type: feat
status: active
date: 2026-04-12
origin: docs/brainstorms/2026-04-12-zenoh-localhost-transport-requirements.md
---

# feat: Add Zenoh localhost pubsub transport

## Overview

Add Zenoh as a localhost-only pubsub transport backend in dimos, using LCM encoding for message serialization. This validates Zenoh's performance against LCM before committing to a broader network-capable integration.

## Problem Frame

dimos needs a transport that works across unreliable networks, WANs, and heterogeneous link types. Zenoh is the candidate. Before adopting it, the team needs to see it pass the existing localhost transport benchmarks with comparable performance to LCM. (see origin: `docs/brainstorms/2026-04-12-zenoh-localhost-transport-requirements.md`)

## Requirements Trace

- R1. `ZenohPubSub` conforming to `PubSub[TopicT, MsgT]`
- R2. LCM encoding via mixin composition
- R3. `ZenohService` with singleton session
- R4. `ZenohTransport` wrapper
- R5. Peer mode, localhost only
- R6. GlobalConfig `transport` flag switches `_get_transport_for()`
- R7. Both `ZenohTransport` (LCM-encoded) and `pZenohTransport` (pickle)
- R8. Pass spec conformance tests
- R9. Pass benchmarks with comparable localhost performance
- R10. Unit tests for `ZenohPubSub`
- R11. Unit tests for `ZenohTransport`

## Scope Boundaries

- Localhost peer mode only — no router, no network endpoints
- Pubsub only — no RPC
- Two encoding variants: LCM-typed + pickle. No raw bytes, JPEG
- No QoS configuration — default Zenoh QoS for localhost benchmarking (per-topic QoS is trial-period work)
- No security, SHM optimization, or discovery configuration
- (see origin for full list)

## Context & Research

### Relevant Code and Patterns

**Encoder mixin composition (the critical pattern):**
- `LCMEncoderMixin` (`dimos/protocol/pubsub/encoders.py:105`) calls `msg.lcm_encode()` → `super().publish(topic, encoded_bytes)`. Transport-agnostic.
- `LCMTopicProto` requires `.topic: str` and `.lcm_type: type[DimosMsg] | None`
- Composition: `class LCM(LCMEncoderMixin, LCMPubSubBase): ...` (`dimos/protocol/pubsub/impl/lcmpubsub.py:140`)
- `PickleEncoderMixin` uses `pickle.dumps(msg)` / `pickle.loads(msg)` — also transport-agnostic

**Session singleton (DDSService pattern):**
- Module-level `_participants: dict[int, DomainParticipant] = {}` with `threading.Lock()`
- `start()` creates if not exists, `stop()` does NOT delete shared resource
- Property accessor raises `RuntimeError` if not started

**Transport wrapper pattern (DDSTransport, `dimos/core/transport.py:289`):**
- `__init__` creates topic + pubsub instance
- `start()`/`stop()` with `_started` flag and `threading.RLock()`
- `broadcast()` and `subscribe()` auto-start if needed
- `subscribe()` wraps callback to strip topic: `lambda msg, topic: callback(msg)`

**GlobalConfig (`dimos/core/global_config.py`):**
- `pydantic_settings.BaseSettings` with `extra="ignore"`
- Fields auto-cascade: env vars (`DIMOS_` prefix from pydantic-settings convention) → `.env` → defaults
- Singleton at module level: `global_config = GlobalConfig()`
- Already imported in `blueprints.py` (line 28)

**`_get_transport_for()` (`dimos/core/blueprints.py:200`):**
- Checks `transport_map` for explicit overrides first
- Then: `use_pickled = getattr(stream_type, "lcm_encode", None) is None`
- Returns `pLCMTransport(topic)` or `LCMTransport(topic, stream_type)`

**Existing `ZenohTransport` stub:** `dimos/core/transport.py:323` — empty `class ZenohTransport(PubSubTransport[T]): ...`

### Institutional Learnings

- Previous PR #1296 followed the right file structure but missed encoder integration. Its `ZenohService` used `zenoh.open(config)` with `config.insert_json5()` for settings — viable approach.
- PR #927 was rejected for not using "LCM types under Zenoh" — confirms the encoder mixin approach is expected.

## Key Technical Decisions

- **Reuse `lcmpubsub.Topic` for Zenoh's LCM-encoded variant**: It satisfies `LCMTopicProto` and avoids extracting a shared type (more churn than value for this scope). The import path `from dimos.protocol.pubsub.impl.lcmpubsub import Topic` is unusual but pragmatic. For the pickle variant, use a plain `str` topic (matches `pLCMTransport` pattern).
- **`ZenohPubSubBase` implements `AllPubSub[Topic, bytes]`**: Zenoh natively supports wildcard key expressions (`dimos/**`), so `subscribe_all()` maps directly to a wildcard subscription. This matches how `LCMPubSubBase` implements `AllPubSub` via regex.
- **No message pump thread**: Zenoh dispatches subscriber callbacks on its internal threads. Unlike `LCMService` (which needs `_lcm_loop`), `ZenohService` just opens a session. Simpler.
- **Session lifecycle**: Follow DDSService — no explicit shutdown hook. Zenoh's Python binding (`zenoh.Session`) is reference-counted and closes when all references are dropped. Module-level dict keeps sessions alive for the process lifetime, which is correct for the singleton pattern.
- **Skip LCM configurators when transport=zenoh**: `_run_configurators()` runs LCM-specific system checks (multicast setup). When Zenoh is selected, these are unnecessary. Gate on `global_config.transport`.
- **Key expression prefix**: Use `dimos/` prefix for all Zenoh key expressions (e.g., `dimos/cmd_vel`). Forward-compatible with the multi-robot `dimos/robots/{id}/...` convention from the protocol overview.

## Open Questions

### Resolved During Planning

- **Topic type for Zenoh**: Reuse `lcmpubsub.Topic` for LCM-encoded variant (has `.topic: str` + `.lcm_type`). Pickle variant uses plain `str`. Rationale: minimal churn, matches existing pLCM pattern.
- **LCM configurators**: Skip when `global_config.transport != "lcm"`. They check multicast setup which is LCM-specific.
- **Session cleanup**: No explicit shutdown hook needed. Zenoh session closes on GC. Module-level dict keeps it alive for process lifetime (same as DDSService).

### Deferred to Implementation

- **Zenoh's callback threading model**: Verify that Zenoh callbacks are safe to invoke from multiple internal threads simultaneously. If not, may need a callback dispatcher queue.
- **`zenoh` package version pinning**: Check latest stable on PyPI and pin appropriately.

## High-Level Technical Design

> *This illustrates the intended approach and is directional guidance for review, not implementation specification. The implementing agent should treat it as context, not code to reproduce.*

```
MRO composition (same pattern as LCM):

class ZenohPubSubBase(ZenohService, AllPubSub[Topic, bytes]):
    publish(topic, message_bytes) → session.declare_publisher(topic).put(bytes)
    subscribe(topic, callback)    → session.declare_subscriber(topic, on_sample)

class Zenoh(LCMEncoderMixin, ZenohPubSubBase):
    # MRO: Zenoh → LCMEncoderMixin → ZenohPubSubBase → ZenohService → AllPubSub
    # publish(topic, dimos_msg) → encode → super().publish(topic, bytes)
    # subscribe(topic, cb)     → super().subscribe(topic, decode_wrapper(cb))

class PickleZenoh(PickleEncoderMixin, ZenohPubSubBase):
    # Same pattern with pickle encoding

Transport wrappers:
    ZenohTransport(topic, type)  → uses Zenoh (LCM-encoded)
    pZenohTransport(topic)       → uses PickleZenoh (pickle-encoded)

Blueprint switch:
    global_config.transport == "zenoh" → _get_transport_for() returns ZenohTransport/pZenohTransport
```

## Implementation Units

- [ ] **Unit 1: ZenohService — session singleton**

**Goal:** Implement the Zenoh session management layer following DDSService pattern.

**Requirements:** R3, R5

**Dependencies:** None (foundation for everything else)

**Files:**
- Create: `dimos/protocol/service/zenohservice.py`
- Modify: `dimos/protocol/service/__init__.py`
- Test: `dimos/protocol/service/test_zenohservice.py`

**Approach:**
- Module-level `_sessions: dict[str, zenoh.Session]` with `threading.Lock()`
- `ZenohConfig(BaseConfig)` with `mode: str = "peer"` (only peer mode for now)
- Session key derived from config (mode + endpoints hash)
- `start()` opens session if not exists, `stop()` is a no-op for shared session
- Property `session` returns the shared session or raises `RuntimeError`

**Patterns to follow:**
- `dimos/protocol/service/ddsservice.py` — exact singleton pattern
- `dimos/protocol/service/spec.py` — `Service[ConfigT]` base class

**Test scenarios:**
- Happy path: `start()` creates a session, `session` property returns it
- Happy path: Two `ZenohService` instances with same config share one session
- Happy path: `stop()` does not close the shared session
- Edge case: `session` property before `start()` raises `RuntimeError`
- Edge case: `start()` called twice is idempotent (no error, same session)

**Verification:**
- Tests pass. A ZenohService can be instantiated and started without errors on localhost.

---

- [ ] **Unit 2: ZenohPubSubBase — core publish/subscribe**

**Goal:** Implement the raw bytes pub/sub over Zenoh, conforming to `AllPubSub[Topic, bytes]`.

**Requirements:** R1, R5

**Dependencies:** Unit 1

**Files:**
- Create: `dimos/protocol/pubsub/impl/zenohpubsub.py`
- Modify: `dimos/protocol/pubsub/impl/__init__.py`
- Test: `dimos/protocol/pubsub/impl/test_zenohpubsub.py`

**Approach:**
- Inherits from `ZenohService` and `AllPubSub[Topic, bytes]`
- `publish(topic, message)`: get-or-create a `zenoh.Publisher` for the topic's key expression, call `publisher.put(message)`
- `subscribe(topic, callback)`: declare a `zenoh.Subscriber` with `on_sample` handler that extracts `sample.payload.to_bytes()` and calls `callback(bytes, topic)`
- `subscribe_all(callback)`: subscribe to `dimos/**` wildcard
- Publisher cache: `dict[str, zenoh.Publisher]` with lock (avoid declaring per-publish)
- Subscriber tracking: `list[zenoh.Subscriber]` for cleanup in `stop()`
- `stop()`: undeclare all publishers and subscribers, then `super().stop()`
- Topic uses `lcmpubsub.Topic` (has `.topic: str` which becomes the key expression)

**Patterns to follow:**
- `dimos/protocol/pubsub/impl/lcmpubsub.py:76` — `LCMPubSubBase` structure
- PR #1296's `zenohpubsub.py` — publisher caching and subscriber cleanup patterns

**Test scenarios:**
- Happy path: publish bytes on a topic, subscriber receives them
- Happy path: multiple subscribers on same topic all receive the message
- Happy path: unsubscribe function stops delivery to that callback
- Happy path: `subscribe_all` receives messages from any `dimos/` key
- Edge case: publish before any subscriber — no error, message is lost (fire-and-forget)
- Edge case: unsubscribe called twice — no error (idempotent)
- Edge case: publish after `stop()` — logs error, does not crash
- Integration: start/stop lifecycle — publishers and subscribers are cleaned up on stop

**Verification:**
- All unit tests pass. The class can publish and receive bytes on localhost with zero configuration.

---

- [ ] **Unit 3: Encoder composition — Zenoh + LCMEncoderMixin**

**Goal:** Compose `LCMEncoderMixin` and `PickleEncoderMixin` with `ZenohPubSubBase` to create typed pub/sub classes.

**Requirements:** R2, R7

**Dependencies:** Unit 2

**Files:**
- Modify: `dimos/protocol/pubsub/impl/zenohpubsub.py` (add composed classes)
- Modify: `dimos/protocol/pubsub/test_spec.py` (add to conformance matrix)
- Test: `dimos/protocol/pubsub/impl/test_zenohpubsub.py` (extend)

**Approach:**
- `class Zenoh(LCMEncoderMixin, ZenohPubSubBase): ...` — one line, MRO does the work
- `class PickleZenoh(PickleEncoderMixin, ZenohPubSubBase): ...` — same pattern
- Add both to spec test matrix in `test_spec.py`:
  - `Zenoh` variant tested with `Topic(topic="dimos/test/spec", lcm_type=Vector3)` and `Vector3` messages
  - `PickleZenoh` variant tested with plain topic string and arbitrary Python objects
- Export from `__init__.py`

**Patterns to follow:**
- `dimos/protocol/pubsub/impl/lcmpubsub.py:140-149` — `LCM`, `PickleLCM` composition
- `dimos/protocol/pubsub/test_spec.py:119-125` — LCM test matrix entry

**Test scenarios:**
- Happy path: `Zenoh` class encodes a `Vector3` message via LCM, subscriber decodes it back to `Vector3`
- Happy path: `PickleZenoh` class encodes arbitrary Python dict, subscriber receives identical dict
- Happy path: Spec conformance — publish/subscribe/unsubscribe/multiple_subscribers/ordering all pass
- Edge case: Publishing a `bytes` value directly through `Zenoh` (LCMEncoderMixin passes bytes through without encoding)
- Error path: Subscribing with a topic that has `lcm_type=None` — `DecodingError` raised on decode, message silently dropped by mixin

**Verification:**
- `uv run pytest dimos/protocol/pubsub/test_spec.py` passes with Zenoh variants in the matrix.

---

- [ ] **Unit 4: ZenohTransport and pZenohTransport wrappers**

**Goal:** Create transport wrapper classes that the Blueprint system can instantiate, replacing the stub.

**Requirements:** R4, R7

**Dependencies:** Unit 3

**Files:**
- Modify: `dimos/core/transport.py` (replace stub with implementation)
- Test: `dimos/core/test_zenoh_transport.py`

**Approach:**
- `ZenohTransport(topic: str, type: type, **kwargs)` — wraps `Zenoh` instance, stores `LCMTopic(topic, type)`. kwargs forwarded to `ZenohConfig`.
- `pZenohTransport(topic: str, **kwargs)` — wraps `PickleZenoh` instance, stores plain string topic. Same kwargs passthrough.
- Both follow the DDSTransport pattern: `_started` flag, `threading.RLock()`, auto-start on first use
- `broadcast(_, msg)` → `self.zenoh.publish(self.topic, msg)`
- `subscribe(callback, selfstream)` → `self.zenoh.subscribe(self.topic, lambda msg, topic: callback(msg))`
- `__reduce__` for pickling support (same as LCMTransport)

**Patterns to follow:**
- `dimos/core/transport.py:289` — `DDSTransport` (lock pattern, auto-start)
- `dimos/core/transport.py:80` — `pLCMTransport` (pickle variant with string topic)
- `dimos/core/transport.py:112` — `LCMTransport` (typed variant with LCMTopic)

**Test scenarios:**
- Happy path: `ZenohTransport` broadcasts a DimosMsg, subscriber callback receives decoded message
- Happy path: `pZenohTransport` broadcasts arbitrary Python object, subscriber receives it
- Happy path: Auto-start — calling `broadcast()` before `start()` starts the transport implicitly
- Happy path: `subscribe()` returns an unsubscribe callable that works
- Edge case: `stop()` then `start()` — transport can be restarted
- Integration: Two `ZenohTransport` instances on same topic share one Zenoh session

**Verification:**
- Unit tests pass. Transport wrappers correctly delegate to underlying pubsub.

---

- [ ] **Unit 5: GlobalConfig integration and blueprint switch**

**Goal:** Add `transport` field to GlobalConfig and wire `_get_transport_for()` to use Zenoh when configured.

**Requirements:** R6

**Dependencies:** Unit 4

**Files:**
- Modify: `dimos/core/global_config.py` (add `transport` field)
- Modify: `dimos/core/blueprints.py` (modify `_get_transport_for()`, gate configurators)
- Test: `dimos/core/test_blueprints.py` (add test for Zenoh transport selection)

**Approach:**
- Add `transport: str = "lcm"` to `GlobalConfig`
- In `_get_transport_for()`: after the `transport_map` check, branch on `global_config.transport`:
  - `"zenoh"` → return `pZenohTransport(topic)` or `ZenohTransport(topic, stream_type)` using same `use_pickled` heuristic
  - `"lcm"` (default) → existing behavior unchanged
- In `_run_configurators()`: skip LCM configurators when `global_config.transport != "lcm"`
- Import `ZenohTransport, pZenohTransport` at top of `transport.py` (they're already in the same file)

**Patterns to follow:**
- `dimos/core/global_config.py:35` — field definition style
- `dimos/core/blueprints.py:200-208` — existing transport selection logic

**Test scenarios:**
- Happy path: `global_config.transport = "lcm"` → `_get_transport_for()` returns `LCMTransport` (default, unchanged)
- Happy path: `global_config.transport = "zenoh"` → `_get_transport_for()` returns `ZenohTransport` for typed messages
- Happy path: `global_config.transport = "zenoh"` → `_get_transport_for()` returns `pZenohTransport` for untyped messages
- Happy path: LCM configurators skipped when transport is zenoh
- Edge case: Unknown transport value — falls back to LCM (defensive default)

**Verification:**
- Blueprint test passes. Setting `transport = "zenoh"` in GlobalConfig switches transport selection.

---

- [ ] **Unit 6: Benchmark integration**

**Goal:** Add Zenoh to the benchmark test matrix so performance can be compared directly with LCM.

**Requirements:** R9

**Dependencies:** Unit 3

**Files:**
- Modify: `dimos/protocol/pubsub/benchmark/testdata.py` (add Zenoh test case)

**Approach:**
- Add `zenoh_pubsub_channel()` context manager (create `Zenoh` instance, start, yield, stop)
- Add `zenoh_msggen(size)` that returns `(Topic("dimos/benchmark/zenoh", Image), make_data_image(size))`
- Append `Case(pubsub_context=zenoh_pubsub_channel, msg_gen=zenoh_msggen)` to `testcases`
- Wrap in try/except for missing `zenoh` package (like DDS and Redis do)

**Patterns to follow:**
- `dimos/protocol/pubsub/benchmark/testdata.py:272` — Redis benchmark entry pattern (with availability check)

**Test scenarios:**

Test expectation: none — this unit adds to existing benchmark infrastructure which validates itself by producing heatmap output.

**Verification:**
- `uv run pytest -svm tool dimos/protocol/pubsub/benchmark/test_benchmark.py` includes Zenoh in results without crashes.

---

- [ ] **Unit 7: pyproject.toml dependency**

**Goal:** Add `eclipse-zenoh` as an optional dependency.

**Requirements:** R1 (dependency)

**Dependencies:** None (can be done first or in parallel)

**Files:**
- Modify: `pyproject.toml`

**Approach:**
- Add `zenoh` optional extra group (like `dds` extra): `zenoh = ["eclipse-zenoh>=1.0.0,<2.0"]`
- Add `zenoh` to `all-extras` group so `uv sync --all-extras` installs it
- Exclude from `--no-extra dds` if that pattern exists (check existing extras structure)

**Patterns to follow:**
- Existing optional extras in `pyproject.toml` (look for `dds`, `ros` extras)

**Test scenarios:**

Test expectation: none — dependency declaration, validated by `uv sync`.

**Verification:**
- `uv sync --all-extras --no-extra dds` succeeds and `import zenoh` works in the venv.

## System-Wide Impact

- **Interaction graph:** `_get_transport_for()` is called by `autoconnect()` in the Blueprint system. Changing its return type affects all modules connected via autoconnect. The change is transparent — modules don't know what transport they're on.
- **Error propagation:** Zenoh publish errors should be logged but not crash the module (fire-and-forget for best-effort). Matches LCM behavior.
- **State lifecycle risks:** Zenoh session lives for the process lifetime. Multiple transports share one session. If a transport `stop()` prematurely closed the session, other transports would break. The DDSService pattern (never close shared resources) prevents this.
- **API surface parity:** `ZenohTransport` must match the same `broadcast()`/`subscribe()` contract as `LCMTransport`. No new methods exposed to modules.
- **Unchanged invariants:** Module `In[T]`/`Out[T]` streams, Blueprint `autoconnect()`, the `DimosMsg` protocol — all unchanged. The transport layer is transparent to everything above it.

## Risks & Dependencies

| Risk | Mitigation |
|------|------------|
| Zenoh callback thread safety — callbacks may fire from multiple Zenoh internal threads | Start with the naive approach (no dispatcher). The spec tests exercise concurrent publish which will surface races. Add a dispatcher only if tests fail. |
| Benchmark shows Zenoh significantly slower than LCM for localhost | Acceptable if within same order of magnitude (10x). Zenoh adds protocol overhead vs raw UDP multicast. If >10x slower, investigate session configuration. |
| `eclipse-zenoh` package has Rust compilation step on install | Use pre-built wheels from PyPI. If wheels unavailable for the platform, document the build requirement. |
| Circular import from `transport.py` importing `zenohpubsub.py` | Follow the DDS pattern — guard import with `try/except ImportError` and set `ZENOH_AVAILABLE` flag. |

## Sources & References

- **Origin document:** [docs/brainstorms/2026-04-12-zenoh-localhost-transport-requirements.md](docs/brainstorms/2026-04-12-zenoh-localhost-transport-requirements.md)
- Related code: `dimos/protocol/pubsub/impl/lcmpubsub.py` (LCM composition pattern)
- Related code: `dimos/protocol/service/ddsservice.py` (singleton session pattern)
- Related code: `dimos/core/transport.py` (transport wrapper pattern)
- Related PRs: #1296 (draft Zenoh transport), #927 (rejected earlier attempt)
- External: Zenoh Python API stubs at `~/ghq/github.com/eclipse-zenoh/zenoh-python/zenoh/__init__.pyi`
