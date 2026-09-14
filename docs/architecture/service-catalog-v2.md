# Architecture V2 — service and runtime catalog

Status: proposed. [Architecture and decisions](architecture-v2.md) govern this catalog. Endpoint names below are interface designs, not implemented routes or copies of vendor API paths. External routes pass through trusted ingress/Gateway; `/internal` routes are private and workload-authorized.

## 1. Boundary rules

One service owns each record, database, migration and business transition. Modules within one service may participate in its local transaction; no other service may import its repositories or connect to its database. Services share versioned contracts and small infrastructure utilities, not mutable domain models.

The five business services below are Grad 1 targets. Workers and rendering are separately counted runtimes. Sensitive writes require fresh authority checks; ordinary reads can use explicitly bounded tenant/membership projections. A service does not call Platform on every database query.

## 2. Platform Control

**Purpose:** operate the platform and establish who may act for which store.

**Modules and features:** merchant/staff authentication and recovery; sessions; tenant registry and lifecycle; memberships and fine-grained employee grants; platform-admin identities and MFA; domain claims/verification/routing; subscription plans, invoice cycles, consent, platform-money entries and entitlements. Identity and Tenancy are not separate services. Billing and Domains have distinct process/secret permissions within Platform.

**Owned data/storage:** `platform_db` (PostgreSQL). Global identity and normalized host registries have explicit scoped access. Tenant memberships, subscriptions/invoices and domain associations are tenant-controlled records. Kashier execution evidence and encrypted token references belong to the Billing module; no shopper-order ledger is stored here.

| Major interface | Responsibility |
|---|---|
| `POST /auth/register`, `/auth/sessions`, `/auth/refresh`, `/auth/logout` | Merchant/staff lifecycle; separate platform-admin validation profile |
| `POST /stores`, `GET /stores/{id}/provisioning` | Create idempotently; report capability initialization independently |
| `/stores/{id}/memberships`, `/roles`, `/permissions` | Invite/revoke staff and grant bounded capabilities |
| `/stores/{id}/domains`, `/domains/{id}/status`, `DELETE /domains/{id}` | Claim, inspect gates and revoke; no arbitrary host activation |
| `/billing/plans`, `/billing/subscription`, `/billing/invoices`, `/billing/payment-operations` | Platform subscription obligations and Kashier hosted/consented charge intents |
| `/webhooks/platform-billing/kashier` | Provider-specific verification and append-only receipt ingestion |
| `GET /internal/routing/resolve`, `POST /internal/authorization/check` | Minimal directory/authority results, not global table export |
| `/admin/tenants`, `/admin/billing`, `/admin/operations`, `/admin/support-grants` | Platform-owner operations and explicitly audited support scope |

**Dependencies:** DNS/certificate provider, Kashier through its Billing workers, email delivery adapter for identity messages. Sync login/authority/directory APIs; async lifecycle facts, domain jobs and billing execution. Registration does not wait for Meta or AI.

**Publishes:** `TenantCreated`, `TenantStatusChanged`, `MembershipChanged`, `DomainActivated`, `DomainRevoked`, `SubscriptionEntitlementChanged`. **Consumes:** `StoreCapabilityReady` from Commerce/Experience; verified Kashier observations are local inputs, not another service's financial truth.

**Failure/scaling boundary:** slow certificate issuance or collection must not occupy authentication workers. New login/sensitive authorization fails safely during authority failure. Existing ordinary sessions have bounded validity; billing outages leave invoices pending. Scale API, domain reconciliation and Billing pools independently. Extract Billing only for independent financial operations/team/security needs.

**MVP/later:** onboarding, staff, domain automation and simple subscription invoice/payment lifecycle now. Complex dunning, plan/proration rules, enterprise SSO and independent Billing service later.

## 3. Commerce

**Purpose:** sell and fulfill a store's products with coherent stock, order and customer-money state.

