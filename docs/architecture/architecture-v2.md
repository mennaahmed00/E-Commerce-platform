# Architecture V2 — supervisor review proposal

Status: **PROPOSED — lead-selected after two specialist review rounds; not product or production approval.**

Review date: 14 September 2026. Scope: architecture and documentation only; no application implementation.

## 1. Read this package

| Document | Purpose |
|---|---|
| [Service catalog](service-catalog-v2.md) | Boundaries, APIs, events, dependencies and runtime privileges |
| [Data ownership](data-ownership-v2.md) | Authoritative records, tenancy, duplicate data, constraints and recovery |
| [Communication](communication-v2.md) | Sync calls, asynchronous delivery, workflows and failure handling |
| [Drawn architecture](diagrams-v2.md) | Context, containers, ownership, deployment and sequence diagrams |
| [Supervisor explanation](supervisor-explanation-v2.md) | Plain-English explanation, presentation narrative and trade-offs |
| [Architecture decisions](decisions/adr-v2-001-service-boundaries.md) | Start of the numbered V2 ADR set |

## 2. Authority and explicitly superseded assumptions

The latest user instructions take precedence. [requirements.md](../reference/requirements.md) remains the product/constraint baseline except for the explicit revisions below. [chat-summary.md](../reference/chat-summary.md) is historical reasoning, not current approval. [implementation-proposal.md](../reference/implementation-proposal.md) is Claude's candidate architecture, not an approved design. [Architecture V1](architecture-v1.md) is the prior reviewed baseline; the requested root filename resolved to this existing file. None of these files is edited by V2.

| Earlier position | What changes | Why the new choice is better for the revised project |
|---|---|---|
| Requirements mandate schema-per-tenant; V1 obeyed that constraint | Shared tables with tenant_id and enforced RLS become the default | User expressly authorized reconsideration. Stable schemas and one migration stream per service reduce provisioning/drift work without pretending schema namespaces contain a compromised service |
| Puck/drag-and-drop and trusted component configuration | Ready themes; direct HTML/CSS and declarative config editing; real AI file patches | Satisfies the new editing model. A file/revision lifecycle and restricted renderer address risks absent from the old component-only model |
| V1 provisionally treated Inbox as storefront chat | Real unified social DMs and bidirectional delivery | Channel account ownership, credentials, webhook traffic and delivery policies require a Messaging capability |
| V1 kept theme metadata inside Commerce | Separate Store Experience ownership | Authoring, validation and publication have their own invariants and release cycle, independent of inventory and money |
| V1 had a generic Integrations service | Domain-owned adapters with isolated execution workers | Removes an internal distributed handoff from financial recognition; does not remove the external payment consistency problem |
| AI was a future/stub runtime | Bounded theme-edit AI is a Grad 1 target | Current requirements explicitly require actual chatbot edits; broader autonomous agents remain later |
| One generic payment flow | Commerce: Paymob/Fawry customer sales. Platform Billing: Kashier subscriptions | Different payer, beneficiary, invoice, consent, accounting and entitlement responsibilities cannot share one business ledger |
| Implicit common dashboard/customer model | Separate merchant/admin authorization surfaces; store-local customers and guests | Prevents platform privilege leakage and accidental cross-store customer identity sharing |
| Custom-domain details unresolved | Automated, stateful verification/certificate/routing workflow | Delivers self-service activation with truthful DNS dependencies and safe failure/revocation behavior |
| Future Trust could remain a global Commerce module | Separate restricted Trust service/database when introduced | Intentional cross-store evidence requires a stronger boundary than public Commerce's tenant-local records |

## 3. Review history and actual disagreement

All six configured specialists completed Round 1 successfully. All six completed Round 2. The resumed review reused their completed findings; no failed analysis needed restarting. The pre-write decision summary was shown before creating this package.

