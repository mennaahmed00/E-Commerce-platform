The review is complete. All six configured specialists completed Round 1 and Round 2. I reused the completed findings and resumed only the three rebuttals interrupted by the usage limit. Repository hashes are unchanged; no implementation code was written.

My lead-architect verdict is **MODIFY before implementation**. Claude’s proposal provides useful starting boundaries, but its tenancy substitution, payment transaction flow, authentication contracts, and theme publishing model need correction.

[requirements.md](D:/codex-arc/docs/reference/requirements.md) remains authoritative. [chat-summary.md](D:/codex-arc/docs/reference/chat-summary.md) supplies historical reasoning. [implementation-proposal.md](D:/codex-arc/docs/reference/implementation-proposal.md) remains an unapproved candidate.

The important requirement findings are:

| Finding | Assessment |
|---|---|
| Schema-per-tenant replaced by shared-schema RLS | Direct conflict with the confirmed isolation model. |
| Published themes serve mutable draft data | The described behavior can violate the required save/preview versus publish/live distinction. |
| Arabic/English and RTL | Confirmed requirements need explicit coverage in frontend and theme acceptance criteria; a language setting alone is insufficient. |
| Storefront-only message box | **OPEN:** requirements mention the MVP box under Meta unified messaging, while the proposal excludes Meta. |
| Separate Identity, raw SQL, database-per-service, MongoDB | Candidate technical choices—not automatically requirement violations. MongoDB is explicitly undecided. |
| Deferring real AI, trust scoring and advanced features | Consistent with the locked Grad 1 scope. Future requirements must remain represented in the evolution plan. |

The most significant debate results were:

| Disagreement | Independent positions and challenge outcome | Lead decision |
|---|---|---|
| Separate Identity | Critic initially favored a Commerce module; other specialists defended credential and platform-administration isolation. Critic revised after challenge. | Separate Identity + Tenancy now. Its security benefit exists in current scope. |
| Separate Messaging | Domain favored a module; several others favored a service. After challenge, the separate-service advocates revised because channel complexity and traffic were speculative. | Basic Inbox module in Commerce for Grad 1. |
| Redis | Limited shared cache/limiter versus local facilities for a singleton deployment. | No default Redis dependency; shared enforcement becomes necessary before replicas. |
| Observability | Platform initially proposed several monitoring products; others challenged their operating cost. Platform revised. | Require diagnostic capabilities; keep specific monitoring products optional. |
| Payment initiation | Worker-first versus synchronous payment-session creation after committing the order. | Bounded synchronous initiation using the same durable operation that workers recover. |
| Tenant permissions | Data proposed tenant roles; backend/domain/critic proposed supplemental RLS; security/platform accepted application enforcement with restricted service credentials. | Tenant-scoped database roles and constrained repositories, with an explicitly trusted service boundary. |

These decisions follow requirements, security and correctness before operational simplicity. They are not a majority vote.

The classifications below apply to the specific decision named; accepting a service boundary does not approve every detail of its proposed implementation.

1. **Decisions we ACCEPT**

| Decision | Selected option, reason and trade-off |
|---|---|
| Microservices architecture | Retain independently deployed business services, as required. Control their number to fit Grad 1. |
| Framework direction | NestJS, Next.js, PostgreSQL and a future FastAPI AI boundary. Update obsolete version pins separately. |
| Identity + Tenancy separation | One service owns merchant authentication, employees, permissions, tenant lifecycle and domain registry. Keeping it separate costs provisioning coordination but keeps its credentials outside public Commerce handlers. Do not split Identity and Tenancy into two services now. |
| Core commerce grouping | Keep catalog, variants, inventory, carts, orders, checkout and per-store customers together. Their immediate consistency requirements outweigh separate scaling benefits at this stage. |
| Integrations boundary | Keep provider credentials and payment/shipping execution in a separate service. Retain payment and shipping as modules within that service. |
| Database-per-service principle | One logical database per retained business service on one PostgreSQL instance. This establishes ownership; separate credentials establish access restrictions. Separate instances are unnecessary initially. |
| REST/JSON contracts | Suitable for public APIs, business commands and internal communication. Neither gRPC nor GraphQL is necessary for Grad 1. |
| No general event broker at MVP | Keep RabbitMQ/Kafka out of the default deployment. Durable background processing remains necessary. |
| Puck-based themes | Keep the visual editor, 2–3 templates and Next.js rendering. Correct publishing semantics without replacing the editor. |
| Object storage | Retain R2 for images/assets, with metadata owned by the relevant domain. It adds an external dependency but keeps binary files outside application containers. |
| Monorepo and basic conventions | Retain pnpm workspaces, validation, pagination, consistent errors, timestamps, integer monetary amounts with currency, and order-item snapshots. Shared packages contain contracts and infrastructure helpers—not shared domain repositories. |