**Modules/features:** catalog/categories/variants/pricing, inventory and reservations, carts, store-local customer identity/recovery, guest capabilities, checkout, order snapshots/status, customer commercial journal and payment allocations/refunds, fulfillment/shipping and COD balances. Payment adapters Paymob/Fawry; Shipping adapters Bosta/J&T. These are modules and isolated workers, not a generic Integrations business service. Theme authoring, social conversations, merchant credentials and platform billing are excluded.

**Owned data/storage:** `commerce_db` (PostgreSQL). Customer credentials and sessions are scoped by store. Product media uses a Commerce-owned object namespace. Order snapshots and shipping request snapshots deliberately preserve historical information. Payment/shipping credentials and evidence are restricted module records in this database.

| Major interface | Responsibility |
|---|---|
| `/products`, `/categories`, `/variants`, `/inventory` | Authorized merchant catalog/stock management |
| `GET /public/catalog`, `/public/products/{id}` | Allowlisted public view models with product versions; never customer/secret fields |
| `/customer/accounts`, `/customer/sessions`, `/customer/recovery` | Store-local account lifecycle; explicit customer JWT/session profile |
| `/carts`, `/carts/{id}/items`, `POST /checkout-handoffs` | Scoped guest/account carts; issue one-use trusted-origin transfer |
| `POST /checkouts`, `GET /orders/{id}`, `GET /payment-operations/{id}` | Idempotent local order/reservation creation; pending payment result polling |
| `/orders/{id}/refunds`, `/orders/{id}/fulfillments` | Authorized durable compensation/dispatch requests |
| `/connections/payments`, `/connections/shipping` | Configure merchant-owned providers through write-only secret intake |
| `/webhooks/commerce/paymob`, `/fawry`, `/bosta`, `/jt` | Account-bound receipt ingestion; J&T enabled only after verified contract |
| `POST /internal/events` | Idempotent consumption of authorized Platform facts |

**Dependencies:** Platform for fresh sensitive authority checks and lifecycle projections; payment/shipping providers through isolated workers; object storage for product media. No synchronous dependency on Experience, Messaging or AI to confirm an order. No cross-service database access.

**Publishes:** `CatalogChanged`, `OrderCreated`, `CustomerPaymentApplied`, `OrderFulfillmentEligible`, `FulfillmentChanged`, `StoreCapabilityReady`; future adjudicated `CodOutcomeRecorded/Corrected`. **Consumes:** `TenantCreated`, `TenantStatusChanged`, `SubscriptionEntitlementChanged`, `MembershipChanged` for invalidation. Provider observations and shipment jobs remain local.

**Sync/async:** inventory/order invariants are local ACID. Payment intent creation returns a pending operation; the execution worker obtains the session/reference asynchronously. Shipping, expiry, callback interpretation and reconciliation are background work. The shopper can poll without holding a database lock or worker open on an external API.

**Failure/scaling boundary:** unknown provider outcome is visible and recoverable. Payment/shipping pool exhaustion cannot consume the checkout connection budget. Shared database contention remains a risk. Scale public reads/API and provider workers separately; do not split stock from orders just to add services.

**MVP/later:** required transactional core, guest/registered checkout, Paymob/Fawry targets and first Bosta shipping flow now. J&T after provider contract, advanced returns/discounts/reporting and Trust consumption later. Model amounts/allocations now so future partial prepayment is possible without a binary paid/unpaid redesign; risk-driven policy is not active in Grad 1.

## 4. Store Experience

**Purpose:** own the complete merchant theme lifecycle and serve safe public presentation artifacts.

**Features:** 2–3 ready themes; HTML/CSS/declarative schema/config code editor; draft saves and optimistic concurrency; immutable revisions and file manifests; validation jobs; revision-pinned preview; publish, rollback, retention and public manifest API. Accept AI patches through the same authorized revision path. Separate authoring API, restricted validator and public rendering runtimes.

