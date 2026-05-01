# CONTEXT.md Format

`CONTEXT.md` is the project's **actor catalog** — the Lego book of Durable Object actors. Each entry describes one actor: its name, its responsibility, its public API, and its partitioning key.

## Structure

```md
# {Worker Name}

{One or two sentence description of what this Worker is responsible for.}

## Actors

**Order**:
A single customer order through its lifecycle.
_Avoid_: OrderManager, OrderService, OrderEntity, OrderActor
**API**: `place(items)`, `cancel()`, `addItem(item)`
**Partitioned by**: orderId

**Invoice**:
A bill against a placed Order, ready to be sent to the Customer.
_Avoid_: Bill, Receipt, InvoiceRecord
**API**: `generate(orderId)`, `send()`, `markPaid()`
**Partitioned by**: invoiceId

**Customer**:
A person or organization that places Orders.
_Avoid_: Client, Buyer, User, Account
**API**: `register(profile)`, `updateProfile(...)`, `getContact()`
**Partitioned by**: customerId

## Relationships

- **Order**.`place()` triggers **Invoice**.`generate(orderId)`
- **Invoice**.`send()` reads from **Customer**.`getContact()`

## Example dialogue

> **Dev:** "When a Customer places an Order, do we create the Invoice immediately?"
> **Domain expert:** "No — `Invoice.generate()` is only called once Fulfillment confirms shipment."

## Flagged ambiguities

- "account" was used to mean both **Customer** and **User** — resolved: distinct actors with distinct partition keys (customerId vs userId).
- It was unclear whether `Order.cancel()` also reverses payment, or just records the cancellation. Resolved: `Order.cancel()` is a state transition only; refund is a separate `Payment.refund()` call orchestrated by the caller.
```

## Rules

- **Be opinionated.** When multiple words exist for the same concept, pick the best one and list the others as aliases to avoid.
- **Bare nouns, no suffixes.** `Order`, not `OrderActor`, `OrderService`, or `OrderManager`. The catalog implies actor; the suffix is noise. Two actors competing for the same name is a signal you have *two distinct domain concepts* — give them different domain nouns.
- **Public API IS domain language.** Method names like `place()`, `cancel()`, `generate()` are the verbs domain experts use. Include them. Internal state, helpers, and code organisation stay out.
- **`Partitioned by` is required.** State the key that determines a single DO instance (`orderId`, `customerId`). This carries the cardinality information — don't repeat it in Relationships.
- **Relationships = message flow only.** Express which actor calls which method on which actor. "**Order**.`place()` triggers **Invoice**.`generate()`." Don't restate cardinality.
- **Flag both naming and behavioural conflicts.** Naming ambiguities ("account" meant two things) belong in "Flagged ambiguities." So do behavioural ambiguities (which actor owns a given operation).
- **Keep definitions tight.** One sentence max. Define what the actor IS, not what it does.
- **Only include actors specific to this Worker's responsibility.** Generic infrastructure (rate limiters, request queues, utility patterns) doesn't belong even if used. Before adding an entry, ask: is this an actor unique to this domain, or generic plumbing? Only the former belongs.
- **Group actors under subheadings** when natural clusters emerge. If all actors belong to a single cohesive area, a flat list is fine.
- **Write an example dialogue.** A conversation between a dev and a domain expert that demonstrates how the actors interact naturally and clarifies which actor owns which behaviour.

## Single vs multi-Worker repos

**Single Worker (most projects):** One `CONTEXT.md` at the repo root.

**Multiple Workers:** A `CONTEXT-MAP.md` at the repo root lists the Workers, where they live, and how they communicate:

```md
# Worker Map

## Workers

- [Ordering](./workers/ordering/CONTEXT.md) — receives and tracks customer orders
- [Billing](./workers/billing/CONTEXT.md) — generates invoices and processes payments
- [Fulfillment](./workers/fulfillment/CONTEXT.md) — manages warehouse picking and shipping

## Cross-Worker integration

- **Ordering → Fulfillment**: service binding; Ordering invokes Fulfillment's `startPicking()` directly
- **Fulfillment → Billing**: queue (`shipment-dispatched`); Billing consumes and calls `Invoice.generate()`
- **Ordering ↔ Billing**: shared types `CustomerId` and `Money` (no runtime call)
```

The skill infers which structure applies:

- If `CONTEXT-MAP.md` exists, read it to find Workers
- If only a root `CONTEXT.md` exists, single Worker
- If neither exists, create a root `CONTEXT.md` lazily when the first actor is resolved

When multiple Workers exist, infer which one the current topic relates to. If unclear, ask.
