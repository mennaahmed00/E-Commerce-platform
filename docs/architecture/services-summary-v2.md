# Architecture V2 — Service Distribution Summary

Status: **Proposed**, matching Architecture V2. This is a summary, not a new architecture decision.

The system has **five business services**. Each owns its responsibilities and data; background workers and frontend applications are additional runtimes, not additional business services.

## 1. Main services

| Service | Main responsibilities | Owned database | External integrations |
|---|---|---|---|
| **Platform Control** | Merchant/staff login, store creation, tenancy, employee permissions, platform administration, custom domains, subscription invoices and platform-money records | `platform_db` | **Kashier** for merchant-to-platform payments; DNS/TLS provider for domain activation |
| **Commerce** | Products, prices, inventory, carts, store-local customer accounts, guest checkout, orders, customer payments/refunds, shipping and COD records | `commerce_db` | **Paymob/Fawry** for customer-to-merchant payments; **Bosta** first, **J&T** after API-contract verification |
| **Store Experience** | Ready-made themes, HTML/CSS/declarative config editing, immutable revisions, validation, preview, publishing and rollback | `experience_db` plus object storage for theme files/assets | AI Assistance; Commerce's public catalog API |
| **Social Messaging** | Connected social accounts, unified private-message inbox, conversation history, staff replies and delivery/retry status | `messaging_db` plus private media storage | **WhatsApp Business Platform and Instagram**; replies use the originating account/channel |
| **AI Assistance** | Actual chatbot-driven theme-edit jobs, proposed file patches, explanations and evaluations | `ai_db` | Selected model runtime/provider; scoped Store Experience APIs |

Platform Control, Commerce, Store Experience and Messaging use **NestJS**; AI Assistance uses **FastAPI**. All five databases use **PostgreSQL**.

## 2. Important responsibility boundaries

- **Identity and Tenancy stay together** inside Platform Control. Store customers belong to Commerce, not the merchant/staff identity system. Customer accounts are store-specific; guests can checkout without registering.
- **Inventory and orders stay together** in Commerce so stock reservation and order creation can use one local transaction.
- **No generic Integrations service.** Each business owner keeps its provider adapters. Payment, shipping and domain work runs in separately restricted workers with scoped secrets and permissions.
- **Customer money and platform money stay separate.** Commerce owns Paymob/Fawry order payments; Platform Billing owns Kashier subscription payments. They never share one financial ledger.
- **Themes and social messages are outside Commerce.** Their editing, rendering, account and delivery lifecycles do not belong inside checkout.
- **AI proposes; the merchant approves publication.** Store Experience validates and owns the actual theme revisions. AI cannot directly publish, charge money or write another service's database.

## 3. Supporting applications and workers

| Component | Responsibility |
|---|---|
| API Gateway / ingress | Routing, authentication checks, trusted tenant resolution, request limits and correlation; no business workflows |
| Merchant dashboard | Manage the merchant's products, orders, staff, themes, inbox, integrations, domains and subscription |
| Platform admin dashboard | Manage platform tenants, plans, billing, moderation and operational issues; no automatic unrestricted access to merchant private data |
| Trusted checkout / customer-account application | Handle shopper login and checkout outside merchant-controlled theme content |
| Public renderer and validators | Render approved themes and validate edits in restricted environments without business-database or payment credentials |
| Background workers | Execute payments, shipping, social delivery, domain checks, AI jobs and durable event delivery |

## 4. Data and communication rules

- One logical database per service, initially on **one PostgreSQL instance** with separate credentials. This is not high availability.
- Tenant-owned tables use **`tenant_id` + enforced Row-Level Security (RLS)**, tenant-aware constraints and validated transaction context—not schema-per-tenant.
- Services exchange data through APIs or events. **No direct cross-service database access.** Copies are read-only projections or intentional historical snapshots.
- Interactive reads and local changes use synchronous HTTP APIs. Slow/external work uses durable background jobs.
- Cross-service events use transactional **outbox/inbox delivery over authenticated HTTP**, with deduplication and bounded retries. No event broker is required in Grad 1.
- Save orders and operation intent before contacting payment providers. Verified outcomes update the owner's records; an uncertain timeout requires reconciliation, not a blind duplicate charge or send.

## 5. MVP versus later

**Grad 1 targets:** all five services in narrow form, both dashboards, safe direct/AI theme editing, recoverable checkout, named payment flows, Bosta shipping, real WhatsApp/Instagram receive-and-reply, and automatic custom-domain onboarding.

Custom domains activate only after ownership, DNS, certificate and routing checks pass. Automation does not guarantee instant external DNS changes. Live provider approvals and sandbox acceptance remain open delivery gates.

**Later:** J&T after its verified contract, richer messaging, advanced billing/analytics, and a separate restricted **Trust/Risk service**. MongoDB, Redis, Meilisearch, pgvector, an event broker and Kubernetes are not default Grad 1 infrastructure; add them only for demonstrated needs.

## Further detail

[Architecture V2](architecture-v2.md) · [Full service catalog](service-catalog-v2.md) · [Data ownership](data-ownership-v2.md) · [Communication](communication-v2.md) · [Rendered diagrams](diagrams-v2-rendered.md)
