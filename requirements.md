# Requirements Document — Multi-Tenant E-Commerce SaaS Platform

**Status:** Post-meeting draft — feature list and MVP finalized as of this version
**Purpose:** Working document for the requirements phase. MVP scope below is locked; remaining items are Future / Far-Future roadmap.

---

## 1. Overview

A multi-tenant, Shopify-style e-commerce SaaS platform for the Egyptian/MENA market, with deep AI integration as a core requirement. Merchants (tenants) run their own online stores under the platform; customers shop on merchant storefronts.

---

## 2. Actors

- **Platform Admin** — manages tenants, monitoring, billing oversight (internal)
- **Merchant / Store Owner** — the tenant; manages their store
- **Merchant Staff / Employee** — sub-accounts with limited, role-based permissions
- **Customer / Shopper** — browses and buys from a merchant's storefront
- **AI System** — chatbots, autonomous assistant, recommendation/CRM engine, acting as a system actor on behalf of merchant or customer
- **Factory / Supplier** — far-future actor, lists products/capacity for merchants to place bulk orders

---

## 3. Final Feature List (post-meeting)

1. User can edit his theme + ~5 ready themes, with multi-language support (Arabic + English)
2. Integration with multiple Egyptian payment gateways
3. Integration with multiple shipping companies
4. Integration with Meta for ads
5. Integration with Meta for handling all PMs in one place (WhatsApp / Instagram / etc.)
6. Trust score for all buyers across all stores — tiered risk system (see Section 4.6 for full logic)
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
19. Search for products by uploading their images (visual search)
20. Return and exchange authorization
21. Platform control center — dashboard to manage brands, etc.
22. Instapay confirmation
23. Bulk product upload — take an Excel sheet with all products' info at once instead of one at a time
24. Alert when low on stock
25. Brand owner can set a timer so when time is up an action happens (e.g. a new drop is released)
26. Mobile application
27. Guide brands through the needed legal documents and process
28. Calculate for brand owners (cost / sales / profit, etc.)
29. **Far future:** have all brands gathered and displayed in one place (like Talabat)
30. AI buys on the buyer's behalf and places orders
31. AI recommendations from order history
32. Clothing preview (virtual try-on)

---

## 4. Functional Requirements by Module

### 4.1 Tenant & Store Management
- Store creation / onboarding flow
- Custom domain support per store
- Store settings management
- Guide brands through required legal documents and process (item 27)

### 4.2 Theme & Storefront Builder
- Drag-and-drop theme editor (Puck-based)
- ~5 ready-made themes (MVP: 2–3 themes)
- Multi-language storefront support (Arabic + English), including RTL layout
- Save/preview vs. publish/live distinction
- Drop timer — brand owner sets a countdown; when it ends, a defined action triggers (e.g. a new drop goes live) (item 25)
- **Raw code editing** — merchants can view/edit their theme's underlying code directly (like Shopify's "Edit code"), plus add their own custom code files. AI assistant (5.4) should also be able to edit these files on the merchant's behalf. `[Grad 2 — implementation approach not yet decided; see Section 7.4]`

### 4.3 Product & Catalog Management
- Product CRUD, variants, inventory
- Categories
- Product search (Meilisearch)
- Bulk product upload via Excel sheet (item 23)
- Low-stock alerts (item 24)
- Visual/image-based product search (item 19)
- Clothing preview / virtual try-on (item 32)

### 4.4 Orders & Checkout
- Cart and checkout flow
- Order status lifecycle
- Return and exchange authorization (item 20)
- Instapay confirmation (item 22)
- **Order confirmation automation**:
  - WhatsApp message-based confirmation
  - Simple IVR phone confirmation ("press 1 to confirm")
  - AI voice-conversation confirmation calls `[FUTURE — significant scope: STT/TTS in Arabic, telephony infra]`

### 4.5 Integrations
- **Payment gateways** — multiple Egyptian providers (MVP: 1 gateway integration)
- **Shipping companies** — multiple providers (MVP: 1 delivery integration)
- **Meta Ads** — ad campaign integration
- **Meta unified inbox** — WhatsApp, Instagram, Messenger handled from one place (MVP: basic message box)
- **Integrations marketplace** — a central "app store" where merchants browse and enable/disable available integrations for their store

