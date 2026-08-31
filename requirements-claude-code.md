# MVP Build Requirements — Multi-Tenant E-Commerce SaaS Platform

**Audience:** This document is written for an AI coding agent (Claude Code) to scaffold and build the MVP. It is scoped tightly to the agreed MVP — do not implement anything listed under "Out of Scope" unless explicitly asked.

---

## 1. Project Summary

A multi-tenant, Shopify-style e-commerce platform for the Egyptian/MENA market. Each merchant ("tenant") gets their own store with its own theme, products, orders, and staff. The MVP is the software foundation for a graduation project (Grad 1) and a real startup — build it clean and extensible, but only implement what's listed in Section 5 (MVP Scope).

---

## 2. Tech Stack (fixed — do not substitute)

| Layer | Technology |
|---|---|
| Backend services | NestJS (TypeScript) |
| Frontend (merchant dashboard + storefront) | Next.js (App Router) |
| AI microservice | FastAPI (Python) — stub only for MVP, not built out yet |
| Database | PostgreSQL — **schema-per-tenant** multi-tenancy |
| Cache | Redis |
| Theme/storefront editor | Puck (React-based visual editor) |
| Architecture style | Microservices |

**MongoDB**: not decided yet — do not introduce it. Use PostgreSQL for all MVP data, including flexible/JSON fields (`jsonb` columns), e.g. theme configuration.

---

## 3. Architecture — Microservices Breakdown for MVP

Only these services are needed for the MVP. Each is an independent NestJS app (own repo or own package in a monorepo — pick a monorepo with clear service boundaries unless told otherwise) with its own database schema.

| Service | Responsibility | DB access |
|---|---|---|
| **Core Commerce Service** | Tenants/stores, products, orders, theme assignment, employees/roles | Owns tenant schemas (schema-per-tenant) |
| **Integrations Service** | Payment gateway adapter (1 for MVP), shipping provider adapter (1 for MVP) | Own schema — stores per-tenant integration config/credentials |
| **Messaging Service** | Basic unified message box (receives/displays messages from one channel for MVP) | Own schema — stores inbound messages |
| **Frontend** | Next.js app serving both the merchant dashboard and the public storefront | Calls the above services via REST |

The AI microservice (FastAPI) should exist as a skeleton/stub service with health-check endpoint only — no AI logic yet.

**Tenant isolation rule:** every query in the Core Commerce Service must be scoped to the current tenant's schema. Implement tenant resolution via request-scoped provider (NestJS DI), resolving tenant from subdomain or JWT claim — never trust a client-supplied tenant ID without validating it against the authenticated session.

---

## 4. Data Models (MVP only)

### 4.1 Core Commerce Service

**Tenant / Store** (platform-level table, not inside a tenant schema)
- `id`, `name`, `subdomain`, `custom_domain` (nullable), `theme_id`, `created_at`

**Theme**
- `id`, `name`, `config` (jsonb — Puck-compatible section/block config)
- Seed exactly 2–3 predefined themes for MVP

**Merchant User** (store owner)
- `id`, `tenant_id`, `email`, `password_hash`, `role` = `owner`

**Employee**
- `id`, `tenant_id`, `email`, `password_hash`, `role`, `permissions` (jsonb — which integrations/sections this employee can access)

**Product**
- `id`, `tenant_id`, `name`, `description`, `price`, `stock_quantity`, `images` (array of URLs), `created_at`

**Order**
- `id`, `tenant_id`, `customer_name`, `customer_phone`, `items` (jsonb array of `{product_id, qty, price}`), `status` (`pending` | `confirmed` | `shipped` | `delivered` | `cancelled`), `payment_gateway_used`, `shipping_provider_used`, `created_at`

### 4.2 Integrations Service
- `tenant_integration_config`: `id`, `tenant_id`, `integration_type` (`payment` | `shipping`), `provider_name`, `credentials` (encrypted), `enabled` (bool)

### 4.3 Messaging Service
- `message`: `id`, `tenant_id`, `channel`, `sender`, `content`, `received_at`, `read` (bool)

---

## 5. MVP Scope (build these, and only these)

1. **Theme editor** — Puck-based drag-and-drop editor; merchant can select and customize one of 2–3 pre-built themes; save/publish flow
2. **Create store** — signup/onboarding flow that provisions a new tenant + subdomain
3. **Admin dashboard (merchant-facing)** — merchant logs in, sees their store: products, orders, basic stats
4. **Basic admin dashboard (platform-facing)** — internal/platform admin view listing tenants
5. **Employee system** — merchant can invite employees, assign roles, and limit which integrations each employee can access
6. **2–3 ready themes** — seeded, selectable, editable via the theme editor
7. **1 payment gateway integration** — pick one Egyptian provider (e.g. Paymob) and implement a working integration end-to-end
8. **1 delivery/shipping integration** — pick one shipping provider (e.g. Bosta) and implement a working integration end-to-end
9. **Message box** — a basic unified inbox showing incoming messages from one channel (pick one, e.g. WhatsApp Business API webhook) — display only, no AI response generation yet

**Basic product management (CRUD) and basic checkout/order flow are implicit prerequisites** for the above to function end-to-end (a store with no way to add products or place an order isn't demonstrable) — include them as part of the MVP build even though not separately itemized.

---

## 6. Out of Scope — do NOT build these for MVP

- Any AI feature: RAG chatbot, AI CRM, "Jarvis" smart assistant, speech-to-text, AI product descriptions, fake review detection, AI media buying, AI buying agent, AI recommendations
- Trust score system (tiered risk / prepayment logic)
- Multiple payment gateways or shipping providers beyond the one each
- Meta Ads / Meta unified inbox beyond the single-channel message box
- Visual/image-based product search
- Clothing preview / virtual try-on
- Return and exchange authorization
- Instapay confirmation
- Bulk product upload (Excel)
- Low-stock alerts
- Drop timer feature
- Mobile application
- Legal document guidance for brands
- Cost/sales/profit calculator
- Video streaming, supplier/factory marketplace, unified brand marketplace (all far-future)
- Integrations marketplace / app store UI (MVP has exactly one payment + one shipping integration, hardcoded, no picker UI needed)
- Raw code/file editing for themes (Shopify-style "Edit code", custom code file uploads, AI editing raw theme files) — architecture not yet decided (candidates: LiquidJS + Monaco, GrapesJS, or a Puck "advanced mode"). MVP theme editing is Puck-only, structured fields, no raw code surface.

If asked to implement any of the above, flag that it's outside MVP scope per this document rather than building it silently.

---

## 7. Non-Functional Requirements (MVP-relevant)

- Multi-tenant data isolation must be enforced at the query layer, not just the application layer — no cross-tenant data leakage under any circumstance
- Arabic + English support in the storefront UI, including RTL layout
- Passwords hashed (bcrypt/argon2), never stored plain
- Payment/shipping provider credentials stored encrypted at rest

---

## 8. Suggested Repo Structure

```
/apps
  /core-commerce-service   (NestJS)
  /integrations-service    (NestJS)
  /messaging-service       (NestJS)
  /ai-service              (FastAPI, stub only)
  /web                     (Next.js — dashboard + storefront)
/packages
  /shared-types            (shared TS interfaces/DTOs across services)
```

Use a monorepo (Turborepo or Nx) unless there's a reason to split into separate repos later.
