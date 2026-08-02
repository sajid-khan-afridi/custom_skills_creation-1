# Plan — Write the CoWink TRD

> Working document produced from a full walkthrough of `docs/prds/cowink-prd.md`.
> It records every decision made, the outline of the TRD to be written, the vendor
> evaluation checklist, and the items still open.
>
> **Date:** 2026-08-02 · **Status:** decisions captured, TRD not yet written

---

## Context

`docs/prds/cowink-prd.md` is a v1.0 PRD for **CoWink**, a two-sided creator ↔ advertiser
marketplace derived from a 240-frame Figma file. It deliberately defers every technology
choice to the TRD ("Tech stack: TBD").

The PRD was walked through in 15 plain-language sections, followed by four rounds of
decision questions. All outcomes are recorded below.

**Next deliverable:** `docs/trds/cowink-trd.md`.

**Repo state at time of writing:** `docs/prds/cowink-prd.md`, `docs/prompts/1-prd_trd.md`.
No source code yet.

---

## Writing style for the TRD (non-negotiable)

This was explicit, repeated feedback. Earlier drafts were rejected as too dense and too
technical. The TRD must:

- Explain the **why in everyday terms first**, then the technical decision
- Define every term on first use (escrow, ledger, saga, idempotency, PostGIS, RLS…)
- Use short sections, concrete analogies, and tables over paragraphs
- Be **convincing**, not just descriptive — say what breaks if the decision goes the
  other way
- Stay concise; no filler

---

## Decisions made (do not re-open without cause)

| Area | Decision |
| --- | --- |
| Project | Real product, **team of 1–5**, full PRD scope, ~14–18 months realistic |
| Launch payout countries | **Pakistan + Canada, both at launch** |
| Pricing currency | **USD only** in; local currency out; FX at payout |
| Pakistan payout route | **Payoneer** (PayPal and Stripe do not serve Pakistan) |
| Mobile | **Responsive from day one** |
| Architecture | **One application, nine internal modules** |
| Undesigned flows | **Specified in the TRD now**, marked design-pending |
| Language | **TypeScript everywhere** — Next.js + Node |
| Database | **Supabase** (managed PostgreSQL) |
| Auth | **Supabase Auth** (verify TOTP + SMS coverage for PK/CA) |
| Hosting | **Vercel** (web) + **Supabase** (data) + a small always-on worker (Railway/Render/Fly) |
| Real-time | **Supabase Realtime** |
| Escrow provider | **Not chosen** — run the scored vendor evaluation below |
| Work handover | **Dedicated "submit work" screen** with approve / request-changes |
| Revenue model | **10–15% of the creator's earnings**, deducted at release |

### Deviations from the PRD — write these up as deliberate changes

1. **Creator-side fees contradict the design.** `Pay to Post` (`1:16823`) charges the
   advertiser a platform fee. Under the chosen model the advertiser pays budget +
   card processing only; CoWink's cut comes out at release.
2. **PayPal-only payouts (AC-8.4) cannot serve Pakistan.** Payoneer replaces it there.
3. **Two currencies at launch** moves FX from the PRD's Phase 3 into Phase 1.
4. **W-9 / W-8BEN only make sense for a US entity.** Until registration is decided, the
   tax layer must be pluggable.

### Blocking open item

**O-1: Company registration country — UNDECIDED.**

Recommendation: **Canada** — payment providers will onboard you, it is already a launch
market so you learn one set of rules instead of two, it is cheaper than the US, and it is
credible to international brands. Pakistani creators are paid as foreign contractors via
Payoneer.

⚠️ Registering in Pakistan risks disqualifying you from the major payment providers,
which would leave escrow — the core of the product — with nowhere to run. Verify provider
availability before committing.

This needs an accountant and a lawyer, not a technical opinion. Until it is settled the
TRD keeps tax and compliance swappable.

---

## The TRD to write — `docs/trds/cowink-trd.md`

### 1. Overview and how to read this document
Purpose, relationship to the PRD, glossary of every term used.

### 2. Decisions register
The table above, each with a one-paragraph plain-language rationale and what breaks if
reversed. Plus the four deviations and the one blocking open item.

### 3. Architecture
One Next.js/Node application with nine enforced internal module boundaries (Identity,
Profile & Channels, Discovery, Jobs & Proposals, Contracts, Payments & Escrow, Tax &
Compliance, Messaging, Notifications). Vercel + Supabase + one worker process.

**The Supabase condition — state prominently:** use Supabase for database, auth, storage
and realtime, but **never** let the browser talk directly to the database for money,
contracts or permissions. Reads and browsing may go direct; money goes through your own
server. Reason: "contract + escrow lock, both or neither", an append-only ledger, and
per-account permissions cannot be safely expressed as row-level database policies.