| Specialist | Independent Round 1 position | Round 2 response |
|---|---|---|
| domain_architect | Five domains: Platform, Commerce, Experience, Messaging, AI; domain-owned adapters | Defended domain cohesion; acknowledged added Experience contracts and mandatory worker isolation |
| backend_architect | Retain Integrations; theme metadata in Commerce | Revised: remove internal financial outcome handoff; separate Experience; retain restricted provider workers |
| data_architect | Retain Provider Operations; theme metadata in Commerce | Revised: domain-local financial application; Experience publication remains locally atomic |
| platform_architect | Retain Integrations; isolated renderer plus Commerce metadata | Revised: ownership improves, but neither alternative reduces the hidden worker/runtime count |
| architecture_critic | Retain Integrations and Commerce theme module to limit scope | Revised: Experience is now a substantial domain; generic Integrations distributes money coordination |
| security_architect | Retain Integrations and Commerce theme module; catalog ambiguously placed ledgers in Integrations | Corrected ledgers to their business owners; retained dissent in favor of a dedicated execution boundary unless equivalent worker privileges are demonstrated |

Lead decision is **not majority voting**. Local commercial correctness, the confirmed theme lifecycle and explicit secret isolation justify the selected topology. Security's dissent becomes a release gate: API processes cannot decrypt provider secrets, execution workers cannot arbitrarily update business ledgers, and renderers cannot access business databases. If these controls cannot be demonstrated, reopen ADR-001; do not quietly co-locate all secrets in a broad runtime.

Commerce is not too broad merely because it owns many tables. Catalog, stock, cart conversion and orders share checkout invariants. Themes and social DMs do not; those leave Commerce. Payment and shipping execution remain modules of their commercial owner, with separate processes rather than new business authorities.

## 4. Selected architecture shape

Use **five coarse-grained, independently owned services with modular internals**, not a service per entity and not a single unrestricted application process.

| Service | Purpose | Grad 1 |
|---|---|---|
| Platform Control | Merchant/staff identity, tenancy, memberships, platform administration, domains and platform Billing | Required |
| Commerce | Catalog, inventory, carts, store-local customers/guests, checkout, orders, customer money and fulfillment | Required |
| Store Experience | Theme library, editable files/config, revisions, validation, previews, publication and rollback | Required |
| Social Messaging | Connected social accounts, unified inbox, replies and delivery recovery | Required; WhatsApp and Instagram targets |
| AI Assistance | Actual constrained theme-edit jobs, proposed patches and execution evaluation | Required narrow capability |

Supporting runtimes are real work: thin API Gateway, trusted merchant/admin/checkout web applications, public storefront renderer, isolated validators, and domain/provider/message/AI workers. They are not extra authoritative business services. See the runtime permission matrix in the service catalog.

Identity and Tenancy remain modules of Platform Control. Platform Billing is another separately permissioned module, not a shopper-payment feature. Trust/Fraud is a later separate service. Analytics, returns, discounts and notifications start as modules of the relevant owner. No synchronous checkout chain through Billing, Messaging, Experience or AI is permitted.

## 5. Decision register

ACCEPT means retain the underlying decision, subject to the controls documented here. MODIFY changes its scope or implementation architecture. REJECT removes the proposed default. DEFER means do not deploy it without an implemented consumer and evidence. Detailed alternatives, risks and reconsideration triggers are in the ADRs.

