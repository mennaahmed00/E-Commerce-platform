# ADR-V2-008 — Automatic activation with independent ownership, TLS and routing gates

Status: **PROPOSED — lead-selected, awaiting product/supervisor acceptance.** Date: 14 September 2026.

[Architecture V2](../architecture-v2.md) · [Service catalog](../service-catalog-v2.md) · [Data ownership](../data-ownership-v2.md) · [Communication](../communication-v2.md)

## Context

Entering a hostname cannot transfer DNS ownership or force global propagation/certificate issuance. The product wants fast self-service activation without human platform operations.

## Decision

Platform owns normalized unique host claims and routing generations. Use managed custom-hostname/TLS automation, Cloudflare for SaaS proposed, behind a provider interface. Track ownership proof, DNS target, hostname readiness, certificate/DCV readiness and store eligibility separately. Activate only after all gates; retry/reconcile automatically with visible reasons. Supply a platform subdomain during pending setup.

## Alternatives considered

Manual operations; arbitrary host routing immediately after entry; self-managed ACME/custom proxy controller; assuming every DNS provider supports a universal apex CNAME; hard-coded guaranteed activation time.

## Consequences and stage impact

Managed certificates reduce student operating burden and automate renewal. External DNS authority, provider plan/quotas, CAA and certificate gates remain dependencies. Merchant DNS action or authorized supported DNS integration may be necessary; no human platform step is required on the normal path.

## Risks and acceptance gates

Require fresh proof on reassignment; prevent duplicate/conflicting claims; unknown hosts fail closed. Revoke generations and invalidate caches before serving more tenant content; routing cache expiry cannot fall back to stale mappings. Check active route before cached HTML. Limit verification to DNS/approved edge endpoints, not arbitrary server-side URL fetching.

## Future reconsideration conditions

Revisit provider choice for cost, limits, apex/wildcard requirements or deployment constraints. Confirm operational revocation/renewal targets under failure. Self-managed issuance requires explicit ownership of challenge routing, key recovery and renewal operations.
