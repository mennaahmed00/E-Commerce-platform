# ADR-V2-007 — Standalone social Messaging with PostgreSQL

Status: **PROPOSED — lead-selected, awaiting product/supervisor acceptance.** Date: 14 September 2026.

[Architecture V2](../architecture-v2.md) · [Service catalog](../service-catalog-v2.md) · [Data ownership](../data-ownership-v2.md) · [Communication](../communication-v2.md)

## Context

The clarified message box is a unified inbox of external private messages with replies through their original platforms, not website chat. OAuth/asset ownership, tokens, delivery rules and webhook traffic are independent of checkout.

## Decision

Messaging owns WhatsApp/Instagram connections, provider-scoped identities, conversations, messages, send intents, delivery evidence and account policy state. Use PostgreSQL with relational constraints/indexes and bounded JSONB, private media storage and durable local jobs. Persist verified receipts before acknowledgment; preserve channel/account/recipient on every reply.

## Alternatives considered

Website-chat module in Commerce; social transport in generic Integrations with conversations elsewhere; MongoDB solely because payloads differ; direct browser-to-provider secrets; scraping personal social sessions.

## Consequences and stage impact

Standalone ownership isolates credentials and channel workload and supports future conversational commerce. Additional database/processes and account approval work are real cost. Text-first MVP supports real receive/reply; arbitrary media and full historic import are not assumed.

## Risks and acceptance gates

Single-use store/session-bound connection state, token encryption, exact account mapping, dispatch-time policy checks, current send authority and per-account limits are mandatory. Accepted is not delivered. Unknown sends require supported reconciliation or review, not blind retry. Never merge identities by name or matching phone alone.

## Future reconsideration conditions

Add channels/media/backfill as provider capability and product demand justify. Introduce MongoDB or search only after measured access/storage limitations. AI sending/ordering requires new user authority and confirmation contracts, not inherited theme-edit access.
