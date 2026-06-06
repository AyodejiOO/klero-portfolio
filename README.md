# Klero — modular operations platform for Nigerian logistics

> Replacing exercise books, Excel, and WhatsApp groups across logistics
> businesses, one paying-customer-justified module at a time.

Klero started as a Terminal Operating System (TOS) for Inland Container
Depots and grew into a multi-product platform: a web app for terminal
operators, a mobile app for clearing agents, and a mobile app for fuel
station attendants. Built end-to-end — schema, RLS, Edge Functions, web
UI, two Expo apps, payments, SMS, AI document checks, the lot.

This repo is a **portfolio overview**. Source lives in a private repo —
available on request.

---

## What's in the platform

### 🏗️ Terminal OS (web)
The system of record for an Inland Container Depot. Replaces the
collection of paper logs and chat groups a typical Nigerian terminal
runs today.

- **Import** — gate-in / gate-out, yard moves, shift handover, bond
  transfers, container document tracking
- **Export** — gate ops, yard, vessel and container management
- **Billing** — invoices, payments, shift reconciliation
- **Empties** — survey, repair, depot tally
- **Reports** — owner dashboard with anti-leakage views
- **Clearing-agent portal** — agents see their containers, charges,
  delivery bookings, proforma PDFs from a single dashboard
- **Public importer tracking** — `klero.app/t/<token>` deep-link page
  importers receive via WhatsApp

> _Screenshot: Terminal OS dashboard_
> _Screenshot: Anti-leakage report_

### 📱 Klero Agent (iOS + Android, Expo)
Native mobile companion for clearing agents — typically managing 5–50
containers across multiple terminals at any time.

- Cross-tenant dashboard: containers, charges, and urgency in one
  list regardless of which terminal they're at
- Register a new container: photograph Bill of Lading + Customs
  Release, AI checks image quality, terminal staff approves
- Delivery booking, proforma invoice request, full status timeline
- PIN + Face ID / Fingerprint, offline-aware
- WhatsApp + push notifications on every status change

> _Screenshot: Agent dashboard_
> _Screenshot: Container registration document upload_

### ⛽ Klero Station (iOS + Android, Expo)
Built for fuel-station attendants in the freight-ops module. Replaces
the paper sheet that gets falsified.

- Driver presents OTP-secured fuel allocation; attendant verifies, dispenses
- Pump-meter photo before and after; AI OCR pre-fills the
  litres/amount on the confirm screen (attendant verifies and submits)
- First-login geo-capture pins the station to its real coordinates,
  flags fraud (attendant logged in 40 km from the registered station)
- Offline-resilient: transactions queue if the station's connectivity
  drops, sync when it's back

> _Screenshot: New transaction screen_
> _Screenshot: Dispensing animation + confirm_

---

## Architecture highlights

```
            Internet
                │
        ┌───────┼───────────────────────┐
        ▼       ▼                       ▼
   Next.js 14    Expo SDK 55       Expo SDK 55
   (Vercel)       Agent app        Station app
        │             │                  │
        └──────┬──────┴──────────────────┘
               ▼
        Supabase Edge Functions (Deno)
        + PostgREST + RLS
               │
               ▼
        Postgres (Supabase, Cape Town region)
```

**Monorepo:** pnpm 9 + Turborepo. Web in `apps/web`, mobile in
`apps/mobile` (agent) and `apps/station`. Shared types, formatters,
and constants in `packages/shared` — including `formatNaira()`,
`formatDate()` for Nigerian DD/MM/YYYY, demurrage rules, VAT
calculations.

**Backend pattern:** all business logic in Supabase Edge Functions
(Deno). No separate API server. Service-role DB client only inside
Edge Functions; never in the browser or mobile binary.

**Auth:** dual-mode by user class.
- Terminal managers use Supabase Auth.
- Ground staff (gate clerks, yard checkers, cashiers) and clearing
  agents and station attendants use a PIN-mint-your-own-JWT scheme —
  signed with the same `SUPABASE_JWT_SECRET` so PostgREST and RLS
  trust them natively.