**Owned data/storage:** `experience_db` (PostgreSQL) owns template definitions/versions, tenant draft heads, revision ancestry/hashes, validation evidence, publication pointer/generation and AI job associations. R2/S3-compatible storage holds private source artifacts and approved public assets under Experience ownership. AI is not a second theme authority.

| Major interface | Responsibility |
|---|---|
| `GET /themes`, `POST /stores/{id}/theme-initializations` | Ready-theme selection and idempotent capability initialization |
| `/theme/files`, `POST /theme/revisions` | File/config reads and expected-base saves; no arbitrary filesystem paths |
| `POST /theme/revisions/{id}/validations`, `/previews` | Queue validation and issue revision-bound preview capability |
| `POST /theme/publications`, `/theme/rollbacks` | Exact revision/validator checks, publish permission, atomic pointer update |
| `POST /theme/ai-jobs`, `GET /theme/ai-jobs/{id}` | Authorized edit request and job status projection |
| `POST /theme/proposals/{id}/accept` | Apply permitted patch against its base; conflicts require review |
| `GET /internal/publications/{tenant}` | Minimal active manifest/render contract; no drafts or secrets |
| `POST /internal/theme-proposals`, `/internal/validation-results` | Workload- and job-scoped result acceptance, not autonomous publication |

**Dependencies:** Platform authority/lifecycle; object storage; Commerce public catalog API; AI for asynchronous optional assistance. Renderer receives immutable approved artifacts and bounded public data. Template initialization can retry independently of account creation.

**Publishes:** `ThemePublished`, `ThemeEditRequested`, `StoreCapabilityReady`. **Consumes:** `TenantCreated`, `TenantStatusChanged`, entitlement changes, `ThemePatchProposed`, `AIJobFailed`. It does not maintain a second authoritative domain registry. Catalog caching is optional/read-only, not a competing product store.

**Failure/scaling boundary:** AI failure leaves direct editing available; validation failure leaves live revision intact. Authoring failure need not invalidate cached published artifacts, subject to routing freshness. Resource-limit renderer/validator independently; checkout does not execute theme content.

**MVP/later:** required file/config editing, safe rendering and actual AI proposal integration now. Plugin ecosystems, arbitrary scripts/builds, richer theme grammar and a theme marketplace are later reviews, not enabled defaults.

## 5. Social Messaging

**Purpose:** a real unified private-message inbox, with replies delivered via the original social account.

**Features:** WhatsApp/Instagram connection lifecycle; OAuth/embedded signup state; account ownership validation; encrypted tokens; inbound receipts; conversations and channel identities; history/unread/assignment state; staff replies; outgoing policy/rate controls; attachment metadata and safe media handling; delivery receipts, revocation, recovery and audit. Channel adapters belong here, not in Commerce or a generic Integrations service.

**Owned data/storage:** `messaging_db` (PostgreSQL) for relational identity/conversation/delivery records, indexed histories and bounded JSONB provider metadata. Private object storage for approved retained media. No MongoDB requirement. Do not retain every raw payload indefinitely.

| Major interface | Responsibility |
|---|---|
| `/channels/connections`, `/channels/oauth/callback`, `/channels/{id}/disconnect` | Single-use session/store-bound connection and revocation |
| `/webhooks/social/whatsapp`, `/webhooks/social/instagram` | Provider handshake/authentication, account resolution and durable receipt |
| `GET /conversations`, `/conversations/{id}/messages` | Tenant/channel-scoped paginated inbox/history |
| `POST /conversations/{id}/replies` | Durable authorized send intent, payload-bound idempotency |
| `/conversations/{id}/read-state`, `/assignment` | Staff-specific read and responsibility state |
| `GET /messages/{id}/delivery`, `/connections/{id}/health` | Queued/accepted/delivered/failed/unknown visibility |
| `/internal/events` | Platform eligibility and revocation consumption |

