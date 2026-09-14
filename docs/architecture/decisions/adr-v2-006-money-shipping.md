# ADR-V2-006 — Separate customer and subscription money; recoverable fulfillment

Status: **PROPOSED — lead-selected, awaiting product/supervisor acceptance.** Date: 14 September 2026.

[Architecture V2](../architecture-v2.md) · [Service catalog](../service-catalog-v2.md) · [Data ownership](../data-ownership-v2.md) · [Communication](../communication-v2.md)

## Context

Paymob/Fawry collect shopper payments for merchants; Kashier collects platform subscriptions. Provider execution evidence and commercial recognition are different facts. Delayed payments, COD and shipping retries invalidate a paid/unpaid or rollback-on-timeout model.

## Decision

Commerce owns customer obligations/collections/refunds/allocations and shipment eligibility; Platform Billing owns invoices/cycles/consent/schedule/platform entries/entitlements. Domain-local processors recognize verified evidence atomically with state and subsequent work. Isolated workers execute fixed authorized operations. Bosta is first shipping adapter; J&T follows verified Egypt API access.

## Alternatives considered

Both ledgers in Integrations; platform collection and merchant payouts by default; provider calls inside checkout transactions; redirects as proof; unconditional shipment creation; generic accounting ERP.

## Consequences and stage impact

Customer payment application no longer crosses an internal business-service database boundary. External uncertainty remains. Append-only money entries and immutable snapshots preserve explainability. Merchant-owned settlement is a working assumption requiring onboarding confirmation. Shipping and COD remittance need distinct records.

## Risks and acceptance gates

Validate account/mode/reference/amount/currency and effect uniqueness. Late payment after reservation release enters resolution/refund review. Kashier provides token charging, not managed subscriptions: automatic renewal needs consent and enabled capability; hosted invoice payment is the initial fallback. J&T auth/idempotency/lookup and provider approvals remain OPEN.

## Future reconsideration conditions

Reopen for marketplace custody/payouts, regulated payment segregation, independent payment operators, complex proration/dunning or substantial logistics orchestration. New business model/compliance requirements are not merely new adapter parameters.