Verify at write time: PostGIS availability, Supabase Auth TOTP, SMS coverage for PK/CA.

### 4. Data model
Core tables with the reasoning:
- `User` ≠ `Account` ≠ `Organization`, joined by `Membership` (role + account scope +
  validity window)
- Session carries `{user_id, active_account_id, active_role}`; every query scoped by it
- Money as `{amount_minor: bigint, currency: char(3)}` from migration #1 — no floats
- One `deal_request` with a `source` discriminator covering interest / proposal /
  direct offer
- Network requirement as tri-state `threshold(n) | unlimited | excluded`
- Append-only `contract_event` stream as the source of truth for timeline, activity log,
  notifications and PDFs
- IANA timezone names, never UTC offsets
- Contract ID format `CW-YYYYMMDD-XXXXX` (replaces the URL-hostile `20251202-Q4&S`)

### 5. Identity, permissions and onboarding
Supabase Auth setup; the two separate OAuth systems (social **login** vs social
**channel data access**) with encrypted refresh tokens and a re-consent flow;
permissions stored as data not code; server-side deny-by-default; verification state as
an authorization input; distributed rate limiting; server-side wizard state with
config-driven step lists.

### 6. The money architecture
The five invariants as automated tests, not prose. Append-only double-entry ledger.
Pricing Service as the single authority for fees and tax. Money enters at posting, locks
at acceptance. The atomic accept as a **saga with idempotency keys plus a reconciliation
sweep** (your database and the provider are separate systems — no single transaction
spans them). Daily reconciliation with drift alerting. Every payout check fails closed.
Creator-side fee mechanics at release.

### 7. Vendor evaluation and integrations
The escrow checklist (below) as a scored, dated task. Tax vendor, KYC/AML vendor,
Payoneer, four social APIs, maps/geocoding, email, SMS, storage. Every provider behind
a wrapper — never called from business logic.

### 8. Discovery and the matching engine
Postgres + PostGIS. **Eligibility as one shared, reason-returning pure function**
(`{eligible, failures:[{rule, required, actual, remediation}]}`) used by both search
filtering and the apply gate. No randomness, no internal clock in scoring. Weights in a
settings store, version-stamped per search. New-account boost as a setting (cold-start
risk R-5). Denormalized search profiles. Estimated counts + stable tiebreakers.
Canonical URL query-param schema.

### 9. Contracts, delivery and disputes
Full milestone state machine including the undesigned states:
`draft → funded → visible → submitted → approved → released`, plus
`revision_requested / disputed / cancelled / refunded`. The dedicated submit-work screen.
Dispute intake, evidence, resolution, partial release. Authorize **actions, not records**
(`allowed_actions` list). Archived server-generated PDFs.

### 10. Messaging, notifications and reviews
Supabase Realtime. **System messages stored as structured data, never rendered
sentences.** Unread counts. Contract-gated reviews with the 30-day window.

### 11. Tax and compliance (pluggable pending O-1)
Entity type × tax residence drives the form; the user never chooses. Withdrawal hard-
blocked with the specific missing form named. Signature records name + timestamp + IP.
Tax data in separate, stricter storage — own key, masked, excluded from logs, analytics,
error reports and non-production copies.

### 12. Cross-cutting
Security (S-1..S-10), one shared audit log, accessibility with the results list as the
map's equal, strings externalised from day one, shared loading/empty/error components,
validation defined once server-side, the analytics event taxonomy behind §1.3's KPIs,
and the job scheduler (channel refresh, access expiry, review windows, tax renewals,
reconciliation).

**Channel refresh is infrastructure, not a cron:** ~200k calls/day at 100k users ⇒ queue,
per-provider quota budgeting, backoff, staleness prioritization, and degrade-to-stale
rather than fail.

