# ADR Format

ADRs live in `docs/adr/` and use sequential numbering: `0001-slug.md`, `0002-slug.md`, etc.

Create the `docs/adr/` directory lazily — only when the first ADR is needed.

## Template

```md
# {Short title of the decision}

{1-3 sentences: what's the context, what did we decide, and why.}
```

That's it. An ADR can be a single paragraph. The value is in recording *that* a decision was made and *why* — not in filling out sections.

## Optional sections

Only include these when they add genuine value. Most ADRs won't need them.

- **Status** frontmatter (`proposed | accepted | deprecated | superseded by ADR-NNNN`) — useful when decisions are revisited
- **Considered Options** — only when the rejected alternatives are worth remembering
- **Consequences** — only when non-obvious downstream effects need to be called out

## Numbering

Scan `docs/adr/` for the highest existing number and increment by one.

## When to offer an ADR

All three of these must be true:

1. **Hard to reverse** — the cost of changing your mind later is meaningful
2. **Surprising without context** — a future reader will look at the code and wonder "why on earth did they do it this way?"
3. **The result of a real trade-off** — there were genuine alternatives and you picked one for specific reasons

If a decision is easy to reverse, skip it — you'll just reverse it. If it's not surprising, nobody will wonder why. If there was no real alternative, there's nothing to record beyond "we did the obvious thing."

### What qualifies

- **Architectural shape.** "We're using a single Worker with all DO classes in one binding." "**Order** state is split across two DOs — the live `Order` and an immutable `OrderHistory` log."
- **Integration patterns between actors or Workers.** "**Order** and **Invoice** communicate via Queue, not direct RPC, because invoice failures shouldn't block order placement." "The **Ordering** Worker invokes the **Fulfillment** Worker via service binding rather than HTTP fetch, because we want them to share a deploy boundary and avoid auth round-trips."
- **Technology choices that carry lock-in.** Storage tier (DO storage vs KV vs D1 vs R2), queue mechanism, auth provider, deployment target. Not every library — just the ones that would take a quarter to swap out.
- **Boundary and scope decisions.** "**Customer** owns customer data; other actors reference it by `customerId` only." "**Order** is partitioned by `orderId`, not `customerId`, because we expect concurrent updates per customer to dwarf concurrent updates per order." The explicit no-s are as valuable as the yes-s.
- **Deliberate deviations from the obvious path.** "We're using fetch between Workers instead of service bindings because we plan to deploy them in different Cloudflare accounts." Anything where a reasonable reader would assume the opposite. These stop the next engineer from "fixing" something that was deliberate.
- **Constraints not visible in the code.** "We can't put **Customer** state in DO storage because GDPR requires regional isolation; we use D1 with regional replication instead." "Response times must be under 200ms because of the partner API contract."
- **Rejected alternatives when the rejection is non-obvious.** If you considered a single mega-DO and picked separate **Order** + **Invoice** for subtle reasons, record it — otherwise someone will suggest merging them in six months.
