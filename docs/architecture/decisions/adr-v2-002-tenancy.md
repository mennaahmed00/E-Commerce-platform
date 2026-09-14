# ADR-V2-002 — Pooled tenant tables with enforced RLS

Status: **PROPOSED — lead-selected, awaiting product/supervisor acceptance.** Date: 14 September 2026.

[Architecture V2](../architecture-v2.md) · [Service catalog](../service-catalog-v2.md) · [Data ownership](../data-ownership-v2.md) · [Communication](../communication-v2.md)

## Context

The old requirements mandated schema-per-tenant; the user explicitly permits changing that constraint. V1's schema/role fleet adds migrations and partial provisioning without isolating a service allowed to choose every tenant role.

## Decision

Use shared tenant_id tables plus RLS within each service's separate PostgreSQL database. Require non-owner runtime roles, no superuser/BYPASSRLS, FORCE RLS, read/write checks, transaction-local context and tenant-qualified foreign keys/unique keys. Global directory/auth records have explicitly scoped access.

## Alternatives considered

Schema and roles per tenant; database per tenant; shared tables with application filters only; pooled default with dedicated placement for selected tenants.

## Consequences and stage impact

One migration stream per service and simpler provisioning fit four application students. Larger indexes, tenant skew, selective restore and resource isolation are harder. Dedicated placement is a future escape hatch, not two implemented operational paths now.

## Risks and acceptance gates

Missing context, connection reuse, background jobs, exports, object/cache keys and constraints must be isolation-tested. RLS does not contain full application compromise. Queue/hostname bootstrap uses narrowly scoped resolver/dispatcher roles, never blanket business-table bypass.

## Future reconsideration conditions

Add dedicated placement when contractual isolation, exceptional tenant load or restore requirements justify it. If live V1 schemas exist, inventory/backfill/validate/cut over per service with retained rollback sources; do not assume a destructive empty-database migration.
