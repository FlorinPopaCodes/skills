---
name: improve-actor-composition
description: Find composition opportunities in a Cloudflare Workers + Durable Objects codebase — moves on the actor graph (merge, split, repartition, state-relocate) that improve depth, leverage, and locality. Informed by the actor catalog in CONTEXT.md and the decisions in docs/adr/. Use when the user wants to improve actor composition, fix anemic / god / coordinator actors, repartition wrong keys, or make the codebase more testable and AI-navigable.
---

# Improve Actor Composition

Surface architectural friction in the actor graph and propose **composition moves** that improve depth, leverage, and locality. Four move types are in scope:

- **Merge** — collapse anemic actors, coordinator pass-throughs, or phantom seams into a deeper actor.
- **Split** — break a god actor into two with clearer responsibilities and sharper partition keys.
- **Repartition** — fix a wrong partition key (the most common DO design mistake).
- **State-relocate** — collapse shared external state into the single actor that should own it.

The aim is testability and AI-navigability — actors deep enough to absorb behaviour, with seams that earn real leverage.

## Glossary

Use these terms exactly in every suggestion. Consistent language is the point — don't drift into "component," "service," "API," or "boundary." Full definitions in [LANGUAGE.md](LANGUAGE.md).

- **Actor** — a Durable Object class with an interface, an implementation, and a partition key that gives every instance an identity.
- **Interface** — everything a caller must know to use the actor: RPC method signatures, invariants, idempotency, ordering, error modes, partition-key implications, state-durability semantics.
- **Implementation** — the code inside the DO class.
- **Depth** — leverage at the interface: a lot of behaviour behind a small RPC surface. **Deep** = high leverage. **Shallow** = interface nearly as complex as the implementation.
- **Seam** — where an interface lives; a place behaviour can be altered without editing in place. (Use this, not "boundary.")
- **Adapter** — a concrete thing satisfying an interface at a seam (test fake DO vs production DO; service binding vs HTTP fetch).
- **Leverage** — what callers get from depth.
- **Locality** — what maintainers get from depth: change, bugs, knowledge, **and state ownership** concentrated in one actor.

Actor-specific terms for the anti-patterns this skill detects:

- **Partition key** — the key that determines a single actor instance.
- **Anemic actor** — state bag with getters/setters; no behaviour owned. The actor analog of a shallow module.
- **God actor** — multi-purpose actor with a fuzzy partition key; usually a candidate to split.
- **Coordinator actor** — pure pass-through; just routes calls between other actors. Fails the deletion test.
- **Chatty composition** — a single user-facing call fans out across many actors; latency and partial-failure surface.
- **Phantom seam** — RPC between two actors that always deploy together; the seam earns no leverage.

Key principles (see [LANGUAGE.md](LANGUAGE.md) for the full list):

- **Deletion test**: imagine deleting the actor. If complexity vanishes, it was a pass-through. If complexity reappears across N callers, it was earning its keep.
- **The interface is the test surface.**
- **One adapter = hypothetical seam. Two adapters = real seam.**

This skill is _informed_ by the project's actor catalog. `CONTEXT.md` (per [grill-with-catalog/CONTEXT-FORMAT.md](../grill-with-catalog/CONTEXT-FORMAT.md)) names the actors, their public APIs, and their partition keys; ADRs record decisions the skill should not re-litigate.

## Process

### 1. Explore

Read the project's actor catalog (`CONTEXT.md`) and any ADRs in the area you're touching first.

Then use the Agent tool with `subagent_type=Explore` to walk the codebase — DO classes, the Workers that bind them, the storage they touch, and the calls between them. Don't follow rigid heuristics. Note where you experience friction:

- **Anemic actors** — DO classes that are mostly getters/setters with no real behaviour owned.
- **Coordinator actors** — DO classes that just call other DOs and return; fail the deletion test.
- **God actors** — DO classes that handle multiple unrelated responsibilities; partition key is fuzzy or arbitrary.
- **Chatty composition** — a single user-facing operation crosses N actors; reasoning about it requires bouncing between many DO classes.
- **Phantom seams** — RPC between two actors that always deploy in the same Worker and have no second adapter.
- **Wrong partition keys** — hot DOs, lost isolation, or query patterns that don't fit (e.g. cross-instance queries are common).
- **Shared external state** — two actors implicitly coupled via shared KV / D1 / R2; eventual consistency surprises bite.
- **Hard to test through the RPC interface** — often a downstream symptom of one of the above.

