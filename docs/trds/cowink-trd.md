# Technical Requirements Document — CoWink

| Field | Value |
| --- | --- |
| **Product** | CoWink — Creator ↔ Advertiser Marketplace |
| **Version** | 1.0 (Draft) |
| **Date** | 2026-08-02 |
| **Source PRD** | [`docs/prds/cowink-prd.md`](../prds/cowink-prd.md) v1.0 |
| **Decisions log** | [`docs/plans/cowink-trd.md`](../plans/cowink-trd.md) |
| **Scope** | Full PRD scope — every epic, both deal types, both launch countries |
| **Team assumption** | 1–5 people |
| **Status** | Awaiting one blocking decision (company registration country — see §14) |

---

## 1. How to read this document

The PRD said **what** CoWink must do. This document says **how** it will be built, and
**why** each choice was made rather than the obvious alternative.

It is written for someone who is new to this kind of system. Every technical term is
explained the first time it appears. Where a decision is genuinely a judgement call, you
will find what breaks if it goes the other way — so you can overrule it knowingly.

### 1.1 Glossary

Read this once. Everything after it assumes you have.

| Term | What it means |
| --- | --- |
| **Escrow** | Money taken from the buyer and frozen by a third party. Neither side can touch it until agreed conditions are met. This is the core of CoWink. |
| **PSP** (Payment Service Provider) | The company that actually moves money — takes cards, holds funds, sends payouts. |
| **Ledger** | A record of every money movement. Like a bank statement you can only add to. |
| **Double-entry** | Every movement is recorded twice — where money left, where it arrived. If the two sides don't match, something is wrong and you know immediately. |
| **Minor units** | Storing money as whole numbers. `$50.15` is stored as `5015` cents, never as `50.15`. |
| **Idempotency** | Sending the same request twice has the same effect as sending it once. Prevents double-charging when a connection drops and the app retries. |
| **State machine** | A written list of the states a thing can be in, and which moves between them are allowed. Anything not on the list is rejected. |
| **Saga** | A multi-step process across two systems that can't share one transaction. Each step is retryable and reversible, so the whole thing can be completed or unwound. |
| **OAuth** | The "log in with Google" mechanism. Also used here to read a creator's real follower count from their channel, with their permission. |
| **TOTP** | The 6-digit code from an authenticator app that changes every 30 seconds. |
| **RLS** (Row Level Security) | A PostgreSQL feature that decides, per row, who is allowed to see it. Supabase's headline feature. |
| **PostGIS** | A PostgreSQL add-on that understands maps — "who is within 50km of here". |
| **Denormalized** | A pre-calculated copy of data, kept for speed. Slightly stale, much faster. |
| **Webhook** | The PSP calling *your* server to say "a payment succeeded", instead of you asking repeatedly. |
| **FX** | Foreign exchange — converting one currency to another. |
| **KYC / AML** | Checking a person is who they say they are, and is not on a government watchlist, before you send them money. Legally required. |
| **PCI-DSS SAQ-A** | The lightest level of card-security compliance. You qualify only if card numbers never touch your servers. |

### 1.2 Relationship to the PRD

This document follows the PRD except in four places, which are deliberate changes, not
oversights. They are listed in **§2.3**.

Where the PRD marked something `[GAP]` (not designed) or `[DEFECT]` (wrong in the
design), this document either resolves it or carries it forward with an owner. §14 tracks
all of them.

---

## 2. Decisions register

### 2.1 The decisions

| # | Decision | Choice |
| --- | --- | --- |
| D-1 | Launch payout countries | **Pakistan + Canada, both at launch** |
| D-2 | Pricing currency | **USD only** in; local currency out; FX at payout |
| D-3 | Pakistan payout route | **Payoneer** |
| D-4 | Mobile | **Responsive from day one** |
| D-5 | System shape | **One application, nine internal modules** |
| D-6 | Undesigned flows | **Specified now**, marked design-pending |
| D-7 | Language | **TypeScript everywhere** — Next.js + Node |
| D-8 | Database | **Supabase** (managed PostgreSQL) |
| D-9 | Authentication | **Supabase Auth** |
| D-10 | Hosting | **Vercel** + **Supabase** + one always-on worker |
| D-11 | Real-time | **Supabase Realtime** |
| D-12 | Escrow provider | **Not yet chosen** — scored evaluation, §7.1 |
| D-13 | Work handover | **Dedicated submit-work screen** |
| D-14 | Revenue model | **12% of the creator's earnings**, deducted at release |
| D-15 | Search | **PostgreSQL + PostGIS**, behind a swappable interface |

### 2.2 Why, and what breaks otherwise

**D-1 — Both countries at launch.** Wider reach immediately. The cost is that two payout
routes, two currencies and two tax regimes all go live before a single deal has completed.
*If reversed:* launching one country first would cut roughly three months of compliance
work and concentrate less risk in the most dangerous part of the product.

**D-2 — USD pricing only.** Every job, budget, fee and ledger entry is in one currency.
Conversion happens once, at payout. *If reversed:* prices, minimums, fees and comparisons
would exist in several currencies at once, and every "is this budget above your floor?"
check would need a conversion — with a rate that changes hourly.

**D-3 — Payoneer for Pakistan.** PayPal does not operate in Pakistan for receiving money,
and neither does Stripe. Payoneer is what Pakistani freelancers already use. *If
reversed:* there is no reversal available — the PRD's stated method simply does not work
there. ⚠️ Verify current availability before building (§7.5).

**D-4 — Responsive from day one.** Creators work from phones. *If reversed:* retrofitting
240 screens later costs far more than building it in, and you likely lose creator supply
in the meantime (PRD Risk R-8).

**D-5 — One application, nine modules.** The hardest problem in this system is "create the
contract and lock the money — both or neither". Inside one application that is
straightforward. *If reversed:* split across separate services, it becomes a distributed
transaction problem — one of the genuinely hard problems in computing — taken on before
the first customer.

**D-8/D-9 — Supabase.** PostgreSQL underneath (so PostGIS works and you are not locked
in), plus login, file storage and live updates included. For a team of five, that is three
systems you do not have to build. **This comes with one hard condition — see §3.3.**

**D-14 — Creator-side fee.** 12% deducted when money is released. Creators already expect
this from Upwork and Fiverr; advertisers see a clean price when comparing; and you never
chase an invoice because it is taken automatically. *If reversed:* charging the advertiser
makes budgets look higher at exactly the moment they are comparing options.

**D-15 — PostgreSQL for search.** The PRD targets 100,000 profiles. That is small. A
dedicated search cluster would add a second datastore to keep in sync for no benefit at
this size. *If reversed:* revisit past ~1 million profiles or if faceted counts get heavy.

### 2.3 Deviations from the PRD

