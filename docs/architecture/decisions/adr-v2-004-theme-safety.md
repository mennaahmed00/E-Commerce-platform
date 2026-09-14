# ADR-V2-004 — File-based themes, isolated rendering and bounded AI proposals

Status: **PROPOSED — lead-selected, awaiting product/supervisor acceptance.** Date: 14 September 2026.

[Architecture V2](../architecture-v2.md) · [Service catalog](../service-catalog-v2.md) · [Data ownership](../data-ownership-v2.md) · [Communication](../communication-v2.md)

## Context

The revised product abandons Puck/drag-drop and requires HTML/CSS/config editing plus actual AI changes. Merchant-controlled presentation is untrusted, even without scripts. Checkout and identity cannot share that trust surface.

## Decision

Experience owns immutable file manifests/revisions, validation evidence and publication pointers. Store blobs in object storage; upload and validate before an atomic pointer switch. Permit restricted HTML/CSS, declarative JSON config/schema and a small bounded template grammar. AI returns base-revision patches; merchant accepts and separately publishes. Run validators/evaluators without business credentials or arbitrary network/filesystem access.

## Alternatives considered

Puck/component-only editor; arbitrary merchant JavaScript or builds; AI direct live writes; shared privileged rendering; keeping metadata as a Commerce module with the same mandatory runtime isolation.

## Consequences and stage impact

Ready themes plus code editing satisfy the new workflow and give deterministic preview/rollback. No-JS/declarative-schema restrictions limit customization. Separate Experience adds contracts but isolates a coherent lifecycle from orders. Global publication visibility is eventual, not a multi-system atomic switch.

## Risks and acceptance gates

Parsed HTML/CSS/URL restrictions, escaping, controlled assets, CSP, sandboxed preview, separate registrable trust domains and resource budgets are cumulative. Keep checkout/accounts/admin outside merchant markup. Prevent path traversal, prompt injection, stale-base overwrite and deleting referenced artifacts. Merchant deception remains a moderation risk.

## Future reconsideration conditions

Reconsider executable scripts/plugins only with a separately reviewed sandbox and permissions system. Broaden template grammar based on real theme needs. Move or scale renderer independently; never interpret service extraction itself as sufficient sandboxing.