| Major decision from Claude/V1 | V2 classification | Selected option; trade-off and stage |
|---|---|---|
| Service-per-capability direction | MODIFY | Five coarse domains; fewer distributed checkout invariants, more internal module discipline |
| Thin API Gateway | ACCEPT | Routing, trusted context, limits and correlation; no business orchestration |
| Identity/Tenancy separation | MODIFY | One Platform Control service, distinct modules; avoids remote tenant/identity provisioning transactions |
| Commerce responsibilities | MODIFY | Keep transactional core and commercial adapters; remove themes/social inbox |
| Separate Storefront | MODIFY | Store Experience owns authoring/publication; restricted public renderer is a separate runtime, not a second data owner |
| Generic Integrations service | REJECT | Domain-owned operations; restricted provider processes are required now |
| Messaging service | MODIFY | Standalone real social capability with its own channel adapters and PostgreSQL |
| AI service | MODIFY | Real bounded theme work in Grad 1, not stub or unrestricted agent |
| Schema-per-tenant default in V1 | REJECT | Shared tenant_id + RLS; explicit user-authorized requirements change |
| Shared-schema + RLS in Claude | MODIFY | Adopt principle with enforced roles, tenant-aware constraints and tested context propagation; not merely query filters |
| Database per service | ACCEPT | Five logical PostgreSQL databases, distinct credentials; one initial instance is not five failure domains |
| MongoDB for Messaging | REJECT | PostgreSQL relational envelope plus JSONB; reconsider only after measured data/workload need |
| Redis baseline | DEFER | No financial authority; shared throttling uses edge/DB coordination initially. Add when measured need warrants |
| Meilisearch | DEFER | PostgreSQL search initially; future tenant-scoped rebuildable projection |
| pgvector | DEFER | Theme editing uses bounded supplied files, not a retrieval platform; add only for a delivered retrieval feature |
| Raw SQL with pg | ACCEPT | Parameterized repositories, migration ownership and transaction-scoped clients; no controller SQL or permanent ban on a justified query builder |
| REST transport | ACCEPT | Versioned HTTP/JSON APIs and asynchronous HTTP deliveries |
| REST-only synchronous workflows | REJECT | Durable jobs, outbox/inbox and state machines are mandatory in Grad 1 |
| No event broker in MVP | ACCEPT | No broker does not mean no events or background processing |
| Universal shared internal JWT secret | REJECT | Asymmetric workload identities plus destination/action allowlists and separate user delegation |
| External-to-internal JWT design | MODIFY | Preserve user subject and authorization provenance; a worker cannot mint merchant permissions |
| Host/token tenant resolution | MODIFY | Verified hostname registry for public requests; membership for staff; account mapping for callbacks; reject mismatches |
| Payment inside checkout SQL transaction | REJECT | Commit intent first; execute externally; apply verified result locally; compensate or reconcile |
| Fire-and-forget shipping | REJECT | Durable eligible shipment operation and recoverable state machine |
| Failure handling/retries/idempotency | MODIFY | Unknown is a real state; payload-bound keys, duplicate effect guards, bounded retry and visible manual review |
| Mutable draft used by live store | REJECT | Immutable revisions and atomic validated publication pointer |
| Docker Compose | ACCEPT | Development/assessment and modest single-host demonstration, not automatic HA |
| Shared app superuser/.env secrets | REJECT | Per-process credentials, role grants and secret scopes; migration credentials are offline from runtime |
| Observability and recovery | MODIFY | Structured redacted logs, traces/correlation, metrics, audit and backlog/reconciliation visibility from Grad 1 |
| Grad 1 breadth | MODIFY | Narrow vertical slices of confirmed features; no unused infrastructure or silently substituted chat/AI demos |
| Kubernetes, service mesh, dedicated tenant placement, autonomous Trust/AI ordering | DEFER | Add only after the stated product, security and operational gates |

## 6. Tenancy decision

| Criterion | Schema per tenant | Shared tables + tenant_id + RLS — selected | Dedicated placement/hybrid |
|---|---|---|---|
| Scalability | Schema/role inventory and migration fan-out grow with tenants | Stable object count; grow tables and indexes with capacity planning | Isolate large/regulated tenants, but placement and relocation become systems to operate |
| Complexity/migrations | Partial tenant upgrades and schema drift | One service migration stream, resumable data backfills | Multiple placement cohorts and migration paths |
| Security | Grants help; namespaces alone do not isolate | Database guard against missing filters; trusted context and restricted roles are essential | Dedicated credentials/resources can contain selected tenants more strongly |
| Developer experience | Dynamic schema selection and per-tenant bootstrap | Stable SQL, predictable tests; every tenant table must participate | Hardest local testing and operational diagnosis |
| Performance | Smaller per-tenant indexes; still shared hardware | Tenant-leading composite indexes and quotas; benchmark skew and contention | Independent resources at added cost |
| Recovery | Tenant-local export may be easier; cross-service restore still coordinated | Selective tenant restore requires extraction/replay, not a simple whole-DB restore | Better isolated restore where separately provisioned |
| Team/project fit | Unjustified default overhead for four application students | Best baseline with automated isolation tests | Startup escape hatch, not Grad 1 machinery |

