# Architecture V2 — data ownership and consistency

Status: proposed. [Architecture](architecture-v2.md) · [Catalog](service-catalog-v2.md) · [Communication](communication-v2.md) · [Diagrams](diagrams-v2.md)

## 1. One writer for each business fact

One initial PostgreSQL instance contains five logical databases. Each service has its own migrations, runtime roles, outbox, consumer deduplication and recovery records. No database links, cross-database SQL, shared writable schema, direct analytics joins or another service's credentials are allowed. A shared physical instance saves operating effort but remains a shared capacity/failure boundary.

| Canonical owner | Authoritative records | Storage | Explicit exclusions |
|---|---|---|---|
| Platform Control | Merchant/staff identity, memberships, grants, tenant lifecycle, normalized domain claims/routes, plans, subscription invoices/entries/consent/cycles/entitlements | platform_db; restricted Billing/Domain evidence and credential tables | Customer-sale balances, store customer credentials, theme source, DMs |
| Commerce | Products/variants/prices, stock/reservations, store-local customers/sessions, carts, immutable order snapshots, customer journal/allocations/refunds/COD balance, shipments | commerce_db; Commerce product-media namespace; restricted payment/shipping operation/evidence records | Platform subscription ledger, merchant identity, editable themes, social message bodies |
| Store Experience | Template versions, draft heads, immutable revision manifests, validation evidence, publication generations/pointers, preview capabilities, accepted AI proposal association | experience_db; immutable private files and approved asset namespace | Prices/stock/customer credentials, authoritative domain registry, AI execution history |
| Social Messaging | Connected assets/tokens, OAuth state, channel identities, conversations/messages/read state, send intents/attempts/receipts, policy status | messaging_db; JSONB extensions; private media namespace | Shared customer identity, order/money authority, platform billing |
| AI Assistance | Jobs/attempts, bounded proposal history, model/config versions, evaluation records | ai_db; optional expiring temporary object namespace | Authoritative theme revisions or publish pointers; customer/order/DM replicas by default |
| Trust — future | Minimized cross-store outcome evidence/corrections, keyed subjects, scores/policies and assessment audit | Dedicated trust_db when launched | Raw merchant-accessible cross-store histories or shared login accounts |

Provider observations are **evidence**, not financial authority. A receipt may be duplicated, late, contradictory or insufficient. Only Commerce posts customer commercial effects; only Platform Billing posts subscription effects. Execution workers have append/operation permissions, not unrestricted journal updates.

## 2. Tenant and global data boundaries

Use stable tenant UUIDs, never merchant-selected schema names. Every tenant-owned row carries tenant_id, including child/join tables, attachments, audit entries, business operations and private object metadata. A parent ID alone is not tenant isolation.

Within an owner database:

- Tenant-qualified unique keys enforce store-local email, SKU, slug and external-reference rules where appropriate.
- Foreign keys include tenant_id on both sides: an order line cannot reference a sibling tenant's order/product merely because the UUID exists.
- RLS read policies and write checks enforce the validated transaction tenant. Enable and FORCE RLS on tenant tables; runtime roles are not owners, superusers or BYPASSRLS roles.
- Start a short transaction on one checked-out pg client, establish tenant context locally, execute repositories on that client, then commit/rollback before release. Never use a global mutable tenant variable, session-persistent search path, pool-per-tenant or independent pooled query mid-transaction.
- Missing/invalid context denies access. Malformed or conflicting host/path/token context is rejected before database work. Background jobs and exports use the same discipline.
- Parameterize values; allowlist unavoidable identifiers. SQL helpers receive the transaction client and authorized context, not raw browser tenant selectors.

Global Platform tables include merchant identity, hostname uniqueness and tenant directory. They use explicit privileges and actor/lookup constraints rather than a fake null-tenant admin policy. Merchant membership links remain scoped; a global identity is not a global store customer account.

Bootstrap access must be explicit. Routing resolves host through a narrow Platform directory API before tenant RLS context exists. Provider intake resolves a verified account/operation through its owner's minimal account mapping. Dispatchers may claim cross-tenant queue envelopes with queue-specific roles, then execute tenant-scoped work. They do not receive a universal business-table bypass. Quarantine unknown webhook mappings; never choose a default store.

These are proposed safeguards informed by PostgreSQL's documented RLS semantics, not a claim that RLS contains arbitrary service compromise. See [ADR-002](decisions/adr-v2-002-tenancy.md) and the [PostgreSQL policy reference](https://www.postgresql.org/docs/current/ddl-rowsecurity.html).

## 3. Customer identity, money and historical snapshots

