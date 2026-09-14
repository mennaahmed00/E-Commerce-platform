# Architecture V2 — drawn architecture

Status: proposed; Grad 1 unless explicitly marked Future. These diagrams use Mermaid and follow C4 context/container semantics without requiring the experimental C4 parser. The [architecture](architecture-v2.md), [catalog](service-catalog-v2.md) and [communication contracts](communication-v2.md) define the precise boundaries. See [rendered diagrams](diagrams-v2-rendered.md) for portable SVG images.

Legend: solid arrows are synchronous requests or local writes; dashed arrows in structural diagrams are durable asynchronous facts/jobs or read-only projections. Databases belong to exactly one service. A worker inside an ownership boundary is a separate restricted runtime, not permission to share all credentials. Sequence-diagram dashed arrows are replies; durable delivery is explicitly labeled.

## 1. C4 level 1 — system context

```mermaid
flowchart LR
  Merchant["Merchant and store staff"]
  Admin["Platform owners / admins"]
  Customer["Shopper / guest"]
  System["Multi-tenant commerce platform<br/>Stores, checkout, themes, social inbox, subscriptions"]
  Sales["Paymob + Fawry<br/>Customer pays merchant"]
  Billing["Kashier<br/>Merchant pays platform"]
  Shipping["Bosta first<br/>J&T after API contract"]
  Social["WhatsApp Business + Instagram<br/>Private messages and replies"]
  Edge["Managed DNS / TLS edge<br/>Cloudflare for SaaS proposed"]
  Model["Selected AI model provider/runtime"]
  Merchant -->|"Merchant dashboard"| System
  Admin -->|"Separate admin dashboard"| System
  Customer -->|"Public store and trusted checkout"| System
  System <-->|"Payment operations / verified outcomes"| Sales
  System <-->|"Subscription collection / outcomes"| Billing
  System <-->|"Delivery operations / tracking"| Shipping
  System <-->|"Account connection, DMs and delivery"| Social
  System <-->|"Domain readiness / routing"| Edge
  System -->|"Bounded theme-only context"| Model
  classDef platform fill:#dbeafe,stroke:#1d4ed8,color:#172554;
  class System platform;
```

The system does not collect shopper money into the platform account by default. Provider account approval and DNS control remain external dependencies.

## 2. C4 level 2 — containers and ownership

```mermaid
flowchart TB
  Users["Merchants / admins / shoppers"] --> Edge["TLS edge + ingress<br/>Verified host and route eligibility"]
  Edge --> Web["Trusted web entrypoints<br/>Merchant / Admin / Checkout and accounts"]
  Edge --> Render["Public renderer<br/>Restricted template evaluation"]
  Web --> Gateway["Thin API Gateway<br/>Route, authenticate, limit, correlate"]
  subgraph Services["Five business ownership boundaries"]
    P["Platform Control<br/>Identity, tenancy, domains, Billing"]
    C["Commerce<br/>Catalog, stock, customers, orders, payments, shipping"]
    E["Store Experience<br/>Files, revisions, validation, publish"]
    M["Social Messaging<br/>Connections, conversations, replies"]
    A["AI Assistance<br/>Theme patch jobs and evaluation"]
  end
  Gateway --> P
  Gateway --> C
  Gateway --> E
  Gateway --> M
  E -.->|"Durable theme-edit command"| A
  A -.->|"Proposal, never publish"| E
  Render -->|"Approved manifest"| E
  Render -->|"Public view model"| C
  P --> PD[("platform_db")]
  C --> CD[("commerce_db")]
  E --> ED[("experience_db")]
  M --> MD[("messaging_db")]
  A --> AD[("ai_db")]
  E --> Objects["Object storage<br/>Immutable theme files / controlled assets"]
  C --> Media["Commerce media namespace"]
  M --> Private["Private social media namespace"]
  P --- PW["Restricted Billing + domain workers"]
  C --- CW["Restricted payment + shipping workers"]
  E --- EW["Restricted validator<br/>No business DB access"]
  M --- MW["Channel delivery workers"]
  A --- AW["Bounded inference worker"]
  PW --> PK["Kashier / DNS and certificate APIs"]
  CW --> CP["Paymob / Fawry / Bosta / J&T"]
  MW --> Meta["WhatsApp / Instagram"]
  AW --> Model["Model runtime/provider"]
  classDef control fill:#ede9fe,stroke:#7c3aed,color:#2e1065;
  classDef commerce fill:#dbeafe,stroke:#2563eb,color:#172554;
  classDef experience fill:#dcfce7,stroke:#16a34a,color:#14532d;
  classDef messaging fill:#ffedd5,stroke:#ea580c,color:#7c2d12;
  classDef ai fill:#fce7f3,stroke:#db2777,color:#831843;
  class P,PD control;
  class C,CD commerce;
  class E,ED experience;
  class M,MD messaging;
  class A,AD ai;
```

