# ADR-V2-001 — Five domains; domain-owned provider execution

Status: **PROPOSED — lead-selected, awaiting product/supervisor acceptance.** Date: 14 September 2026.

[Architecture V2](../architecture-v2.md) · [Service catalog](../service-catalog-v2.md) · [Data ownership](../data-ownership-v2.md) · [Communication](../communication-v2.md)

## Context

V1 had Platform identity, broad Commerce and generic Integrations. Confirmed social DMs and merchant-edited theme files/AI change the boundaries. Six independent reviews and six rebuttals completed; security retained a preference for dedicated execution isolation.

## Decision

Select Platform Control, Commerce, Store Experience, Social Messaging and AI Assistance. Keep stock/cart/order invariants in Commerce. Paymob/Fawry and shipping operations belong to Commerce; Kashier and domain automation belong to Platform; social adapters belong to Messaging. Execute providers in separately restricted processes. Experience owns authoring/publication; rendering/validation still require independent isolation.

## Alternatives considered

One modular application; V1 plus isolated theme renderer; five services retaining Integrations and Commerce themes; entity-level microservices. Security's preferred alternative keeps Integrations as execution-evidence owner, never ledger owner.

## Consequences and stage impact

Removes an internal provider-outcome-to-ledger network handoff. Adds Experience contracts and lifecycle coordination outside checkout. Five business services do not mean five containers. Shared databases/hosts still share failure fate; explicit module and runtime permissions cost implementation effort.

## Risks and acceptance gates

API processes cannot decrypt provider secrets. Workers claim authorized operations and append evidence but cannot arbitrarily rewrite orders, journals or memberships. Callback intake is append-limited, not literally read-only. Domain processors recognize effects. Validate process, secret, role, network and connection budgets before claiming isolation.

## Future reconsideration conditions

Reopen if privilege isolation cannot be demonstrated, independent payment operators/compliance require a stronger boundary, provider releases repeatedly disrupt Commerce, or Experience contracts prevent Grad 1 delivery. A theme module remains a possible simplification but never permits trusted-process template execution.