Customer accounts are keyed by store. The same email/phone in two stores may represent separate accounts; one account's reset/session cannot authorize the other. Guest carts/orders use opaque, scoped capabilities and verified recovery flows, not a phone-number lookup. Merchant/staff identities are a separate authority in Platform.

Commerce order creation snapshots product description/SKU, selected options, unit amounts, tax/discount/shipping components, currency, address/contact and policy version. Later product/address edits do not rewrite an existing order. Product references remain useful, but historical rendering must not depend on a product still existing.

Represent monetary values in integer minor units with currency and documented rounding. Record order obligation, required prepayment, collected/refunded amounts and outstanding COD via immutable commercial entries/allocations. Do not infer all semantics from one paid boolean. Duplicate provider event IDs and duplicate commercial effects are different problems: two callbacks can describe the same capture/refund, so recognition uniqueness includes provider account, transaction/effect identity and domain operation.

Subscription invoices and payment allocations use the same accounting principles in **Platform's separate ledger**, with unique invoice-cycle/charge-intent constraints. A Kashier receipt cannot match a Commerce order operation. Do not build a full accounting ERP, but never overwrite accepted financial history to repair a mistake; append reversals/corrections with reasons.

Shipping owns a delivery snapshot linked to its Commerce order. Carrier-delivered, customer-collected COD and carrier-remitted funds are distinct facts. Shipment retries reuse the authorized operation; exceptions preserve the order and audit trail.

## 4. Deliberate duplication — not two writable truths

| Logical data | Authoritative source | Where a copy appears | How it is written/updated | Consistency and recovery |
|---|---|---|---|---|
| Tenant status/entitlement | Platform | Small Commerce/Experience/Messaging eligibility projections; active AI job eligibility | Platform transaction + outbox; consumers atomically apply version and dedup | Eventual; proposed 60-second authority cache maximum for ordinary staff actions; current sensitive checks; rebuild through owner snapshot/version contract |
| Host to tenant/generation | Platform domain registry | Gateway/router cache | Narrow authenticated resolver plus activation/revocation invalidation | Proposed max30-second cache; expiry without refresh fails closed. No cached HTML bypasses the gate |
| Membership/grants | Platform | Short-lived claim/cache in callers | Signed claims plus invalidation and fresh authority checks | Removal is not instantly global; sensitive publish/refund/send/provider-connect checks use current authority |
| Public product view | Commerce | Public renderer's bounded cache; optional later Experience/search projection | Public API response with version; future outbox updates | Display may lag. Checkout always reprices/reserves at source. Clear/rebuild cache; never write price back |
| Product/address at sale | Commerce catalog/customer current state | Commerce order and shipment snapshots | Copied inside local order/fulfillment transaction | Deliberately immutable history, not a synchronization defect |
| Provider result | Provider report, interpreted by owning domain | Receipt/evidence and recognized commercial entry in the same owner DB | Intake persists evidence; trusted processor deduplicates and posts effect | Local application atomic, external arrival eventual; reconcile against provider records |
| Theme file revision | Experience | Object blobs, immutable manifest, renderer/CDN asset copies | Upload complete blobs, verify hashes, then commit metadata; publish exact revision | Metadata is authoritative pointer. Copies immutable; cache by tenant/revision/renderer version; retained manifests allow rebuild |
| AI source and result | Experience source; AI owns execution | Expiring AI input snapshot/proposal; Experience job/proposal status projection | Job-scoped source read, versioned result outbox, idempotent proposal acceptance | Base-revision conflicts never silently overwrite human edits; replay result/status query recovers missing projection |
| Social message | Messaging | Dashboard view, attachment cache and provider counterpart | Verified inbound receipt or authorized local send intent | Our DB owns local inbox view; provider owns delivery fact. Preserve channel IDs and reconcile only where API supports it |
| Trust subject/outcomes — future | Commerce adjudicated facts; Trust owns score | Trust evidence and Commerce's frozen assessment/policy snapshot | Outbox with outcome versions/corrections; restricted assessment API | Eventually current across stores; frozen checkout decision remains explainable |

Owner snapshots used for projection rebuild carry a watermark/version. Buffer or replay subsequent outbox events; never assume a snapshot and concurrent event stream are automatically aligned. A smaller Grad 1 consumer can re-fetch an aggregate on a detected version gap. Do not discard missing-version facts silently.

## 5. Writing data needed in multiple databases

There is no client-side two-database write and no distributed SQL transaction.

1. Platform writes a new tenant and an outbox record in one transaction.
2. Its dispatcher delivers TenantCreated independently to Commerce and Experience with stable event ID and tenant/version.
3. Each consumer verifies producer/action, establishes tenant context and commits its initialization, dedup record and StoreCapabilityReady outbox together.
4. Platform records capability readiness. A duplicate has no duplicate effect; a missing/failed consumer remains visibly preparing and retries.
5. Messaging and AI initialize on first authorized use or a lifecycle fact. Their availability does not block account creation or basic store setup.