Workers shown beside an owner use that owner's narrowly scoped operation/evidence contracts, not broad API credentials. All five databases initially share one PostgreSQL instance. Callback intake is detailed in the deployment and provider flow diagrams.

## 3. Service communication — no event broker in Grad 1

```mermaid
flowchart LR
  G["Gateway / routing cache"] -->|"Host and authority queries"| P["Platform Control"]
  C["Commerce"] -->|"Current sensitive authority"| P
  E["Store Experience"] -->|"Current sensitive authority"| P
  M["Social Messaging"] -->|"Current sensitive authority"| P
  P -.->|"Tenant and entitlement facts<br/>Outbox to consumer inboxes"| C
  P -.->|"Tenant and entitlement facts"| E
  P -.->|"Tenant and entitlement facts"| M
  P -.->|"Active job eligibility"| A["AI Assistance"]
  P -.->|"Domain generation invalidation"| G
  C -.->|"Capability ready"| P
  E -.->|"Capability ready"| P
  E -.->|"ThemeEditRequested"| A
  A -->|"Scoped source read"| E
  A -.->|"ThemePatchProposed / failed"| E
  R["Public rendering orchestration"] -->|"Approved manifest"| E
  R -->|"Public catalog read"| C
  E -.->|"Publication invalidation"| R
  C -.->|"Optional public cache invalidation"| R
  EVID["Each producer owns its outbox<br/>Each consumer commits dedup + effect<br/>Authenticated HTTP; at least once"]
```

The opposite-direction arrows are authority checks and independent lifecycle facts, not a synchronous circular checkout chain. Local payment/shipping jobs do not traverse another business service. No event-bus box is hidden in the delivery mechanism.

## 4. Storage ownership and intentional copies

```mermaid
flowchart TB
  P["Platform owns tenant / domain / entitlement truth"] --> PD[("platform_db<br/>Identity and platform-money ledger")]
  C["Commerce owns stock / orders / customer money"] --> CD[("commerce_db<br/>Snapshots, operations and evidence")]
  E["Experience owns revisions / publication"] --> ED[("experience_db<br/>Manifests, validation, live pointer")]
  M["Messaging owns channel identities / DMs"] --> MD[("messaging_db<br/>Conversations and delivery")]
  A["AI owns execution and proposals"] --> AD[("ai_db<br/>Jobs, model version, minimal history")]
  P -.->|"Versioned facts, never cross-DB writes"| Proj["Small eligibility copies in other owners<br/>Deduplication + freshness policy"]
  C -.->|"Public data only"| Cache["Renderer product cache<br/>Read-only and rebuildable"]
  CD -->|"Local transaction at sale"| Snap["Order / address snapshots<br/>Intentionally immutable"]
  E -->|"Complete objects before metadata commit"| Obj["Immutable object blobs<br/>Hashes and retention roots"]
  E -.->|"Scoped base revision snapshot"| A
  A -.->|"Patch proposal via API<br/>Human accept and publish"| E
  ED -.->|"Publication generation"| Pub["Renderer and CDN asset copies<br/>Tenant + revision + renderer version"]
  PD -.->|"Active host/generation with bounded cache age"| Route["Routing cache<br/>Cannot bypass revocation gate"]
```

Dashed copies are not writable masters. No owner writes another owner's database. Provider receipts and financial entries share an owner but remain distinct records with explicit recognition links.

## 5. Conceptual ERD — important local invariants

