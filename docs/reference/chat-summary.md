# Chat History Summary — E-Commerce SaaS Grad Project

**Purpose:** Full context dump of everything discussed so far, to paste into a new chat so it continues with complete context.

---

## 1. Who / What

- Mostafa — CS student at Nile University.
- Building a **two-semester graduation project** (Grad 1 = MVP, Grad 2 = expanded scope) that is also intended to become a **real startup**.
- Project: a **multi-tenant, Shopify-style e-commerce SaaS platform** for the Egyptian/MENA market, with a heavy AI layer (the team's original direction was pushed back on by their supervisor for not being AI-heavy enough, hence the strong AI feature set).
- **Team of 6:**
  - 2 members — AI-only
  - 1 member — Backend + DevOps
  - 3 members — Backend + Frontend (+ help with AI work too)
- Team background: had just finished the **IBM Full Stack Developer course on Coursera**, specifically the Node.js and React portions, before this stack was chosen — so they already know plain Node.js and plain React fundamentals.

---

## 2. Tech Stack (confirmed)

| Layer | Choice |
|---|---|
| Backend | **NestJS** (TypeScript, runs on Node.js/Express under the hood) |
| Frontend (merchant dashboard + storefront) | **Next.js** (App Router) |
| Mobile | React Native / Expo |
| AI microservice | **FastAPI** (Python) |
| Database | **PostgreSQL**, **schema-per-tenant** multi-tenancy |
| Cache | Redis |
| Search | Meilisearch |
| Vector store | pgvector |
| Object storage | Cloudflare R2 |
| Custom domains | Wildcard DNS + Cloudflare for SaaS |
| Storefront drag-and-drop theme editor | **Puck** (React-native visual editor) |
| Overall architecture style | **Microservices** — required by their supervisor, and also the team's own preference |
| MongoDB | **Not decided.** Best-fit candidate use case if adopted: AI chat/conversation history (high-volume, variable-shaped, little need for joins). Not needed for flexible product/theme data since Postgres `jsonb` already covers that. Decision: don't add it unless a real pain point emerges. |

**AI models discussed for the AI microservice:**
- Arabic RAG chatbot: Qwen2.5-7B via Ollama
- RFM customer segmentation: K-Means
- Churn prediction: XGBoost
- Arabic sentiment analysis (on reviews): CAMeL-BERT
- AI-generated product descriptions: fine-tuned Flan-T5
- Fake review detection (model not yet specified)
- Training environment: CPU laptops + Google Colab Pro (T4) for heavier jobs

### Stack reasoning already covered (don't re-litigate unless asked)
- **NestJS over plain Express**: NestJS *is* Node.js (built on Express/Fastify), just more structured (modules, DI, decorators, native TypeScript). For a 6-person team building multi-tenant SaaS, NestJS's dependency injection makes tenant-scoped data isolation and auth enforcement structural rather than something every developer has to remember to check manually. Better for team consistency and for a startup that will be maintained/hired-for long-term.
- **Next.js over plain React**: the storefront is public-facing and needs SEO (Next.js gives SSR/SSG out of the box) and needs wildcard/custom-domain routing (Next.js middleware handles this cleanly). Plain React would need this built manually.
- **Should still learn plain Express and plain React fundamentals** (which they already have via IBM course) since NestJS/Next.js are additive layers on top, not replacements — understanding the underlying layer helps when debugging.
- **Puck chosen for the theme/drag-and-drop editor** over Craft.js and GrapesJS:
  - Puck is React-native and JSON-driven (define a component config → editor produces JSON → same config + a `Render` component displays the live page). This mirrors Shopify's own sections/schema model.
  - Craft.js is headless — would require building the entire editor UI from scratch (too much work for the team/timeline).
  - GrapesJS is HTML/CSS-string based, not native React components — works with Next.js but needs iframe/SSR workarounds, and is a style mismatch with the rest of a React-native codebase. GrapesJS's one edge: native MJML email-builder support, and it natively supports raw HTML/CSS viewing/editing (relevant to Section 7 below).

---

## 3. Full Feature List (32 items, finalized post-meeting)

1. User can edit his theme + ~5 ready themes, with multi-language support (Arabic + English)
2. Integration with multiple Egyptian payment gateways
3. Integration with multiple shipping companies
4. Integration with Meta for ads
5. Integration with Meta for handling all PMs in one place (WhatsApp / Instagram / etc.)
6. **Trust score** for all buyers across all stores — tiered risk system:
   - Low-risk (high history of accepting COD) → checkout stays fully Cash-on-Delivery, no friction
   - Medium-risk (some rejected shipments; below 50% "taking percent") → system requires partial prepayment before shipping (e.g. "pay 20% now, rest on delivery")
   - High-risk (frequently rejects shipments) → system requires full prepayment, or flags for manual merchant review
   - Only the computed "taking percent" is visible to a store owner — never another store's raw order history
7. Lots of dashboards for everything for the merchants, integrated with ads data and everything
8. Auto order confirmation through WhatsApp, or phone calls through AI, or any system like "press 1 to confirm"
9. Multi-role for store owners and their employees, so the owner can limit the integrations his employees can access
10. A "store"/marketplace of all available integrations inside the platform, so store owners can pick and enable the ones they want to use
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
23. Bulk product upload — Excel sheet with all products' info at once
24. Alert when low on stock
25. Brand owner can set a timer so when time is up an action happens (e.g. a new drop is released)
26. Mobile application (**clarified: merchant-facing only, planned for Grad 2**)
27. Guide brands through the needed legal documents and process
28. Calculate for brand owners (cost / sales / profit, etc.)
29. **Far future:** have all brands gathered and displayed in one place (like Talabat)
30. AI buys on the buyer's behalf and places orders — **clarified meaning**: a customer on a store's social media (WhatsApp/Instagram) can finalize an entire order just by chatting with the AI chatbot (conversational commerce), not a fully autonomous unsupervised buying agent
31. AI recommendations from order history
32. Clothing preview (virtual try-on)

**Raw code theme editing (added after the 32-item list, not numbered in original list):** merchants should be able to view/edit their theme's underlying code directly (like Shopify's "Edit code"), plus add their own custom code files — and the AI assistant should be able to edit these files on the merchant's behalf too. Architecture for this is **not yet decided** (see Section 7).