| # | PRD says | This document says | Why |
| --- | --- | --- | --- |
| DEV-1 | Advertiser pays the platform fee at posting (`1:16823`) | Advertiser pays budget + processing only; CoWink's 12% comes out at release | D-14 |
| DEV-2 | "Cards and PayPal only" (AC-8.4, Non-goal 9) | Payoneer added for Pakistan payouts | PayPal does not serve Pakistan |
| DEV-3 | Multi-currency is Phase 3 (AC-8.11) | FX required at launch | Two launch countries, two currencies |
| DEV-4 | W-9 / W-8BEN are the tax forms (EP-13) | Tax layer is pluggable until §14 O-1 is answered | Those are US forms and only apply if CoWink is a US entity |

Each of these needs product sign-off before build.

---

## 3. Architecture

### 3.1 The shape

```
        ┌─────────────────────────────────────────┐
        │  Browser  (Next.js, responsive)         │
        └───────────────┬─────────────────────────┘
                        │
          reads ────────┼──────── everything else
          (direct)      │         (through the server)
                        │
     ┌──────────────────▼──────────────────┐
     │  CoWink server (Next.js API + Node) │
     │                                      │
     │  Identity  Profile  Discovery        │
     │  Jobs      Contracts  Payments       │
     │  Tax       Messaging  Notifications  │
     └───────┬──────────────────┬───────────┘
             │                  │
     ┌───────▼────────┐  ┌──────▼──────────────────┐
     │   Supabase     │  │  Worker process          │
     │  Postgres      │  │  (Railway / Render / Fly)│
     │  Auth          │  │                          │
     │  Storage       │  │  channel refresh          │
     │  Realtime      │  │  reconciliation           │
     └────────────────┘  │  expiries, reminders      │
                         └───────────────────────────┘
                                    │
                    ┌───────────────▼────────────────┐
                    │  External providers (§7)       │
                    │  PSP · Payoneer · tax · KYC ·  │
                    │  YouTube/TikTok/IG/X · maps ·  │
                    │  email · SMS                   │
                    └────────────────────────────────┘
```

### 3.2 The nine modules

They live in one codebase and one deployment. The boundaries are enforced by folder
structure and lint rules, not by network calls.

| Module | Owns |
| --- | --- |
| **Identity** | Logins, sessions, 2FA, organizations, accounts, roles, permissions |
| **Profile & Channels** | Profiles, connected social accounts, reach refresh, verification |
| **Discovery** | Search, geo queries, filters, eligibility, ranking |
| **Jobs & Proposals** | Job posts, deal requests, their state machines |
| **Contracts** | Contracts, milestones, delivery, disputes, events, PDFs |
| **Payments & Escrow** | Ledger, holds, releases, fees, invoices, payouts |
| **Tax & Compliance** | Forms, expiry, document storage, withdrawal gating |
| **Messaging** | Threads, delivery, system cards, unread counts |
| **Notifications** | Fan-out to email (and later desktop/mobile) per preference |

**Rule:** a module may call another module's public functions. It may never read another
module's tables directly. That single rule is what makes any module extractable later.

### 3.3 ⚠️ The Supabase condition

Supabase's headline feature lets the browser talk to the database directly, with **RLS**
policies deciding who can see what.

**Use that for reading and browsing. Never for money, contracts, or permissions.**

Why this matters:

| What we need | Why RLS can't safely do it |
| --- | --- |
| "Create the contract and lock the money — both or neither" | Spans your database *and* an external PSP. No database policy covers that. |
| An append-only ledger where balances always reconcile | Requires multi-step server logic and provider calls, not a row filter. |
| Permissions that depend on which brand you are currently inside | Expressible in RLS, but as policies complex enough that a mistake is invisible and the cost is someone's money. |

**So:** reads that are already public (profiles, published job posts, search results) may
go direct with RLS. Everything that writes, and everything involving money, goes through
your own server with the service key. RLS stays enabled everywhere as a second line of
defence — never as the only one.

### 3.4 The worker

Some work must not run inside a web request: it is slow, scheduled, or must survive a
deploy.

| Job | Frequency |
| --- | --- |
| Refresh follower counts from the four platforms | Continuous, staleness-prioritized |
| Reconcile our ledger against the PSP | Daily |
| Expire time-bounded team access + 7-day warnings | Daily |
| Close review windows (30 days after completion) | Daily |
| Tax form renewal reminders | Daily |
| Retry stuck sagas (§6.4) | Every few minutes |
| Refresh denormalized search profiles | Event-driven + nightly sweep |

Run one small always-on Node process with a queue. Supabase's `pg_cron` can trigger
lightweight jobs, but the channel refresh (§7.4) needs real queue semantics — retries,
backoff, per-provider rate budgets — so use a proper queue library backed by Postgres or
Redis.

### 3.5 Environments

`local` → `preview` (per pull request) → `staging` → `production`.

**Non-production databases must never contain real tax numbers, real card references, or
real personal data.** Seed them with generated data.

---

## 4. Data model

Table sketches, not final migrations. Types are PostgreSQL.

### 4.1 The three identities

The most important structural decision in the whole system: **a person, a brand, and a
company are three different things.**

```sql
-- The human who logs in
user_account (
  id              uuid pk,
  email           citext unique not null,
  email_verified  boolean not null default false,
  timezone        text not null,          -- IANA name: 'America/Montreal'
  locale          text not null default 'en',
  status          text not null,          -- live | suspended | closed
  created_at      timestamptz not null
)

-- The legal company (may be absent for solo users)
organization (
  id            uuid pk,
  legal_name    text not null,            -- 'Coca Cola.INC'
  country       char(2) not null,
  created_at    timestamptz not null
)

-- The thing that owns jobs, contracts and money
account (
  id               uuid pk,
  organization_id  uuid null references organization,
  kind             text not null,         -- creator | advertiser
  representation   text not null,         -- self | broker
  display_name     text not null,         -- 'Fanta', 'MrBeast'
  payout_currency  char(3) not null,
  home_location    geography(point) null,
  created_at       timestamptz not null
)

-- Who may do what, where, and until when
membership (
  id           uuid pk,
  user_id      uuid not null references user_account,
  account_id   uuid not null references account,
  role         text not null,             -- full_control | manager | accountant
  valid_from   timestamptz not null,
  valid_until  timestamptz null,          -- AC-12.4 auto-revoke
  unique (user_id, account_id)
)
```

**Why `account` and not `user`:** a job post is owned by *Fanta*, not by *Alice*. When
Alice leaves, the contracts stay. This is the single most common modelling mistake in
marketplace software, and it is very expensive to fix later.

**Role is not a column.** A person may act as creator *and* advertiser (PRD §2.1). The
session carries `{ user_id, active_account_id, active_role }`, and every query is scoped
by `active_account_id`. See §5.2.