### 13. Build sequence
| Window | Focus |
| --- | --- |
| **Start immediately, parallel** | Company registration · payment provider evaluation and onboarding · social API approvals · tax vendor selection (calendar-bound; more developers don't help) |
| Months 1–3 | Identity, accounts, organizations, permissions, onboarding, channel verification, profiles |
| Months 3–5 | Discovery, map, filters, job posting (both deal types), proposals, offers, hiring |
| Months 5–9 | Money end to end — escrow, milestones, delivery, approval, release, payout, tax gating |
| Months 9–12 | Disputes, messaging, reviews, notifications |
| Months 12–16 | Organizations and billing, both countries live, languages, accessibility, polish |

Money gets a thin end-to-end path early on purpose — vendor surprises should surface in
month 5, not month 14.

### 14. Open questions and risks
O-1 registration (blocking), the PRD's Q-1..Q-10, and R-1..R-11 restated with the
technical mitigation now chosen for each.

---

## Escrow vendor evaluation checklist (goes into §7)

Any "no" in the must-have block eliminates the candidate. Test 2, 3 and 4 first — most
providers fail there.

**Must have**
1. Hold funds on behalf of two parties until instructed to release
2. **Release part** of a held amount, not all-or-nothing
3. **Freeze** a release while a dispute is open
4. **Refund from held funds** back to the payer
5. Pay out to **Canada**
6. Pay out to **Pakistan**, or work cleanly alongside Payoneer
7. **USD in, local currency out**, at a rate that can be shown to the user
8. Will onboard a marketplace registered in the chosen country
9. States clearly whether **you** or **they** are legally the fund holder

**Should have**
10. Hosted card fields (card numbers never touch your servers)
11. Same request twice ⇒ charged once
12. Reports reconcilable against your own ledger, daily
13. Saved payment methods with a designated primary
14. PayPal accepted as a pay-in method
15. Test mode that simulates disputes and refunds

**Nice to have**
16. Tax form collection built in
17. Identity / sanctions screening built in
18. Webhooks on every money event

---

## Design guidance already settled (carry into the TRD)

### Identity and access
- **Three entities:** `User` (login) ≠ `Account` (brand owning jobs and money) ≠
  `Organization` (legal entity), joined by `Membership`. Time-bounded access (AC-12.4)
  requires a job scheduler.
- **Role is session state, not a column.** Switching account re-scopes everything.
- **Permissions stored as data** — AC-12.2's matrix is `[INFERRED]` and unsigned.
- **Two distinct OAuth systems** — social login vs social channel data access. Separate
  subsystems, scopes and token storage; re-consent flow for silent revocation.
- Verification state is an authorization input (AC-1.7).
- Distributed rate limiting from day one (AC-1.3, AC-11.6, S-9).
- Store IANA timezone names, never UTC offsets.

### Money
- Money type from migration #1; no floating point anywhere.
- Append-only double-entry ledger; balances computed, never stored-and-edited.
- Daily reconciliation against the provider, with drift alerting.
- Idempotency keys on every money-moving endpoint.
- A Pricing Service is mandatory — kills R-7 and the three contradictory fee models.
- Buy a tax engine; tax is jurisdiction × entity × date, not a flat 3%.
- Money enters at posting, locks at acceptance — two distinct ledger events.
- Every payout check fails closed.

### Contracts and deals
- Three doors, one destination ⇒ one `deal_request` with a `source` discriminator.
- Slot counting is a race condition ⇒ row lock or optimistic concurrency.
- Define the milestone state machine now, despite missing screens.
- One append-only `contract_event` stream; timeline, activity log, notifications and PDF
  are projections of it.
- PDFs are legal artifacts — generate server-side and archive the file.
- Authorize actions, not records.
- Published job posts immutable in budget/deal type/requirements; snapshot criteria.

### Discovery and matching
- Eligibility as one shared, reason-returning pure function.
- No randomness, no internal clock in scoring — reproducible by construction.
- Ranking weights in a settings store, version-stamped per search; new-account boost as
  a setting (R-5 cold start — 30% of the score rewards past work).
- Reach fit is job-dependent ⇒ cannot be fully precomputed.
- Denormalized search profiles; eventual consistency is fine.
- Network requirement is tri-state — never a nullable integer.
- Numbered pagination forces a count ⇒ estimated/capped counts + stable tiebreakers.

### Platform and operations
- Wrap all external providers; never call them from business logic.
- Turn INV-1..INV-5 into automated tests, not documentation.
- Channel refresh is infrastructure, not a cron.
- A job scheduler is required — channel refresh, access expiry, review windows, tax
  renewals, reconciliation.
- Server-side wizard state; step lists are configuration so denominators are derived.
- Reputation aggregates precomputed; lifetime stats ledger-derived, not counters.
- Sub-score suppression until ≥3 completed contracts — one serializer, read-time.
- Analytics event pipeline is a launch requirement (§1.3's 8 KPIs).
- Shared loading/empty/error components as a design-system deliverable (R-10).
- Validation defined once server-side; client renders returned errors.
- System messages stored as structured data, never rendered sentences.
- One shared audit log for money, permissions and contract transitions.
- Results list must be the accessible equal of the map, not a fallback.
- Externalise all strings from day one.
- Tax data in separate, stricter storage.

---

## Definition of done for the TRD

1. Every decision in the register appears with a plain-language rationale.
2. Every `[GAP]`, `[DEFECT]` and `[INFERRED]` marker in the PRD is either resolved,
   assigned an owner, or explicitly listed as still open.
3. All ten of the PRD's Appendix B questions are answered or carried forward.
4. All five money invariants (INV-1..INV-5) map to a named automated test.
5. The four deviations from the PRD are stated as deliberate changes, not omissions.
6. A reader new to the domain can follow it without looking anything up — check by
   reading §3 and §6 aloud.