Use one database per service, not one database per tenant. Separate global control tables from tenant-owned tables. Require explicit read/write policies, non-owner runtime roles without superuser/BYPASSRLS, FORCE RLS on tenant tables, transaction-local context, and tenant-inclusive unique/foreign keys. PostgreSQL documents privileged-role bypass and the special behavior of integrity checks; policies are not a substitute for schema constraints. [PostgreSQL row security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)

RLS prevents classes of accidental cross-tenant access. It does **not** contain a fully compromised service allowed to set another tenant context. No architecture here claims otherwise. [ADR-002](decisions/adr-v2-002-tenancy.md) defines migration and future placement conditions.

## 7. Communication and commercial correctness

Interactive CRUD, authority checks, public catalog reads and local checkout decisions use bounded synchronous APIs. Provider execution, webhook processing, social delivery, domains, validation and AI are background jobs. Cross-service facts travel via producer-owned transactional outboxes and consumer-owned deduplication inboxes over authenticated HTTP.

Commerce atomically creates the order snapshot, reserves stock, converts a cart once and persists payment intent. A worker calls Paymob/Fawry after commit. The UI receives pending status and polls for a session/reference; this avoids provider secrets in interactive handlers. Verified evidence is durably recorded, then a trusted Commerce processor applies deduplication, monetary entries, order eligibility and subsequent work in one transaction. Never hold database transactions across provider calls.

Paymob/Fawry customer receipts and refunds belong to Commerce. Kashier subscription invoices, cycles, consent, receipts and entitlements belong to Platform Billing. The default assumption is merchant-owned customer-payment accounts settling to merchants; the platform receives only its subscription money. A marketplace collection/payout model would require a separate review, not a hidden adapter change.