### 4.2 Money

```sql
-- Used everywhere money appears
amount_minor   bigint not null    -- 5015 means $50.15
currency       char(3) not null   -- 'USD'
```

**Never `numeric`, never `float`, never a bare number.** `0.1 + 0.2` does not equal `0.3`
in floating-point arithmetic. Across a million payments that becomes real money you cannot
account for.

Currency is stored even though pricing is USD-only (D-2), because creator payout floors
and payouts themselves are in local currency, and adding a currency column to a live money
table is one of the most painful migrations there is.

### 4.3 Channels

```sql
channel (
  id                uuid pk,
  account_id        uuid not null references account,
  network           text not null,       -- youtube | tiktok | instagram | x
  external_id       text not null,
  handle            text not null,
  follower_count    bigint null,
  verified_at       timestamptz null,    -- null = never verified, never displayed
  last_checked_at   timestamptz null,    -- drives the Check-In filter
  refresh_state     text not null,       -- ok | stale | revoked | error
  unique (network, external_id)
)

channel_token (            -- separate table, separate access, encrypted
  channel_id     uuid pk references channel,
  access_token   bytea not null,
  refresh_token  bytea not null,
  scopes         text[] not null,
  expires_at     timestamptz
)
```

**AC-4.3 is absolute:** if `verified_at is null`, the follower count is never rendered
anywhere. There is no self-reported fallback.

### 4.4 Requirements — the tri-state

A brand can say three different things about a network (AC-4.4):

```sql
job_requirement (
  job_post_id  uuid not null,
  network      text not null,
  mode         text not null,   -- threshold | unlimited | excluded
  min_followers bigint null,    -- only when mode = 'threshold'
  primary key (job_post_id, network)
)
```

`excluded` means **"do not deal"** — the creator must have **no connected account** on
that network. A nullable integer cannot express this: `null` would be ambiguous between
"any amount" and "none allowed". Getting this wrong silently matches the wrong creators.

### 4.5 Deal requests — three doors, one table

Interest, proposal and direct offer all end in a contract, so they are one entity with a
label (PRD EP-6):

```sql
deal_request (
  id            uuid pk,
  source        text not null,   -- interest | proposal | direct_offer
  job_post_id   uuid null,       -- null for direct offers
  creator_id    uuid not null references account,
  advertiser_id uuid not null references account,
  status        text not null,   -- see §9.1
  amount_minor  bigint null,     -- proposals carry their own terms
  currency      char(3) null,
  created_at    timestamptz not null
)
```

Three parallel tables would mean three state machines that must behave identically
forever. They would not.

### 4.6 Contracts and events

```sql
contract (
  id              uuid pk,
  public_id       text unique not null,   -- 'CW-20251202-K4M9P'
  deal_request_id uuid not null,
  kind            text not null,          -- digital | appearance
  status          text not null,
  -- appearance-only
  starts_at       timestamptz null,
  ends_at         timestamptz null,
  duration_min    int null,
  location        geography(point) null,
  location_label  text null
)

contract_event (           -- append-only. Never updated, never deleted.
  id           bigserial pk,
  contract_id  uuid not null,
  seq          int not null,
  type         text not null,
  actor_id     uuid null,
  payload      jsonb not null,
  created_at   timestamptz not null,
  unique (contract_id, seq)
)
```

**`contract_event` is the source of truth.** Four features are *projections* of it:

- the Contract Timeline (AC-7.5 — "derived from real events, never hard-coded")
- the Recent Activity log (AC-7.6)
- notifications
- the downloadable PDF

One write, four readers. The PRD's requirement is satisfied by construction rather than by
discipline.

**Contract ID format:** `CW-YYYYMMDD-XXXXX`, where `XXXXX` is 5 uppercase alphanumeric
characters excluding `0/O/1/I/L`. This replaces the design's `20251202-Q4&S` — an `&`
breaks URLs, CSV exports and email clients (PRD AC-7.1 `[DEFECT]`).

### 4.7 Search projection

```sql
creator_search_profile (
  account_id       uuid pk,
  display_name     text,
  location         geography(point),
  influence_areas  geography(multipolygon),
  rating_avg       numeric(3,2),
  rating_count     int,
  success_rate     numeric(5,2),
  completed_count  int,
  min_digital_minor    bigint,
  min_appearance_minor bigint,
  networks         jsonb,          -- {youtube: {followers, verified_at, last_checked_at}}
  connected_networks text[],       -- for cheap 'excluded' checks
  last_active_at   timestamptz,
  refreshed_at     timestamptz
)
```

Rebuilt when channels refresh, a contract completes, or a review publishes. Being a few
minutes stale is fine — follower counts only change once a day anyway.

Indexes: GiST on the geography columns, GIN trigram on `display_name`, B-tree on the
numeric filters.

---

## 5. Identity, permissions and onboarding

### 5.1 Authentication

Supabase Auth provides email/password, OAuth logins, and TOTP two-factor.

| PRD requirement | How |
| --- | --- |
| AC-1.1 email + one OAuth provider | Supabase Auth; start with Google |
| AC-1.2/1.4 verification email, 24h expiry, friendly re-send screen | Supabase Auth + custom template |
| AC-1.3 resend limited to 1/min, 5/hour | Our own rate limiter (§5.5) — do not rely on the provider's defaults |
| AC-1.5 ≥12 chars, not a breached password | Password policy + **Have I Been Pwned range API**: send the first 5 characters of the hash, never the password itself |
| AC-11.5/11.7 real second factor | **TOTP required** for any account with escrow authority. SMS as secondary. Security question demoted to account recovery only. |
| AC-11.6 4-digit SMS OTP, countdown, 3 sends / 15 min | SMS provider via Supabase Auth; our rate limiter enforces the cap |

⚠️ **Verify before building:** Supabase Auth's TOTP enrolment flow, and SMS deliverability
and cost for **Pakistan** and **Canada**. If SMS to Pakistan is unreliable or expensive,
TOTP becomes the only second factor and that is acceptable.

**AC-1.7 is an authorization rule, not a UI state.** `email_verified = false` blocks
posting a job, sending a proposal, and adding funds — checked server-side on the endpoint.

### 5.2 Session scope

Every authenticated request resolves to:

```ts
type Ctx = {
  userId: string
  activeAccountId: string
  activeRole: 'creator' | 'advertiser'
  permissions: Set<Permission>
}
```

`activeAccountId` is set when the user picks an account from the header dropdown and is
stored in the session, not the URL. **Every query in every module takes `Ctx` and filters
by `activeAccountId`.** A single query that forgets is a cross-tenant data leak.

Enforce it structurally: module data-access functions accept `Ctx` as their first
argument, and a lint rule forbids raw database clients outside those functions.

