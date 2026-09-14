# ADR-V2-010 — Future cross-store Trust as a restricted separate owner

Status: **PROPOSED — lead-selected, awaiting product/supervisor acceptance.** Date: 14 September 2026.

[Architecture V2](../architecture-v2.md) · [Service catalog](../service-catalog-v2.md) · [Data ownership](../data-ownership-v2.md) · [Communication](../communication-v2.md)

## Context

The historical product proposes cross-store COD behavioral scoring while customer accounts remain store-local. This deliberately crosses tenant boundaries and creates privacy, evidence-quality and abuse risks unlike ordinary Commerce data.

## Decision

Defer scoring/enforcement in Grad 1. When launched, give Trust a separate service/database and restricted assessment API. Consume adjudicated, source-attributed, versioned COD outcomes/corrections via outbox. Use versioned keyed phone pseudonyms, minimized evidence and policy/score versions. Commerce freezes the assessment/prepayment policy chosen for an order and remains decision owner.

## Alternatives considered

Global raw history in public Commerce; global customer identity by phone; unrestricted merchant phone lookup; plain phone hashes as anonymity; trusting every carrier failure as buyer rejection.

## Consequences and stage impact

Creates a defensible boundary for intentional cross-store processing and future corrections. Adds eventual evidence arrival and an unavailable/unknown assessment state. Keyed identifiers remain linkable personal data; they do not establish identity certainty or legal permission.

## Risks and acceptance gates

Threshold/minimum evidence, legal basis, retention, correction/appeal, poisoning controls, phone recycling/shared numbers and outage fallback must be decided before launch. Never expose raw source histories to merchants. Do not implement a silent reject/accept default while these are OPEN.

## Future reconsideration conditions

Activate only after product/privacy/security approval and evidence-quality evaluation. Revisit model/policy versions and tenant deletion/recomputation as facts evolve. Any AI risk inference remains auditable and cannot directly mutate stock or charge a customer.