**Dependencies:** Platform for account authority; Meta channel APIs/webhooks; controlled media fetch/storage. Ordinary receive/reply has no Commerce or AI dependency. Future AI ordering must use explicit Commerce APIs with confirmation, not direct DB writes.

**Publishes:** `MessageReceived`, `MessageDeliveryChanged`, `ChannelConnectionChanged`. In Grad 1 these feed its dashboard/status surface; no dummy subscribers are needed. **Consumes:** tenant/entitlement/membership changes. Normalized channel observations are internal inputs.

**Sync/async:** dashboard read and send acceptance are synchronous; actual send, receipt handling, media fetch and status recovery asynchronous. Polling is the Grad 1 dashboard baseline; SSE/WebSockets are optional UX evolution and never the durable delivery mechanism.

**Failure/scaling boundary:** per-account dispatch serialization/quotas prevent one account's rate limits blocking others. Account revocation stops unsent work. Timeout does not authorize blind resend. Scale webhook intake and dispatch separately; persist first, acknowledge second.

**MVP/later:** real incoming text and merchant replies for both named channels; safe unsupported-media indication/reference instead of broken rendering. Confirm media matrix and retention before launch. More channels, richer media, backfill and AI conversations later. Do not promise complete historical DM import across providers.

## 6. AI Assistance

**Purpose:** bounded model execution for actual merchant theme-edit proposals, independently operated by the AI team.

**Features:** idempotent job acceptance/status/cancellation, bounded prompt/file context, provider abstraction, patch generation/explanation, execution quotas, evaluation and safety telemetry. No authority to publish themes, charge money, send DMs or modify business databases.

**Owned data/storage:** `ai_db` (PostgreSQL) owns jobs, attempt history, minimal conversation/proposal records, model/config version and evaluation metadata. Temporary scoped source snapshots expire. Large temporary artifacts use its private object namespace if necessary; no vector database is required for bounded file editing.

**Interfaces:** `POST /internal/theme-edit-jobs`, `GET /internal/jobs/{id}`, `POST /internal/jobs/{id}/cancel`; job-scoped Experience revision read/proposal submission contracts. No public general-purpose model/tool endpoint. Experience accepts the user's request and records a durable dispatch; AI owns execution status.

**Dependencies:** Experience file contracts and selected model runtime/provider. Sync job acceptance/status; async inference and result delivery. **Consumes:** `ThemeEditRequested`, tenant cancellation/status signals relevant to active work. **Publishes:** `ThemePatchProposed`, `AIJobFailed`; duplicate delivery must not produce duplicate accepted revisions.

**Failure/scaling boundary:** model timeout/cost exhaustion cannot break direct editing or checkout. Separate API/orchestration from inference execution; bound tokens, file count, bytes, concurrency and wall time. Model tools never provide arbitrary shell, package install, network or credential access.

**MVP/later:** actual theme patching/evaluation now. Retrieval, conversational orders, voice, buying agents, CRM and autonomous campaigns later with new permissions/confirmation ADRs. Model selection and data-processing terms remain OPEN; a stub is not feature completion.

## 7. Supporting runtimes and enforced permissions

These are deployment responsibilities, not additional service databases. Processes may reuse a domain image with different entrypoints, credentials and resource limits. Do not merge secret scopes merely to reduce Compose lines.