### 5.3 Permissions

The PRD's permission matrix (AC-12.2) is marked `[INFERRED]` and has never been signed
off. **Therefore it is stored as data, not written into code.**

```sql
role_permission (role text, permission text, primary key (role, permission))
```

Seeded from the PRD's table. Changing it is a data change with an audit entry, not a
release. Checks are `ctx.permissions.has('contract.release')` — deny by default (AC-12.6).

**Time-bounded access (AC-12.4):** the daily worker revokes memberships past
`valid_until` and warns managers 7 days ahead.

### 5.4 The two OAuth systems — keep them apart

This confusion is common and expensive.

| | **Login OAuth** | **Channel OAuth** |
| --- | --- | --- |
| Purpose | "Sign in with Google" | "Let CoWink read my YouTube stats" |
| Scopes | identity only | read-only channel analytics |
| Tokens | held by Supabase Auth | held by us, encrypted, long-lived |
| Failure | user can't log in | reach data goes stale |
| Revoked by user | must sign in again | **silent** — we only find out on next refresh |

They are separate subsystems that share a protocol and nothing else. Signing in with
Google must never grant channel access, and vice versa.

**Revocation handling:** when a refresh fails with an auth error, set
`channel.refresh_state = 'revoked'`, keep the last known figures, mark them stale, and
prompt the user to reconnect. Never delete the channel.

### 5.5 Rate limiting

Needed from day one for: verification resends (AC-1.3), OTP sends (AC-11.6), login
attempts, and fraud velocity limits on signup, posting and withdrawal (PRD S-9).

One shared counter store. Postgres is sufficient at launch scale; move to Redis if
contention appears.

### 5.6 Onboarding

**AC-2.9** (resume where you left off) and **AC-2.2** (the design's step counts
contradict each other) both point the same way: onboarding state lives on the server, and
the step list is configuration.

```sql
onboarding_state (
  account_id      uuid pk,
  flow            text not null,     -- creator | advertiser
  completed_steps text[] not null,
  data            jsonb not null
)
```

The server returns the ordered step list for the role; the UI renders `Step N of M` from
its length. The denominator is never typed into a component.

**AC-2.4:** at least one verified channel is required to pass the connect step. Enforced
server-side.

---

## 6. Money architecture

The most dangerous part of the system. Everything here is deliberately conservative.

### 6.1 The five invariants — as tests, not prose

The PRD states five non-negotiable rules. Each maps to a named automated test that runs in
CI. A rule with no test is a rule that quietly stops being true.

| Invariant | Test |
| --- | --- |
| **INV-1** A contract can never exist unfunded | `contract_never_exists_without_hold` — property test: after any interleaving of accept/failure, no contract row lacks a matching hold |
| **INV-2** The ledger is append-only and double-entry | `ledger_entries_are_immutable` (UPDATE/DELETE rejected by trigger) + `every_transaction_balances_to_zero` |
| **INV-3** Sum of holds equals the PSP balance | `daily_reconciliation_detects_drift` — inject a discrepancy, assert the alert fires |
| **INV-4** Every state change is an allowed transition | `no_illegal_transitions` — exhaustively attempt every from→to pair, assert only the documented ones succeed |
| **INV-5** Money endpoints are idempotent | `same_idempotency_key_charges_once` — fire N concurrent identical requests, assert one effect |

### 6.2 The ledger

```sql
ledger_transaction (
  id              uuid pk,
  kind            text not null,     -- deposit | hold | release | fee | payout | refund | correction
  idempotency_key text unique not null,
  contract_id     uuid null,
  created_at      timestamptz not null
)

ledger_entry (
  id              bigserial pk,
  transaction_id  uuid not null references ledger_transaction,
  account_ref     text not null,     -- 'escrow:fanta' | 'creator_balance:mrbeast' | 'platform_fee'
  direction       text not null,     -- debit | credit
  amount_minor    bigint not null,
  currency        char(3) not null
)
```

Rules enforced by database triggers, not convention:

- entries are **insert-only** — UPDATE and DELETE raise an exception
- every transaction's debits must equal its credits
- balances are always `SUM(entries)`, never a stored column
- mistakes are fixed with a `correction` transaction, never by editing history

### 6.3 Money flow

```
Advertiser posts a job
   → charge budget + processing fee            [deposit]
   → funds sit in the account's escrow balance

Advertiser accepts a creator
   → lock the slot amount                      [hold]        ← atomic with contract creation

Creator submits work → advertiser approves
   → release the milestone                     [release]
   → deduct 12% platform fee                   [fee]
   → credit the creator's balance

Creator withdraws
   → BLOCKED unless valid tax form + KYC on file
   → send via PSP (Canada) or Payoneer (Pakistan)  [payout]
   → FX applied here, rate recorded on the transaction
```

**DEV-1 in practice.** The `Pay to Post` screen shows the advertiser:

```
Base amount            $5,000.00
Processing fee            $XX.XX      ← from the Pricing Service
Tax (if applicable)       $XX.XX      ← from the tax engine
────────────────────────────────
Total                  $5,0XX.XX
```

No platform fee line. CoWink's 12% appears on the *creator's* side at release, itemised on
their statement.

### 6.4 The atomic accept — a saga, not a transaction

**The problem:** "create the contract and lock the money, both or neither" (INV-1) spans
your database and an external PSP. No single database transaction covers both.

**The solution** — three steps, each retryable, each reversible:

```
1. RESERVE   Write deal_request → 'accepting' + a saga record.
             Call the PSP to place a hold, with an idempotency key.
2. COMMIT    On PSP success: in ONE database transaction, create the
             contract, write the ledger hold, append contract_event,
             mark the saga complete.
3. SETTLE    Publish notifications and refresh projections.
```

Failure handling:

| Fails at | Action |
| --- | --- |
| Step 1, PSP declines | Mark the request `accept_failed`. Show the exact shortfall (AC-6.6 — the design says "$3,000", not "insufficient funds"). Nothing else changed. |
| Step 1, PSP times out | Saga stays `pending`. The worker retries with the **same idempotency key** — the PSP returns the original result rather than holding twice. |
| Step 2, database fails after PSP success | Saga is `hold_placed, contract_missing`. The worker retries the commit. If it cannot complete within the deadline, it releases the hold and alerts. |

**The reconciliation sweep runs every few minutes** and is what makes this safe. Without
it, a saga stuck between steps is a silently frozen customer payment.

**Slot races (AC-6.5):** two managers accepting simultaneously could oversubscribe a
4-slot job. `SELECT ... FOR UPDATE` on the job post row during step 1.

### 6.5 The Pricing Service

The PRD found three contradictory fee models across the design (AC-5.4, AC-8.2, Risk R-7).
The fix is structural: **one module answers every "what does this cost?" question, and no
screen ever computes money.**