Kashier supports token-based repeated charges but does not manage subscription plans/schedules; our Billing module must own those. Automatic renewal requires verified consent and appropriate account capability; manual hosted renewal is the proposed fallback until that gate is met. [Kashier recurring payments](https://developers.kashier.io/docs/accept-payments/recurring)

Fawry can report delayed and expired outcomes. Align reservation policy with the selected payment method; never mark a voucher/reference as paid. Late payment after stock release is a recorded exception requiring stock re-evaluation or refund/review. [Fawry payment notifications](https://developer.fawrystaging.com/docs/server-apis/payment-notifications/server-notification-v2)

Shipping eligibility is a Commerce decision. Bosta is the proposed first adapter; J&T Express fits the same capability contract, but its Egypt account/API details remain OPEN. A provider status does not automatically prove customer culpability or COD remittance. All external exactly-once claims are rejected; ambiguous effects need provider-supported lookup/idempotency or visible review.

## 8. Themes, storefronts and AI

Provide 2–3 Arabic/English, RTL-aware starter themes. Merchants edit HTML, CSS and schema/config text. V2 interprets schema/config as **declarative JSON/JSON Schema**, not executable application code. Allow a small, versioned template grammar for public catalog substitutions; reject arbitrary merchant JavaScript, packages, server code and dynamic network/file access.

Experience owns private draft heads, immutable revision manifests, validation evidence and publication pointers in PostgreSQL. Immutable files/assets live in object storage. Save and AI-accept operations use an expected base revision. Preview pins one revision. Publish checks exact validation version and artifact completeness, then atomically switches the pointer. Rollback chooses a compatible earlier validated revision; it does not roll back orders or products.

AI receives only authorized theme files and a bounded public design context. It proposes path-limited patches. Experience checks authorization, revision conflicts, paths, types, HTML/CSS/config and quotas; merchant review creates a draft and explicit publication is a separate action. Prompt text and file comments cannot grant tools or publication authority.

Public storefronts and previews are isolated from privileged applications, preferably on a different registrable domain from trusted merchant/admin/checkout hosts. Preview frames are sandboxed and receive no trusted cookies. Rendering/validation processes receive bounded artifacts and public data, not database or provider credentials. Parsed HTML/CSS restrictions, escaping, controlled asset URLs, CSP and resource limits are cumulative controls, not alternatives. Trusted checkout/account screens cannot include merchant templates.

Cart handoff to trusted checkout uses a short-lived, one-use, store-bound opaque capability via POST. Checkout recalculates prices, stock and shipping from Commerce. Merchant-editable forms cannot collect payment or account credentials through platform widgets. Merchant deception remains an abuse risk even without scripts; moderation/reporting and suspension are required.

## 9. Social inbox and dashboards

Messaging owns each connected account, token, channel-scoped participant, conversation, message and delivery attempt. Both inbound DMs and dashboard replies are persisted. Replies return through the originating channel/account; never merge identities by display name or matching phone alone. Staff message-read/send permissions are distinct.

Instagram integration targets professional accounts and approved messaging access, not arbitrary personal inboxes. Its Send API requires the recipient to have initiated contact. Provider policy is checked again at dispatch. [Meta's Instagram Send API](https://www.postman.com/meta/instagram/folder/uxudqu0/send-api)

WhatsApp uses the official Business Platform, not scraping a personal WhatsApp session. Connection, webhook and outbound policy contracts are designed now; exact production permissions, current messaging-window/template rules and account onboarding must pass provider verification before launch. Some Meta documentation requests were rate-limited during this review; this package does not claim all account capabilities were verified.

Merchant dashboard: own-store catalog, orders, fulfillment, customer records, staff permissions, themes/AI previews, connected providers, inbox, domains and subscription status. Platform admin dashboard: tenant lifecycle, platform plans/invoices, operational health, provider onboarding incidents, moderation and audited support access. Platform admins do not automatically get all merchant DMs, payment secrets or customer exports.

## 10. Automatic custom domains

Platform Control owns a global normalized hostname registry and durable onboarding workflow. Select Cloudflare for SaaS behind a provider abstraction as the proposed managed DNS/TLS edge. Give each store a platform subdomain while its custom domain is pending.

Track claim ownership, DNS routing, certificate validation/readiness and routing activation separately. Activate only when all required gates and store eligibility pass. Cloudflare's documented production gates include active hostname, active SSL and DNS pointing at the SaaS target. [Cloudflare hostname validation](https://developers.cloudflare.com/cloudflare-for-platforms/cloudflare-for-saas/domain-support/hostname-validation/)

Immediately attempt checks, reconcile with bounded backoff and show precise pending/action-required states. Human platform operators are not part of the normal activation path. The merchant must control DNS or explicitly authorize a supported DNS integration. Incorrect DNS, CAA restrictions, external CDN conflicts, issuance delays and provider outages remain possible. Apex-domain support is conditional on the merchant DNS/provider plan; never prescribe a universal apex CNAME.

No store content is served to an unverified/unknown host. Revocation, suspension and reassignment invalidate routing generations; check routing eligibility before serving cached HTML. Renewal failure degrades to an actionable state while the platform subdomain remains available according to tenant policy. [ADR-008](decisions/adr-v2-008-custom-domains.md)

## 11. Infrastructure, security and operations

Use NestJS for the four application services, FastAPI for AI, Next.js for trusted web applications, a restricted storefront renderer, PostgreSQL and S3-compatible object storage (R2 proposed). Pin mutually supported versions before implementation; historical image tags are not approval. Raw pg access remains behind repositories and transaction-scoped clients.

Docker Compose is the Grad 1 development/assessment topology. One PostgreSQL instance hosts five logical databases. Expose only ingress; keep databases, workers and internal APIs private. Use non-root images, resource/connection budgets, health checks, service-specific secret injection and one-shot service-owned migrations. Start ordering is not readiness. [Docker Compose readiness guidance](https://docs.docker.com/compose/how-tos/startup-order/)

Use distinct asymmetric user and workload trust profiles, destination/action authorization, current checks for sensitive staff writes, host-only cookies, CSRF protection, refresh rotation, platform-admin MFA and redacted audit logs. No service self-asserts new user permissions. Provider callback verification is provider-specific; do not pretend every provider signs identical raw bytes.

Minimum observability: correlated structured logs, error/latency metrics, worker heartbeat, oldest queued job, webhook lag, unknown payment/send counts, inventory-expiry lag, domain gate age, AI quota/cost and publication validation failures. Propagate trace context through HTTP and jobs. Prefer a managed telemetry backend over a student-operated observability cluster.

Back up databases, object manifests/files and recoverable encryption keys separately from the application host. Encrypt backups, restrict access and test restoration plus reconciliation. Proposed assessment targets are RPO <=24 hours and RTO <=4 hours; these are unmeasured acceptance targets, not achieved SLAs. Real-money startup launch needs reviewed PITR/availability/retention targets and restore drills. One host, PostgreSQL instance and ingress remain single points of failure; backups do not make them highly available.

## 12. Grad 1 delivery and future evolution

The six-person team has four application developers and two AI specialists. Five business services plus workers is substantial; this plan does not assert a semester schedule without estimates. Share transport/schema tooling, not domain repositories. Assign vertical slices and enforce module boundaries in CI.

| Stage | Deliver and demonstrate | Explicitly excluded |
|---|---|---|
| Grad 1 foundation | Isolation tests, onboarding/staff, two dashboards, per-store customers/guests, catalog/COD checkout, ready-theme publication | Global customer identity, drag/drop |
| Grad 1 commerce | Paymob and Fawry adapter targets; recovery/refunds workflow; Bosta shipping; Kashier subscription invoice/hosted payment flow | Marketplace settlement, general accounting engine, automatic charging without consent |
| Grad 1 experience | Direct HTML/CSS/config editing, safe preview/rollback, actual AI patch proposal and merchant approval | Arbitrary merchant JS/builds, autonomous publish |
| Grad 1 social/domains | WhatsApp and Instagram receive/reply targets; automated domain lifecycle and visible failures | Website-chat substitution, arbitrary personal-account scraping, guaranteed instant DNS |
| Grad 2 | J&T adapter after account contract; advanced returns/analytics, richer social media support; consented renewal automation if not ready earlier | Infrastructure introduced only for a roadmap slide |
| Future/startup | Restricted Trust service, conversational ordering/voice/CRM, implemented search/retrieval, dedicated tenant placement, replicas and stronger recovery | Raw cross-store histories exposed to merchants |

Start provider access and account eligibility checks immediately, in parallel with foundation work and AI evaluations. Sandbox demonstrations prove integration behavior, not production account approval. If assessment requires live processing or a different adapter order, explicitly rebaseline delivery; do not call a simulated provider or canned AI response complete.

Evolution triggers: add a broker for measured fan-out/backlog/independent-consumer needs; Redis for measured shared throttling/caching/queue value; Meilisearch for unmet search requirements; pgvector for a delivered retrieval use case. Extract Billing, payment execution or logistics only when independent operational ownership, security or load justifies their contracts. Extract stock/orders only after accepting the resulting distributed reservation design.

## 13. OPEN assumptions and release gates

| Question | Proposed working assumption / required gate |
|---|---|
| Launch providers and sandbox acceptance | Both named customer gateways and social channels are Grad 1 targets; Bosta first, J&T later. Supervisor must confirm grading/live acceptance |
| Who receives shopper funds? | Merchant-owned gateway accounts; no platform custody/payout orchestration. Confirm commercial onboarding model before implementation |
| Subscription pricing, trial, grace, cancellation and automatic renewal | Platform-owned policy; one simple plan/hosted invoice first. No automatic debit without consent and enabled provider capability |
| Delayed-payment stock reservation | Configurable per-method expiry, explicit late-payment recovery. Product owner must choose durations and refund authority |
| Shipping dispatch | Merchant confirms fulfillment after persisted payment/COD eligibility; auto-dispatch is a future configurable policy |
| Theme restrictions and checkout origin | Declarative config/no merchant JS; trusted platform checkout/account screens accepted as the security default |
| Revocation freshness | Proposed <=30-second routing cache lifetime, <=60-second ordinary staff authority cache; sensitive actions require current authority. Verify under failure, not just healthy events |
| Social account binding/retention/history | One provider asset bound to one store initially; text-first real inbox, safe attachment references, no promise of complete historical import. Set retention/deletion and supported media before launch |
| Model choice and privacy | Model-provider adapter, bounded file-only context and budget. Choose model/account/data-processing terms after evaluation; no unused vector stack |
| Domain limits and apex behavior | Normalize/verify each hostname; confirm provider plan, quota, apex features and supported DNS automation |
| Trust policy | Future separate owner. Thresholds, minimum evidence, correction/appeal, consent/legal basis and unavailable-score fallback remain unresolved |

## 14. Evidence and validation status

The architecture decisions are engineering judgments informed by the repository and two review rounds. Provider sources establish specific capabilities, not approval of this system. Paymob documents processed callbacks and intention correlation; those inform the receipt workflow. [Paymob transaction callbacks](https://developers.paymob.com/paymob-docs/manage-callback/transaction-callbacks)

Bosta documents delivery creation and status webhooks, including configurable callback headers. These do not establish a universal callback signature or exactly-once create API. [Bosta delivery webhooks](https://docs.bosta.co/docs/how-to/get-delivery-status-via-webhook/)

J&T Egypt production API contracts, all providers' ambiguous-create lookup/idempotency behavior, and live account approvals remain unverified release gates. Validate authentication, signature fixtures, duplicate/out-of-order callbacks, refunds, cancellation, rate limits and reconciliation against each selected account before enabling production. Documentation diagrams and links are checked separately; no application behavior has been implemented or load-tested by this review.

## 15. ADR index and package checks

| ADR | Decision |
|---|---|
| [001](decisions/adr-v2-001-service-boundaries.md) | Five domains and restricted domain-owned execution |
| [002](decisions/adr-v2-002-tenancy.md) | Shared tenant tables with enforced RLS |
| [003](decisions/adr-v2-003-communication-consistency.md) | Local ACID, durable jobs and outbox HTTP |
| [004](decisions/adr-v2-004-theme-safety.md) | File-based themes, isolated rendering and bounded AI |
| [005](decisions/adr-v2-005-identity-security.md) | Actor profiles, trusted tenancy and workload identity |
| [006](decisions/adr-v2-006-money-shipping.md) | Customer/platform money separation and fulfillment |
| [007](decisions/adr-v2-007-social-inbox.md) | Standalone social Messaging with PostgreSQL |
| [008](decisions/adr-v2-008-custom-domains.md) | Automatic domain activation gates |
| [009](decisions/adr-v2-009-storage-operations.md) | Minimal storage and honest deployment/recovery |
| [010](decisions/adr-v2-010-future-trust.md) | Future restricted cross-store Trust |

All 16 Mermaid diagrams were parsed/rendered to [portable SVGs](diagrams-v2-rendered.md). A final security documentation check found no blockers; a data check identified and prompted corrections to optional guest-account/provider-receipt relationships and commit-time theme revision concurrency. These checks validate the documentation, not application behavior. The four baseline document hashes were unchanged after package creation.
