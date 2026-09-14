# Architecture V2 — supervisor explanation

Status: proposed after six independent specialist reviews and a completed challenge round. [Full architecture](architecture-v2.md) · [Drawings](diagrams-v2.md) · [Rendered drawings](diagrams-v2-rendered.md)

## The whole system in plain English

We are building a platform where merchants open online stores. Each merchant manages their own products, orders, employees, theme and social inbox. Platform administrators manage the platform itself, not automatically every merchant's private data. Shoppers can create an account for a particular store or buy as guests.

The backend has five business owners:

| Service | How to explain its job |
|---|---|
| Platform Control | Knows who runs each store, who may do what, whether the store is active, its domains and what it owes the platform |
| Commerce | Knows what is for sale, what stock is available, what customers ordered, what they paid and what must be shipped |
| Store Experience | Owns what a store looks like: theme files, edits, safe previews, published versions and rollback |
| Social Messaging | Brings WhatsApp/Instagram private messages into one inbox and sends replies through the correct account |
| AI Assistance | Suggests actual theme-code changes in a controlled job; it cannot publish or touch money by itself |

Each service has its own PostgreSQL database and credentials, initially on one database server. This gives clear ownership without operating five database clusters. Within each database, stores share tables, but every tenant row identifies its store and database policies enforce isolation. The policy is a guard against mistakes, not a promise that a fully compromised service cannot harm multiple stores.

A shopper sees a published theme and public product information. When they buy, trusted checkout recalculates the order and reserves stock inside Commerce. A background worker asks Paymob or Fawry to handle payment. Only verified evidence can update the order's money records. Shipping proceeds through an authorized, recoverable operation, with Bosta first and J&T after its API contract is available.

A merchant paying a subscription is a different transaction. Platform Billing creates the invoice and uses Kashier. It never marks a customer order paid. The two flows may share programming utilities, but they do not share balances, credentials or business authority.

For design, the merchant starts from a ready-made theme and edits HTML, CSS or declarative configuration, directly or with AI. The system shows a diff and safe preview. Publication selects a validated immutable version; failed edits do not replace the live store. Merchant markup never becomes trusted checkout, account or administrator code.

For social messages, the system remembers the platform, connected account and conversation. A reply from our dashboard goes back through that same account. It records whether delivery is queued, accepted, delivered, failed or unknown; it does not treat clicking Send as successful delivery.

For custom domains, the platform automatically verifies the claim, checks DNS and certificate readiness, then activates routing. The merchant may need to change DNS or authorize automation. No platform employee is needed for normal activation, but external DNS and certificate issuance cannot be guaranteed instantaneous. A platform subdomain is the fallback.

## How to explain this to a supervisor — a short presentation

1. **Start with the context drawing.** “This platform serves merchants, shoppers and platform administrators. The external systems are payment providers, carriers, social networks, a DNS/TLS provider and an AI model.”
2. **Show the five-service drawing.** “We split by business responsibility. We keep inventory and orders together because checkout needs an atomic stock decision. Themes and social messaging are separate because they have different data and failure behavior.”
3. **Show data ownership.** “Each business fact has one writer. Other components get API responses or read-only copies. They cannot join or edit another service's database.”
4. **Trace one payment.** “First we save an order and intent. Then a worker contacts the provider. We verify and record the outcome. If the provider times out, we investigate the existing operation rather than making another charge.”
5. **Trace one AI edit.** “AI proposes a patch to a specific theme version. Our backend checks it; the merchant reviews and explicitly publishes it. The model has no database or deployment shell.”
6. **Close with the trade-off.** “Grad 1 uses PostgreSQL, object storage and durable workers, without a broker or multiple unused databases. The ownership and message contracts allow later scaling, but this single-host deployment is not high availability.”

Use diagrams 1–4 for the overview, 8 for correctness, 11 for the inbox, 12 for domains and 13 for safe AI editing. Diagram 15 shows the real runtime cost; diagram 16 is future scope, not delivered Grad 1 functionality.

## Why V2 is substantially different from V1