```ts
type Quote = {
  lines: { label: string; amountMinor: number }[]
  totalMinor: number
  currency: string
  quoteId: string       // recorded on the resulting transaction
  expiresAt: string
}

priceJobPost(ctx, draft): Quote
priceRelease(ctx, milestoneId): Quote     // includes the 12% deduction
pricePayout(ctx, amount, destination): Quote
```

Fee rates live in configuration with an effective-date, so a change is a settings edit with
an audit entry. The `quoteId` is stored on the transaction, so you can always answer "why
was this person charged that?" months later.

### 6.6 Card handling

Card numbers never touch CoWink infrastructure. Use the PSP's hosted fields or SDK, which
keeps you at **PCI-DSS SAQ-A** — the lightest compliance level (PRD AC-8.1, S-3).

**AC-8.3 `[DEFECT]`:** the design's "Secured by AES-256 encryption" is a claim about data
at rest, not a payment guarantee. Replace with copy naming the PSP and the actual
compliance level. Requires legal review.

### 6.7 Reconciliation

Daily, in the worker:

1. Pull the PSP's balance and transaction list for the period
2. Compare against `SUM(ledger_entry)` per escrow account
3. On any mismatch: alert immediately, and **do not auto-correct**

Drift means either a bug or a missed webhook. Both need a human.

---

## 7. Vendors and integrations

Every provider sits behind a wrapper interface in `src/integrations/`. Business logic never
imports a vendor SDK directly. Swapping a provider — which will happen — should be one
file, not fifty.

### 7.1 ⚠️ Escrow provider — scored evaluation, not a snap decision

This is the single biggest technical decision in the project and it is **not yet made**.
Score every candidate. **Any "no" in the must-have block eliminates them.**

**Must have**

| # | Capability |
| --- | --- |
| 1 | Hold funds on behalf of two parties until instructed to release |
| 2 | **Release part** of a held amount, not all-or-nothing |
| 3 | **Freeze** a release while a dispute is open |
| 4 | **Refund from held funds** back to the payer |
| 5 | Pay out to **Canada** |
| 6 | Pay out to **Pakistan**, or work cleanly alongside Payoneer |
| 7 | **USD in, local currency out**, at a rate that can be shown to the user |
| 8 | Will onboard a marketplace registered in the chosen country (§14 O-1) |
| 9 | States clearly whether **you** or **they** are legally the fund holder |

**Should have**

| # | Capability |
| --- | --- |
| 10 | Hosted card fields (SAQ-A scope) |
| 11 | Idempotency support |
| 12 | Reports reconcilable against our ledger, daily |
| 13 | Saved payment methods with a designated primary |
| 14 | PayPal accepted as a pay-in method |
| 15 | Test mode that simulates disputes and refunds |

**Nice to have:** tax form collection (16), KYC/sanctions screening (17), webhooks on every
money event (18).

**Test 2, 3 and 4 first** — they eliminate candidates fastest. Most providers send money
well and can do none of them.

**Item 9 is a legal question, not a technical one.** If *you* are the fund holder, you may
be a regulated money business in your country of registration. Get advice before building.

### 7.2 Social platforms

| Platform | Notes |
| --- | --- |
| **YouTube** | Most stable, best documented. Build first. Daily quota, not a rate limit — budget it. |
| **TikTok** | Requires application and approval. Start the application now (§13). |
| **Instagram** | Requires the creator to have a Business or Creator account linked to a Facebook Page. Expect drop-off. |
| **X** | Paid tiers. Model the cost before committing. |

All four behind one interface:

```ts
interface ChannelProvider {
  authUrl(state: string): string
  exchangeCode(code: string): Promise<Tokens>
  fetchStats(tokens: Tokens): Promise<{ followers: number; handle: string }>
}
```

### 7.3 Other providers

| Purpose | Notes |
| --- | --- |
| **Payoneer** | Pakistan payouts (D-3). Verify current API access and onboarding requirements. |
| **Tax engine** | Buy, do not build. Selection depends on §14 O-1. |
| **KYC/AML** | Mandatory before payout. Not in the design at all (PRD AC-13.8). |
| **Maps + geocoding** | Confirm the licence permits commercial marketplace use. Cache geocode results aggressively — they are billed per call and locations rarely change. |
| **Email** | Verification, notifications, invoices. Set up SPF/DKIM/DMARC properly or verification emails land in spam and onboarding dies. |
| **SMS** | OTP. Check Pakistan and Canada coverage and cost. |
| **Storage** | Supabase Storage for avatars and attachments. **Tax documents go in a separate bucket** with its own access policy (§11.4). |

### 7.4 Channel refresh — infrastructure, not a cron

AC-2.5 requires a daily refresh. At the PRD's target scale:

```
100,000 accounts × ~2 channels = ~200,000 calls/day ≈ 2.3/second sustained
```

YouTube enforces a **daily quota**, not a per-second rate — you cannot simply back off and
retry later in the day. Design accordingly:

- one queue per provider, with a configured daily budget
- prioritize by staleness × account activity — active creators refresh first
- exponential backoff on transient errors; `revoked` on auth errors
- **degrade, never fail:** keep the last known figures and mark them stale (PRD Risk R-3)
- `last_checked_at` is user-visible — it drives the `Check-In` filter and the
  `Last check in DD/MM/YYYY` stamp

---

## 8. Discovery and the matching engine

### 8.1 Storage

PostgreSQL + PostGIS against `creator_search_profile` / `job_search_profile` (§4.7), behind:

```ts
interface SearchProvider {
  searchCreators(ctx: Ctx, q: CreatorQuery): Promise<Page<CreatorResult>>
  searchJobs(ctx: Ctx, q: JobQuery): Promise<Page<JobResult>>
}
```

Revisit at ~1 million profiles or if faceted counts become heavy.

**Free-text disambiguation (AC-3.3).** One box accepts a place *or* a name. Run both
lookups in parallel: if geocoding returns a confident match, apply the geo filter and show
the `Location Active` chip; otherwise treat it as a name query. The rule is explicit and
unit-tested.

**Pagination (AC-3.8).** Numbered pages require a total count. Use an estimated count
capped at a threshold (`500+`) rather than `COUNT(*)` on every search, and cap deep paging.
The sort must include stable tiebreakers (§8.3) or rows repeat across pages.

**Filter state (AC-3.6)** is encoded in the URL under a documented, versioned query-param
schema. The moment someone shares a filtered link, that schema is a public contract.

### 8.2 ⭐ Eligibility — one shared function

The highest-leverage design decision in discovery.

