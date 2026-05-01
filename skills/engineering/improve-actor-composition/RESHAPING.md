# Reshaping

How to safely split, repartition, or relocate state in an actor graph. Assumes the vocabulary in [LANGUAGE.md](LANGUAGE.md) — **actor**, **partition key**, **god actor**, **coordinator actor**.

This file covers the three **non-merge** move types. For merge moves, see [DEEPENING.md](DEEPENING.md).

The shared property of these moves: they all involve **state migration**. Unlike a merge — which collapses logic, often without touching durable state — a split / repartition / relocate changes which actor instance owns which state. That makes them more disruptive, and demands a cutover plan.

## Move types

### Split (god actor → two actors)

Take a god actor — multi-purpose, fuzzy partition key — and break it into two with sharper responsibilities and (usually) sharper partition keys.

Process:

1. **Identify the cleavage line.** Which methods stay, which leave. Each new actor should have a coherent, narrow responsibility.
2. **Pick partition keys for both halves.** They may differ from the original. If they do, this is also a repartition — combine guidance with that section.
3. **Pick the API between the two new actors.** Sync RPC, queue, or no direct call (caller orchestrates). Apply the **state-coordination probe** from [grill-with-catalog](../grill-with-catalog/SKILL.md).
4. **Migration**: see "Cutover" below.

Tests: each new actor gets its own tests at its RPC interface. Add cross-actor tests for any flow that crosses the new seam, but only enough to verify the seam — not enough to recreate the old monolithic test surface.

### Repartition (wrong key → right key)

The actor exists, but its partition key is wrong. Symptoms: hot DO instances (one key sees too much traffic), lost isolation (operations on different logical entities serialize through the same instance), or query patterns that don't fit (cross-instance queries are common).

Process:

1. **Decide the new key.** Apply the **partitioning probe** from [grill-with-catalog](../grill-with-catalog/SKILL.md). What query patterns does the new key support? What does it cost?
2. **Decide the cutover strategy.** See "Cutover" below.
3. **Plan reverse migration if it goes wrong.** Repartitioning is among the harder moves to reverse — guarantee an escape hatch before committing.

Tests: existing tests should mostly pass with the new key; add tests for the migration itself, exercised during the cutover window.

### State-relocate (shared external state → single owner)

Two or more actors implicitly coupled by shared external state — a KV namespace, a D1 table, an R2 bucket — that nobody owns. Symptoms: eventual consistency surprises, partial-write races, unclear "source of truth" when actors disagree.

Process:

1. **Pick the new owner.** Usually the actor whose responsibilities most centre on this state.
2. **Convert other actors' direct storage access into RPC calls** against the new owner's interface.
3. **Migrate the state.** Often this means changing storage tier (e.g. shared KV → DO storage) — surface as an ADR trigger.
4. **Migration**: see "Cutover" below.

Tests: the new owner is the test surface for this state. Other actors are tested with a mock or in-memory adapter of the owner's RPC interface.

## Cutover

All three move types involve durable state changing identity (which DO instance owns it) or location (which storage tier holds it). Pick a cutover strategy:

- **Big-bang** — write the new shape, migrate all state at once, switch traffic. Cheapest if downtime is acceptable; viable only when state is small or traffic is paused.
- **Dual-write window** — both old and new shapes accept writes for a period; reads serve from new; old is cleaned up after. Most common for live systems. Requires careful idempotency and conflict resolution.
- **Read-through migration** — new shape accepts writes; reads check new first, fall back to old if missing, lazily migrate on read. Lower migration risk; longer migration tail.

Apply the **migration-cost probe** when grilling the candidate: which strategy fits this move? Is the migration reversible if something goes wrong? What observability do you need during the window?

## Seam discipline (still applies)

- **One adapter means a hypothetical seam. Two adapters means a real one.** When a split creates a new RPC seam between the two new actors, ask: are there two adapters that justify it? If the only adapter is production-to-production, the seam may be hypothetical — consider whether the split is actually solving the problem, or just moving complexity.
- The split-or-merge probe applies to **both halves** of a split: each new actor needs to pass the deletion test on its own.
