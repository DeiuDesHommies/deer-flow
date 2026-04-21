# Multi-Replica Deployment Infrastructure for DeerFlow

**Date**: 2026-04-20
**Status**: Draft for external RFC publication
**Target upstream**: `bytedance/deer-flow` `release/2.0-rc`

---

## 1. Motivation

DeerFlow's single-process defaults work well for local development, but multi-replica deployments expose a common set of missing infrastructure primitives:

1. **Cross-replica SSE delivery**
2. **Cross-replica shared memory backends**
3. **User ↔ thread reverse lookup across replicas**
4. **Model-level runtime health counters**

These are not product-specific requirements; they are generic operational needs for any deployment with:

- multiple gateway pods
- multiple worker replicas
- shared persistence backends
- sticky-less or reconnect-prone clients

This RFC groups those gaps into one umbrella proposal so maintainers can reason about them as deployment infrastructure, not as isolated downstream patches.

---

## 2. Goals

1. Define the missing infrastructure primitives for multi-replica DeerFlow deployments.
2. Upstream the low-risk parts first (especially Redis-backed stream bridge).
3. Clarify which parts are good candidates for core upstream support and which may remain optional/experimental.
4. Align these capabilities with already-merged persistence and auth foundations in `release/2.0-rc`.

## 3. Non-Goals

1. Force every deployment to use Redis/Postgres/Mongo.
2. Replace existing single-process defaults.
3. Introduce data migration requirements for current users.
4. Conflate user feedback on runs with model-level health counters.

---

## 4. Current Gaps

### 4.1 Stream Bridge

`stream_bridge` already has the abstraction, and Redis is visibly planned in upstream, but the implementation is still a stub in current 2.0-rc.

This is the clearest low-risk upstream candidate because:

- abstraction already exists
- config schema already exists
- semantics are deployment-facing and generic

### 4.2 Memory Backends

Current upstream memory defaults are file-based. Multi-replica deployments need shared backends such as PostgreSQL or MongoDB so memory survives pod boundaries.

The requirement here is **pluggable storage**, not a single mandatory database.

### 4.3 Thread Mapping

Upstream `persistence.thread_meta` provides forward thread metadata, but many deployments also need a runtime-friendly reverse lookup abstraction: given a user namespace, enumerate the user's threads efficiently in a cross-replica-safe way.

This may land either as:

- a thin adapter over `thread_meta`
- additional helper APIs in persistence
- or an optional separate store abstraction

### 4.4 Model Counters

Upstream persistence already includes run feedback, but **model-level call/success/failure counters are a different concept**. They are useful for operational health and model routing decisions, but are not the same as user rating data.

This area likely needs the most RFC discussion before upstream code is proposed.

---

## 5. Proposed Capability Buckets

### A. Redis Stream Bridge

- backend: Redis Streams
- contract: preserve current `StreamBridge` API
- behavior: `XADD` for publish, `XRANGE` for replay, `XREAD` for blocking subscribe
- status: best first upstream candidate

### B. Pluggable Memory Backends

- backend choices: PostgreSQL / MongoDB (optional)
- contract: keep current storage abstraction
- requirement: no regression for file-backed default mode

### C. Thread Mapping over Persistence

- keep current namespace-KV API for callers
- internally adapt to official persistence/thread metadata where possible
- preserve Redis/Mongo/SQLite alternatives for deployments that still need them

### D. Model Runtime Counters

- model_name keyed counters
- call/success/failure/positive/negative totals
- should remain conceptually separate from run feedback

---

## 6. Backward Compatibility

All four capability areas should be additive:

- single-process defaults continue working
- file memory continues working
- auth and persistence paths do not regress
- deployments opt into multi-replica backends explicitly via config

---

## 7. Interaction with Existing Upstream Work

### Persistence (`thread_meta`, `runs`, `feedback`)

This RFC builds on the persistence foundation already merged into `release/2.0-rc`.

### Per-user filesystem isolation

Pluggable PG/Mongo memory should not fight the existing per-user filesystem direction. Instead, it should be positioned as an optional deployment backend for replica-safe operation.

### Auth improvements

Trusted-header auth and multi-replica infrastructure are complementary but separable. The auth RFC can move independently.

---

## 8. Open Questions

1. Should Redis stream bridge be merged before the broader umbrella RFC concludes, since it is already scaffolded?
2. Should thread reverse lookup live inside persistence or remain an adapter layer?
3. Should model counters be upstreamed as a separate capability later instead of inside the umbrella path?
4. How much operational complexity are maintainers willing to carry for optional PG/Mongo/Redis backends?

---

## 9. Recommended Sequence

1. RFC publication for umbrella alignment
2. PR #4 Redis Stream Bridge
3. PR #5 Pluggable Memory Storage (if RFC feedback does not oppose)
4. Thread mapping adapter/helper discussion
5. Model counter proposal only after maintainers confirm appetite