```ts
type EligibilityResult = {
  eligible: boolean
  failures: {
    rule: 'reach' | 'price_floor' | 'geography' | 'verification'
    network?: string
    required: number | string
    actual: number | string
    remediation?: string
  }[]
}

function evaluateEligibility(
  creator: CreatorSnapshot,
  job: JobCriteria,
  now: Date,
): EligibilityResult
```

Three properties, all deliberate:

1. **It returns reasons, not a boolean.** AC-3.10 requires naming the specific unmet
   requirement *and the user's current value*: "you need 200K subscribers, you have 187K".
   A boolean cannot produce that sentence.
2. **It is pure** — no database, no network, no clock. `now` is passed in. This makes the
   PRD's 200-profile golden-set test (§3.3) trivial to write and perfectly reproducible.
3. **It is used in exactly two places** — filtering search results, and gating the apply
   button. One implementation means they can never disagree. Two implementations means a
   creator eventually sees a job they cannot apply to, with no explanation.

The four rules (PRD §3.1):

| Rule | Check |
| --- | --- |
| **Reach** | `Match All` → every specified network passes. `Match Any` → at least one. `unlimited` always passes. `excluded` → the network must **not** appear in `connected_networks`. |
| **Price floor** | Job's per-creator budget ≥ the creator's stored minimum for that deal type |
| **Geography** | Appearance deals only: job location falls within the creator's influence areas |
| **Verification** | Every network used to establish eligibility is verified and fresher than the staleness threshold |

### 8.3 Ranking

Weighted score over normalized signals (PRD §3.2). Weights live in a settings table with an
effective-date, and the active version is stamped on every search response — so "why did
results change last Tuesday?" is answerable.

| Signal | Default |
| --- | --- |
| Rating | 0.25 |
| Job success rate | 0.20 |
| Reach fit — closeness to requirement without excessive overshoot | 0.20 |
| Recency of activity | 0.15 |
| Geographic proximity | 0.10 |
| Completed contract volume | 0.10 |

**Reproducibility rules:**
- no randomness anywhere in scoring
- no `Date.now()` inside the scorer — time is an input
- ties break by most-recent activity, then account age (this is also what makes pagination
  stable)

**Cold start (PRD Risk R-5).** 30% of the score rewards past work, so a brand-new verified
creator scores zero on two signals before doing anything wrong. A marketplace that buries
new creators runs out of supply. Add a `new_account_boost` setting: a decaying bonus for
accounts verified within N days. It is configuration so it can be tuned during launch and
switched off later without a release.

**Precomputation split:** rating, success rate, volume and recency are stored on the search
profile. **Reach fit depends on the specific job**, so it is computed per query.

### 8.4 Tests

| Test | Requirement |
| --- | --- |
| Golden set — 200 profiles × 50 jobs, hand-labelled | **100% correct.** It is deterministic; anything less is a bug. |
| Ranking snapshot | Any weight change surfaces as a reviewed diff |
| New-creator starvation | A zero-contract creator who fully satisfies a query must appear above page 3 |
| Boundaries | Reach exactly at threshold; budget exactly at floor; `excluded` with a connected account; `Match Any` with one qualifying network; stale check-in |

---

## 9. Contracts, delivery and disputes

The PRD's largest gap (AC-7.9, Risk R-1 — Critical/Certain). These flows have no screens.
They are specified here anyway, because they determine the PSP requirements in §7.1 and the
contract data model. Screens follow.

### 9.1 State machines

Illegal transitions are rejected by the code, not merely undrawn (INV-4).

**Deal request**
```
draft → submitted → waiting_response ─┬→ accepted → (contract created)
                                      ├→ rejected
                                      ├→ withdrawn
                                      └→ expired
```

**Job post**
```
draft → published ─┬→ filled
                   ├→ closed        (auto-rejects pending interest, notifies each — AC-6.7)
                   └→ expired
```

**Milestone** — including the undesigned states
```
draft
  → funded              (escrow hold placed — AC-7.4, before the counterparty sees it)
  → visible
  → submitted           (creator delivers — §9.2)
  ├→ approved → released
  ├→ revision_requested → submitted
  └→ disputed → (resolved_release | resolved_refund | resolved_split)
cancelled / refunded reachable from funded, visible, submitted
```

**Contract**
```
active → completed
      → cancelled
      → disputed → active | cancelled | completed
```

Statuses are normalized: the design's `Ended & Payed` becomes `ended_paid`, and
`Funded In Escro` becomes `funded_in_escrow` (AC-8.13 `[DEFECT]`).

### 9.2 Work handover (D-13)

A dedicated submit screen, not chat.

```sql
deliverable (
  id            uuid pk,
  milestone_id  uuid not null,
  submitted_by  uuid not null,
  note          text,
  links         text[],           -- published post URLs
  files         jsonb,            -- storage keys + checksums
  submitted_at  timestamptz not null
)
```

The advertiser gets **Approve** or **Request changes** (with a required reason). Both write
a `contract_event`.

**Why not chat.** This step releases money. When two people disagree about whether work was
delivered, "scroll through the conversation" is exactly what escrow exists to avoid. A
deliverable record gives a timestamped, immutable answer to "what was handed over, and
when?"

Chat attachments may be added later for drafts and back-and-forth (PRD AC-9.8), but they
are not the handover.

**Note:** CoWink never verifies that content was actually published (PRD Non-goal 3).
Approval is a human decision. That makes this screen the entire delivery mechanism.

### 9.3 Disputes

```sql
dispute (
  id            uuid pk,
  milestone_id  uuid not null,
  opened_by     uuid not null,
  reason        text not null,
  status        text not null,   -- open | evidence | under_review | resolved
  resolution    text null,       -- release | refund | split
  split_minor   bigint null,
  resolved_by   uuid null,
  opened_at     timestamptz not null,
  resolved_at   timestamptz null
)
```

Flow: **open** (either party, before release) → **evidence** (both sides submit, fixed
window) → **review** (platform decides, target SLA 5 business days) → **resolve**
(full release, full refund, or a split).

The moment a dispute opens, the milestone's hold is **frozen** at the PSP. This is why
checklist items 2, 3 and 4 in §7.1 are must-haves: a provider that can only send money
cannot resolve a dispute.

Every step writes a `contract_event` and notifies both parties.

### 9.4 Payouts (PRD AC-8.12 — no screens exist)

```sql
payout_method (
  id            uuid pk,
  account_id    uuid not null,
  kind          text not null,      -- bank | payoneer | paypal
  country       char(2) not null,
  details       jsonb not null,     -- tokenized; no raw bank numbers
  is_primary    boolean not null
)

payout_request (
  id                uuid pk,
  account_id        uuid not null,
  amount_minor      bigint not null,
  currency          char(3) not null,
  fx_rate           numeric null,
  destination_minor bigint null,
  status            text not null,  -- requested | blocked | processing | paid | failed
  block_reason      text null,      -- names the specific missing form (AC-13.2)
  requested_at      timestamptz not null
)
```