Apply the **deletion test** to anything you suspect is anemic, coordinator, or a phantom seam: would deleting it concentrate complexity, or just move it? A "yes, concentrates" is the signal you want.

### 2. Present candidates

Present a numbered list of composition opportunities. For each candidate:

- **Move type** — merge, split, repartition, or state-relocate
- **Files / actors** — which DO classes / Workers / storage are involved
- **Problem** — why the current shape is causing friction (name the anti-pattern: anemic, god, coordinator, chatty, phantom seam, wrong partition, shared state)
- **Solution** — plain English description of the proposed move
- **Benefits** — explained in terms of locality, leverage, and how tests would improve

**Use CONTEXT.md vocabulary for the actors, and [LANGUAGE.md](LANGUAGE.md) vocabulary for the architecture.** If `CONTEXT.md` defines `Order`, talk about "the Order actor" — not "the OrderHandler DO," and not "the Order service."

**ADR conflicts**: if a candidate contradicts an existing ADR, only surface it when the friction is real enough to warrant revisiting the ADR. Mark it clearly (e.g. _"contradicts ADR-0007 — but worth reopening because…"_). Don't list every theoretical refactor an ADR forbids.

**Out of scope**: cross-Worker integration patterns (queue vs RPC vs binding), storage-tier choices (DO vs KV vs D1 vs R2), and other ADR-shaped decisions are *not* moves on the actor graph. If they come up, surface as ADR triggers — don't roll them into composition candidates.

Do NOT propose interfaces yet. Ask the user: "Which of these would you like to explore?"

### 3. Grilling loop

Once the user picks a candidate, drop into a grilling conversation. Walk the design tree with them — constraints, dependencies, the shape of the new actor(s), what sits behind each seam, what tests survive.

Use the universal actor-design lenses from [grill-with-catalog/SKILL.md](../grill-with-catalog/SKILL.md):

- **Partitioning probe** — especially for split and repartition moves.
- **Split-or-merge probe** — for any move; the discipline behind composition.
- **State-coordination probe** — for any move that changes who calls whom.

Plus deepening-specific lenses for this skill:

- **Deletion test challenge** — walk the callers. Would deleting this actor concentrate complexity, or just move it?
- **Phantom-seam probe** — is there only one impl behind this RPC? Is the seam hypothetical? If so, can we collapse it?
- **Anti-pattern check** — does the candidate match the test for anemic, god, or coordinator? Apply each.
- **Migration-cost probe** (split, repartition, state-relocate only) — what's the cutover plan? Dual-write window? Is the partition-key change reversible? See [RESHAPING.md](RESHAPING.md).

Side effects happen inline as decisions crystallize:

- **Naming a new actor after a concept not in `CONTEXT.md`?** Add the entry to `CONTEXT.md` — same discipline as `/grill-with-catalog` (see [CONTEXT-FORMAT.md](../grill-with-catalog/CONTEXT-FORMAT.md)). Create the file lazily if it doesn't exist.
- **Sharpening a fuzzy actor name or method during the conversation?** Update `CONTEXT.md` right there.
- **User rejects the candidate with a load-bearing reason?** Offer an ADR, framed as: _"Want me to record this so future composition reviews don't re-suggest it?"_ Only offer when the reason would actually be needed by a future explorer. See [ADR-FORMAT.md](../grill-with-catalog/ADR-FORMAT.md).
- **Want to explore alternative APIs for the new actor(s)?** See [INTERFACE-DESIGN.md](INTERFACE-DESIGN.md).
- **Working out testing strategy for the move?** See [DEEPENING.md](DEEPENING.md) for merges, [RESHAPING.md](RESHAPING.md) for split / repartition / state-relocate.