```mermaid
erDiagram
  COMMERCE_CUSTOMER |o--o{ COMMERCE_ORDER : "optional store-local account"
  COMMERCE_CART ||--o| COMMERCE_ORDER : "one conversion"
  COMMERCE_ORDER ||--|{ COMMERCE_ORDER_LINE : "tenant-qualified FK"
  COMMERCE_ORDER ||--o{ COMMERCE_RESERVATION : "stock held locally"
  COMMERCE_VARIANT ||--o{ COMMERCE_RESERVATION : "tenant-qualified FK"
  COMMERCE_ORDER ||--o{ CUSTOMER_MONEY_ENTRY : "recognized effects"
  CUSTOMER_OPERATION ||--o{ PROVIDER_RECEIPT : "execution evidence"
  PROVIDER_RECEIPT |o--o{ CUSTOMER_MONEY_ENTRY : "optional provider-backed recognition"
  PLATFORM_SUBSCRIPTION ||--o{ PLATFORM_INVOICE : "unique billing cycle"
  PLATFORM_INVOICE ||--o{ PLATFORM_MONEY_ENTRY : "separate platform ledger"
  EXPERIENCE_REVISION ||--|{ EXPERIENCE_FILE_REF : "immutable manifest"
  EXPERIENCE_REVISION ||--o{ EXPERIENCE_VALIDATION : "manifest and validator version"
  EXPERIENCE_REVISION ||--o{ EXPERIENCE_PUBLICATION : "validated pointer selection"
  MESSAGING_CONNECTION ||--o{ MESSAGING_CONVERSATION : "provider asset scope"
  MESSAGING_CONVERSATION ||--o{ MESSAGING_MESSAGE : "channel identity retained"
  MESSAGING_MESSAGE ||--o{ MESSAGING_DELIVERY_ATTEMPT : "unknown is possible"
```

This is a conceptual invariant map, not final DDL. All relationships stay within the named owner's database; every tenant-owned relation includes tenant_id. Guest orders have no customer-account relationship. Local order obligations/COD entries need no provider receipt; entries record an explicit source type/reference. A provider callback may recognize zero effects, and multiple receipts for one capture must not create multiple monetary entries.

## 6. Merchant registration and store preparation

```mermaid
sequenceDiagram
  actor Merchant
  participant P as Platform Control
  participant PD as platform_db
  participant C as Commerce
  participant CD as commerce_db
  participant E as Store Experience
  participant ED as experience_db and objects
  Merchant->>P: Register / create store with idempotency key
  P->>PD: TX identity, tenant, owner membership, outbox
  PD-->>P: Commit
  P-->>Merchant: Store ID, platform subdomain, preparing state
  par Durable per-recipient delivery
    P->>C: TenantCreated through outbox HTTP
    C->>CD: TX dedup, store settings, readiness outbox
    C-->>P: Durable accepted acknowledgment
  and Independent theme preparation
    P->>E: TenantCreated through outbox HTTP
    E->>ED: Initialize ready-template draft idempotently
    E-->>P: Durable accepted acknowledgment
  end
  C->>P: StoreCapabilityReady
  E->>P: StoreCapabilityReady
  P->>PD: Update capability progress with dedup
  Note over P,E: Failed capability retries independently. No cross-service DB write
  Merchant->>E: Validate and publish selected template
  Note over Merchant,E: AI and social onboarding do not block account creation
```

## 7. Customer browsing, store-local accounts and checkout

```mermaid
sequenceDiagram
  actor Shopper
  participant G as Ingress and Gateway
  participant P as Platform directory
  participant R as Public renderer
  participant E as Experience
  participant C as Commerce
  participant T as Trusted checkout and account web
  participant DB as commerce_db
  Shopper->>G: GET store host
  G->>P: Resolve verified active host or fresh cache
  P-->>G: Tenant and routing generation
  G->>R: Trusted tenant context
  R->>E: Approved publication manifest
  E-->>R: Immutable validated revision
  R->>C: Public catalog view model
  C-->>R: Public product data
  R-->>Shopper: Safe storefront HTML
  Shopper->>C: Add items via scoped public cart API
  Shopper->>C: Request trusted checkout handoff
  C-->>Shopper: Short-lived one-use store/cart capability
  Shopper->>T: POST capability to trusted origin
  T->>C: Redeem and bind trusted cart session
  alt Registered customer
    Shopper->>T: Store-local login / register
    T->>C: Validate store-scoped account
  else Guest
    Note over T,C: No account required. Scoped guest capability
  end
  T->>C: Checkout request and idempotency key
  C->>DB: TX reprice, reserve stock, snapshot order, convert cart, durable intent
  DB-->>C: Commit or stock conflict
  C-->>T: Order and pending payment operation, or COD state
  Note over C,DB: No provider call while transaction is open
```