Routing: **Canada** → PSP payout. **Pakistan** → Payoneer.

Every request is checked against tax form validity **and** KYC status before anything
moves, and **fails closed** — if the check cannot be confirmed, the payout is blocked, never
assumed fine.

The FX rate used is recorded on the transaction and shown to the creator before they
confirm.

### 9.5 Authorization

**Authorize actions, not records** (AC-7.7). Both parties see the identical contract
payload plus an `allowed_actions` list computed from role and state:

```json
{ "contract": { ... }, "allowedActions": ["submit_deliverable", "open_dispute"] }
```

The UI renders buttons from that list. The server re-checks on every action. Neither side
ever sees a different version of the truth.

### 9.6 PDFs

Generated server-side and **archived to storage**, not re-rendered on demand. A contract
downloaded in 2026 must look identical in 2029. Includes all terms, milestones and the full
activity log (AC-7.8).

---

## 10. Messaging, notifications and reviews

### 10.1 Messaging

One thread store, two entry points: the global inbox and the in-contract tab (AC-9.1).
Threads are scoped to a job or contract — there is no open-ended chat.

Delivery uses **Supabase Realtime** (D-11), which also powers the live contract timeline
(AC-9.5). Target: ≤2s p95, no manual refresh (AC-9.6).

**System messages are structured data, never sentences** (AC-9.4):

```json
{
  "type": "contract_created",
  "payload": { "contractId": "...", "title": "1 video",
               "dueDate": "2025-04-14", "amountMinor": 30000, "currency": "USD" }
}
```

The client renders the card and the `View details` link. If you store the finished English
sentence instead, you can never translate it (§12.3), never restyle it, and never add the
button.

Unread counts are a per-user, per-thread read cursor, driving the notification badge
(AC-9.7).

### 10.2 Notifications

One fan-out module reading per-user preferences (AC-11.4's Desktop × Mobile × Email
matrix). Email is the only channel wired at launch; the matrix is stored from day one so
adding channels is configuration.

Every notification is triggered by a `contract_event` or an equivalent domain event —
never by a scattered call inside business logic.

### 10.3 Reviews

Contract-gated: only a counterparty on a **completed** contract may review, one per contract
per side (AC-10.5). Window closes 30 days after completion — enforced by the daily worker.

Four sub-scores plus a star rating and free text. Sub-scores are the mean of per-contract
ratings, shown as an integer percentage, and **hidden entirely until ≥3 completed
contracts** (AC-4.1). That suppression is applied in **one serializer**, not per endpoint —
otherwise it leaks somewhere.

Reviews are immutable once published; the reviewed party may post one public response
(AC-10.4).

---

## 11. Tax and compliance

⚠️ **This section is pluggable pending §14 O-1.** The forms below assume a US entity, which
is what the PRD assumes. If CoWink registers in Canada or Pakistan, the *shape* below holds
but the *forms* change.

### 11.1 Determination

Tax status is derived from **entity type × tax residence**. The user never chooses their own
form (AC-13.1) — asking people to self-select guarantees wrong filings.

```ts
interface TaxProvider {
  determineForm(entityType, residence): FormType
  validate(form): ValidationResult
  isValid(accountId): Promise<{ valid: boolean; missing?: FormType; expiresAt?: Date }>
}
```

Swapping jurisdictions means a new implementation, not a rewrite of the payout code.

### 11.2 Gating

Withdrawal is hard-blocked until a valid, unexpired form is on file, and **the block
message names the specific missing form** (AC-13.2, AC-8.9). Not "blocked" — "we need your
W-8BEN before you can withdraw."

**W-8BEN expires** on 31 December of the third year after signing. The worker prompts 60
days before (AC-13.4).

### 11.3 Signatures

Typed-signature certification records the typed name, timestamp and source IP, stored
immutably (AC-13.3). This is what makes the certification legally meaningful.

### 11.4 Sensitive data

Tax identifiers (TIN / SSN / EIN) and uploaded documents are the most sensitive data in the
product.

- encrypted at rest under a **dedicated key**, separate from the general database key
- masked in every UI after entry
- **excluded from all logs, analytics events, error reports and non-production databases**
- documents in a separate storage bucket, PDF only, size-capped, virus-scanned, access
  confined to compliance staff and access-logged
- retained per statutory minimum (typically 7 years) and **surviving account closure** —
  closure anonymises the profile but preserves the financial and compliance record (PRD S-7)

### 11.5 KYC / AML

Not in the design at all, and mandatory before international payout. Identity verification
plus sanctions and PEP screening, run before the first payout and re-run on a schedule.
Fails closed.

---

## 12. Cross-cutting

### 12.1 Security

| Ref | Requirement |
| --- | --- |
| S-1 | Password + **mandatory TOTP** for any account with escrow authority. SMS secondary. Security question is recovery-only. |
| S-2 | Per-account, per-role checks server-side on every request. Deny by default. |
| S-3 | No raw card data on CoWink infrastructure. PSP hosted fields, SAQ-A scope. |
| S-4 | Sensitive data classes per §11.4. |
| S-5 | TLS 1.3, HSTS, secure/httpOnly/SameSite cookies. |
| S-6 | GDPR/UK-GDPR, PIPEDA, and Pakistani data rules. Requires lawful-basis mapping, access/erasure/portability workflows, DPAs with every subprocessor, a cross-border transfer mechanism, 72-hour breach notification, and a published retention schedule. |
| S-8 | Reporting and moderation for profiles, job posts, proposals and messages. **No moderation UI exists in the design** — needs building. |
| S-9 | Velocity limits on account creation, posting and withdrawal. Detection of collusive advertiser↔creator pairs cycling escrow to fabricate reputation. |
| S-10 | Managed secret store, documented rotation, no credentials in source. |

### 12.2 Audit log

One shared, append-only log for every money movement, permission change and contract state
transition (AC-X.6): actor, timestamp, before/after, request ID.

Built once as shared plumbing. If each feature invents its own, you cannot answer a
regulator's question and you will rewrite it under pressure.

### 12.3 Internationalization

- **All strings externalised from day one** (AC-14.2). Cheap now; retrofitting across 240
  screens is months of miserable work.
- Dates, numbers and currency render per the user's locale; contract dates additionally show
  the timezone (AC-14.3).
- Layouts tolerate **40% string growth** without truncation (AC-14.4) — German and Finnish
  routinely need it.
- RTL support, launch languages, and whether user-written content is translated remain open
  (AC-14.5).

### 12.4 Accessibility

WCAG 2.2 Level AA (AC-X.2). The design carries no accessibility annotations, so an audit is
required.