### 4.6 Trust & Fraud Prevention — Tiered Buyer Trust Score
Cross-store buyer trust score, tracking customers (likely keyed by phone number) across *all* stores to flag COD-cancellation risk. Only the computed "taking percent" is visible to a store owner — never another store's raw order history.

**Risk tiers and checkout behavior:**
- **Low-risk** (high history of accepting COD shipments) → checkout stays fully Cash-on-Delivery, no added friction
- **Medium-risk** (some rejected shipments; percentage flagged, below 50%) → system automatically requires partial prepayment before shipping (e.g. "pay 20% now via card/wallet, rest on delivery") — filters out casual/careless orders without a full block
- **High-risk** (frequently rejects shipments) → system requires full prepayment, or flags the order for the merchant to manually review/reject

Architecturally requires a shared/global data layer outside the schema-per-tenant isolation model (see Section 7).

### 4.7 Dashboards
- Merchant dashboard — sales, orders, ads performance, AI CRM insights, all integrated in one view
- Employee dashboard — scoped by role/permissions
- **Platform control center** — internal admin dashboard to manage brands/tenants across the platform (item 21)
- AI insights/analytics dashboard — may be part of merchant dashboard or standalone
- Brand financial dashboard — cost / sales / profit calculation for brand owners (item 28)

### 4.8 Auth & Multi-Tenancy / Access Control
- Merchant authentication
- Customer authentication: per-store accounts (not one platform-wide identity), and **optional** — guest checkout supported, matching the pattern used by both Shopify and Vondera. Phone number is still captured at checkout regardless of account status, since the trust score depends on it, not on having an account.
- Tenant data isolation (schema-per-tenant)
- Multi-role access for store owner + employees, including per-integration permission limits — employee system (item 9)

### 4.9 Mobile Application
- Native/cross-platform mobile app (item 26) — scope (merchant-facing, customer-facing, or both) to be defined

### 4.10 Far-Future / Startup Roadmap Modules
- **Video streaming for online courses** — merchants selling courses can host/stream video content on the platform (item 17)
- **Supplier/factory marketplace** — factories list products/capacity; store owners contact them for bulk/wholesale orders (item 18)
- **Unified brand marketplace** — all brands on the platform gathered and displayed in one consumer-facing place, similar to Talabat (item 29)

---

## 5. AI Features — Full List

### 5.1 Customer-Facing AI
- Arabic RAG chatbot (Qwen2.5-7B via Ollama) for storefront customer support
- AI order confirmation — WhatsApp bot / IVR / AI voice call flow
- Visual/image-based product search (item 19)
- Clothing preview / virtual try-on (item 32)
- AI recommendations generated from order history (item 31)
- **AI buys on the buyer's behalf** — an agent that can place orders for the customer automatically (item 30) `[ambitious — needs a defined trigger/permission model, e.g. reorder subscriptions or explicit customer authorization per purchase]`

### 5.2 Merchant-Facing AI — CRM & Insights
- AI CRM layer: RFM segmentation (K-Means), churn prediction (XGBoost), Arabic sentiment analysis (CAMeL-BERT)
- Fully autonomous AI CRM `[needs a scoped-down MVP version — autonomous within merchant-approved guardrails, not fully unsupervised]`

### 5.3 Merchant-Facing AI — Product Tools
- AI-generated product descriptions (fine-tuned Flan-T5)
- Fake review detection

### 5.4 Merchant-Facing AI — Smart Assistant ("Jarvis")
- AI chatbot/assistant for store owners that can:
  - Edit the store's theme on the owner's behalf
  - Edit/manage anything in the store (products, orders, settings) on the owner's behalf
  - Act as a general smart assistant across the merchant dashboard
- Speech-to-text interface for conversational/voice interaction
  - Architecturally an agent with *write access* via tool-calling, not just Q&A — needs a defined action/permission boundary; strong candidate for text-only, limited-write scope pre-MVP, with voice and full autonomy as later phases

### 5.5 Merchant-Facing AI — Marketing
- AI for media buying — AI-assisted/managed ad spend and targeting across integrated ad platforms