- Each token class carries a `pin_version` claim that's bumped on
  PIN reset, role change, or deactivation, with revocation propagating
  to every Edge Function worker within ~30 seconds.

**RLS:** every tenant-scoped table has policies on both `USING` and
`WITH CHECK`, gated on `facility_id` from `auth.jwt() ->
'app_metadata' ->> 'facility_id'`. Agents are platform-level
(cross-facility by design) and gate on a `current_agent_id_if_active()`
SECURITY DEFINER helper that also enforces subscription state —
revoking access on the next query, no JWT rotation required.

**Financial integrity:** `overrides` and `container_moves` are
append-only (INSERT-only triggers); state transitions are
status-guarded RPCs (`UPDATE … WHERE status = 'X'`, assert affected
rows = 1); idempotency keys on every webhook handler; constant-time
HMAC compare on Paystack callbacks.

**Nigerian-specific:** Naira formatting, calendar-day demurrage
(day 0 = gate-in, no charge), 7.5% VAT, first_name / last_name
(never full_name), Africa/Lagos timezone, Termii for SMS, Paystack
for payments, WhatsApp as the primary user notification channel.

---

## Process — built with Claude Code agents

The bulk of the code in this platform was written by Claude (Sonnet /
Opus) under explicit human direction — spec, plan, build, test, audit,
fix. The workflow:

1. **Spec each module** as a design doc before any code. Threat model
   in the same doc.
2. **Plan-first** — weekly plans mapping features to user-justified
   outcomes, broken down to per-task acceptance criteria.
3. **Build with subagent fan-out** — parallel agents for unrelated
   features, single agent for tightly coupled work.
4. **Test the abuse paths**, not just the happy path. Every feature
   gets at least one "what if a malicious tenant did X" test.
5. **Adversarial security audit** every meaningful release, against a
   written security standard. Findings remediated one commit at a time.
6. **Commit per finding / per feature.** Atomicity makes revert cheap.

Working effectively with frontier-model agents is itself a skill —
knowing when to fan-out vs serialise, when to verify vs trust, how to
write a prompt that gets a useful result on the first try, when the
agent is wrong and needs to be reset. This is most of what I'd bring
to an Implementation Engineer role.

---

## Tech stack

| Layer | Choice |
|---|---|
| Frontend | Next.js 14 (App Router), Tailwind CSS, custom design tokens |
| Mobile | Expo SDK 55, expo-router, expo-secure-store, expo-local-authentication |
| Backend | Supabase Edge Functions (Deno 2) |
| Database | PostgreSQL on Supabase (Africa / Cape Town region) |
| Auth | Supabase Auth (managers) + custom PIN-mint JWT (ground / agent / station) |
| AI | Anthropic Claude API for document quality and pump-meter OCR |
| Payments | Paystack (Nigerian) |
| SMS / OTP | Termii (Nigerian) |
| Messaging | WhatsApp Business API |
| Hosting | Vercel (web), EAS (mobile), Supabase (data + functions) |
| Dev | TypeScript, pnpm 9, Turborepo, Vitest, GitHub Actions |

---

## Outcomes

<!--
  Replace these with real figures once Klero is comfortable sharing.
  Specifics beat adjectives — name a terminal, a month-live, a volume.
-->

- Active pilots running at Nigerian terminal operators
- Multiple paying customers — built only after the previous feature
  proved out, never on speculation
- Replaces an average of ~4 separate tools per customer (paper log,
  Excel, WhatsApp group, ad-hoc PDFs over email)
- Clearing-agent app cuts container clearance time by reducing the
  back-and-forth with the terminal office

---

## Source access

The source repo is private to protect customer-specific schema and
Nigerian domain detail that competitors could lift directly.

Happy to:
- Walk through the code on a screen-share
- Add reviewer access to specific modules
- Discuss any architectural decision in detail

📩 **ayodejiogunniyi@gmail.com**

---

_Built by Ayodeji Ogunniyi, 2025–2026._