| Change | Short defense |
|---|---|
| Shared tables + tenant_id + RLS instead of per-tenant schemas | One migration stream per service is a better student-team default; tenant-aware constraints and tested roles enforce the actual boundary |
| Remove generic Integrations | Payment outcomes can be recognized in the same owner database as orders/invoices; restricted workers still isolate secrets and timeouts |
| Separate Store Experience | File authoring, validation, revisions and publication now form a complete capability, not a small theme-settings table |
| Separate real social Messaging | External account credentials, channel rules and uncertain delivery are not checkout responsibilities |
| Real AI theme assistance | Directly satisfies the updated requirement; broader autonomous commerce remains future work |
| Distinct customer and platform payments | Different money owners require different invoices, journals, credentials and policies |
| Explicit trusted UI origins | Merchant theme code must not become payment, login or admin code |

## The disagreement worth explaining

The security reviewer preferred a dedicated Integrations service because its credential boundary is obvious to audit. The domain reviewer argued that it adds a distributed financial handoff while grouping unrelated provider operations. Backend, data, platform and critic reviewers revised toward the domain proposal after debating those costs.

The lead selected domain-owned execution **with required separate process/credential/role controls**, not by vote. Ordinary APIs cannot decrypt provider keys. Provider workers cannot freely edit financial journals or memberships. Trusted processors apply verified evidence. If those boundaries cannot be demonstrated, the decision must be reopened. Separating a service by name would not itself fix shared credentials or a shared host.

## Questions a supervisor is likely to ask

**Why not split products, inventory and orders?** Their checkout invariants currently benefit from one transaction. Splitting them would add distributed stock reservation and compensation before there is an independent team or load requirement.

**Why not put everything in a monolith?** That remains simpler to deploy, but confirmed untrusted theme processing, social credentials and model execution need failure/security boundaries now. Five coarse owners with modular internals balance those requirements; this is not a microservice per table.

**Why PostgreSQL for messages?** Messages still have store, account, conversation, ordering, delivery and permission relationships. JSONB accommodates provider metadata without adding MongoDB's backup/query/operating model prematurely.

**No broker—can it still be asynchronous?** Yes. Durable database jobs and transactional outboxes are delivered by workers over authenticated HTTP. A broker can replace delivery transport later; it does not remove the need for deduplication or correct financial state transitions.

**Does every service store the same customer?** No. Commerce owns store-local accounts. Messaging owns channel identities. Only an explicit verified association can link them. Copies of public product data or order snapshots have documented purposes and are not writable masters.

**What if payment succeeds while our worker crashes?** The operation remains unresolved. We use a verified callback or supported provider lookup/idempotency to reconcile it. If the result cannot be established, the UI and operator workflow show unknown; we do not blindly charge again.

**Can an AI-generated theme break checkout?** It may produce an invalid or deceptive design, which is why validation, preview, restricted rendering, moderation and human publication exist. Checkout and account screens do not execute merchant templates and revalidate commercial data independently.

**Will a custom domain work immediately?** The platform starts automation immediately and needs no normal human operator. The domain becomes active only after ownership, DNS, TLS and routing are ready. External DNS control and propagation are real dependencies, not problems software can wish away.

**Is this realistic for six students?** It is an ambitious revised scope, especially with four application developers. Deliver narrow vertical slices, begin provider approvals early, let the two AI students work against the bounded revision contract, and defer unused infrastructure/advanced features. Do not claim all live provider approvals or a semester schedule without evidence.

## What is MVP, and what is not?

Grad 1 targets: both dashboards, merchant/staff/store setup, catalog and stock, store-local customers/guests, recoverable checkout, Paymob/Fawry, first Bosta shipping adapter, Kashier subscription invoice/payment, ready themes with direct/AI editing, WhatsApp/Instagram receive/reply and automated domains. Delivery is staged, not all-at-once.

Open acceptance choices: live versus sandbox providers; J&T timing; settlement recipient/account onboarding; plan/grace/renewal policy; delayed-payment reservation duration; supported social media/history/retention; model/privacy terms; domain plan/limits; recovery targets. These need explicit supervisor/product confirmation, not silent assumptions.

Later: J&T after contract, richer channels/media, advanced analytics/returns, consented renewal automation if not ready earlier, Trust scoring, conversational ordering/voice/CRM, search/vector projections and dedicated tenant placement. Redis, broker and Kubernetes are additions only when demonstrated needs justify them.

## One-sentence conclusion

Architecture V2 keeps money and stock decisions close to their owner, separates the new theme/social/AI capabilities, and makes uncertain external work recoverable without assuming a student team can operate a startup-scale platform from day one.