---

## 4. MVP — Software Scope (Grad Project 1, locked in meeting)

- Theme editor
- Create store
- Basic admin dashboard
- Admin dashboard
- Employee system
- 2–3 themes
- 1 payment gateway integration
- 1 delivery/shipping integration
- Message box

Everything else in the 32-item list is Future (Grad 2) or Far-Future (startup roadmap) by default, unless explicitly pulled forward.

Implicit prerequisites for the MVP to actually be demoable (agreed via reasoning, not explicitly listed by the team): basic product CRUD and a basic checkout/order flow.

---

## 5. Key Decisions Made During Discussion

- **Architecture**: microservices — confirmed as a hard supervisor requirement, and the team's own preference.
  - Candidate service boundaries discussed: **Core Commerce Service** (tenants, products, orders, checkout, employees/roles), **AI Service** (FastAPI), **Trust Score Service** (naturally standalone — the one genuinely cross-tenant service), **Integrations Service** (payment/shipping/Meta adapters — isolated so external API outages don't take down checkout), **Chat/Messaging Service** (unified inbox + AI conversational commerce).
  - Caution raised: true microservices add real operational cost (service discovery, inter-service auth, distributed transactions, deployment complexity for 6 people). A "modular monolith" (NestJS with clearly separated modules/schemas) was floated as a lower-risk starting point, splitting only AI/trust-score into real services — but since microservices is a hard requirement, this is more of a "how aggressively to split" question, not whether to.
- **Customer accounts**: per-store (not one platform-wide identity), and **optional** — guest checkout supported. This matches confirmed research: both **Shopify** (accounts optional, guest checkout available) and **Vondera** (has a `Customer Login` API endpoint but also supports guest/session-based checkout per their API docs) work this way. Phone number is still captured at checkout regardless of account status, since the trust score depends on phone number, not account status.
- **Trust score validation**: researched and confirmed this is a real, validated pattern in the MENA COD market — Shopify has native per-merchant fraud scoring (Low/Medium/High), and third-party apps built specifically for MENA/Africa (e.g. CODShield, COD Sentry, Codify) do phone-number-based COD risk scoring. Their **cross-store shared trust score** (one customer's behavior at Store A informing Store B) is more ambitious than anything found — a genuine differentiator, not just a checkbox feature.
- **AI buying agent (item 30) clarified**: not an autonomous agent making purchase decisions — it's conversational commerce (customer finalizes order via chat on WhatsApp/Instagram with the AI chatbot).

---

## 6. Study Plan Discussed (for the team, given IBM Node/React background)

Recommended order:
1. Quick Express pass (a few hours) — just enough to recognize what NestJS abstracts (middleware, routing, req/res cycle). Skip if IBM course covered it well.
2. NestJS fundamentals — modules, controllers, providers/services, dependency injection. Study **request-scoped providers** specifically, since that's how schema-per-tenant DB routing gets implemented cleanly.
3. Next.js App Router — file-based routing, server vs. client components (main source of confusion), the `middleware.ts` file for wildcard subdomain routing.
4. Multi-tenant data isolation pattern — schema-per-tenant with TypeORM or Prisma (pick one, don't switch later), using `AsyncLocalStorage` or NestJS request-scoped providers to thread tenant context without passing `tenantId` manually everywhere.
5. Auth + tenant resolution — how a request maps to a tenant (subdomain parsing, custom domain lookup, JWT with tenant claims).
6. AI integration contract — agree on the FastAPI microservice's API contract (REST or gRPC) early, so the 3 generalists and the 2 AI-only members aren't blocked on each other.

---

## 7. Open / Deferred Architecture Decisions

### 7.1 Raw code theme editing + AI code editing — **decision deferred, no option chosen yet**
Puck has no underlying "file" to show a merchant (it's JSON-config-driven, not file-based), so this can't just be a Puck tweak — it needs a separate system or a different editor entirely. Three options discussed, none chosen:
1. **Replace/supplement the theme layer with a Liquid-style file-based system**: LiquidJS (actively maintained, TypeScript-supported, Shopify-compatible Liquid template engine for Node.js) + an embedded Monaco code editor (`@monaco-editor/react`, the actual VS Code editor) + per-tenant theme files stored as text (Postgres table or Cloudflare R2 objects). Closest match to literally recreating Shopify's "Edit code" experience.
2. **Keep Puck as the default no-code path, add raw-code editing as a separate "Advanced" mode** — mirrors what Shopify itself actually does (visual sections editor by default, "Edit code" for power users). This was the recommended approach, since it doesn't throw away the Puck work already planned for MVP.
3. **Switch the entire theme editor to GrapesJS** instead of Puck, since it natively supports raw HTML/CSS viewing/editing. Not recommended as a full switch, since it would undo the reasoning that led to picking Puck.
- **AI editing of raw code files**: would follow a read/edit/write/validate pattern (same pattern Claude Code itself uses) — needs a lint/validation step before publishing, since raw code gives the AI room to break something in a way Puck's typed JSON fields don't allow.
- **Security note raised**: raw **CSS** injection is low-risk (worst case, a merchant's own store looks broken). Raw **JavaScript** injection is a real XSS risk in a multi-tenant setup where all storefronts render through the same Next.js app — would need sandboxing (iframe, CSP, script sanitization) designed deliberately before allowing it, not bolted on casually.

### 7.2 Trust score — needs finalizing later
- Exact percentage thresholds for medium vs. high risk, and exact prepayment percentages, still need real numbers (currently just "below 50%" as the flagged threshold, with illustrative "pay 20% now" example — not finalized).
- Whether phone number is stored plain or hashed in the shared trust-score table — for MVP scope, plain text with restricted admin-only access was suggested as acceptable, with hashing flagged as a "future: consider" item.

### 7.3 AI Smart Assistant ("Jarvis") — write-access boundary not yet defined
- This is architecturally an **agent with tool-calling access** to store management functions (theme edits, product CRUD, settings) — not just a Q&A chatbot like the customer-facing RAG bot.
- Needs a defined boundary: which actions the AI can take autonomously vs. which require merchant confirmation. Not decided — explicitly deprioritized ("forget about it for now") in the most recent discussion.
- Voice layer (STT/TTS) is a separate concern from the agent's reasoning/action layer — should be added only after the text-based agent works.

### 7.4 AI Buying Agent (item 30) — trigger model needs definition
Even after clarifying it's conversational-commerce-based (not fully autonomous), the exact trigger/authorization model isn't defined: is it (a) automated reordering of previously-bought items, (b) an agent acting on a standing customer instruction ("buy X when price drops"), or (c) something else. Each has different authorization/payment implications.

### 7.5 "Fully autonomous AI CRM" (item 15) — needs scoping
Flagged that "autonomous" needs to be defined concretely before it can be a real requirement — likely needs to mean "autonomous within merchant-approved guardrails," not literally unsupervised, for it to be buildable/defensible.

### 7.6 Explicitly deprioritized / set aside for later (from most recent Q&A round)
- Trust score exact thresholds — left for Grad 2.
- Whether to even have customer accounts at all — **resolved**: optional, per-store (see Section 5).
- AI smart assistant write-access boundary — set aside for now.
- Mobile app — merchant-facing only, Grad 2.

---

## 8. Documents Already Produced

Two markdown files have been created and iterated on throughout this conversation:

1. **`requirements.md`** — the full planning-phase requirements document: overview, actors, the 32-item feature list, functional requirements by module, tiered trust score logic, AI features section, non-functional requirements, architecture notes (microservices overview, trust score, Jarvis assistant, AI buying agent, raw code editing options), and the locked MVP scope. Team-split and "Open Questions"/"Next Steps" sections were later removed at the user's request to keep the doc focused.
2. **`requirements-claude-code.md`** — a separate, implementation-focused document written specifically to hand to an AI coding agent (Claude Code) to scaffold the MVP. Contains: fixed tech stack table, MVP-only microservice breakdown (Core Commerce, Integrations, Messaging, AI stub, Frontend), concrete data models (Tenant, Theme, Merchant User, Employee, Product, Order, integration config, messages), the 9-item MVP scope mapped to implementation, an explicit **"Out of Scope"** list (every AI feature, trust score, raw code editing, and all Future/Far-Future items) so the coding agent doesn't build ahead of scope, non-functional requirements, and a suggested monorepo folder structure.

---

## 9. Where the Planning Process Is Now

- Requirements-gathering meeting(s) have happened; the 32-item feature list and MVP scope are locked.
- **Next planned step (not yet done in this conversation)**: the diagramming phase — Use Case diagram (scoped to MVP first, Future items noted separately) → ERD → System architecture diagram → Sequence diagrams for key flows (checkout with trust-score branching, AI chatbot query, tenant provisioning).
- Architecture decisions still open: raw code editing approach (Section 7.1), exact trust-score thresholds (7.2), Jarvis assistant permission boundary (7.3), AI buying agent trigger model (7.4), MongoDB adoption (Section 2), and how aggressively to split microservices vs. starting more monolithic (Section 5).