## 8. Customer payment — Paymob or Fawry

```mermaid
sequenceDiagram
  actor Shopper
  participant C as Commerce API and domain processor
  participant DB as commerce_db
  participant W as Restricted payment worker
  participant PSP as Paymob / Fawry
  participant I as Commerce callback intake
  C->>DB: TX order, reservation, authorized payment intent
  W->>DB: Claim authorized operation and read minimal snapshot
  W->>PSP: Create payment session/reference with stable correlation
  alt Definitive session result
    PSP-->>W: Session/reference, not paid proof
    W->>DB: Append execution evidence
    Shopper->>C: Poll operation status
    C-->>Shopper: Hosted checkout URL or Fawry reference
    Shopper->>PSP: Pay through provider
  else Timeout or crash
    W->>DB: Mark unknown if possible. Lease recovery otherwise
    Note over W,PSP: Lookup/idempotency if supported. Otherwise review, no blind new charge
  end
  PSP->>I: Payment outcome callback
  I->>I: Provider-specific verification and account mapping
  I->>DB: Persist verified receipt with dedup
  DB-->>I: Commit
  I-->>PSP: Acknowledge durable receipt
  C->>DB: TX recognize effect, journal, order state, next work
  alt Reservation still valid and outcome eligible
    Note over C,DB: Order ready for authorized fulfillment
  else Paid after reservation expiry
    Note over C,DB: Paid-needs-resolution. Re-evaluate stock or durable refund/review
  end
  Note over Shopper,C: Browser return never marks an order paid
```

## 9. Subscription payment — Kashier, a separate money flow

```mermaid
sequenceDiagram
  actor Merchant
  participant B as Platform Billing
  participant DB as platform_db
  participant W as Restricted Kashier worker
  participant K as Kashier
  participant I as Platform billing intake
  participant S as Entitlement consumers
  Merchant->>B: Choose plan / pay invoice
  B->>DB: TX invoice, unique cycle, authorized collection intent
  W->>DB: Claim Billing operation only
  W->>K: Hosted payment or consent-backed token operation
  K-->>W: Execution result / payment URL
  Merchant->>K: Complete hosted payment when required
  K->>I: Payment/refund callback
  I->>DB: Verify and durably append receipt
  I-->>K: Acknowledge
  B->>DB: TX dedup, platform ledger, invoice, entitlement, outbox
  B->>S: SubscriptionEntitlementChanged via durable delivery
  Note over B,K: Platform owns schedule and consent. Kashier does not manage subscription plans
  Note over DB,S: Consumers store small entitlement projections, never this ledger
```

## 10. Shipping — Bosta first, J&T after contract verification

```mermaid
sequenceDiagram
  actor Staff
  participant C as Commerce
  participant DB as commerce_db
  participant W as Restricted shipping worker
  participant Carrier as Bosta / J&T
  participant I as Shipping callback intake
  Staff->>C: Confirm fulfillment
  C->>DB: TX check payment/COD eligibility, delivery snapshot, shipment intent
  W->>DB: Claim authorized operation
  W->>Carrier: Create shipment with stable reference
  alt Known result
    Carrier-->>W: Shipment ID / tracking / label
    W->>DB: Append execution evidence
  else Ambiguous create
    Note over W,Carrier: Reconcile by supported lookup or review before retry
  end
  Carrier->>I: Tracking observation
  I->>DB: Authenticate and persist receipt
  I-->>Carrier: Durable acknowledgment
  C->>DB: TX apply valid transition and fulfillment fact
  C-->>Staff: Status / actionable failure
  Note over C,DB: Delivered is not necessarily COD remitted or buyer-risk evidence
```

## 11. Unified private-message inbox

