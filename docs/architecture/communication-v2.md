# Architecture V2 — communication and workflows

Status: proposed. [Architecture](architecture-v2.md) · [Services and runtime permissions](service-catalog-v2.md) · [Data ownership](data-ownership-v2.md) · [Sequence diagrams](diagrams-v2.md)

## 1. Three mechanisms, not one synchronous chain

| Mechanism | Use | Completion means |
|---|---|---|
| Synchronous HTTP/JSON | Authorized CRUD, login, status, public catalog, authority/directory lookup | The owner committed a local effect or returned a bounded read; provider completion is not implied |
| Owner-local durable job | Payment/shipping execution, webhook processing, reservation expiry, domain checks, validation, AI inference | Job intent is persisted. Actual execution may still be queued, unknown or failed |
| Asynchronous event over authenticated HTTP | Tenant/entitlement facts, capability readiness, AI request/result and publication invalidation | Producer committed a fact; each consumer must persist/deduplicate its own effect |

No RabbitMQ/Kafka in Grad 1. PostgreSQL queues/outboxes are durable delivery infrastructure, not fire-and-forget promises. Choose one maintained PostgreSQL job approach or a small audited claim/lease implementation during implementation; do not casually combine several job frameworks. REST transport can carry asynchronous messages; REST-only synchronous business processing is rejected.

## 2. Synchronous call map

| Caller | Receiver | Contract and limits |
|---|---|---|
| Trusted dashboards/customer web | Gateway → owning service | User/customer/guest profile, validated tenant and resource/action scope; owner enforces authorization |
| Gateway | Platform directory | Normalized public host → active tenant/generation; proposed maximum routing-cache age 30 seconds |
| Owning services | Platform authorization | Sensitive publish, refund, staff/provider/domain changes and send authorization; fail safely if current authority unavailable |
| Public renderer orchestration | Experience publication API | Approved revision/manifest only; draft access requires separate preview capability |
| Public renderer orchestration | Commerce public API | Bounded public catalog model; no account/order/secret fields; checkout does not trust displayed snapshots |
| Experience | AI job/status API | Idempotent background acceptance/status; no long synchronous inference request |
| AI | Experience job-scoped revision API | Exact approved source snapshot, size limits and no broader tenant enumeration |
| Provider execution workers | Their configured provider | Bounded timeout and allowlisted destination after local intent commit; never an open SQL transaction |
| Future checkout | Trust assessment | Bounded query before order transaction; freeze policy/result and explicit unavailable-score behavior |

Never route internal service calls through a browser endpoint that permits forged tenant headers. Internal services may use private service DNS and receiver-validated identities; they do not require Gateway as a bottleneck for every job. Public rendering orchestration fetches data; the untrusted template evaluator itself has no network access.

## 3. Event and durable command map

| Producer / authoritative owner | Message | Consumer | Consumer effect |
|---|---|---|---|
| Platform | TenantCreated | Commerce, Experience | Idempotent owner-local setup and capability-ready fact |
| Commerce / Experience | StoreCapabilityReady | Platform | Provisioning progress; does not give Platform write access to their DBs |
| Platform | TenantStatusChanged | Commerce, Experience, Messaging, active AI orchestration; routing invalidation | Update versioned eligibility/cancel unstarted work; safety checks still enforce freshness |
| Platform | SubscriptionEntitlementChanged | Commerce, Experience, Messaging, AI where needed | Small entitlement projection; no copied subscription ledger |
| Platform | MembershipChanged | Gateway/owner authority caches | Invalidation, not permission grant from an untrusted event |
| Platform | DomainActivated / DomainRevoked | Routing/renderer cache control | Invalidate mapping generation; Platform remains sole registry owner |
| Commerce | CatalogChanged | Optional public cache invalidation; later search/AI read models | Invalidate/rebuild public projection, never modify authoritative stock |
| Commerce | OrderCreated / CustomerPaymentApplied / OrderFulfillmentEligible / FulfillmentChanged | Owner-local dashboard/notification modules; future analytics/Trust as authorized | Commercial facts emitted only after domain validation |
| Experience | ThemePublished | Renderer/publication cache | Select new publication generation; immutable artifact cache remains reusable |
| Experience | ThemeEditRequested | AI | Deduplicated execution job with scoped source/base revision; command, not theme publication |
| AI | ThemePatchProposed / AIJobFailed | Experience | Proposal/job status projection; no direct draft overwrite or publish |
| Messaging | MessageReceived / MessageDeliveryChanged / ChannelConnectionChanged | Local dashboard notifications; later authorized automation | Conversation/delivery updates; ordinary inbox has no Commerce dependency |

