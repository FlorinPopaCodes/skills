# Deepening

How to safely merge a cluster of shallow / anemic / coordinator actors into a deeper one, given its dependencies. Assumes the vocabulary in [LANGUAGE.md](LANGUAGE.md) — **actor**, **interface**, **seam**, **adapter**.

This file covers **merge** moves only. For split, repartition, and state-relocate moves, see [RESHAPING.md](RESHAPING.md).

## Dependency categories

When assessing a merge candidate, classify its dependencies. The category determines how the deepened actor is tested across its seam.

### 1. In-process

Pure computation, the same actor's own storage, in-memory helpers — no I/O outside the DO instance. Always merge — collapse the methods into one actor and test through the new RPC surface directly. No adapter needed.

This is the cleanest case: anemic actors that just hold state used by one caller, coordinator actors that just delegate to in-process helpers, phantom seams between actors that share a partition key.

### 2. Local-substitutable

Dependencies that have a local test stand-in — the Cloudflare Workers test harness for DOs and bindings, in-memory KV / D1 / R2 simulators. Mergeable if the stand-in exists. The deepened actor is tested with the stand-in running in the test suite. The seam is internal; no port at the actor's RPC interface.

### 3. Remote but owned (Ports & Adapters)

Other actors in the same or different Workers — another DO across an RPC boundary, service binding, or queue. Define a **port** (interface) at the seam. The deep actor owns the logic; the transport is injected as an **adapter**. Tests use an in-memory adapter that mimics the dependency's RPC surface. Production uses the real RPC / service binding / queue adapter.

Recommendation shape: *"Define a port at the seam, implement a service-binding adapter for production and an in-memory adapter for testing, so the logic sits in one deep actor even though it's deployed across an RPC boundary."*

### 4. True external (Mock)

Third-party services (Stripe, Twilio, OpenAI, etc.) reached via `fetch` from the Worker or the actor — anything you don't control. The deepened actor takes the external dependency as an injected port; tests provide a mock adapter.

## Seam discipline

- **One adapter means a hypothetical seam. Two adapters means a real one.** Don't introduce a port unless at least two adapters are justified (typically production + test). A single-adapter seam is just indirection — and in actor systems, that often means a phantom seam between two DOs that always deploy together.
- **Internal seams vs external seams.** A deep actor can have internal seams (private helpers used by its own tests) as well as the external seam at its RPC interface. Don't expose internal seams through the RPC surface just because tests use them.

## Testing strategy: replace, don't layer

- Old unit tests on shallow / anemic / coordinator actors become waste once tests at the deepened actor's RPC interface exist — delete them.
- Write new tests at the deepened actor's RPC interface. The **interface is the test surface**.
- Tests assert on observable outcomes through the RPC surface, not storage internals or helper behaviour.
- Tests should survive internal refactors — they describe behaviour, not implementation. If a test has to change when storage layout changes, it's testing past the interface.