**The map is the hard part.** Rather than attempting keyboard-navigable map pins, treat the
**results list as a complete, equal alternative** — every result reachable, filterable and
actionable from the list alone. Cheaper and genuinely better.

Wizard step changes must be announced to screen readers.

### 12.5 Shared UI states

The design supplies almost no loading, empty or error states (AC-3.12, AC-5.8, AC-X.5, Risk
R-10). Build **one** set of components — `<Loading>`, `<Empty>`, `<ErrorState>` — and require
their use. Otherwise 240 screens each invent their own.

### 12.6 Validation

Defined **once**, server-side, using a shared schema (Zod). The client imports the same
schema for instant feedback, but the server is authoritative and the client renders the
errors the server returns.

### 12.7 Analytics

The PRD's eight KPIs (§1.3) are unmeasurable without instrumentation. Define the event
taxonomy up front — `signup_verified`, `onboarding_completed`, `channel_connected`,
`job_published`, `contract_funded`, `milestone_released`, `payout_completed`,
`dispute_opened` — with a documented schema per event, emitted from the domain layer.

### 12.8 Performance budgets

| Target | Value |
| --- | --- |
| p95 page interaction | < 300ms |
| p95 API response | < 500ms |
| Discovery first meaningful paint | < 1.5s |
| Filtered search, server time p95 @ 100k profiles | < 500ms |
| Message delivery p95 | < 2s |

Wire up measurement in month 1. Performance targets discovered at the end are always missed.

---

## 13. Build sequence

### 13.1 Start immediately, in parallel

These are **calendar time, not work time**. More developers do not speed them up, and each
can block launch.

| Task | Why now |
| --- | --- |
| **Company registration** (§14 O-1) | Blocks PSP onboarding, which blocks escrow, which is the product |
| **Escrow provider evaluation + onboarding** | Marketplace onboarding takes weeks to months |
| **Social platform API applications** | TikTok and Instagram require approval and can decline |
| **Tax vendor selection** | Depends on registration; start the shortlist now |

### 13.2 Delivery order

| Window | Focus |
| --- | --- |
| **Months 1–3** | Identity, accounts, organizations, permissions, onboarding, YouTube verification, profiles |
| **Months 3–5** | Discovery, map, filters, job posting (both deal types), proposals, offers, hiring |
| **Months 5–9** | Money end to end — escrow, milestones, delivery, approval, release, payout, tax gating |
| **Months 9–12** | Disputes, messaging, reviews, notifications |
| **Months 12–16** | Organizations and billing, both countries live, remaining social platforms, languages, accessibility, polish |

**Money gets a thin end-to-end path early on purpose.** Fund → hold → release → payout,
even with rough screens, by month 6. Vendor surprises should surface in month 5, not month
14.

**Realistic estimate: 14–18 months to full launch for a team of 3.**

---

## 14. Open questions and risks

### 14.1 Blocking

| # | Question | Owner | Blocks |
| --- | --- | --- | --- |
| **O-1** | **Where is CoWink registered?** Recommendation: **Canada** — PSPs will onboard you, it is already a launch market, cheaper than the US, credible to brands. ⚠️ Pakistan registration risks disqualifying you from major PSPs, leaving escrow with nowhere to run. | Founder + accountant + lawyer | §11 entirely, §7.1 item 8, tax vendor |
| **O-2** | Exact fee percentages and who pays each component (PRD Q-1) | Product / Finance | Pricing Service config |
| **O-3** | Sign off the permission matrix (PRD Q-5, AC-12.2 `[INFERRED]`) | Product | §5.3 seed data |

### 14.2 Carried forward from the PRD

| # | Question | Status |
| --- | --- | --- |
| Q-3 | Mobile scope | **Resolved** — responsive from day one (D-4) |
| Q-4 | How are disputes resolved? | Platform arbitration assumed (§9.3). Needs legal sign-off on the policy and SLA. |
| Q-6 | `Check-In` staleness threshold | Open. Recommend 7 days as a configurable default. |
| Q-7 | What is a "Creator Network"? | **Resolved for modelling** — a hired slot filled by exactly one Account, which may itself be broker-type. Needs product confirmation. |
| Q-8 | Can one creator take multiple slots on a `Many` job? | Open. Recommend no, for v1. |
| Q-9 | What does `Share Feedback` share to? | Open — design. |
| Q-10 | Intro video uploaded or embedded? | Recommend **embed from a connected channel** — avoids storage, transcoding, moderation and CDN cost, and reinforces verification. |

### 14.3 Design work still required

Everything the PRD marked `[GAP]`:

- dispute, cancellation and refund screens (§9.3)
- milestone submission and approval screens (§9.2)
- creator payout screens (§9.4)
- profile edit screens for both roles (AC-4.7)
- loading, empty and error states throughout (§12.5)
- validation error states in the job wizard (AC-5.8)
- invite acceptance, pending invites, seat limits, last-Full-Control guard (AC-12.7)
- moderation and reporting UI (S-8)
- mobile layouts for all 240 frames (D-4)
- full copy pass — `Exlopre`, `Funded In Escro`, `Ended & Payed`, `$5,00`, inconsistent step
  counts (Risk R-11)

### 14.4 Risks and their technical mitigation

| Risk | Mitigation in this document |
| --- | --- |
| **R-1** Escrow with no dispute path | §9.3 specified; §7.1 items 2–4 make it a PSP hard requirement |
| **R-2** Global launch vs limited compliance design | §11 pluggable; fails closed; buy compliance |
| **R-3** Social API dependency | §7.4 degrade-to-stale, provider interface, quota budgeting |
| **R-4** Money-path correctness | §6 — five invariants as tests, double-entry ledger, minor units, idempotency, daily reconciliation |
| **R-5** Cold start | §8.3 `new_account_boost` + the starvation test |
| **R-6** Off-platform disintermediation | Product concern. Keep escrow, disputes and reputation genuinely valuable. |
| **R-7** Contradictory fee model | §6.5 Pricing Service — one authority, no client-side maths |
| **R-8** No mobile design | Resolved by D-4; design work in §14.3 |
| **R-9** Weak second factor | §5.1 — TOTP mandatory, security question demoted to recovery |
| **R-10** Undesigned states | §12.5 shared state components |
| **R-11** Copy quality | Copy pass before string extraction (§12.3) |

---

## Appendix A — Definition of done for this document

1. ✅ Every decision appears with a plain-language rationale
2. ✅ Every PRD `[GAP]` / `[DEFECT]` / `[INFERRED]` marker is resolved or carried forward
3. ✅ All ten Appendix B questions answered or tracked (§14.2)
4. ✅ All five invariants map to a named automated test (§6.1)
5. ✅ The four deviations stated as deliberate changes (§2.3)
6. ⬜ Read §3 and §6 aloud to a non-technical reader — they should follow it without
   looking anything up