Do not deploy fake consumers merely to draw arrows. A publication/route cache may use a local invalidation hook or authenticated control endpoint with bounded TTL fallback. Domain events without a current external consumer remain owner-local records/contracts until required.

Payment, shipping and subscription execution are **local durable commands within their owner**, not cross-service requests to a generic Integrations service. External callbacks are untrusted observations until verified and recognized by that owner.

## 4. Outbox, inbox and retry contract

Every cross-service envelope contains event/command ID, schema version, producer, tenant where applicable, aggregate ID/version, occurred-at, correlation/causation ID and minimal payload. Transport credentials authenticate the sender; envelope fields alone do not. Consumers allowlist producer/message/action combinations and validate payload schemas.

1. Producer commits state and outbox together.
2. Dispatcher claims eligible per-recipient deliveries using bounded leases; no global ordering claim.
3. Deliver over authenticated HTTP with timeout and a stable ID. Consumer commits deduplication plus its effect and outgoing facts atomically before acknowledging.
4. Lost acknowledgment causes redelivery. Duplicate acceptance returns the recorded outcome without repeating the business effect.
5. Retry only classified transient failures, with bounded exponential backoff/jitter and provider Retry-After handling where documented. Expired leases recover crashes; they do not make a remote side effect unique.
6. Persist attempts, next eligibility, errors and dead-letter/review status. Operators replay the original identity through the same transition path, not by editing tables or generating a new charge key.

An idempotency key is scoped to tenant, business domain, operation and request hash. Reusing a key with another payload returns conflict. Retention must exceed documented retries/callback windows; monetary recognition uniqueness remains after short-lived API response caching expires.

Aggregate versions and state transition rules prevent stale status regression. Detect gaps; re-fetch owner state or replay missing events. Different operations may legitimately complete out of order. Refunds and payment corrections are new facts, not a stale-event rule that discards all post-success messages.

**External ambiguity:** after a timeout or worker crash, first use documented provider idempotency/reference lookup. If neither can establish the outcome, mark unknown and require review; do not automatically re-charge, re-create a shipment or re-send a DM. Local locks, leases and deduplication cannot prove that a remote system did not already act.

## 5. Authentication and tenant resolution

| Request class | Tenant/authority source | Enforcement |
|---|---|---|
| Merchant/staff dashboard | Authenticated Platform identity plus chosen authorized membership | Reject path/token mismatch; resource owner checks read/write/action grant |
| Platform administrator | Separate admin identity/session and explicit operation/support scope | MFA, current authority and audit; no general Commerce RLS-bypass connection |
| Anonymous public storefront | Verified normalized host in active Platform directory | Strip client-supplied forwarding/context headers; trust only configured edge; unknown host fails closed |
| Customer account | Commerce-issued store-bound account/session | Separate issuer/audience/type validation; same email in another store has no authority |
| Guest cart/order | Opaque tenant-bound capability | Expiry/ownership checks, one-use handoff; no phone-number authentication |
| Provider callback | Verified provider endpoint/account/operation mapping | Tenant claimed in payload/path is not authoritative; quarantine missing/conflicting mappings |
| Background job | Authorized tenant/action persisted by owning domain | Fresh workload identity; current eligibility for unstarted sensitive actions; never reuse an expired browser token |