```mermaid
sequenceDiagram
  actor Staff
  actor Customer
  participant Meta as WhatsApp / Instagram
  participant I as Messaging callback intake
  participant M as Social Messaging
  participant DB as messaging_db
  participant W as Channel delivery worker
  Staff->>M: Connect account with store-bound authorization
  M->>Meta: Approved account authorization flow
  Meta-->>M: Account grant / scoped token
  M->>DB: Bind verified provider asset and encrypt token
  Customer->>Meta: Send private message
  Meta->>I: Inbound event
  I->>DB: Verify, resolve account, durably append receipt
  I-->>Meta: Acknowledge after commit
  M->>DB: TX dedup, channel-scoped thread/message, notification
  Staff->>M: Read unified inbox
  M-->>Staff: WhatsApp and Instagram threads with channel identity
  Staff->>M: Reply with idempotency key
  M->>DB: TX authorization, message and send intent
  M-->>Staff: Queued, not delivered
  W->>DB: Claim and check current connection/policy
  W->>Meta: Send via original account/channel/recipient
  alt Accepted
    Meta-->>W: Provider message ID
    W->>DB: Persist accepted evidence
    Meta->>I: Delivery observation where supported
    M->>DB: Apply deduplicated delivery state
  else Unknown send outcome
    Note over W,Meta: Reconcile if supported. Otherwise review, no blind resend
  end
```

## 12. Custom domain — automated but conditional

```mermaid
sequenceDiagram
  actor Merchant
  participant P as Platform Domains
  participant DB as platform_db
  participant W as Domain reconciliation worker
  participant DNS as Merchant authoritative DNS
  participant CF as Managed custom-hostname and TLS provider
  participant G as Routing gate / cache
  Merchant->>P: Enter hostname
  P->>DB: Normalize, unique claim, ownership challenge, job
  P-->>Merchant: Platform URL and exact required DNS/proof steps
  Merchant->>DNS: Change records or grant supported automation
  W->>CF: Idempotent hostname setup / fetch status
  par Independent readiness checks
    W->>DNS: Verify ownership and correct routing target
    DNS-->>W: Observed proof and routing
  and Certificate and hostname checks
    W->>CF: Read hostname and certificate/DCV states
    CF-->>W: Separate readiness states
  end
  W->>DB: Persist observations and next retry
  alt All gates pass and store eligible
    P->>DB: TX activate new routing generation and outbox
    P->>G: Durable invalidation / active mapping
    P-->>Merchant: Active HTTPS domain
  else External prerequisite or failure
    P-->>Merchant: Pending / action required with reason
    Note over W,CF: Automatic bounded retry. No human platform step or instant guarantee
  end
  opt Disconnect, suspension or ownership loss
    P->>DB: Revoke generation
    P->>G: Invalidate. Expired mapping cannot serve content
  end
```

## 13. Theme editing — direct code and actual AI patches

```mermaid
sequenceDiagram
  actor Merchant
  participant E as Store Experience
  participant DB as experience_db
  participant A as AI Assistance
  participant O as Immutable object storage
  participant V as Restricted validator
  participant R as Isolated preview / public renderer
  Merchant->>E: Select ready theme and read base revision
  alt Direct HTML/CSS/config edit
    Merchant->>E: Save files with expected base revision
  else AI chatbot edit
    Merchant->>E: Request change against base revision
    E->>DB: TX authorized job association and outbox
    E->>A: Durable ThemeEditRequested
    A->>E: Read scoped file snapshot
    A->>A: Bounded model work. No shell or business DB
    A->>E: Durable proposal with patch and base revision
    E-->>Merchant: Diff and explanation
    Merchant->>E: Accept exact proposal
  end
  E->>E: Check current permission, paths, limits and expected base
  E->>O: Upload immutable candidate files
  E->>DB: TX conditional expected-head check, revision, draft-head update, validation job
  alt Head changed concurrently
    DB-->>E: Conflict and rollback of metadata transaction
    E-->>Merchant: Reload or merge. No overwrite and no live change
    Note over DB,O: Unreferenced uploaded objects become eligible for later GC
  else Revision committed
    DB-->>E: New draft revision
    E->>V: Job-scoped artifacts, no privileged credentials
    V-->>E: Validation bound to manifest and validator version
    E->>DB: Record result
    alt Validation fails
      E-->>Merchant: Validation errors. Previous live revision unchanged
    else Validation passes
      E-->>Merchant: Expiring revision-pinned preview capability
      Merchant->>R: Open sandboxed preview
      R-->>Merchant: Render without trusted cookies
      Merchant->>E: Explicit publish revision and expected publication generation
      E->>DB: TX permission-bound validated selection, conditional pointer switch, outbox
      E->>R: Durable publication invalidation after successful commit
      Note over DB,R: Visibility is eventual. Publication conflict leaves live pointer unchanged
    end
  end
```

