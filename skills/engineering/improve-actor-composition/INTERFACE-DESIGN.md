# Interface Design

When the user wants to explore alternative interfaces for a chosen composition candidate, use this parallel sub-agent pattern. Based on "Design It Twice" (Ousterhout) — your first idea is unlikely to be the best.

Uses the vocabulary in [LANGUAGE.md](LANGUAGE.md) — **actor**, **interface**, **seam**, **adapter**, **leverage**, **partition key**.

## Process

### 1. Frame the problem space

Before spawning sub-agents, write a user-facing explanation of the problem space for the chosen candidate:

- The constraints any new interface would need to satisfy (invariants, idempotency, ordering, partition-key implications, state-durability semantics)
- The dependencies it would rely on, and which category they fall into (see [DEEPENING.md](DEEPENING.md) for merges, [RESHAPING.md](RESHAPING.md) for split / repartition / state-relocate)
- The move type (merge, split, repartition, state-relocate) and any migration constraints
- A rough illustrative code sketch to ground the constraints — not a proposal, just a way to make the constraints concrete

Show this to the user, then immediately proceed to Step 2. The user reads and thinks while the sub-agents work in parallel.

### 2. Spawn sub-agents

Spawn 3+ sub-agents in parallel using the Agent tool. Each must produce a **radically different** interface for the new actor (or actors, if a split).

Prompt each sub-agent with a separate technical brief — file paths, coupling details, dependency category, what sits behind the seam, partition-key options. The brief is independent of the user-facing problem-space explanation in Step 1. Give each agent a different design constraint:

- Agent 1: "Minimise the RPC surface — aim for 1–3 entry points max. Maximise leverage per entry point."
- Agent 2: "Maximise flexibility — support many use cases and extension points."
- Agent 3: "Optimise for the most common caller — make the default case trivial."
- Agent 4 (if applicable): "Design around ports & adapters for cross-actor or cross-Worker dependencies."
- Agent 5 (if applicable, for split or repartition): "Pick a different partition key and redesign the interface around that key."

Include both [LANGUAGE.md](LANGUAGE.md) vocabulary and CONTEXT.md vocabulary in the brief so each sub-agent names things consistently with the architecture language and the project's actor catalog.

Each sub-agent outputs:

1. Interface (RPC method signatures, params — plus invariants, idempotency, ordering, error modes)
2. Partition key and its rationale
3. Usage example showing how callers use it
4. What the implementation hides behind the seam (state model, internal helpers)
5. Dependency strategy and adapters (see [DEEPENING.md](DEEPENING.md) / [RESHAPING.md](RESHAPING.md))
6. Trade-offs — where leverage is high, where it's thin

### 3. Present and compare

Present designs sequentially so the user can absorb each one, then compare them in prose. Contrast by **depth** (leverage at the interface), **locality** (where change concentrates, including state ownership), **seam placement**, and **partition-key choice**.

After comparing, give your own recommendation: which design you think is strongest and why. If elements from different designs would combine well, propose a hybrid. Be opinionated — the user wants a strong read, not a menu.