| Runtime | Database/data permission | Secrets and network | Failure behavior |
|---|---|---|---|
| Edge/TLS ingress + thin Gateway | No domain DB; narrow Platform directory/authority APIs | Own workload/delegation keys; validated upstream host metadata | Unknown/conflicting tenant fails closed; routing gate precedes cached HTML |
| Merchant web / platform-admin web / trusted checkout-account web | No direct DB; owner APIs | Host-specific sessions; no provider secrets; admin authority separate | Distinct auth profiles/origins; shared source packages do not share privilege |
| Platform API/domain processor | Platform-owned authorized business records | No provider decryption key in ordinary auth/API processes | Fresh sensitive checks fail safely; outbox work survives process restart |
| Platform Billing execution worker | Claim authorized Billing operations; append execution evidence; no membership or commercial journal writes | Kashier-only key/token scope and allowlisted egress | Unknown charge pending reconciliation; independent pool |
| Platform domain worker | Domain jobs and minimal registry transitions through constrained paths | DNS/certificate token limited to platform zone; no money keys | Retries independent gates; does not activate from a single success flag |
| Commerce API/domain processor | Tenant-scoped commercial records and accepted evidence; ledger transition authority | No provider credential decryption key | Applies money/order/outbox atomically; budgets connections separately |
| Commerce payment execution workers | Authorized payment/refund operations and append-only evidence; no direct order/journal edits | Only relevant Paymob/Fawry secret scope | Provider timeouts consume bounded worker capacity, not checkout threads |
| Commerce shipping execution worker | Authorized shipments/snapshots and append evidence; no money journal edits | Shipping credentials only; no payment secrets | Shipment remains recoverable; cannot invent dispatch eligibility |
| Callback intake per owner | Restricted account mapping read and receipt append/dedup; no business-state mutation | Provider-specific verification material only | Acknowledge only after durable receipt; quarantine unmapped/invalid observations |
| Experience API/domain processor | Experience metadata/pointers with user authorization | Restricted object access; no Commerce DB or provider keys | Exact revision validation and publish permissions enforced |
| Validator/template evaluator | No domain DB; job-scoped input/output capability | Read-only temporary files, no arbitrary egress, no privileged cookies | Bounded CPU/memory/time/output; termination leaves live revision unchanged |
| Public renderer | No domain DB; approved manifest/public catalog contracts | No secrets/customer account data; controlled public assets only | Serves only active host and approved revision; no merchant server execution |
| Messaging API/processor and delivery workers | API business state; delivery worker narrowly scoped outbound operations/evidence | Channel-specific tokens only in connection/delivery components | No payment credentials; per-account policy/quota and ambiguous-send review |
| AI API/orchestration and inference workers | Only ai_db job state; scoped Experience contracts | Model credential only where needed; never platform/provider master keys | Quotas/cancellation; failures cannot publish |
| Service-local dispatchers/outbox delivery | Queue-specific claim access across tenants; no blanket business RLS bypass | Own service workload identity, fixed destination allowlists | Establish tenant context from authorized persisted job; lease expiry/replay |
| One-shot migration/backup roles | Only their service database; privileged maintenance access explicitly controlled | Not injected into API/worker deployments | Serialized migrations; backups checked for complete data, not filtered RLS exports |

Provider setup uses a separate write-only credential intake path to encrypt secrets without making them retrievable by ordinary handlers; workers alone have decryption authority. A callback requiring a shared verification key may retain charge-capable authority depending on the provider's key design: document that residual risk and isolate intake accordingly. Distinct process names alone do not establish least privilege.

Minimum runtime accounting is five business APIs **plus** gateway/ingress, three trusted web entrypoints, renderer, validator, provider execution pools, callbacks, domain/outbox processors, Messaging dispatch and AI inference. Logical pools can share a process only where their secret/resource scopes genuinely match. Demo deployment is not a five-container system or a highly available cluster.

## 8. Future Trust/Risk service — not deployed in Grad 1

Own `trust_db`: minimized keyed phone pseudonyms, source-attributed COD outcome evidence/corrections, policy/score versions and assessment audit. Receive only adjudicated Commerce outcomes through outboxes. Expose a restricted checkout-assessment API with sufficient-evidence/unknown status; merchants receive the permitted computed result, not raw histories or an unrestricted phone search.

No shared customer accounts are created. Future Commerce requests an assessment before its local checkout transaction, freezes the chosen policy/prepayment requirement on the order, and records fallback behavior. Consent/legal basis, retention, poisoning resistance, disputes, phone recycling and unavailable-score policy are release gates. See [ADR-010](decisions/adr-v2-010-future-trust.md).
