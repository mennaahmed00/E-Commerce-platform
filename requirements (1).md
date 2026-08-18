# Requirements Document — Multi-Tenant E-Commerce SaaS Platform

**Status:** Draft / Planning phase
**Purpose:** Working document for requirements-gathering meetings. Items are grouped by module; MVP vs. future-roadmap split to be finalized as a team.

---

## 1. Overview

A multi-tenant, Shopify-style e-commerce SaaS platform for the Egyptian/MENA market, with deep AI integration as a core requirement. Merchants (tenants) run their own online stores under the platform; customers shop on merchant storefronts.

**Team split:**
- 2 members — AI-only
- 1 member — Backend + DevOps
- 3 members — Backend + Frontend (+ AI support)

**Planned scope split:** This document intentionally lists everything under consideration. Each item should be tagged `[MVP]` (Grad Project 1) or `[FUTURE]` (Grad Project 2 / post-grad roadmap) once the team agrees in the requirements meetings.

---

## 2. Actors

- **Platform Admin** — manages tenants, monitoring, billing oversight (internal)
- **Merchant / Store Owner** — the tenant; manages their store
- **Merchant Staff / Employee** — sub-accounts with limited, role-based permissions
- **Customer / Shopper** — browses and buys from a merchant's storefront
- **AI System** — chatbots, autonomous assistant, recommendation/CRM engine, acting as a system actor on behalf of merchant or customer

---

## 3. Feature List (as raw-captured, unordered)

1. User can edit his theme + ~5 ready themes, with multi-language support (Arabic + English)
2. Integration with multiple Egyptian payment gateways
3. Integration with multiple shipping companies
4. Integration with Meta for ads
5. Integration with Meta for handling all PMs in one place (WhatsApp / Instagram / etc.)
6. Trust score for all buyers across all stores — to know if a customer doesn't take his shipment a lot, so store owners can decide to cancel his order
   - Not all data visible to each store owner — only the "taking percent" is visible, and only flagged if it's below 50%; the store owner then decides
	-Low-risk buyer (high history of accepting COD shipments) → checkout stays fully Cash-on-Delivery, no friction.
	-Medium-risk buyer (some rejected shipments) → system automatically requires a partial prepayment before the order ships — e.g. "pay 20% now via card/wallet, pay the rest on delivery." This isn't a full block, just enough of a commitment to filter out casual/careless orders.
	-High-risk buyer (frequently rejects shipments) → system could require full prepayment or suggest the merchant reject/manually review the order.
7. Lots of dashboards for everything for the merchants, integrated with ads data and everything
8. Auto order confirmation through WhatsApp, or phone calls through AI, or any system like "press 1 to confirm"
9. Multi-role for store owners and their employees, so the owner can limit the integrations his employees can access
10. A "store" of all available integrations inside the platform, so store owners can pick and enable the ones they want to use
11. AI chatbot for store owners that edits the theme for them
12. AI chatbot / smart assistant for the store owner — a "Jarvis"-style assistant
13. The AI chatbot can edit anything in the store for the owner (products and everything)
14. Speech-to-text so the assistant can be interacted with like Jarvis
15. AI CRM that handles all CRM work autonomously
16. AI for media buying
17. **Far future:** video streaming, so people selling online courses can sell them on the platform
18. **Far future:** a supplier/factory area — factories showcase products, store owners browse and contact factories to place large wholesale orders
19. Search for products by uploading their images.
20. Return and exchange authorization
21. Platform control center: dashboard to manage brands,..etc
22. Instapay confirmation
23. Bulk product upload: take an excel sheet with all products’ info at once instead of providing one at a time 
24. Alert when low on stock
25. Brand owner can set a timer so when time is up an action happens(for instance: a new drop is released) 
26. Mobile application
27. Guide brands through the needed legal documents and process
28. Calculate for brand owners [cost-sales-profit..etc]
29 **Far future**:have all brands gathered and displayed (zay talabat) 
---

## 4. Functional Requirements by Module

