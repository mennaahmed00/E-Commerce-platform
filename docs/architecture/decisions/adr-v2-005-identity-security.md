# ADR-V2-005 — Separate actor profiles, trusted tenant context and workload identity

Status: **PROPOSED — lead-selected, awaiting product/supervisor acceptance.** Date: 14 September 2026.

[Architecture V2](../architecture-v2.md) · [Service catalog](../service-catalog-v2.md) · [Data ownership](../data-ownership-v2.md) · [Communication](../communication-v2.md)

## Context

Claude's universal internal signing secret and ambiguous Host/token resolution risk forged authority. Store-local customers, guest checkout and separate platform administration now need explicit identity profiles.

## Decision

Platform owns merchant/staff and distinct admin identity profiles; Commerce owns store-local customer credentials/sessions and guest capabilities. Gateway resolves public tenancy from an active verified directory and staff tenancy from membership. Validate issuer/audience/type/expiry/algorithm. Use asymmetric workload identities with destination/action allowlists and separate bounded user delegation. Owners enforce resource permissions.

## Alternatives considered

One universal JWT profile/key; trusting client tenant headers; global customer login by email/phone; private network as sole authentication; platform-admin unrestricted database access.

## Consequences and stage impact

Explicit profiles and current sensitive checks reduce confused-deputy and stale-privilege risks. Key rotation, service contracts, CSRF/session handling and bounded revocation freshness require implementation. Existing accepted remote operations cannot always be cancelled immediately.

## Risks and acceptance gates

Host-only secure cookies, CSRF/Origin checks, refresh rotation/reuse detection, admin MFA, minimal secrets and redacted audits are required. Unknown tenant fails closed. Sensitive publication/refund/send/connect checks require current authority; ordinary caches have tested bounds. A worker cannot self-assert merchant authority.

## Future reconsideration conditions

Adopt managed identity or mTLS/workload federation when their operating benefit is demonstrated. Review support access and new actor types explicitly. Never turn scaling into a reason to accept a universal credential or unlimited RLS bypass.