---

## 6. Non-Functional Requirements

- Multi-tenant data isolation — security requirement, not just architectural detail
- Performance under concurrent tenants
- Arabic language support across UI (RTL) and AI layer
- Scalability of the AI microservice
- Availability / uptime expectations
- Data privacy — trust score's cross-store data, and any AI agent with write access to merchant data or purchasing authority (item 30 especially)

---

## 7. Architecture Notes

### 7.0 Overall Architecture
- **Microservices-based** (required by supervisor, also the team's own preference).
- Core stack: NestJS (backend services), Next.js (frontend), FastAPI (AI microservice), PostgreSQL (schema-per-tenant).
- Candidate service boundaries: core commerce service (tenants, products, orders, checkout, employees/roles), AI service (FastAPI), trust score service (naturally standalone — the one genuinely cross-tenant service), integrations service (payment/shipping/Meta adapters, isolated so external API outages don't affect checkout), chat/messaging service (unified inbox + AI conversational commerce).
- MongoDB alongside Postgres — under consideration, not yet decided. Best-fit candidate use case if adopted: AI chat/conversation history (Jarvis assistant, customer chatbot, conversational checkout) — high-volume, variable-shaped data with little need for relational joins. Not needed for flexible product/theme data, since Postgres `jsonb` already covers that.

### 7.1 Trust Score (Tiered)
- Shared/global schema (outside tenant schemas) storing: normalized phone number, total orders, fulfilled orders, computed fulfillment percentage, current risk tier.
- Implemented as a dedicated module (e.g. `trust-score`) within the core NestJS API, own DB connection/schema — no separate microservice needed at MVP scale.
- Flow: Order module queries trust-score module at checkout → returns `{ percentage, tier }` → checkout flow branches: low-risk = normal COD, medium-risk = partial prepayment required, high-risk = full prepayment or manual merchant review.
- Only a computed result is ever exposed cross-tenant, never raw order history — privacy enforced at the API contract level.

### 7.2 AI Smart Assistant ("Jarvis")
- Distinct from the customer-facing RAG chatbot — an agent with tool-calling access to store management functions, not just Q&A.
- Needs a defined action/permission boundary: which actions the AI can take autonomously vs. which require merchant confirmation.
- Voice layer (STT/TTS) is separate from the agent's reasoning/action layer — add after the text-based agent works.

### 7.3 AI Buying Agent (item 30)
- Needs a clear trigger model before design: is this (a) automated reordering of previously-bought items, (b) an agent acting on a standing customer instruction ("buy X when price drops"), or (c) fully autonomous purchasing without per-order confirmation? Each has very different authorization/payment implications — flag for further discussion before scoping.

### 7.4 Raw Code Theme Editing + AI Code Editing
- **Decision deferred** — not a simple extension of the Puck-based editor (4.2); Puck has no underlying "file" to show a merchant, so this requires either a separate file-based theming system or a different editor library entirely.
- Options under consideration, to be decided later:
  - Replace/supplement the theme layer with a Liquid-style file-based system (e.g. LiquidJS) + embedded code editor (e.g. Monaco) + per-tenant theme files in storage.
  - Keep Puck as the default no-code path, add raw-code editing as a separate "Advanced" mode (mirrors Shopify's own sections-editor + Edit-code split).
  - Switch the theme editor entirely to a library with native HTML/CSS editing (e.g. GrapesJS) instead of Puck.
- AI editing of raw code files (item 13/14, tied to the Jarvis assistant in 5.4) would follow a read/edit/write/validate pattern on whatever file format is chosen — needs a validation/lint step before publishing, since unlike Puck's typed fields, raw code gives the AI room to produce something broken.

---

## 8. MVP — Software Scope (Grad Project 1, agreed in meeting)

- Theme editor
- Create store
- Basic admin dashboard
- Admin dashboard
- Employee system
- 2–3 themes
- 1 payment gateway integration
- 1 delivery/shipping integration
- Message box

*Everything else in Sections 3–5 not listed above is Future (Grad Project 2) or Far-Future (startup roadmap), by default, unless the team explicitly pulls an item forward.*