Core Commerce is therefore **not too broad because it contains products, inventory and orders**. The original candidate becomes harder to justify when it also owns merchant credential authority and platform administration. Moving those to Identity addresses that concern without distributing checkout across more services.

2. **Decisions we MODIFY**

| Decision | Architecture V1 selection | Grad 1 cost and future effect |
|---|---|---|
| Storefront Service | Keep Store Design as a distinct module inside Commerce; Next.js renders storefronts. | Removes a deployment and provisioning participant. Theme releases share Commerce deployment until extraction is justified. |
| Messaging Service | Keep Basic Inbox as a distinct Commerce module using PostgreSQL tenant schemas. | Shares Commerce resources, requiring limits. Extract for channel workers, independent ownership or measured contention. |
| API Gateway | Thin routing, request limits, authentication, trusted tenant context and correlation. | Domain services retain business authorization and transaction ownership. |
| External/internal JWT design | Separate user delegation from authenticated service identity; use asymmetric signing and explicit validation profiles. | More explicit contracts and key management; removes universal signing authority. |
| Tenant resolution | Separate platform/dashboard requests from storefront-host resolution. Reject conflicting tenant selectors. | Resolves registration/bootstrap and custom-domain routing gaps. |
| Raw SQL with `pg` | Select it as the concrete V1 default behind repositories and short transaction units. Remove the permanent query-builder prohibition. | Requires disciplined mappings and tests; permits a later builder under the same persistence contract. |
| Database access and migrations | Distinct runtime, tenant-access and migration roles; bounded pools; service-owned schema provisioning. | More provisioning work, with explicit permission boundaries. |
| Payment workflow | Persist order, reservation and operation before external execution; reconcile provider outcomes. | Adds intermediate states and recovery work essential even in a sandbox. |
| Shipping workflow | Persist shipment intent and execute it through retryable work after Commerce establishes eligibility. | Shipping becomes recoverable without delaying checkout. |
| Retries/idempotency | Persist stable operation identifiers, deduplication records, attempt state and recovery status. | Adds storage/state-machine work; supports later brokers and workers. |
| Theme publication | Mutable drafts plus immutable revisions and one atomic published-version pointer per store. | Preserves preview/live separation and gives a clean rollback path. |
| Frontend/API connection | Same-origin browser requests handled by Next.js server routes, which call Gateway with trusted host and authentication context. | Resolves HttpOnly-cookie versus bearer-header mismatch and preserves storefront domain identity. |
| Docker Compose | Keep it for development/demo, with private networking, scoped secrets, readiness and controlled migrations. | Accept single-host limitations; keep production availability a separate decision. |
| Observability | Require logs, correlation, counters, audit records and durable-work visibility. | Avoid mandatory operation of several telemetry products. |
| Delivery sequence | Build frontend/backend vertical slices together: onboarding, publishing, COD order, payment, shipping, inbox. | Finds integration failures earlier than leaving the frontend until last. |