## 14. Employee authorization and revocation

```mermaid
sequenceDiagram
  actor Staff
  participant G as Gateway
  participant P as Platform authority
  participant E as Experience owner
  participant DB as experience_db
  Staff->>G: Publish revision for selected store
  G->>G: Validate issuer, audience, token type and tenant selection
  G->>E: Scoped delegation with original subject
  E->>P: Current publish permission and tenant eligibility
  alt Permission valid
    P-->>E: Bounded authority result
    E->>DB: TX tenant-scoped exact-revision publication and audit
    E-->>Staff: Publication generation
  else Removed staff / wrong store / unavailable authority
    P-->>E: Denied or unavailable
    E-->>Staff: No publication
  end
  Note over G,E: Workload identity authenticates caller. It does not grant user permissions
```

## 15. Deployment/infrastructure — honest Grad 1 runtime view

```mermaid
flowchart TB
  Internet["Browsers and provider callbacks"] --> TLS["Managed TLS edge / custom hostnames"]
  TLS --> Ingress["Only public ingress<br/>Host validation and route allowlist"]
  subgraph Host["Grad 1 Compose host — NOT high availability"]
    Ingress --> Web["Trusted merchant / admin / checkout web runtimes"]
    Ingress --> Public["Public renderer<br/>Separate trust origin"]
    Web --> GW["Thin Gateway"]
    GW --> APIs["Platform / Commerce / Experience / Messaging APIs<br/>AI job API private"]
    Ingress --> Intake["Owner-specific callback intake<br/>Restricted receipt append roles"]
    APIs --> PG[("One PostgreSQL instance<br/>Five owner databases; distinct roles")]
    Intake --> PG
    PG --> Work["Owner processors and dispatchers<br/>Claim jobs, dedup, local transitions"]
    Work --> ProviderW["Isolated payment / shipping / Billing / domain workers<br/>Provider-specific keys and limits"]
    Work --> SocialW["Social dispatch workers<br/>Channel credentials only"]
    Work --> AIW["AI inference worker<br/>Bounded model tools"]
    Work --> Validate["Restricted validator/evaluator<br/>No DB, no arbitrary egress"]
    Public -->|"Approved contracts only"| APIs
    Migrate["One-shot owner migrations<br/>No runtime DDL credentials"] --> PG
  end
  ProviderW --> Providers["Paymob / Fawry / Kashier / Bosta / J&T / DNS-TLS APIs"]
  SocialW --> Meta["WhatsApp / Instagram"]
  AIW --> Model["Selected model runtime/provider"]
  APIs --> Obj["R2/S3-compatible objects<br/>Owner-scoped keys and private drafts/media"]
  Work -.-> Telemetry["Redacted logs / metrics / traces / audit views"]
  APIs -.-> Telemetry
  PG -.-> Backup["Encrypted off-host backups<br/>DB + objects + key recovery; restore drills"]
  Obj -.-> Backup
```

Separate process roles, network permissions, secret injection and resource limits are mandatory. There are more runtimes than five APIs. A shared host, database instance and ingress remain common failure domains. Startup evolution adds managed recovery/replicas as justified; Kubernetes is not part of this diagram.

## 16. Future only — cross-store Trust without shared customer accounts

```mermaid
sequenceDiagram
  participant C as Commerce outcome owner
  participant T as Future Trust service
  participant TD as trust_db
  actor Shopper
  participant Checkout as Commerce checkout
  participant CD as commerce_db
  C->>T: Durable adjudicated COD outcome or correction
  T->>TD: TX dedup, source/version, minimized subject evidence, score
  Shopper->>Checkout: Checkout request
  Checkout->>T: Authorized assessment for this checkout context
  T-->>Checkout: Computed assessment, policy version, observed-at or unknown
  Checkout->>Checkout: Apply approved policy or explicit fallback
  Checkout->>CD: TX freeze assessment/prepayment policy, reserve stock, order
  Note over T,Checkout: No merchant raw cross-store history or unrestricted phone search
  Note over T,TD: Privacy, evidence quality, disputes and unknown policy are release gates
```

Future AI ordering must follow authorized Commerce commands and confirmation, not transform a social message into direct stock/payment database writes. It is not enabled by the Grad 1 theme-assistance permission set.
