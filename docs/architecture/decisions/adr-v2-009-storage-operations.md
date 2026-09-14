# ADR-V2-009 — PostgreSQL and object storage; minimal stateful infrastructure

Status: **PROPOSED — lead-selected, awaiting product/supervisor acceptance.** Date: 14 September 2026.

[Architecture V2](../architecture-v2.md) · [Service catalog](../service-catalog-v2.md) · [Data ownership](../data-ownership-v2.md) · [Communication](../communication-v2.md)

## Context

The project has six students, four building application/platform features. Multiple database technologies, unused search/vector stores and a broker increase backup, monitoring and deployment work before features use them.

## Decision

Use one initial PostgreSQL instance with five logical owner databases and scoped credentials, plus R2/S3-compatible object storage. Keep parameterized pg repositories and service-owned SQL migrations. Compose is the development/demo topology; private ingress, separate runtime privileges, resource budgets, redacted telemetry and off-host tested recovery are required. Defer MongoDB, Redis, Meilisearch, pgvector, broker and Kubernetes defaults.

## Alternatives considered

One shared application database; separate physical clusters per service immediately; MongoDB for flexible payloads; always-on Redis/search/vector containers; a full self-hosted observability stack.

## Consequences and stage impact

Reduces technology count and permits local transactions/durable work in familiar storage. One PostgreSQL host and Compose node are single failure domains. Queue workloads and noisy tenants need indexes, fairness and connection budgets. Backup recovery is not high availability.

## Risks and acceptance gates

Test RLS, migrations, concurrent checkout, crash recovery, object/manifests, key restoration and external reconciliation. Maintain audit/backlog/domain/AI cost visibility. Proposed assessment RPO24h/RTO4h are unmeasured targets, not production SLAs. Pin supported runtime/dependency versions before implementation.

## Future reconsideration conditions

Add managed PITR/failover and replicas for real availability requirements. Redis needs a measured coordination/cache/queue use; search/vector need implemented consumers and tenant-safe rebuilds. Dedicated DB instances or orchestration need workload/security/operating evidence, not anticipated startup scale alone.