This same mechanism maintains lifecycle/entitlement projections. It is at-least-once delivery plus idempotent effects, not exactly-once transport. Delivery status is stored per recipient; one unavailable consumer must not block the other.

For data inside one owner database, prefer a local transaction. There is no benefit in turning every module transition into a network event. Provider calls and object uploads remain outside those transactions even when their metadata shares the database.

## 6. Theme/object storage integrity

Store files under generated tenant/revision/content-hash keys; never concatenate arbitrary merchant paths into filesystem or object-access authority. Manifest entries map allowlisted logical paths to immutable objects, sizes, MIME types and hashes. Draft heads are mutable pointers; revision contents are not.

Upload to staging/private storage, enforce byte/type limits, validate/transform eligible assets, verify all required objects exist, then commit a revision marked ready. That transaction conditionally checks the expected draft head, inserts the revision and updates the head together; a pre-upload check alone is insufficient. A concurrent change rolls back the metadata transaction and returns conflict. Validation evidence binds manifest hash, validator/template version and outcome. Publication atomically checks readiness and expected publication generation, updates pointer/generation and writes ThemePublished. CDN propagation is eventual; the publication transaction does not switch every browser globally at once.

If object upload succeeds and metadata commit fails, unreferenced objects are later garbage-collected after a grace period. If validation/publish fails, the previous publication remains live. GC traverses published revisions, drafts, pending jobs and retained rollback roots; never delete referenced blobs merely by age. Backup restoration must recover matching DB manifests and objects; recheck hashes before serving.

Personalized responses, drafts, previews, DMs and customer data are never publicly cached. Published public assets may be immutable cached copies; access revocation applies to routing/private content, not a claim that already downloaded public files can be recalled.

## 7. Storage technology decisions

| Technology | V2 use |
|---|---|
| PostgreSQL | All five authoritative databases, indexed transactional records, JSONB provider metadata, durable work and deduplication |
| pg + SQL migrations | Parameterized owner-local repositories; explicit transaction scopes; versioned service migrations |
| R2/S3-compatible objects | Theme sources/artifacts, product assets and private retained media with separate credentials/namespaces |
| MongoDB | Not needed for initial messaging envelope/history; adding it would add another backup/query/migration discipline |
| Redis | Deferred cache/coordinated queue option, never journal/stock/payment truth. Separate durable-queue Redis from disposable caches if adopted |
| Meilisearch | Deferred rebuildable, tenant-filtered search index only when PostgreSQL search proves inadequate |
| pgvector | Deferred owner-scoped retrieval projection for an implemented AI use case; tenant filtering precedes model context assembly |
| Broker | Deferred transport, not a new data owner. Existing event IDs/outbox contracts survive later adoption |

## 8. Indexes, migration and recovery gates

Start with workload-driven tenant-leading indexes: catalog slug/SKU, order creation/status, inventory variant, conversation updated time, message conversation/time, provider account/reference, operation eligibility/lease. Use keyset pagination for message/order histories, bounded queries, row locks or conditional stock updates and deterministic lock ordering. Measure large-tenant skew; no unconditional claim that either schema layout is faster.

Budget the total PostgreSQL connection count across all APIs, workers and replicas; no pool per tenant. Queue scheduling scans need bounded batches, fair tenant/account allocation, retention cleanup and metrics. Long backfills must not monopolize checkout locks.

Each service applies expand/backfill/contract migrations with a serialized migrator, lock timeouts and resumable backfills. Old/new workers and event contracts remain compatible during rolling changes. Do not rely on application rollback to reverse destructive schema changes.

If V1 is only documentation, there is no live tenant data migration. If schemas already contain data, first inventory them, map tenant identifiers, create tenant-qualified constraints/RLS, backfill per service with count/hash checks, then use a bounded write freeze/delta catch-up and verified routing cutover. Retain a read-only rollback source until reconciliation; never drop old schemas as part of this documentation task. Dedicated tenant placement later requires a similarly explicit export/replay/ownership-transfer protocol.

Restore tests include database/object/key recovery, tenant isolation and outbox/provider reconciliation. Whole-instance backups are not tenant-selective restore. A future tenant restore must reconstruct its records and reconcile subsequent external effects; rewinding a database cannot undo a charge or sent DM.

Retention/deletion rules must distinguish financial/audit obligations from DM bodies, AI snapshots and provider raw payloads; exact periods are OPEN. Use restricted, encrypted exports, auditable deletion workflows and documented backup expiry. A plain phone hash is not anonymity; future Trust uses keyed pseudonyms and an explicit privacy review.
