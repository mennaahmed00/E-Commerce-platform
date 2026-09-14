# ADR-V2-003 — Local ACID; durable jobs and outbox HTTP without a broker

Status: **PROPOSED — lead-selected, awaiting product/supervisor acceptance.** Date: 14 September 2026.

[Architecture V2](../architecture-v2.md) · [Service catalog](../service-catalog-v2.md) · [Data ownership](../data-ownership-v2.md) · [Communication](../communication-v2.md)

## Context

External payments, DMs, object uploads and DNS cannot join a PostgreSQL transaction. Synchronous REST-only processing and fire-and-forget work lose recoverability. A broker would add operational cost before substantial fan-out exists.

## Decision

Use local transactions for owner invariants; commit state plus outbox or job intent together. Deliver cross-service messages through authenticated HTTP with per-recipient progress; consumers commit deduplication/effect together. Model payment, provisioning and publication as explicit persisted workflows with compensation, not a generic saga engine.

## Alternatives considered

Synchronous chains; broker-first RabbitMQ/Kafka; two-phase commit; Redis-only queues; fire-and-forget promises. A maintained PostgreSQL job library may implement claim/lease mechanics without changing ownership.

## Consequences and stage impact

At-least-once delivery and eventual projections are explicit. A missing acknowledgment is recoverable. Queue fairness, leases, schema versions, retention, gap handling, dead letters and operator replay still require engineering; no-broker is not no-infrastructure.

## Risks and acceptance gates

Payload-bound idempotency keys and unique commercial effects are separate controls. Unknown external effects require documented provider lookup/idempotency or review. Local leases cannot guarantee external exactly-once behavior. Never hold SQL transactions across network calls.

## Future reconsideration conditions

Introduce a broker for measured delivery/fan-out/backlog/independent-consumer needs. Preserve outbox IDs and consumer deduplication after migration. Use a workflow engine only when workflow volume/versioning/visibility outweighs operating and learning cost.