### 4.1 Tenant & Store Management
- Store signup / onboarding flow
- Custom domain support per store
- Subscription / plan tiers (future consideration)
- Store settings management

### 4.2 Theme & Storefront Builder
- Drag-and-drop theme editor (Puck-based)
- ~5 ready-made themes provided out of the box
- Multi-language storefront support (Arabic + English), including RTL layout
- Save/preview vs. publish/live distinction

### 4.3 Product & Catalog Management
- Product CRUD, variants, inventory
- Categories
- Product search (Meilisearch)

### 4.4 Orders & Checkout
- Cart and checkout flow
- Order status lifecycle
- **Order confirmation automation**:
  - WhatsApp message-based confirmation `[MVP candidate]`
  - Simple IVR phone confirmation ("press 1 to confirm") `[MVP candidate]`
  - AI voice-conversation confirmation calls `[FUTURE — significant scope: STT/TTS in Arabic, telephony infra]`

### 4.5 Integrations
- **Payment gateways** — multiple Egyptian providers (e.g. Paymob, Fawry, Kashier — to be finalized)
- **Shipping companies** — multiple providers (e.g. Bosta, Mylerz — to be finalized)
- **Meta Ads** — ad campaign integration
- **Meta unified inbox** — WhatsApp, Instagram, Messenger handled from one place
- **Integrations marketplace** — a central "app store" where merchants browse and enable/disable available integrations for their store

### 4.6 Trust & Fraud Prevention
- **Cross-store buyer trust score**: tracks customers (likely keyed by phone number) across *all* stores on the platform to flag high COD-cancellation-risk buyers, so store owners can decide to cancel/hold orders.
  - Only the computed fulfillment percentage is exposed to a store owner, and only surfaced as a flag when it drops below 50% — no raw cross-store order history is shared.
  - Architecturally requires a **shared/global data layer** outside the schema-per-tenant isolation model (see Section 7 — Architecture Notes).
  - Needs a defined policy: what counts as a strike, how the score is calculated, data privacy handling.

### 4.7 Dashboards
To be broken into distinct use cases per actor rather than one generic requirement:
- Merchant dashboard — sales, orders, ads performance, AI CRM insights, all integrated in one view
- Employee dashboard — scoped by role/permissions
- Platform admin dashboard — tenant management, monitoring
- AI insights/analytics dashboard — may be part of merchant dashboard or standalone

### 4.8 Auth & Multi-Tenancy / Access Control
- Merchant authentication
- Customer authentication (per-store vs. platform-wide — **open decision**)
- Tenant data isolation (schema-per-tenant)
- **Multi-role access for store owner + employees**, including per-integration permission limits (e.g. an employee can be restricted from certain integrations)

### 4.9 Far-Future / Startup Roadmap Modules
- **Video streaming for online courses** — allow merchants selling online courses to host/stream video content directly on the platform
- **Supplier / factory marketplace** — a dedicated area where factories list products/production capacity, and store owners browse and contact them to place bulk/wholesale orders

---

## 5. AI Features — Full List

Grouping every AI capability discussed so far, customer-facing and merchant-facing.

### 5.1 Customer-Facing AI
- **Arabic RAG chatbot** (Qwen2.5-7B via Ollama) for storefront customer support — answers customer questions, defined scope for what data it can access, escalation path to human support
- **AI order confirmation** — WhatsApp bot / IVR / AI voice call flow (see 4.4)

### 5.2 Merchant-Facing AI — CRM & Insights
- **AI CRM layer**:
  - RFM customer segmentation (K-Means)
  - Churn prediction (XGBoost)
  - Arabic sentiment analysis on reviews (CAMeL-BERT)
  - **Fully autonomous AI CRM** — handling CRM actions/decisions without merchant micromanagement `[ambitious — needs scoped-down MVP version, e.g. autonomous *within defined guardrails* rather than fully unsupervised]`

### 5.3 Merchant-Facing AI — Product Tools
- AI-generated product descriptions (fine-tuned Flan-T5)
- Fake review detection

