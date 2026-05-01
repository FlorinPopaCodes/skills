# Language

Shared vocabulary for every suggestion this skill makes. Use these terms exactly — don't substitute "component," "service," "API," or "boundary." Consistent language is the whole point.

## Terms

**Actor**
A Durable Object class with an interface, an implementation, and a partition key that gives every instance an identity. The unit of composition.
_Avoid_: module (too generic — actors carry identity and durable state), service (overloaded), DO (acronym; use the noun).

**Interface**
Everything a caller must know to use the actor correctly. Includes the RPC method signatures, but also invariants, idempotency, ordering, error modes, partition-key implications, and state-durability semantics.
_Avoid_: API, signature (too narrow — those refer only to the type-level surface).

**Implementation**
What's inside the DO class — its body of code and storage layout. Distinct from **Adapter**: the production DO and the test fake DO are two adapters of the same actor; both have implementations.

**Depth**
Leverage at the interface — the amount of behaviour a caller (or test) can exercise per unit of interface they have to learn. An actor is **deep** when a large amount of behaviour sits behind a small RPC surface. An actor is **shallow** when the interface is nearly as complex as the implementation — usually because it's anemic.

**Seam** _(from Michael Feathers)_
A place where you can alter behaviour without editing in that place. The *location* at which an actor's interface lives. Cross-actor RPC boundaries are seams; so are the boundaries between a Worker and the DOs it binds. Choosing where to put the seam is its own design decision, distinct from what goes behind it.
_Avoid_: boundary (overloaded with DDD's bounded context).

**Adapter**
A concrete thing that satisfies an interface at a seam. Describes *role* (what slot it fills), not substance (what's inside). Examples: a production DO and its in-memory test fake; a service binding and an HTTP-fetch adapter for cross-Worker calls.

**Leverage**
What callers get from depth. More capability per unit of interface they have to learn. One actor pays back across N call sites and M tests.

**Locality**
What maintainers get from depth. Change, bugs, knowledge, **and state ownership** concentrate at one actor rather than spreading across many. Fix once, fixed everywhere.

**Partition key**
The key that determines a single actor instance. The identity-defining property of an actor. Wrong partition key is the most common DO design error — leads to hot keys, lost isolation, or query patterns that don't fit.

**Anemic actor**
State bag with getters/setters; no behaviour owned. The actor analog of a shallow module — its interface is nearly as complex as its implementation, and the real logic lives in callers. Usually a merge candidate.

**God actor**
Multi-purpose actor with a fuzzy partition key; handles multiple unrelated responsibilities. Usually a split candidate; the split often surfaces sharper partition keys for each new actor.

**Coordinator actor**
Pure pass-through that just routes calls between other actors. Fails the deletion test — deleting it would concentrate complexity nowhere, just move it to the caller. Usually a merge candidate.

**Chatty composition**
A single user-facing call fans out across many actors. Reasoning about a feature requires bouncing between many DO classes; latency adds up; partial-failure surface grows. Usually a split-or-merge issue elsewhere in the graph.

**Phantom seam**
RPC between two actors that always deploy together and have only one adapter (the production one). The seam is hypothetical — no leverage earned. Usually a merge candidate.

## Principles

- **Depth is a property of the interface, not the implementation.** A deep actor can be internally composed of small, mockable parts — they just aren't part of the interface. An actor can have **internal seams** (private to its implementation, used by its own tests) as well as the **external seam** at its RPC surface.
- **The deletion test.** Imagine deleting the actor. If complexity vanishes, the actor wasn't hiding anything (it was a pass-through). If complexity reappears across N callers, the actor was earning its keep.
- **The interface is the test surface.** Callers and tests cross the same seam. If you want to test *past* the interface — poking at storage directly, or reaching through to internal helpers — the actor is probably the wrong shape.
- **One adapter means a hypothetical seam. Two adapters means a real one.** Don't introduce a seam unless something actually varies across it. Production-DO + test-fake counts as two adapters; production-DO + production-DO doesn't.
- **Locality includes state ownership.** A deep actor doesn't just have rich behaviour — it owns the state that behaviour acts on. State scattered across actors (or across actors and shared external storage) leaks locality.

## Relationships

- An **Actor** has exactly one **Interface** (the RPC surface it presents to callers and tests).
- An **Actor** has exactly one **Partition key** (which defines instance identity).
- **Depth** is a property of an **Actor**, measured against its **Interface**.
- A **Seam** is where an **Actor**'s **Interface** lives.
- An **Adapter** sits at a **Seam** and satisfies the **Interface**.
- **Depth** produces **Leverage** for callers and **Locality** for maintainers.
- **Anemic**, **God**, and **Coordinator** are anti-patterns of the **Actor** itself.
- **Chatty composition** and **Phantom seam** are anti-patterns of the **Actor graph** (the set of actors and the calls between them).

## Rejected framings

- **Depth as ratio of implementation-lines to interface-lines** (Ousterhout): rewards padding the implementation. We use depth-as-leverage instead.
- **"Interface" as the TypeScript `interface` keyword or a class's public methods**: too narrow — interface here includes every fact a caller must know, including partition-key implications and state-durability semantics.
- **"Boundary"**: overloaded with DDD's bounded context. Say **seam** or **interface**.
- **"Module" applied to actors**: actors carry identity and durable state, which generic modules don't. Say **actor**.
