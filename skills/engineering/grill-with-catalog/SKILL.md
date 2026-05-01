---
name: grill-with-catalog
description: Grilling session for Cloudflare actor systems. Challenges your plan against the existing actor catalog, sharpens actor names, partition keys, and message flow, and updates documentation (CONTEXT.md, ADRs) inline as decisions crystallise. Use when the project is built on Workers + Durable Objects and you want to stress-test a plan against its actor model.
---

<what-to-do>

Interview me relentlessly about every aspect of this plan until we reach a shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one-by-one. For each question, provide your recommended answer.

Ask the questions one at a time, waiting for feedback on each question before continuing.

If a question can be answered by exploring the codebase, explore the codebase instead.

</what-to-do>

<supporting-info>

## Domain awareness

`CONTEXT.md` is the project's **actor catalog** — the Lego book of Durable Object actors that compose this system. Each entry is one actor with its public API and partitioning key. During codebase exploration, look for it alongside any ADRs.

### File structure

Most repos have a single Worker:

```
/
├── CONTEXT.md
├── docs/
│   └── adr/
│       ├── 0001-order-and-invoice-via-queue.md
│       └── 0002-do-storage-for-order-state.md
└── src/
```

If a `CONTEXT-MAP.md` exists at the root, the repo has multiple Workers. The map lists each Worker, points to its `CONTEXT.md`, and describes how Workers communicate (service binding / queue / fetch):

```
/
├── CONTEXT-MAP.md
├── docs/
│   └── adr/                          ← cross-Worker decisions
├── workers/
│   ├── ordering/
│   │   ├── CONTEXT.md
│   │   └── docs/adr/                 ← Worker-specific decisions
│   └── billing/
│       ├── CONTEXT.md
│       └── docs/adr/
```

Create files lazily — only when you have something to write. If no `CONTEXT.md` exists, create one when the first actor is resolved. If no `docs/adr/` exists, create it when the first ADR is needed.

## During the session

### Challenge against the catalog

When the user uses an actor name or method that conflicts with the existing entries in `CONTEXT.md`, call it out immediately. "Your catalog defines `Order.cancel()` as a state-only transition, but you seem to mean it also reverses payment — which is it?"

### Sharpen fuzzy language

When the user uses vague or overloaded terms, propose a precise canonical term. "You're saying 'account' — do you mean **Customer** or **User**? Those are different actors with different partition keys."

### Discuss concrete scenarios

When actor relationships are being discussed, stress-test them with specific scenarios. Invent scenarios that probe edge cases and force the user to be precise about which actor owns which behaviour.

### Partitioning probe

For every actor proposed or modified, grill the partition key. "What determines a single instance of this actor — `orderId`, `customerId`, or both? What if two users share that key — does it cause a hot DO or lose isolation? What query patterns become impossible if you pick this key?" Wrong partitioning is the most common DO design error; surface it before code is written.

### Split-or-merge probe

When an actor is proposed (or feels too big), apply the split-or-merge test. "If you merged this actor into **Order**, would complexity vanish or just shift one level deeper? If you split it into two, what's the API across the new seam — and would it just be a chatty internal protocol?" Composition only works when the seams are real.

### State-coordination probe

When two actors call each other (sync or async), probe the moments they're inconsistent. "When **Order**.`place()` calls **Inventory**.`reserve()`, what does the world look like in between? If `Inventory.reserve()` succeeds but `Order.place()` fails — who's the source of truth? Is the transient inconsistency a bug or fine?" Eventual consistency is unavoidable in actor systems; make it conscious.

### Cross-reference with code

When the user states how something works, check whether the code agrees. The actor catalog's `**API**` field should match the DO class's public methods 1:1. If you find a contradiction, surface it: "Your `Order` catalog entry lists `cancel()`, but the DO class only has `cancelLineItem(id)` — is the catalog stale, or is `cancel()` a different operation?"

### Update CONTEXT.md inline

When an actor or method is resolved, update `CONTEXT.md` right there. Don't batch these up — capture them as they happen. Use the format in [CONTEXT-FORMAT.md](./CONTEXT-FORMAT.md).

Include the public surface a domain expert would recognise as the system's verbs — actor names and public API method names. Internal state, helpers, and code organisation stay out.

### Offer ADRs sparingly

Only offer to create an ADR when all three are true:

1. **Hard to reverse** — the cost of changing your mind later is meaningful
2. **Surprising without context** — a future reader will wonder "why did they do it this way?"
3. **The result of a real trade-off** — there were genuine alternatives and you picked one for specific reasons

If any of the three is missing, skip the ADR. Use the format in [ADR-FORMAT.md](./ADR-FORMAT.md).

Common actor-design ADR triggers worth watching for:

- **Storage tier per actor** (DO storage vs KV vs D1 vs R2) when the choice is non-obvious
- **Sync RPC vs queue** between two actors when failure-coupling is the deciding factor
- **Partition key choice** when multiple valid keys exist
- **Cross-Worker integration mode** (service binding vs fetch vs queue)

</supporting-info>