Version pins also need modification. Node 20 is EOL and Next.js 14 is outside the normal support policy. Proposed replacements are Node 24 LTS and Next.js 16, with NestJS/Puck compatibility checked before pinning dependencies. [Node release status](https://nodejs.org/en/about/previous-releases), [Next.js support policy](https://nextjs.org/support-policy).

3. **Decisions we REJECT**

| Rejected decision | Reason and replacement |
|---|---|
| Silently replacing schema-per-tenant with shared-schema RLS | It contradicts [requirements.md:124](D:/codex-arc/docs/reference/requirements.md:124). Shared-schema RLS remains a legitimate alternative only through an explicit requirements change. |
| Calling payment inside the Commerce transaction and rolling back on failure | A remote payment cannot be rolled back by PostgreSQL. Commit recoverable local state before the external call. |
| Fire-and-forget shipping | A crash can permanently lose shipment creation. Use a durable intent and recovery worker. |
| Prohibiting “anything async” | Reliable provider integration needs background delivery/reconciliation even without a broker. |
| A universal internal JWT signing secret | Every holder can forge other identities. Use distinct signing identities, destination audiences and receiver-enforced caller permissions. |
| Shared privileged database credentials and shared secret delivery | Database names and containers do not isolate services when every process receives the same authority. |
| Serving editable `puck_data` merely because its theme is marked published | Later drafts can become live. Public reads must follow the immutable publication pointer. |
| Letting the proposal’s “fixed” or “out of scope” wording override requirements | Technical proposal wording cannot redefine confirmed product behavior. |

The database credential issue is especially concrete: the proposal initializes `app` through `POSTGRES_USER` and then uses it for every service. The official image creates that initialization user as a superuser; PostgreSQL superusers bypass RLS, including forced policies. [PostgreSQL image documentation](https://github.com/docker-library/docs/blob/master/postgres/content.md), [PostgreSQL RLS documentation](https://www.postgresql.org/docs/16/ddl-rowsecurity.html).

4. **Decisions we DEFER**

| Decision | Revisit when |
|---|---|
| MongoDB for Messaging | A demonstrated conversation workload or storage requirement warrants another database engine. Use PostgreSQL now. |
| Redis runtime | Shared throttling, multiple Gateway replicas or measured caching needs justify it. |
| Product/session caching | Ownership, invalidation and failure behavior are defined and the benefit is demonstrated. |
| Meilisearch deployment | Product search enters implementation scope. Preserve the requirement; omit the idle container. |
| pgvector and `ai_db` | A real retrieval feature needs embeddings. |
| Mandatory running AI stub | An agreed academic/demo deliverable needs it. FastAPI remains the future boundary; a canned response is not evidence of AI functionality. |
| RabbitMQ/Kafka and BullMQ | Fan-out, scheduling or throughput exceeds the simple durable-job design. |
| Independent Storefront/Messaging deployments | Publishing isolation, channel processing, ownership or measured load warrants extraction. |
| Raw-code themes, autonomous AI, trust scoring, advanced analytics and additional providers | Their Grad 2 scope and permissions are defined. |
| Kubernetes, separate database clusters and a full telemetry stack | Availability, scale or operating needs warrant them. |

5. **Resulting proposed Architecture V1**

V1 has three business services: **Identity & Tenancy, Commerce, and Integrations**. Gateway and Next.js are separate HTTP applications. Store Design and Basic Inbox retain explicit ownership within Commerce.

This gives five HTTP applications, plus service-owned worker processes and provisioning/migration jobs. Workers are real implementation and operational work, even though they reuse service code and images.

| Component | Owned capability and data | APIs/dependencies | Deployment and failure rationale |
|---|---|---|---|
| Next.js | Merchant/staff dashboard, basic platform views, storefront rendering; no authoritative business database | Calls Gateway; renders published theme revisions | Frontend deployment and rendering can scale independently. Tenant-aware caching is mandatory. |
| Gateway | Ingress policy and request context; no business records | Routes to services; uses Identity’s directory | Stateless ingress. Business-service failures affect their routes rather than forcing every route unhealthy. |
| Identity & Tenancy | Merchant credentials, sessions, employees, permissions, tenant lifecycle, verified domains and provisioning status | Authentication, membership, directory and provisioning contracts | Isolates credential authority. Failure affects login, provisioning and unresolved tenant lookups. |
| Commerce | Catalog, inventory/reservations, carts, customers, orders, business payment/fulfillment state and commercial settings | Commerce APIs; requests provider operations; owns product assets in R2 | Keeps checkout invariants local. Its failure affects ordering and the co-located modules. |
| Store Design module | Theme drafts, immutable revisions, publication pointer, theme asset metadata | Draft/preview/publish/live-theme contracts; R2 | Independently owned module, co-deployed for Grad 1. No participation in checkout transactions. |
| Basic Inbox module | Conversations, participants, messages and access capabilities | Staff inbox and customer send/read APIs | Bounded public workload. No shared repositories with orders/catalog. |
| Integrations | Encrypted provider configurations, payment attempts, shipments, webhook receipts, provider references and reconciliation work | Payment/shipping commands, status APIs and verified callbacks | Isolates provider credentials and execution. Separate payment/shipping concurrency limits prevent one exhausting the other. |
| Future AI service | Model execution, retrieval projections and AI execution records as needed | Authorized domain APIs and approved event feeds | Independent Python/compute boundary; ordinary merchant operations must continue without AI. |

Each service may access only its own database. Module ownership also matters inside Commerce: an Inbox repository does not become available to the Orders module merely because both run in one process.

The database layout is one PostgreSQL instance containing `identity_db`, `commerce_db` and `integrations_db`. Each contains tenant schemas for tenant-owned records. Controlled global metadata—such as the tenant registry—is explicitly distinguished from tenant business data.

I select the data architect’s tenant-role approach for V1 because it provides an enforceable check against accidental qualified access to another schema:

- Separate migration owners from non-owner, non-superuser runtime identities.
- Tenant roles receive only their own schema and necessary table privileges.
- A bounded service pool uses trusted registry mappings to select the tenant role and schema within a short transaction.
- Repositories receive that transaction’s scoped client. They cannot obtain a general pool.
- Commit or roll back before returning the connection; discard it if cleanup is uncertain.
- Verify that sibling-schema reads/writes fail and that connection reuse does not retain tenant context.

This adds role provisioning alongside schema provisioning. Supplemental RLS is an alternative additional guard, not a replacement for required schemas.

The service’s pool identity can select its authorized tenant roles, so full service compromise still crosses that service’s tenants. Stronger containment would require separately held credentials or another enforcement boundary. PostgreSQL explicitly treats schemas as namespaces governed by privileges; `search_path` alone is not authorization. [PostgreSQL schema documentation](https://www.postgresql.org/docs/16/ddl-schemas.html).

Use same-schema foreign keys, tenant-local uniqueness, positive quantities, nonnegative amounts, unique cart conversion, unique operation keys and immutable order snapshots. Cross-service identifiers remain opaque references. Transactions use one checked-out `pg` client throughout. [node-postgres transaction guidance](https://node-postgres.com/features/transactions).

Provisioning is a recoverable workflow: Identity records a pending tenant and provisioning intent; each service’s isolated provisioner initializes only its own schema and grants; acknowledgments advance the tenant to active. Partial failures remain visible and retryable. Registration never requires an already-existing tenant.

Synchronous communication is deliberately limited:

| Caller → recipient | Purpose |
|---|---|
| Browser → Next.js → Gateway | Same-origin application requests, authentication transport and trusted storefront host context |
| Gateway → Identity | Login/session/employee operations and authoritative directory/status lookups |
| Gateway → Commerce | Catalog, cart, checkout, orders, themes and inbox |
| Gateway → Integrations | Authorized configuration and operation-status requests |
| Commerce → Integrations | Bounded payment-session initiation after local commit; idempotent command/status APIs |
| Provider → Gateway → Integrations | Explicitly routed callbacks authenticated by provider-specific verification |

There is no synchronous reverse callback from Integrations to Commerce while Commerce waits for its payment initiation call.

The durable communication map uses PostgreSQL outboxes/jobs and authenticated HTTP delivery:

| Producer | Work or outcome | Consumer |
|---|---|---|
| Identity | Initialize tenant resources | Owning service provisioners |
| Commerce | Payment operation requested | Integrations |
| Integrations | Verified payment outcome | Commerce |
| Commerce | Shipment requested after eligibility | Integrations |
| Integrations | Shipment creation/tracking outcome | Commerce |
| Each owner | Reconciliation, expiry or retry work | Its own worker |
| Store Design | Publication changed | Local cache invalidation; future indexing consumers |

These are durable commands and facts, not a general event-bus deployment. Record an outbox entry atomically with the state change that requires it; record consumer deduplication atomically with applying the outcome. Duplicate delivery remains possible. [Transactional outbox guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html).

The selected checkout/payment sequence is:

1. Commerce verifies tenant and cart ownership, validates quantities, calculates authoritative totals and snapshots prices.
2. One local transaction reserves stock, converts the cart once, creates the pending order, and records the payment operation and delivery work. Retain conditional stock updates.
3. After commit, Commerce may synchronously request a payment session within a bounded time budget.
4. Integrations durably records and claims the operation before contacting the provider. Request handling and workers use the same operation and provider idempotency key.
5. Verified callbacks or reconciliation establish the outcome. Creating a session or receiving a browser redirect does not prove payment.
6. Commerce applies the outcome idempotently and records eligible shipping work in a local transaction.
7. Definitive failure or expiry releases stock once. Late success after cancellation/expiry records the payment and enters explicit compensation or review.

Timeout means **unknown/pending**, not declined. A lease prevents competing local workers from normally claiming the same operation; it cannot alone prevent duplicate external effects after a crash. Provider idempotency or reference-based reconciliation is required. Where neither is available, ambiguous attempts require review before resubmission.

COD confirms locally without depending on the payment provider. Shipping failure leaves a visible pending/failed fulfillment operation; it does not erase the order. Never silently convert a failed card attempt into COD.

Retries use stable operation IDs, payload binding, bounded concurrency, backoff, attempt limits and operator-visible recovery states. A merchant-triggered shipping workflow remains a possible product choice, but it still requires persisted operation state and idempotency.

Authentication has two distinct responsibilities:

- **Workload identity:** each calling service has its own signing identity. Receivers validate issuer, audience, type and expiry, then enforce a caller/operation allowlist. A caller cannot grant itself privileges through asserted scopes.
- **User authority:** Gateway supplies verifiable, short-lived delegation for merchant/staff actions. Downstream services enforce permissions and resource ownership. Background workers use persisted, previously authorized operations and restricted workload permissions—not expired employee tokens.

Identity alone holds its merchant-token signing key. Gateway has a separate delegation key. Customer credentials remain per-store and Commerce-validated; guests receive narrowly scoped cart/conversation capabilities. Phone numbers are contact identifiers, not proof of ownership. JWT validation profiles must prevent tokens intended for one purpose being accepted for another. [JWT best-current-practice guidance](https://www.rfc-editor.org/rfc/rfc8725.html).

Storefront tenancy comes from normalized, verified domains at trusted ingress. Dashboard tenancy comes from authenticated membership. Platform login, registration and administration have explicit bootstrap/control-plane routes. Reject path/token/host disagreements where those selectors apply; do not assume `app.platform…` identifies a merchant tenant.

Use host-only Secure/HttpOnly cookies, CSRF controls, refresh rotation/reuse detection and a defined revocation policy. Sensitive permission/configuration changes require current authority; caching must not silently preserve disabled staff or suspended tenants indefinitely.

Other current security boundaries include per-integration permissions, provider callback verification, encrypted provider credentials confined to Integrations, tenant-partitioned SSR/cache/object keys, Puck component validation, safe content rendering, scoped uploads and verified domain activation. Raw merchant code execution remains deferred.

The operational baseline is intentionally small: PostgreSQL, the application processes, their durable workers/provisioners, R2 and appropriate TLS ingress. Redis, MongoDB, search/vector infrastructure, brokers and telemetry containers are absent by default.

Compose needs readiness conditions and controlled migration jobs; ordinary startup ordering does not wait for dependencies to become ready. Hosted backend/database ports stay private, with optional local debugging bound appropriately. [Docker startup guidance](https://docs.docker.com/compose/how-tos/startup-order/).

| Failure | Required V1 behavior |
|---|---|
| Identity unavailable | Login/onboarding and unresolved tenant lookups fail clearly; no guessed tenancy or indefinite stale authorization. |
| Provider timeout | Persist unknown outcome and reconcile; avoid blind duplicate submission. |
| Worker crashes after commit | Durable work remains claimable after recovery. |
| Duplicate/out-of-order callback | Deduplicate and validate permitted state transitions. |
| Shipment creation fails | Retain actionable fulfillment state and retry/review controls. |
| Inbox traffic spikes | Enforce payload, rate and concurrency limits; extract if measured contention persists. |
| R2 upload/publish fails | Preserve the previous publication; retry or clean up unreferenced assets. |
| PostgreSQL/Compose host fails | Accept demo downtime; use tested backups/restoration. This deployment does not provide high availability. |

Observability must include redacted structured logs, request/operation correlation, latency/error counters, worker heartbeats, oldest pending work, unresolved payments, database saturation and durable audit records. Define who inspects failures and how they replay or resolve them. Specific tracing, dashboard and error-tracking products remain optional.

Implementation verification should prioritize tenant isolation, pooled-connection reuse, employee permissions, concurrent checkout, duplicate callbacks, crash recovery, reservation expiry/late payment and draft/live publication. No such implementation tests were run in this document-only review.

Grad 1 starts with singleton services and bounded pools/workers. Before adding Gateway replicas, introduce coordinated throttling through Redis or a suitable upstream facility. Scale Commerce, workers and rendering independently when measurements justify it; budget their combined database connections.

Grad 2 can introduce real FastAPI capabilities, Meilisearch, pgvector, channel workers and extracted Store Design/Messaging services. Trust initially remains a restricted module with its own global connection/schema, matching the requirements’ architecture notes. It consumes fulfillment outcomes and exposes computed scores, never another merchant’s raw order history. Later extraction requires explicit privacy and ownership decisions.

Startup evolution can add managed PostgreSQL recovery/failover, service replicas across failure domains, stronger credential isolation and brokers for multiple independent consumers. Inventory/order separation should wait until its benefits justify distributed reservation complexity.

The remaining **OPEN** product decisions are:

- Whether Grad 1’s message box includes Meta transport.
- Whether sandbox payment/shipping integrations satisfy assessment acceptance.
- Whether there is a separate Grad 1 academic AI deliverable.
- Exact dashboard audiences and customer-account/custom-domain delivery timing.
- Availability, recovery and revocation-freshness targets.
- Provider-specific payment, webhook and shipment contracts.
- Future trust thresholds/privacy policy, AI purchase/write permissions and raw-theme execution design.

Architecture V1 provisionally assumes a storefront basic inbox and sandbox integrations. Those assumptions are visible scope decisions; they do not amend the authoritative requirements.