### 5.4 Merchant-Facing AI — Smart Assistant ("Jarvis")
- **AI chatbot/assistant for store owners** that can:
  - Edit the store's theme on the owner's behalf (natural-language → theme changes)
  - Edit/manage anything in the store (products, orders, settings) on the owner's behalf
  - Act as a general smart assistant across the merchant dashboard
- **Speech-to-text** interface so the assistant can be used conversationally (voice in, not just text)
  - ⚠️ Architectural note: this is a large feature — it implies an AI agent with *write access* to store data via tool-calling (not just a Q&A chatbot), plus a voice pipeline (STT, and likely TTS for responses). Needs to be scoped carefully; strong candidate to start as **text-only, read + limited-write actions** for MVP, with voice and full autonomous editing pushed to later phases.

### 5.5 Merchant-Facing AI — Marketing
- **AI for media buying** — AI-assisted or AI-managed ad spend/targeting across integrated ad platforms (e.g. Meta Ads)

---

## 6. Non-Functional Requirements

- **Multi-tenant data isolation** — security requirement, not just architectural detail
- **Performance under concurrent tenants**
- **Arabic language support** across UI (RTL) and AI layer
- **Scalability of the AI microservice**
- **Availability / uptime** expectations
- **Data privacy** — particularly for the cross-store trust score feature (shared customer data across independent businesses), and for any AI agent with write access to merchant data

---

## 7. Architecture Notes

### 7.1 Trust Score
- Requires a shared/global schema (outside tenant schemas) storing: normalized phone number, total orders, fulfilled orders, computed fulfillment percentage.
- Implemented as a dedicated module (e.g. `trust-score`) within the core NestJS API, with its own DB connection/schema separate from tenant schemas — no new microservice needed at MVP scale.
- Flow: Order module queries trust-score module before/at order confirmation → only `{ percentage, flagged }` returned to the tenant → merchant decides manually whether to proceed.
- Only exposes a computed result, never raw cross-store order history — privacy is enforced at the API contract level.

### 7.2 AI Smart Assistant ("Jarvis")
- Distinct from the customer-facing RAG chatbot — this is an **agent with tool-calling access** to store management functions (product CRUD, theme edits, settings), not just a Q&A system.
- Needs a defined action/permission boundary: which actions the AI can take autonomously vs. which require merchant confirmation.
- Voice layer (STT/TTS) is a separate concern from the agent's reasoning/action layer — can be added after the text-based agent works.

---

## 8. Open Questions for Team Discussion

1. Trust score: global data store design, scoring policy, privacy handling.
2. Customer auth model: per-store accounts or one platform-wide customer identity across all stores?
3. Confirm which payment gateways and shipping companies to target first.
4. AI phone confirmation: IVR only for MVP, or full AI voice conversation?
5. Employee roles: what's the full list of permission scopes needed at MVP?
6. AI smart assistant: what's the actual write-access boundary for MVP (read-only insights vs. limited actions vs. full store control)?
7. AI CRM: define what "autonomous" means in practice — fully unsupervised actions, or autonomous within merchant-approved rules?
8. How far do far-future items (video streaming, supplier marketplace) need to be documented for the defense vs. just mentioned as vision?

---

## 9. MVP (Grad 1) vs. Future Roadmap (Grad 2+)

*To be finalized in requirements meetings — recommend explicitly tagging every item above with `[MVP]` or `[FUTURE]` before moving to diagramming phase.*

**MVP — Grad Project 1**
- [ ] _(fill in during meeting)_

**Future — Grad Project 2 / Startup Roadmap**
- [ ] _(fill in during meeting)_

**Far-Future — Startup Vision (documented, not built)**
- [ ] Video streaming for online course sellers
- [ ] Supplier/factory marketplace area

---

## 10. Next Steps

1. Walk through Sections 4 and 5 as a team; tag each item MVP or Future.
2. Assign an owner per module (aligned with team split).
3. Once locked, proceed to diagramming phase: Use Case diagram → ERD → System architecture diagram → Sequence diagrams for key flows (checkout, AI chatbot query, tenant provisioning, trust score lookup, AI assistant action execution).
