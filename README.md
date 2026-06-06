# Klero — modular operations platform for Nigerian logistics

> Replacing exercise books, Excel, and WhatsApp groups across logistics
> businesses, one paying-customer-justified module at a time.

Klero started as a Terminal Operating System (TOS) for Inland Container
Depots and grew into a multi-product platform spanning terminal
operations, clearing-agent self-service, freight fleet management, and
operational finance. It is built end-to-end: PostgreSQL schema with
row-level security, ~76 Deno Edge Functions, a Next.js web app, two
React Native (Expo) mobile apps, payments, SMS/WhatsApp messaging, and
AI for document and image understanding.

This repo is a **portfolio overview**. The source is private — available
on request.

_Built by **Ayodeji Ogunniyi** with a collaborator, 2025–2026._

**Scale at a glance:** 4 client surfaces (web + 2 mobile + public
tracking) · ~76 Edge Functions · 94 database migrations · 5 distinct
authenticated user classes · Termii (SMS), Paystack (payments),
WhatsApp Business, Anthropic Claude, and DCSA carrier integrations.

---

## Table of contents

1. [Terminal OS (web)](#1-terminal-os-web)
2. [Freight Operations (web)](#2-freight-operations-web)
3. [Clearing-Agent portal & mobile app](#3-clearing-agent-portal--mobile-app)
4. [Station app (mobile)](#4-station-app-mobile)
5. [Public importer tracking](#5-public-importer-tracking)
6. [AI features](#6-ai-features)
7. [Architecture](#7-architecture)
8. [Security model](#8-security-model)
9. [Tech stack](#9-tech-stack)
10. [How it was built](#10-how-it-was-built)
11. [Outcomes](#11-outcomes)

---

## 1. Terminal OS (web)

The system of record for an Inland Container Depot — replacing the paper
logs, spreadsheets, and chat groups a typical Nigerian terminal runs on.

**Import operations**
- Gate-in (single + bulk), gate-out, and bonded gate-out / bond transfer
- Yard management and container repositioning
- Per-container document tracking (Bill of Lading, Delivery Order,
  Customs/NCS release, PAAR, duty receipts) with quality checks
- **Terminal Delivery Order (TDO)** generation as PDF
- **Release-condition engine** — a container can only gate out when every
  gate is satisfied (charges paid, customs released, documents valid, no
  holds); the engine evaluates and explains exactly what's outstanding
- **NCS examination** recording and **OGA (Other Government Agency)
  overrides** with a full audit trail
- **Force-majeure** handling that suspends demurrage accrual for declared
  events

**Export operations**
- Export gate-in / pre-staging / gate-out
- Vessel management with cut-off alerts
- Export container lifecycle and yard-by-vessel views
- Export document validation and "ready for shipment" gating

**Empties**
- Empty-container survey and damage assessment
- Repair tracking and depot tally

**Billing & cash**
- Automated **billing engine**: demurrage (calendar-day, day-0-free),
  storage, handling, VAT at 7.5%, customer-specific rates and tariffs
- Invoice generation, proforma invoices, payment recording, receipts
- **Cash reconciliation** and shift-level cash management
- Paystack online payment + manual payment capture

**Reports & oversight**
- Owner dashboard with operational KPIs
- **Anti-leakage** views — designed to surface the revenue that quietly
  disappears in a manual terminal (un-billed moves, overrides, waivers,
  cash gaps)
- Shipping-line reporting
- AI **daily briefing** summarising the day's exceptions

**Administration**
- Staff management with role-based access (Owner, Ops Manager, Billing
  Officer, Gate Clerk, Yard Checker, Cashier)
- Facility self-signup with CAC document upload
- Clearing-agent registration review queue

> _Screenshots available on request._

---

## 2. Freight Operations (web)

A freight-company control plane for road haulage between terminals,
ports, and inland destinations — the operational-finance layer that sits
on top of moving trucks.

- **Trip lifecycle** — create, AI-estimate fuel, approve budget, dispatch,
  track in-transit, complete, reconcile
- **AI fuel estimation** — distance, cargo weight, truck efficiency, and
  route feed a model that proposes a fuel budget; a human approves
- **Fleet management** — trucks (class, tank capacity, efficiency) and
  drivers
- **Expenses / imprest** — drivers/dispatchers submit expenses with
  receipts; managers approve; advances are requested and reconciled
- **Company wallets** — each company gets a **dedicated virtual account**
  (Paystack DVA) for funding; deposits are verified and reconciled against
  the wallet ledger
- **Settlements** — automated settlement processing across depots
- **Depot allocation** — move funds to depot sub-balances
- **Anomaly scanning** — flags suspicious fuel/expense patterns

> _Screenshots available on request._

---

## 3. Clearing-Agent portal & mobile app

Clearing agents are platform-level actors who manage containers across
**multiple terminals at once** — so they get both a web portal and a
native mobile app.

**Mobile app (iOS + Android, Expo)**
- Cross-terminal dashboard: every container, its charges, and urgency in
  one list regardless of which terminal it sits at
- **Register a container**: photograph Bill of Lading + Customs Release,
  AI checks image quality, the terminal reviews and approves the claim
- Delivery-appointment booking with truck + driver details
- Proforma-invoice request and PDF retrieval
- Full per-container status timeline
- PIN login + Face ID / Fingerprint, offline-aware, push notifications

**Web portal**
- Dashboard, delivery booking, proforma generation for desktop workflows

> _Screenshots available on request._

---

## 4. Station app (mobile)

Built for fuel-station attendants inside the freight module — replacing
the paper sheet that gets falsified.

- Driver presents an **OTP-secured fuel allocation**; the attendant
  verifies before dispensing
- **Pump-meter photo** before and after; AI OCR pre-fills litres and
  amount on the confirm screen (attendant verifies, then submits)
- **First-login geo-capture** pins the station to its real coordinates and
  flags fraud (e.g. an attendant logging in 40 km from the registered
  station)
- **Offline-resilient** — transactions queue if connectivity drops and
  sync when it returns
- PIN login + biometrics

> _Screenshots available on request._

---

## 5. Public importer tracking

- `klero.app/t/<token>` — a tokenised, login-free page importers receive
  over WhatsApp to track their container's status, charges, and what's
  still blocking release
- No account required; the token scopes access to exactly one container

---

## 6. AI features

All AI runs server-side through Edge Functions; the Anthropic Claude API
is the primary model.

- **Document quality validation** — checks clarity, lighting, completeness
  of uploaded clearing documents before a human ever sees them
- **Pump-meter OCR** — reads litres/amount off a fuel-pump photo
- **Damage assessment** — assists empty-container survey
- **General OCR** — extracts structured fields from logistics documents
- **Anomaly detection** — operational and freight-finance anomaly scans
- **Daily briefing** — natural-language summary of the day's exceptions
  for the terminal owner

---

## 7. Architecture

```
                          Internet
                              │
     ┌──────────────┬─────────┼──────────────┬──────────────┐
     ▼              ▼         ▼              ▼              ▼
  Next.js 14     Expo SDK 55  Expo SDK 55   Public         Webhooks
  (Vercel)       Agent app    Station app   tracking page  (Paystack,
     │              │            │              │           Termii)
     └──────┬───────┴────────────┴──────────────┴───────────┘
            ▼
   Same-origin API proxy (/api/edge, /api/rest)  ← web only
            ▼
   Supabase Edge Functions (Deno) ── ~76 functions, all business logic
            ▼
   PostgREST + Row-Level Security
            ▼
   PostgreSQL (Supabase, Africa / Cape Town region) ── 94 migrations
            │
            └── pg_cron (partition roll, scheduled jobs)
```

- **Monorepo** — pnpm 9 + Turborepo. `apps/web`, `apps/mobile` (agent),
  `apps/station`; shared types, validators, formatters, and domain
  constants in `packages/shared` (Naira formatting, DD/MM/YYYY dates,
  demurrage rules, charge codes, container-state machine, zones).
- **No separate API server** — all logic lives in Edge Functions. The
  browser never holds a service-role key; the web app talks to the
  backend through a same-origin proxy that attaches auth server-side.
- **Carrier integration** — DCSA-shaped vessel schedule sync (Hapag-Lloyd)
  cleanly abstracted behind a provider interface, with vessel cut-off
  alerting.
- **Messaging** — WhatsApp Business for inbound + outbound (the primary
  channel for Nigerian users), Termii for SMS/OTP, Expo Push for mobile.

---

## 8. Security model

Security is treated as a building standard, audited adversarially before
each release.

- **Five authenticated user classes**, each a distinct trust level:
  terminal managers (Supabase Auth), terminal ground staff, clearing
  agents, freight staff, and station attendants (the latter four use a
  PIN-mint-your-own-JWT scheme signed with the project JWT secret so
  PostgREST and RLS trust them natively).
- **RLS on every tenant-scoped table**, with predicates in both `USING`
  and `WITH CHECK`, keyed on `facility_id` from the JWT. Agents are
  cross-facility by design and gate on a `SECURITY DEFINER` helper that
  also enforces live subscription state — access revokes on the next
  query, no token rotation required.
- **Token revocation** — every PIN-class JWT carries a `pin_version`
  claim, bumped on PIN reset / role change / deactivation; stale tokens
  are rejected within seconds.
- **Financial integrity** — append-only ledgers (`overrides`,
  `container_moves`) enforced by triggers; status-guarded state
  transitions; idempotency keys on every webhook; constant-time HMAC
  verification on Paystack callbacks; webhook amounts reconciled against
  the DB, never trusted from the payload.
- **Inputs** — validated at every function boundary (regex, length caps,
  pre-decode size caps on base64, MIME sniffing from magic bytes);
  private storage buckets with tenant-prefixed paths and short-lived
  signed URLs for sensitive documents.

---

## 9. Tech stack

| Layer | Choice |
|---|---|
| Frontend | Next.js 14 (App Router), Tailwind CSS, custom design tokens |
| Mobile | Expo SDK 55, expo-router, expo-secure-store, expo-local-authentication |
| Backend | Supabase Edge Functions (Deno 2) — ~76 functions |
| Database | PostgreSQL on Supabase (Africa / Cape Town), 94 migrations, pg_cron |
| Auth | Supabase Auth (managers) + custom PIN-mint JWT (ground / agent / freight / station) |
| AI | Anthropic Claude API — document quality, OCR, damage, anomaly, briefings |
| Payments | Paystack (online payments + dedicated virtual accounts) |
| SMS / OTP | Termii (Nigerian provider) |
| Messaging | WhatsApp Business API (inbound + outbound) |
| Carrier data | DCSA-shaped schedule sync (Hapag-Lloyd) |
| Hosting | Vercel (web), EAS (mobile), Supabase (data + functions) |
| Tooling | TypeScript, pnpm 9, Turborepo, Vitest, GitHub Actions |

---

## 10. How it was built

The bulk of this platform was written by Claude (Sonnet / Opus) agents
under explicit human direction — spec, plan, build, test, audit, fix:

1. **Spec each module** as a design doc, with a threat model, before code.
2. **Plan-first** — features mapped to paying-customer-justified outcomes,
   broken down to per-task acceptance criteria.
3. **Subagent fan-out** — parallel agents for independent features, a
   single agent for tightly coupled work.
4. **Test the abuse paths**, not just the happy path — every feature gets
   a "what if a malicious tenant did X" test.
5. **Adversarial security audit** each release against a written standard;
   findings remediated one commit at a time.
6. **Atomic commits** — one per feature / per finding, so revert is cheap.

Directing frontier-model agents effectively — knowing when to fan out vs
serialise, when to verify vs trust, how to phrase a task for a first-try
result, and when an agent is wrong and needs resetting — is the core
skill this platform demonstrates.

---

## 11. Outcomes

<!--
  Replace with real figures once Klero is comfortable sharing.
  Specifics beat adjectives — name a terminal, a month live, a volume.
-->

- Built and piloted with Nigerian terminal operators
- Multiple paying customers — each module shipped only after the prior one
  proved out, never on speculation
- Replaces ~4 separate tools per customer (paper log, Excel, WhatsApp
  group, ad-hoc PDFs over email)
- The clearing-agent and freight modules cut the back-and-forth that slows
  container clearance and fuel disbursement

---

## Source access

The source repo is private to protect customer-specific schema and
Nigerian domain detail competitors could lift directly. Happy to:

- Walk through the code on a screen-share
- Add reviewer access to specific modules
- Discuss any architectural decision in detail

📩 **ayodejiogunniyi@gmail.com**

---

_Built by Ayodeji Ogunniyi with a collaborator, 2025–2026._