Platform signs merchant/admin tokens asymmetrically with distinct validation profiles. Commerce owns the store-customer token profile. Validate issuer, audience, expiry, token type, allowed algorithms and key IDs. Gateway delegation binds original subject, tenant, audience, action and provenance; downstream callers cannot elevate it. Each workload has its own key identity and receiver-enforced action allowlist. No universal internal signing secret. Apply JWT best-current-practice validation, not merely signature success. [IETF JWT BCP](https://www.rfc-editor.org/rfc/rfc8725.html)

Use TLS on external and internal authenticated hops; private networking alone is not authentication. Initial deployment uses explicit application workload credentials over encrypted connections, without a service mesh; managed mTLS can replace that transport identity later. Browser cookies are host-only, Secure/HttpOnly, with CSRF/Origin controls and separate customer/staff/admin profiles. Refresh rotation/reuse detection and sensitive current-authority checks limit stale access.

Authorization is a point-in-time decision, not globally atomic with a remote revocation. Proposed ordinary authority cache max60 seconds; high-risk operations check current authority and bind the exact action/revision/amount. An accepted irreversible operation may already execute while revocation propagates; cancellation checks apply before dispatch, not a promise to undo external effects already performed.

## 6. Merchant registration/store creation

Platform commits merchant identity, tenant, owner membership and TenantCreated once under an idempotency key. Return account/store ID and preparation state. Commerce initializes store settings; Experience initializes a selected ready-theme draft. They publish capability readiness independently. Neither creates schemas or writes Platform's database.

The platform subdomain is allocated automatically, but public content is enabled only when the store is eligible and has a validated publication. Show a controlled preparing page before that. AI/Meta/provider onboarding remains optional to account creation and cannot hold a global registration transaction open. Failed setup is retryable; cancellation/suspension records durable desired state.

## 7. Browsing, customer accounts and checkout

Ingress resolves active host/generation before serving storefront HTML. Experience supplies the selected validated manifest; Commerce supplies public product data. Cache public output only within tenant/revision/locale/version keys. Grad 1 defaults to dynamic HTML and immutable asset caching, avoiding a second stale HTML routing path.

Customer login/registration and checkout run on trusted platform origins outside merchant markup. An opaque POST handoff binds cart/store and is single-use/short-lived (proposed 60-second lifetime). Do not put capabilities in logged URLs or referrers. Customer account/session access remains store-local; guest checkout needs no account.

Commerce revalidates items, prices, address/serviceability and permitted payment method. Its transaction reserves stock with concurrency control, freezes amounts/policy, creates a pending order, marks the cart converted once and persists the payment/expiry work. If stock is unavailable, the transaction fails without a provider call. Repeated checkout returns the same logical order for the same accepted key/cart.

COD records the accepted COD obligation and fulfillment policy locally. Online methods return a pending operation; worker-created checkout URL/reference becomes available through status polling. The order is durable even if the browser closes or a provider is unavailable.

## 8. Customer payments and platform subscriptions

### Customer → merchant: Paymob / Fawry

1. Commerce records a fixed authorized operation: store/account, order, amount/currency, method, request hash and external correlation reference.
2. Payment worker claims it with restricted privileges and invokes the provider outside any transaction. Persist definitive execution evidence; ambiguous outcomes remain unknown.
3. Callback intake validates the provider-specific signature/credentials and stored account/reference. Persist receipt before acknowledgment. Browser redirects only show progress.
4. Commerce processor validates recognized amount, currency, provider mode/account and legal state transition. Commit receipt application, unique commercial effect, journal/allocations, order state and fulfillment/notification intent together.
5. Reconciliation compares unresolved operations with provider-supported status APIs/records. Refund/cancel are durable authorized operations with their own idempotency and outcome handling.

Paymob's processed callback distinguishes transaction outcomes and correlation from browser navigation. [Paymob callbacks](https://developers.paymob.com/paymob-docs/manage-callback/transaction-callbacks) Fawry's callback format and payment status progression differ; implement each selected API's exact verification contract, not a generic signature guess. [Fawry notification contract](https://developer.fawrystaging.com/docs/server-apis/payment-notifications/server-notification-v2)

Reservation expiry competes safely with payment application through owner-local locks/state guards. Release once. Payment observed after expiry does not silently restore sold stock: enter paid-needs-resolution, re-evaluate availability, and create a refund/review action as policy permits. An old expired/failed callback cannot erase a recognized collection. Partial refund/chargeback correction remains possible as a new effect.

### Merchant → platform: Kashier

Platform Billing creates invoice/cycle and collection intent with unique subscription-cycle identity. Its worker executes the platform's Kashier account operation; verified receipts are recognized locally into the platform ledger and entitlement decision/outbox. Customer orders are not involved.

Billing owns schedule, consent/agreement references, cancellation, renewal retry/dunning and grace. Kashier offers token charging rather than a managed subscriptions scheduler. V2 therefore uses hosted invoice payment first and enables automatic renewal only with verified consent/account capability and an agreed policy. [Kashier recurring contract](https://developers.kashier.io/docs/accept-payments/recurring)

Separate callback namespaces, operation identifiers, account credentials, ledger records and reconciliation reports by money domain. Mode/account mismatch is quarantined. A success redirect never enables a paid plan. Subscription suspension policy must define which actions stop; it must not silently discard already-paid orders or accepted message/payment work.

## 9. Shipping and COD

Commerce owns eligibility: merchant dispatch approval plus the frozen prepaid/COD policy. Create a durable shipment intent containing an immutable delivery snapshot and amount to collect. Shipping worker invokes Bosta, later J&T, through a capability contract: serviceability/options where supported, create, lookup/track, cancel where supported, label and callback normalization.

Persist provider references and receipts. Domain processing updates fulfillment and outgoing facts; duplicate/out-of-order tracking does not regress a completed state. A carrier cancellation/delivery failure does not delete the order or directly imply a customer-risk event. COD collection and remittance need separate reconciliation evidence.

Bosta documents status webhooks and configurable callback headers; use supported authentication plus status lookup where necessary. Do not invent HMAC support. [Bosta webhook guide](https://docs.bosta.co/docs/how-to/get-delivery-status-via-webhook/) J&T Egypt auth, status map and reconciliation capability require an account-specific contract before implementation. Provider adapter tests must include create-timeout/unknown and late cancellation, not only happy-path shipment creation.

## 10. Unified social inbox

Connection state is bound to initiating staff session, store and a single-use OAuth/signup attempt. Verify account eligibility/asset ownership; enforce one connected provider asset per store mapping by default. Encrypt tokens and support refresh/revocation without putting secrets in the browser or AI context.

Inbound: provider → owner intake → verified durable receipt → tenant/account resolution → deduplicated message/thread update → dashboard read/notification. Partition dedup keys by provider/account identity; preserve original event/message references and out-of-order receipt semantics.

Outbound: staff authorization → conversation's originating channel/account → persist message and send intent → dispatch-time policy/permission/account checks → provider → delivery observation. Queued, accepted, delivered, read, failed, blocked-by-policy and unknown are distinguishable; not every provider supplies every state. Dashboard send acceptance is not proof of delivery.

Instagram professional messaging requires appropriate access and prior customer initiation. [Meta Send API](https://www.postman.com/meta/instagram/folder/uxudqu0/send-api) WhatsApp uses official Business Platform account onboarding; verify current conversation-window/template and opt-in rules for the selected API before enabling send. A connected account is not a license for unrestricted messages. Never scrape personal sessions or silently substitute website chat.

Do not infer a shared customer from a matching social name/phone. An explicit, verified store-customer association can link records later without merging channel identities. Fetch media through an allowlisted, size-limited path with malware/content checks; no arbitrary URL fetching or inline unsafe HTML. Text-first Grad 1 can show supported safe references and explicit unsupported-media status.

## 11. Domain activation and theme editing

Domain workflow: normalize/unique claim → fresh ownership proof → DNS target readiness → certificate/DCV readiness → routing generation activation. Some independent checks run concurrently. Every transition is persisted; poll/reconcile with bounded backoff and actionable reasons. The merchant may need DNS changes or authorized DNS automation; no human platform operator is required for normal progress. Do not equate a successful TLS handshake alone with all onboarding gates.

Removal/suspension revokes routing generation before serving more content; events plus maximum cache age bound stale mappings. Reassignment requires fresh proof and invalidation; old certificates or CNAMEs alone do not transfer ownership. Monitor renewal and detach provider hostname resources safely according to state; the platform subdomain remains the fallback within tenant policy. [Cloudflare onboarding gates](https://developers.cloudflare.com/cloudflare-for-platforms/cloudflare-for-saas/start/getting-started/)

Direct theme edit: authorized expected-base save → immutable candidate revision → validation against exact manifest/version → preview → explicit publish. AI edit: Experience persists ThemeEditRequested → AI runs bounded job → returns proposal/base revision → Experience persists proposal → merchant reviews/accepts → same validation/preview/publish path. The post-upload database transaction atomically compares the expected draft head and inserts the revision/head update/job together. Changing a base concurrently returns conflict; a pre-upload comparison does not prevent a race. Publish/rollback also conditionally checks the expected publication generation; no last-writer-wins overwrite.

Validation and rendering reject scripts/event handlers, executable config, dangerous URL/CSS imports, arbitrary forms, filesystem traversal and unbounded template loops/includes. Platform-generated trusted cart widgets/handoffs have fixed destinations outside editable checkout. Preview is private, revision-pinned, expiring and noindex; published output remains subject to tenant routing policy. See [ADR-004](decisions/adr-v2-004-theme-safety.md).

## 12. Failure and verification matrix

| Failure/test | Required observable result |
|---|---|
| Missing/wrong tenant context, pooled connection reuse | Denied or isolated; no sibling rows/objects/cache entries, including jobs and exports |
| Concurrent checkout/duplicate request | One cart conversion; no oversell; same accepted operation or conflict on changed payload |
| Crash after provider acts, before local record | Unknown/reconciliation state, not automatic new charge/shipment/send |
| Forged, duplicate or out-of-order callback | Reject/quarantine invalid data; persist valid receipt; one commercial effect; legal corrections remain supported |
| Payment after inventory expiry | Visible paid-needs-resolution; no silent fulfillment without stock or duplicate release |
| Provider succeeds while consumer/processor is down | Durable receipt/operation survives; delayed recognition visible; replay/reconcile |
| Receiver commits but response is lost | Redelivery acknowledged without duplicate effect |
| AI timeout, injection or stale base | Failed/blocked proposal or conflict; no privileged action or live theme change |
| Missing artifact or malicious template | Validation/publish denied; previous live revision intact; renderer stays resource-bounded |
| DNS wrong, certificate pending/renewal fails | Pending/actionable status; no unauthorized host routing; retry and fallback |
| Account revocation / messaging policy expiry | Unsent work blocked; status shown; no blind resend of unknown messages |
| Tenant suspension while caches exist | Serving gate rejects within tested freshness budget; sensitive operations check current authority |
| PostgreSQL/host unavailable | Explicit unavailable/degraded state; no claim of HA; restore and external-effect reconciliation exercised |

Log correlation and safe IDs, not tokens, payment credentials, full DM bodies or raw customer addresses. Monitor age of unresolved work, not just HTTP success rate. Manual remediation is an audited authorized workflow through the owner; SQL edits are not the operating plan.
