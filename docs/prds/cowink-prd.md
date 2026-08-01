# Product Requirements Document — CoWink

| Field | Value |
| --- | --- |
| **Product** | CoWink — Creator ↔ Advertiser Marketplace |
| **Version** | 1.0 (Draft) |
| **Date** | 2026-08-01 |
| **Source of truth** | [Figma — CoWink](https://www.figma.com/design/TfIlygwIKjVvjXCF2Frf0a/Cowink?node-id=0-1) (240 frames, 1 page) |
| **Scope** | Full platform — every flow present in the design file |
| **Launch market** | Global from day one |
| **Tech stack** | **TBD** — deliberately deferred to the Technical Requirements Document (TRD) |
| **Matching approach** | Deterministic / rules-based. No ML or AI ranking in any phase of this PRD. |

> **Traceability note.** Every requirement below is derived from a frame in the Figma file and carries its node ID (e.g. `1:14850`). Requirements marked **[GAP]** are *not* in the design but are mandatory for a functioning global product — they need design work before build. Requirements marked **[INFERRED]** extend visible UI with behaviour the design implies but does not draw.

---

## 1. Executive Summary

### 1.1 Problem Statement

Brands and content creators transact today over DMs, email threads, and ad-hoc invoices: an advertiser has no reliable way to verify a creator's real reach, price a deal against comparable work, or guarantee delivery before paying — and a creator has no protection against non-payment, no way to filter out deals below their floor rate, and no discoverable storefront. The result is high deal-failure rates, disputed scope, and payment risk on both sides.

### 1.2 Proposed Solution

CoWink is a two-sided marketplace where verified creators and advertisers discover each other by **location, audience reach, and deal type**, agree scope through structured job posts and proposals, and transact through **milestone-based escrow contracts** — with in-app messaging, a shared contract timeline, and reputation scores carried across deals. It supports both **digital content deals** (creator delivers content) and **in-person appearance deals** (creator shows up at a specified location and time), for individual creators, brokers, and multi-brand organizations.

### 1.3 Success Criteria

| # | KPI | Target (12 months post-GA) | Measurement |
| --- | --- | --- | --- |
| SC-1 | **Onboarding completion** — % of signups that reach "Your account is all set!" | ≥ 65% | Funnel event `onboarding_completed` / `signup_verified` |
| SC-2 | **Channel connection rate** — % of onboarded users with ≥ 1 verified social channel | ≥ 90% | Users with `channel.verified_at IS NOT NULL` |
| SC-3 | **Post-to-contract conversion** — % of job posts that produce ≥ 1 funded contract within 14 days | ≥ 40% | Cohort by `job_post.created_at` |
| SC-4 | **Escrow funding success** — % of contract-creation attempts where funding settles first try | ≥ 97% | PSP authorisation success rate |
| SC-5 | **Contract completion** — % of funded contracts reaching all milestones released, no dispute | ≥ 85% | `contract.status = completed` / funded |
| SC-6 | **Time to first offer** — median from job post published → first creator expresses interest | ≤ 24 hours | Event delta |
| SC-7 | **Payout compliance block rate** — % of withdrawal attempts blocked by missing tax forms | ≤ 5% by month 6 | Blocked withdrawals / attempts |
| SC-8 | **Dispute rate** | ≤ 3% of funded contracts | Disputes opened / funded contracts |

---

## 2. User Experience & Functionality

### 2.1 User Personas

| ID | Persona | Description | Design evidence |
| --- | --- | --- | --- |
| P-1 | **Independent Advertiser** | Represents their own brand. Posts jobs, funds escrow, hires creators directly. | `1:21093` "Yourself — I Represent My Own Brand and Identity" |
| P-2 | **Broker / Agency Advertiser** | Runs advertising deals on behalf of someone else; may manage several brand accounts. | `1:21093` "Broker — Someone else represents me and works on my advertising deals" |
| P-3 | **Content Creator** | Individual influencer with audience across YouTube / Instagram / TikTok / X. Receives offers, sends proposals, delivers against milestones. | `1:20992` "Content Creator — I create content and want to collaborate with brands." |
| P-4 | **Creator Network / Broker** | Represents multiple creators; the design calls hired parties "Creator Network(s)". | `1:14421` "Creator Networks", profile type "Broker/Organization" |
| P-5 | **Organization Manager** | Full-control member of a legal entity (e.g. `Coca Cola.INC`) with multiple brand accounts (`Coke`, `Fanta`, `Sprite`). Invites users, assigns roles. | `1:35270` Users table; `1:11077` account switcher |
| P-6 | **Organization Accountant** | Scoped role that sees billing, payment history, invoices, and tax settings — not job/contract authoring. | `1:40283` "User as an accountant"; role `Accountant` in `1:35270` |

**Role duality:** a single account can act as both Creator and Advertiser. The header exposes `Switch To Advertiser` (`1:51568`), so role is a **session context**, not an immutable account attribute.

---

### 2.2 Epic EP-1 — Account Creation & Authentication

**Story.** As a new user, I want to create and verify an account so that I can be trusted by counterparties before money moves.

Screens: `1:21970` Create account · `1:22112` Verify your account · `1:22284` Create Password · `1:22388` Your Profile Is Created

**Acceptance Criteria**

- **AC-1.1** Signup accepts email + a federated option (the design shows an `OR` divider and a `Continue with Email` affordance at `1:33961`). At minimum: email/password and one OAuth provider. Exact providers **TBD in TRD**.
- **AC-1.2** On signup, a verification email is sent to the entered address and the user lands on `Verify your account`, which displays the target address verbatim and a `Resend Verification` control.
- **AC-1.3** `Resend Verification` is rate-limited to 1 send / 60 seconds and 5 sends / hour per address; the button shows a countdown while cooling down.
- **AC-1.4** The verification link expires after 24 hours; an expired link renders a re-send screen, never a raw error.
- **AC-1.5** Password creation enforces ≥ 12 characters and rejects any password appearing in a compromised-credential corpus. Strength is shown inline before submit.
- **AC-1.6** After password creation the user sees `Your Profile Is Created` → "You've successfully created your account. Now you can start with the onboarding", with a single primary CTA into onboarding.
- **AC-1.7** No unverified account may create a job post, send a proposal, or add funds.

---

### 2.3 Epic EP-2 — Role Selection & Onboarding

**Story.** As a new user, I want a guided setup that captures my role, channels, and geography so the marketplace can show me relevant counterparties from my first session.

**Advertiser onboarding — 6 steps** (`1:20992` → `1:21904`)

| Step | Screen | Node | Captured |
| --- | --- | --- | --- |
| 1 of 6 | Choose Profile | `1:20992` | Content Creator \| Advertiser/Branding |
| 2 of 6 | Who represents you | `1:21093` | Yourself \| Broker |
| 3 of 6 | Connect Channels | `1:21200` | OAuth-verified YouTube / TikTok (+ Instagram, X) |
| 4 of 6 | Select Your Location | `1:21392` | Home location (map picker) |
| 5 of 6 | Select Influence Location | `1:21471` | 1..n target markets, `Add Another Location` |
| 6 of 6 | Add Funds | `1:21556`, `1:21774` | Escrow deposit (skippable) |
| — | Profile Created | `1:21904` | "Your account is all set!" |

**Creator onboarding — 5 steps** (`1:55933` → `1:56392`)

| Step | Screen | Node | Captured |
| --- | --- | --- | --- |
| 1 of 5 | Choose Profile | `1:55933` | Role |
| 3 of 5 | Connect Channel | `1:56034`, `1:56201` | Channel OAuth; state flips to `Connected` |
| 4 of 4/5 | Select Location V1 / V2 | `1:56266`, `1:56305` | Home + influence locations |
| 5 of 5 | Minimum Deal Criteria | `1:56162` | Floor price for **Digital content deal** and **Appearance deal** |
| — | Account set | `1:56392` | "Your account is all set!" + `Go to My Profile` |

**Acceptance Criteria**

- **AC-2.1** Every onboarding screen displays an accurate `Step N of M` indicator, a `Back` control, and a persistent `Need help setting up? / Contact Support` link.
- **AC-2.2** **[DEFECT]** The design contains inconsistent step counts on the same flow — `Step 1 of 6` (`1:20992`) vs `Step 1 of 5` (`1:22488`); `Step 3 of 6` and `Step 3 of 5` appear on the same Connect Channels frame (`1:21200`); `Step 4 of 4` on `1:56266` inside a 5-step creator flow. Implementation must derive the denominator from the actual step list for the selected role, and design must be corrected before build.
- **AC-2.3** Channel connection is **OAuth-based ownership verification**, not a handle text field: "Connect your YouTube channel to verify that you are the owner of the account" (`1:21200`). A connected channel displays state `Connected` and its follower/subscriber count.
- **AC-2.4** At least one verified channel is required to continue past step 3 ("Connect at least one channels to begin your personalized experience").
- **AC-2.5** Follower/subscriber counts are refreshed from each platform API on a schedule (target: every 24h) and the profile shows a `Last check in DD/MM/YYYY` stamp (`1:9739`).
- **AC-2.6** Influence locations are multi-valued; `Add Another Location` appends a row with no fixed upper bound below 10.
- **AC-2.7** Advertiser `Add Funds` is **skippable** (`Skip` on `1:21774`); a user who skips can browse but is blocked at contract creation until funded.
- **AC-2.8** Creator minimum-deal criteria are stored as two independent floors (digital, appearance), each in the user's payout currency, and are enforced by EP-3 gating.
- **AC-2.9** A user who abandons onboarding resumes at the last completed step on next login.

---

### 2.4 Epic EP-3 — Discovery, Search, Map & Filters

**Story (advertiser).** As an advertiser, I want to search creators by location, reach, and price so I can shortlist candidates who actually match my campaign.
**Story (creator).** As a creator, I want to browse advertisers and their open job posts so I can find work that clears my minimum rate.

Screens: `1:8226` Explore Creators · `1:9017` Collapsed Map View · `1:9739` Creator View · `1:10947` Filters · `1:51241` Explore Advertisers · `1:52181` Creator-side map · `1:54706` Filters→Brands · `1:54760` Filters→Jobs · `1:54276` Criteria-not-met state

**Acceptance Criteria**

- **AC-3.1** Discovery is a **split map + list** layout: an interactive map with pinned results beside a scrollable result list. The map is collapsible (`Collapsed Map View`), and the collapsed/expanded state persists per user across sessions.
- **AC-3.2** Advertiser-side header copy: "Explore Creators / Find the perfect influencer for your next campaign." Creator-side: "Explore Advertisers / Find the perfect sponsored for your next content."
- **AC-3.3** Search accepts free text matched against **location or counterparty name** (placeholder: "Search by location or creator name", `1:51568`). A `Location Active` chip indicates a geo filter is applied.
- **AC-3.4** Filter panel (`1:10947`) supports:
  - Deal type: `Digital Deal` \| `In-person` \| `Both`
  - `Sphere of influence` (geographic radius/market)
  - `Check-In` (recency of last verified channel check)
  - Budget range with explicit `Min` / `Max` (design values: `$200` / `$500`)
  - Per-network follower thresholds (e.g. `YouTube 100K / 150K`, `TikTok`)
- **AC-3.5** Creator-side filters split into **Brands** (`1:54706`) and **Jobs** (`1:54760`); the Jobs variant adds per-network reach thresholds.
- **AC-3.6** Filters apply without a full page reload and are encoded in the URL so a filtered view is shareable and back-button-safe.
- **AC-3.7** Result cards render: avatar, name, rating `4.93/(66)`, location, niche tags with `+N more` overflow, aggregate reach (`1.2M+ Reach`), and price anchor (`Starting at $500`).
- **AC-3.8** Results are paginated with `Prev / 1 2 3 4 … N / Next` (design shows 8 and 12 page variants). Page size **TBD**; default 12.
- **AC-3.9** Ranking is **deterministic** — see §3. "Top Creators in Demand" (`1:8226`) and "Top Advertisers" (`1:51241`) are rule-derived carousels, not learned recommendations.
- **AC-3.10** **Eligibility gate:** when a creator opens a job post whose requirements they do not meet, the apply action is disabled and replaced with an explicit reason and remediation link — "Can't apply to this job you are close to reach 200K subscribers to meet the criteria" + `See More Followers Details` (`1:54276`). The message must name the specific unmet requirement and the user's current value.
- **AC-3.11** Discovery returns first meaningful paint ≤ 1.5s and a filtered result set ≤ 500ms server time at p95 for a 100k-profile corpus.
- **AC-3.12** **[GAP]** The design shows no empty state, no zero-results state, and no loading skeleton for discovery. All three are required.

---

### 2.5 Epic EP-4 — Profiles & Reputation

**Story.** As a marketplace participant, I want a profile that proves my reach and track record so counterparties can price and trust a deal without leaving the platform.

Screens: `1:58767` Content Creator Profile · `1:11166` Advertiser Profile · `1:9739` Creator drawer · `1:52920` Job post detail

**Creator profile must render** (`1:58767`, `1:9739`):

- Name, verified badge, location, `4.7/5` rating, review count, `100% job succeed`, `Last check in DD/MM/YYYY`
- Per-network stats — `378 M Subscribers`, `66.7 M Followers`, plus per-channel chips (`YouTube 200k`, `Instagram 30K`, `TikTok 2M`)
- `Niche`, `Language(s)`, `Profile` type (`Broker/Organization`)
- Lifetime stats — `2M+ Total Earning`, `40 Total Jobs`, `2000 Total hours`
- **Minimum deal thresholds** — `Digital Deal: Usd 496$ (min)`, `In-Person Deal: Usd 1500$ (min)`
- Long-form bio with `Read More` truncation
- **Intro video** player
- `Location` map
- `Feedback/ Reviews` — four sub-scores (`Work completed 100%`, `Easy communication 98%`, `Professionalism 85%`, `Creator Recommendation 95%`), paginated written reviews with star rating and date, `Share Feedback`
- `Open Profile In New Window` and `Propose an Offer` CTAs

**Advertiser profile must render** (`1:11166`, `1:52920`):

- Legal/brand name, location, rating, review count, `Verified` badge
- `Niche`, `Language(s)`, `Profile Type`, `Payment Method`
- Channel/reach requirements (`YouTube 200k`, `Instagram 10k`, `TikTok unlimited`, `Twitter do not deal`)
- `About Us` with `Read More`, `Location` map
- `Open Job Posts` list with creation date and requirements
- Trust stats — `Job Posted: 38`, `89% Hire Rate`, `Average Rate Paid: $750`, `Member Since 2025`

**Acceptance Criteria**

- **AC-4.1** The four review sub-scores are computed as the mean of per-contract ratings on that dimension, displayed as an integer percentage, and hidden entirely until ≥ 3 completed contracts exist.
- **AC-4.2** `100% job succeed` = completed contracts ÷ funded contracts, over the lifetime of the account.
- **AC-4.3** A channel's stated reach is only displayed if it came from a verified OAuth connection; unverified figures are never rendered.
- **AC-4.4** `Twitter do not deal` is a first-class per-network state meaning *excluded from this deal*, distinct from `unlimited` (any value acceptable) and a numeric threshold.
- **AC-4.5** Reviews are **contract-gated**: only a counterparty on a completed contract may leave one, one review per contract per side.
- **AC-4.6** `Read More` expands in place without navigation.
- **AC-4.7** **[GAP]** No profile-edit screens exist in the file for either role (bio, niche, languages, intro video upload, minimum criteria changes). These require design.

---

### 2.6 Epic EP-5 — Job Posting (Advertiser)

**Story.** As an advertiser, I want a guided job-post wizard so that scope, budget, audience requirements, and deadline are unambiguous before any creator applies.

Two variants share a wizard shell: **Digital** (`1:13582`, 15 frames) and **In-person** (`1:17296`, 8 frames).

| # | Step | Node (digital / in-person) | Behaviour |
| --- | --- | --- | --- |
| 1 | **Type of deal** | `1:13583` / `1:17297` | `To appear in person` — "The Creator Network(s) will need to appear at a location specified by you in order to complete the job." vs `To create digital content` — "…deliver content via a medium you specify…" |
| 2 | **Location** | — / `1:17532` | In-person only. Map picker, "Please select location". |
| 3 | **Title & description** | `1:14012` / — | `Title` + description, **1000-character limit** shown as `Character Limit 1000` |
| 4 | **Creator Networks count** | `1:14421` / — | `One` ("Only a single Creator Network will be associated with this job") \| `Many` ("Two or more… you will need funds for all of them") |
| 5 | **Budget** | `1:14850` / `1:17936` | Digital: `Creators × Per Creator = Total` live calculation. In-person: single `Add Amount`. |
| 6 | **Social media requirements** | `1:15286` / `1:18162` | `Match All` \| `Match Any` toggle; per network (YouTube, TikTok, Instagram, Twitter) a minimum-follower value or `Open to any limit` |
| 7 | **Deadline** | `1:15784` / `1:18460` | Optional — "Do you wanted to set an deadline date" |
| 8 | **Summary** | `1:16205` / `1:18682` | Read-only review: posting profile, title, description, deal type, creator networks, total budget, budget per creator, deadline, location (in-person), social requirements |
| 9 | **Pay to Post** | `1:16823` / `1:19012` | `Payable Amount` with fee breakdown + `Select Method` |

**Acceptance Criteria**

- **AC-5.1** Every step is independently back-navigable; entered values survive back/forward without loss. Drafts persist server-side for 30 days.
- **AC-5.2** `Creators × Per Creator = Total` recalculates on every keystroke and is the authoritative figure carried to payment.
- **AC-5.3** `Match All` requires a creator to satisfy **every** specified network threshold; `Match Any` requires **at least one**. The chosen semantic is stored on the job post and drives AC-3.10 eligibility.
- **AC-5.4** Fee breakdown is itemised before payment — design shows `Base amount: $5,000, Platform fee: $20, Tax (3%): $150` (`1:16823`). **[DEFECT]** $20 is not a plausible flat fee against a $5,000 base, and 3% of $5,000 is $150 while the escrow flow at `1:21556` uses a $12 escrow service fee + $3 processing fee. The commercial fee model must be defined authoritatively before build; the UI must render whatever the pricing service returns rather than hard-coded values.
- **AC-5.5** Tax is computed per the **payer's jurisdiction** at the moment of posting, not a fixed 3%.
- **AC-5.6** Payment method selection lists saved methods with `Primary Method` marked, plus PayPal (`1:16823`).
- **AC-5.7** A published job post is immutable in its budget, deal type, and social requirements; title/description are editable. Any budget change requires closing and re-posting. **[INFERRED]**
- **AC-5.8** **[GAP]** No validation-error states exist in the wizard (empty title, budget below platform minimum, deadline in the past, payment declined). All are required.

---

### 2.7 Epic EP-6 — Proposals, Direct Offers & Interest

**Story (creator).** As a creator, I want to pitch an advertiser with my own terms so I'm not limited to what's publicly posted.
**Story (advertiser).** As an advertiser, I want to see everyone interested in my post and convert one into a contract in a few clicks.

**Creator → Advertiser proposal wizard** (`1:59339`, 22 frames):
Type of deal (`1:59340`) → Location if in-person (`1:62376`) → Proposal text, "Please type here your proposal" (`1:59775`) → Budget (`1:60174`) → Availability (`1:60589`) / Deadline (`1:60819`) → `Review Proposal` summary with posting profile, deal type, total budget, deadline, location (`1:61012`, `1:63618`) → `Congratulations — Your Proposal have been submited` (`1:61288`).

**Advertiser interest management** (`1:22590`, `1:22680`):
Job post header shows `See 4/4 Creators Interested` and `Close Job Post`. The `Interested Creators` list offers per-creator `View Profile`, `Reject`, `I'm Interested`. Accepting produces "Your Contract is all set! You've successfully created a contract. Now you can start chat with the creator in the inbox", followed by a re-funding prompt: "Would you like to keep the job post open for more creators? If you'd like to keep the job open, you'll need to add $3,000 to your escrow for a new contract."

**Pipeline tabs**

| Advertiser (`1:11887`) | Creator (`1:57157`) |
| --- | --- |
| My Job Posts `1:11888` | Received Offers `1:57158` |
| My Direct Offers `1:12284` | My Proposals `1:57472` |
| My Contract `1:12574` | My contracts `1:57618` |
| Proposal `1:12928` | Rejected `1:57803` |
| Rejections `1:13306` | — |

**Acceptance Criteria**

- **AC-6.1** A creator may submit **one active proposal per job post**; re-submission requires withdrawing the prior one.
- **AC-6.2** Proposals carry an explicit state machine: `submitted → (waiting_response | accepted | rejected | withdrawn | expired)`. `My Proposals` renders `Waiting Response…` for `submitted` (`1:57472`).
- **AC-6.3** Every pipeline row shows relative-time provenance — "Initiated Jun 11, 2025 · 3 quarters ago" — and absolute date on hover.
- **AC-6.4** `Rejected` items are retained and listed, never deleted, with the rejecting party and timestamp recorded.
- **AC-6.5** `See N/N Creators Interested` shows *accepted-slots / total-slots* against the `Creator Networks` count from EP-5. When the numerator equals the denominator, no further acceptance is possible without adding escrow funds.
- **AC-6.6** Accepting a creator **atomically** creates a contract and locks the corresponding escrow amount. If escrow is insufficient, the acceptance is refused with the exact shortfall named (design: `$3,000`).
- **AC-6.7** `Close Job Post` transitions the post to `closed`, hides it from discovery, and auto-rejects all pending interest with a notification to each creator.
- **AC-6.8** The creator sees each interested-list card with the counterparty's **local time** ("6:05 PM local time", `1:12574`) resolved from their profile timezone.
- **AC-6.9** Advertiser-initiated **Direct Offers** (`1:12284`) bypass job posts entirely: an advertiser proposes terms to a specific creator from their profile (`Propose an Offer`, `1:9739`), and the creator sees it under `Received Offers`.

---

### 2.8 Epic EP-7 — Contracts, Milestones & Timeline

**Story.** As either party, I want the agreed work broken into funded milestones with a visible timeline so payment and delivery stay in lockstep.

Screens: `1:19267` Digital contract · `1:19537` In-person contract · `1:20227` Contract Details · `1:54867` Creator-side contract · `1:55774` Creator contract details · `1:39243` Ended & Paid

**Contract types**

| | **Digital Content Deal** (`1:19267`) | **Appearance Deal** (`1:19537`) |
| --- | --- | --- |
| Dates | Start date, End date | Start date + time window (`1pm to 4pm`) |
| Duration | — | `Length: 3 hours` |
| Location | — | `Location: Montreal` |
| Structure | `Milestones (N)` with per-milestone deadline | `Appearance Deal Details` |
| Amount | `$5000` `Funded In Escrow` | `$5000` `Funded In Escrow` |

**Acceptance Criteria**

- **AC-7.1** Every contract exposes a human-readable **Contract ID** (design format `20251202-Q4&S`, `1:20227`). **[DEFECT]** `&` in an identifier is URL- and CSV-hostile; specify an alphanumeric+hyphen format.
- **AC-7.2** Contract statuses: `Active Contract`, `Closed`, `Ended & Payed` (`1:39243`) — normalise the last to `Ended & Paid`.
- **AC-7.3** Milestones support `Add New Milestone` post-creation; each carries title, deadline (`Deadline: April 14, 2025`), and amount.
- **AC-7.4** Adding a milestone to a contract requires escrow funding for that milestone **before** it becomes visible to the counterparty.
- **AC-7.5** `Contract Time Line` renders a chronological, append-only ledger: `Contract Created → Funds Added → Video Created → Release Funds` (`1:20569`). Stages are derived from real events, never hard-coded.
- **AC-7.6** `Recent Activity` shows an immutable audit log with actor, action, amount, and date — "Amir's LLC setup a milestone of $1,500.00 to MR Beast", "Amir's LLC released a payment of $1,500.00 to MR Beast for the milestone" (`1:39243`).
- **AC-7.7** Both parties see the identical contract record; only permitted actions differ by role.
- **AC-7.8** Contract detail is downloadable as PDF (`Download PDF`, `1:39243`) including all terms, milestones, and the activity log.
- **AC-7.9** **[GAP]** No screens exist for: milestone submission/delivery by the creator, advertiser approval or rejection of a deliverable, revision requests, contract amendment, early termination, cancellation, refund, or **dispute resolution**. Escrow without a dispute path is not shippable — these are mandatory design work (see §5.2 R-1).

---

### 2.9 Epic EP-8 — Escrow, Payments, Billing & Payouts

**Story.** As an advertiser, I want funds held in escrow and released on delivery so I only pay for work I received. As a creator, I want proof the money exists before I start.

Screens: `1:21556` / `1:21774` Add Funds · `1:16823` Pay to Post · `1:35942` Billing & Payments · `1:37879` Edit Billing · `1:39243` Payment history

**Add Funds panel** (`1:21774`) contains: `Enter Amount`, `Card Details`, `Save this card for future payments`, `Billing Address`, and a `Payment Summary` — `Deposit Amount $5,000` + `Escrow Service Fee $12` + `Processing Fee $3` = `Total Payment $5,015 USD` — with the assurances "Secured by AES-256 encryption. Your funds are held safely in escrow until delivery." and "Need help? Chat with support or visit our Help Center."

**Acceptance Criteria**

- **AC-8.1** Funds are held by a regulated escrow/payment provider. CoWink **never stores raw PAN**; card entry is via a PCI-DSS SAQ-A compliant hosted field or PSP SDK. Provider selection **TBD in TRD**.
- **AC-8.2** The payment summary itemises every component (deposit, escrow service fee, processing fee, tax) and the total, all returned by a pricing service — never computed client-side.
- **AC-8.3** **[DEFECT]** "Secured by AES-256 encryption" is a claim about data at rest, not a payment-security guarantee. Replace with accurate copy naming the PSP and PCI compliance level; legal review required.
- **AC-8.4** Supported billing methods: bank cards (`Visa, Mastercard, AmericanExpress, Discover, diners`) and `Paypal` (`1:37879`). Exactly one method is `Primary Method`; `Make Primary Payment method` reassigns it (`1:38547`).
- **AC-8.5** Escrow release is per-milestone, initiated by the advertiser, and produces both a ledger entry and a notification.
- **AC-8.6** Payment history is searchable and sortable, and each row shows: contract title, `Contract Type`, `Amount`, `Method`, `Track` (`Debited` / `Credited`), the account involved (`Sprite`, `Fanta`), date-time, and row `Actions` (`1:39243`).
- **AC-8.7** Invoices are downloadable as PDF and carry the tax-residence address from EP-13 ("This address will be displayed on invoices").
- **AC-8.8** The creator dashboard surfaces `Active Contracts`, `Total Earning Till Date`, `Pending Payment`, `Receivables` (`1:57618`); each figure links to its underlying list.
- **AC-8.9** **Withdrawals are blocked** until the required tax form is on file (EP-13) — "Before withdrawing funds, all non-U.S. persons must provide their W-8BEN tax information".
- **AC-8.10** All monetary values are stored as minor units in a fixed-point type with an explicit currency code. No floating-point arithmetic anywhere in the money path.
- **AC-8.11** **[GAP — GLOBAL LAUNCH]** The design is **USD-only** (`Total Payment $5,015 USD`, `Usd 496$ (min)`, `$` throughout). A global launch requires multi-currency display, FX handling on cross-currency deals, per-market payout rails, and locale-correct number/currency formatting. None of this is designed. See §5.2 R-2.
- **AC-8.12** **[GAP]** No payout/withdrawal screens exist for creators — no bank/PayPal payout method setup, no withdrawal request, no payout history. Required.
- **AC-8.13** **[DEFECT]** `Funded In Escro` (`1:19267`, `1:20227`) and `Ended & Payed` (`1:39243`) are misspelled; `$5,00` appears where `$5,000` is meant across the wizard summaries. Copy pass required before build.

---

### 2.10 Epic EP-9 — Messaging

**Story.** As a party to a deal, I want conversation and contract state side by side so I never have to ask "where are we?".

Screens: `1:20569` Messages inbox · `1:19832` In-contract messages · `1:55573` Creator-side contract messages

**Acceptance Criteria**

- **AC-9.1** Messaging exists in two entry points backed by one thread store: a global inbox with a searchable conversation list, and an in-contract tab.
- **AC-9.2** Threads are **contract- or job-scoped**; the conversation header names the job ("Food Vlogger With 10M+ Followers") and the counterparty.
- **AC-9.3** The composer is `Type something…` + `Send`; messages show a timestamp (`12:45 PM`).
- **AC-9.4** **System messages** are rendered inline as structured cards, not plain text — e.g. "Advertiser created the contract" followed by `Contract 1: "1 video" Due: April 14, 2025 Amount: $300` with a `View details` action deep-linking to the contract.
- **AC-9.5** A `Contract Timeline` rail sits beside the thread showing live contract state (EP-7 AC-7.5).
- **AC-9.6** Messages deliver in ≤ 2s p95 and the inbox reflects new messages without a manual refresh.
- **AC-9.7** Unread counts drive the notification badge (EP-11 `Increment Message Counter for:`).
- **AC-9.8** **[GAP]** No design for file/media attachments — which a content-delivery workflow requires — nor for typing indicators, read receipts, blocking, or reporting a user. Attachment support and abuse-reporting are mandatory.

---

### 2.11 Epic EP-10 — Reviews & Feedback

**Story.** As a participant, I want structured post-contract feedback so reputation reflects real outcomes.

**Acceptance Criteria**

- **AC-10.1** Feedback captures four dimensions — Work completed, Easy communication, Professionalism, Creator Recommendation — plus a star rating and free text (`1:58767`).
- **AC-10.2** `Share Feedback` is available on each review for sharing outward. **[INFERRED]** — the design shows the control but no target.
- **AC-10.3** Reviews are permitted only after a contract reaches a terminal completed state, within 30 days of completion.
- **AC-10.4** Reviews are immutable once published; the reviewed party may post a one-time public response. **[INFERRED]**
- **AC-10.5** Reviews are paginated on the profile and sorted most-recent-first by default.

---

### 2.12 Epic EP-11 — Account Settings

Screens: `1:33778` Contact Information · `1:34436` Notification Setting · `1:34665` Password & Security

**Acceptance Criteria**

- **AC-11.1** **Contact Information** exposes, each individually editable via its own `Edit` control: `User ID` (immutable), `Name`, `Email Address`, `Emergency Contact`, `Account Status` (`Live`), `Time Zone`, `Language`, `Address`, `Phone`.
- **AC-11.2** Changing email or phone requires re-verification of the new value before it takes effect.
- **AC-11.3** `Close Account` presents a confirmation — "You are about to close your account! Do you want to Close your account?" — and is **refused** while any contract is active or any escrow balance is non-zero. **[INFERRED — mandatory]**
- **AC-11.4** **Notification settings** are a 3×N matrix over **Desktop**, **Mobile**, and **Email** channels, with `Show Notification: All Activity`, `Increment Message Counter for:`, and `Play Sound: Enabled` (`1:34436`).
- **AC-11.5** **Password & Security** supports password change and two-step verification with **two independent methods**: a security question (`What is the name of the city you was born in?` / editable via `Edit Question`) and SMS (`Text Messages — Ending with 4578`, `Change Number`).
- **AC-11.6** The SMS OTP screen (`1:35061`) uses a 4-digit code with a visible expiry countdown (`00:23`) and `Resend OTP`, rate-limited to 3 sends / 15 minutes.
- **AC-11.7** **[DEFECT]** A security question is not an acceptable second factor for an account controlling escrow funds — it is knowledge-based, guessable, and phishable. TOTP or WebAuthn must be added; the security question may remain only as an account-recovery aid. Design work required.
- **AC-11.8** The user's timezone drives every "local time" display across the product (EP-6 AC-6.8).

---

### 2.13 Epic EP-12 — Organizations, Accounts & Roles

**Story.** As an organization manager, I want multiple brand accounts under one legal entity with scoped team access so my agency or enterprise can operate without sharing logins.

Screens: `1:35136` Organization · `1:35270` Users · `1:36292` Add User · `1:36923` Select accounts · `1:11077` Account switcher

**Hierarchy:** `Legal Entity` (`Coca Cola.INC`) → `Accounts` (`Coke`, `Fanta`, `Sprite`) → `Users` with roles. A user may belong to multiple organizations (`Add another organization`, `Create New Organization`).

**Acceptance Criteria**

- **AC-12.1** The header dropdown (`1:11077`) shows the signed-in user, their email, the list of accessible `Account`s, the `Legal Entity`, plus `Add another organization`, `Profile Settings`, and `Sign Out`. Switching account re-scopes the entire session — job posts, contracts, billing, and messages all reflect the active account only.
- **AC-12.2** Roles are `Full Control`, `Manager`, and `Accountant` (`1:35270`). Permissions:

  | Capability | Full Control | Manager | Accountant |
  | --- | --- | --- | --- |
  | Post jobs / accept creators | ✔ | ✔ | ✘ |
  | Create & release milestones | ✔ | ✔ | ✘ |
  | Message creators | ✔ | ✔ | ✘ |
  | View billing & payment history | ✔ | ✔ | ✔ |
  | Manage billing methods | ✔ | ✘ | ✔ |
  | Submit tax forms | ✔ | ✘ | ✔ |
  | Invite users / assign roles | ✔ | ✘ | ✘ |
  | Create organizations & accounts | ✔ | ✘ | ✘ |

  Derived from the accountant view at `1:40283` (billing-only) and the invite flow. **This table is [INFERRED] and requires product sign-off** — the design never states the permission matrix.
- **AC-12.3** `Add User` (`1:36292`) collects `Invite Email` and `Assign Role`, then requires account scoping — "Please select accounts" (`1:36923`). A user's access is per-account, not entity-wide.
- **AC-12.4** Access may be **time-bounded**: the Users table carries `Access Time — June 19, 2026 to Aug 20, 2026`. Access must auto-revoke at the end of the window, with a notification to the manager 7 days prior.
- **AC-12.5** The Users table shows `Name`, `Role`, `Account Access`, `Access Time`, `Joining Date`, `Actions` (edit role, revoke, resend invite), and is searchable and sortable.
- **AC-12.6** Every permission is enforced **server-side**; hiding UI is never the control.
- **AC-12.7** **[GAP]** No design exists for the invite-acceptance experience, pending-invite state, seat limits, or the last-Full-Control-user-cannot-leave guard.

---

### 2.14 Epic EP-13 — Tax & Regulatory Compliance

**Story.** As the platform, I must collect the correct tax documentation per jurisdiction before disbursing funds, so payouts are lawful and reportable.

Screens: `1:42440` W-9 / W-8BEN · `1:41054` Canadian flow · `1:40929` Accountant tax editing

**Flow.** `Tax information required` — "We need some basic tax details to process payments securely and comply with applicable regulations" (`1:40929`) → entity type: `Individual` ("freelancer, contractor, or personal account") vs `Organization / Company` (`1:41418`) → jurisdiction-specific form.

**Forms in the design**

| Form | Applies to | Fields (`1:42632`, `1:43110`, `1:42025`) |
| --- | --- | --- |
| **W-9** | US persons | Name as on tax return, business/disregarded-entity name, Federal Tax Classification, exempt payee code, FATCA exemption code, street/apt/city/ZIP, account numbers, TIN, EIN, perjury certification (4 statements), typed signature, date |
| **W-8BEN** | Non-US individuals | Beneficial owner name, country of citizenship, permanent residence (no P.O. box), mailing address, US TIN (SSN/ITIN), foreign TIN, reference numbers, date of birth, optional tax-treaty claim, certification, typed signature, capacity in which acting |
| **W-8BEN-E** | Non-US entities | Referenced at `1:42851`; **form body not designed** |
| **Canadian** | CA registrants | Account type, company name, `Country of Registration: Canada`, `Business Number`, `Upload Legal Document` (PDF) |

Also captured: `Tax Residence` address ("This address will be displayed on invoices"), `Legal Name of Taxpayer`, `Federal Tax Classification` — each with its own `Edit` control (`1:41306`).

**Acceptance Criteria**

- **AC-13.1** Tax status is determined by the **entity type × tax residence** pair and drives which form is requested. The user is never asked to choose their own form.
- **AC-13.2** Withdrawal is hard-blocked until a valid, unexpired form is on file (see AC-8.9). The block message names the specific missing form.
- **AC-13.3** Typed-signature certification records the typed name, timestamp, and source IP, and is retained as an immutable record.
- **AC-13.4** W-8BEN validity expires on 31 December of the third year after signing; the platform must prompt for renewal 60 days before expiry. **[INFERRED — IRS rule, not in design]**
- **AC-13.5** Uploaded legal documents accept PDF, are size-capped, virus-scanned, and stored encrypted with access confined to compliance staff.
- **AC-13.6** TIN/EIN/SSN values are encrypted at rest with a dedicated key, masked in all UI after entry, and excluded from every log and analytics event.
- **AC-13.7** The `Accountant` role can complete tax forms on the organization's behalf (`1:40283`).
- **AC-13.8** **[GAP — GLOBAL LAUNCH]** The design covers only **US and Canadian** tax. A global launch additionally requires: EU/UK VAT registration and reverse-charge handling, VAT number validation, GST/HST/JCT and equivalents, DAC7 / OECD digital-platform seller reporting, local withholding regimes, and **KYC/AML identity verification with sanctions screening** before payout. None of this exists in the design. This is the single largest gap between the design and the stated global-launch goal — see §5.2 R-2.

---

### 2.15 Epic EP-14 — Internationalization & Localization

**Acceptance Criteria**

- **AC-14.1** A language selector is present in the global footer (`English`) on every screen.
- **AC-14.2** All user-facing strings are externalised; no string is hard-coded in a component.
- **AC-14.3** Dates, times, numbers, and currency render per the user's locale; contract dates additionally show timezone.
- **AC-14.4** Layouts tolerate a 40% string-length increase without truncation or overflow.
- **AC-14.5** **[GAP]** RTL support, the actual set of launch languages, and content translation policy (are bios and job descriptions translated?) are undefined.

---

### 2.16 Cross-Cutting Requirements

- **AC-X.1 Navigation.** Persistent header: `Explore` (spelled `Exlopre` in the design — **[DEFECT]**, fix), `My offers & Proposals`, `Messages`, global `Search`, account dropdown, and `Switch To Advertiser` role toggle. Persistent footer: `Privacy`, `Terms`, `About us`, `Tutorial`, `Contact`, language selector.
- **AC-X.2 Accessibility.** WCAG 2.2 Level AA: ≥ 4.5:1 text contrast, full keyboard operability including the map and every wizard, visible focus, correct labelling of all form fields, and screen-reader announcement of wizard step changes. The design carries no accessibility annotations — **[GAP]**, an audit is required.
- **AC-X.3 Responsive.** **[GAP]** Every frame in the file is **1440px desktop**. There are zero tablet or mobile designs, yet the product references `Mobile Notifications` (`1:34436`). Either responsive breakpoints must be designed, or mobile must be an explicit non-goal for v1 (see §2.17).
- **AC-X.4 Performance.** p95 page interaction < 300ms; p95 API response < 500ms; discovery search < 1.5s to first meaningful paint.
- **AC-X.5 Error handling.** Every network-dependent action defines a loading, empty, and error state. The design supplies almost none — **[GAP]**.
- **AC-X.6 Audit.** Every money movement, permission change, and contract state transition is written to an append-only audit log with actor, timestamp, before/after, and request ID.

---

### 2.17 Non-Goals (v1)

Explicitly **out of scope** for the first release:

1. **AI/ML matching or recommendation.** Discovery and "Top Creators in Demand" are rules-based only (§3). No embeddings, no learned ranking, no LLM features.
2. **Native mobile apps.** Web only. (Mobile *web* responsiveness is an open question — see AC-X.3.)
3. **Automated content delivery/verification.** The platform does not fetch, scrape, or verify that a creator actually published the agreed content; milestone approval is a human decision by the advertiser.
4. **Campaign performance analytics.** No impressions, engagement, click, or ROI reporting on delivered content.
5. **In-platform content production tools.** No brief builder, asset library, or creative review beyond messaging attachments.
6. **Public API / partner integrations** beyond the social-channel OAuth connections required for verification.
7. **Bulk / programmatic hiring.** No CSV import, no bulk offer sending.
8. **Direct creator-to-creator or advertiser-to-advertiser interaction.**
9. **Crypto or alternative payment rails.** Cards and PayPal only (`1:37879`).

---

## 3. Matching & Ranking Requirements (Non-AI)

Per product decision, **no AI or ML is used anywhere in this PRD**. This section replaces the AI System Requirements section and specifies the deterministic logic that must produce the design's ranked surfaces.

### 3.1 Eligibility (hard filter)

A creator is eligible for a job post when **all** of the following hold:

1. **Reach.** For `Match All`, the creator meets or exceeds the minimum follower count on *every* network specified. For `Match Any`, on *at least one*. Networks marked `Open to any limit` always pass. Networks marked `do not deal` must have **no** connected account for that network. (`1:15286`)
2. **Price floor.** The job's per-creator budget ≥ the creator's stored minimum for that deal type — digital or appearance (`1:56162`).
3. **Geography.** For in-person deals, the job location falls within the creator's declared influence locations.
4. **Verification.** Every network used to establish eligibility is OAuth-verified, and its data is fresher than the `Check-In` staleness threshold.

Failing eligibility produces the explicit, reason-bearing block state of AC-3.10 — never a silent hide.

### 3.2 Ranking (ordering of eligible results)

Ranked by a transparent weighted score over normalised inputs. Weights are **configuration, not code**, and are tunable without deploy:

| Signal | Source | Default weight |
| --- | --- | --- |
| Rating (0–5) | Review average | 0.25 |
| Job success rate | Completed ÷ funded contracts | 0.20 |
| Reach fit — closeness to required reach without excessive overshoot | Channel stats | 0.20 |
| Recency of activity | `Last check in`, last contract date | 0.15 |
| Geographic proximity | Distance job ↔ creator | 0.10 |
| Completed contract volume | Lifetime jobs | 0.10 |

- **"Top Creators in Demand" / "Top Advertisers"** = the top N by this score, filtered to accounts active in the last 30 days.
- Ties break by most-recent activity, then by account age.
- The ranking must be **reproducible**: given the same inputs and weight configuration, the same order results.

### 3.3 Validation Strategy

- **Golden-set tests.** A fixture of ≥ 200 creator profiles and ≥ 50 job posts with hand-labelled expected eligibility. Eligibility logic must be **100% correct** on this set — it is deterministic, so anything less is a bug.
- **Ranking regression tests.** Snapshot the ordering for a fixed corpus; any weight change surfaces as an explicit, reviewed diff.
- **No new creator starves.** A creator with zero contracts must still appear in eligible result sets; verify the score floor does not push new accounts below page 3 for a query they fully satisfy.
- **Boundary tests.** Reach exactly equal to the threshold, budget exactly equal to the minimum, `do not deal` with a connected account, `Match Any` with one qualifying network, and stale check-in data.

---

## 4. Technical Specifications

> **Stack is TBD by decision.** This section specifies *what* must exist and *what properties* it must have. Concrete technology selection — languages, frameworks, datastores, PSP, cloud — belongs in the TRD.

### 4.1 Architecture Overview

Logical services (deployment topology TBD):

| Service | Responsibility |
| --- | --- |
| **Identity & Access** | Accounts, sessions, MFA, organizations, accounts, roles, per-account authorization |
| **Profile & Channels** | Profiles, bios, media, OAuth channel connections, scheduled reach refresh, verification state |
| **Discovery** | Search index, geo queries, filters, deterministic eligibility + ranking (§3) |
| **Jobs & Proposals** | Job posts, proposal/offer/interest state machines, eligibility gate at apply time |
| **Contracts** | Contracts, milestones, timeline events, audit log, PDF generation |
| **Payments & Escrow** | Ledger, escrow hold/release, PSP integration, fees, invoices, payouts |
| **Tax & Compliance** | Form collection/validation, expiry, document storage, withdrawal gating, reporting exports |
| **Messaging** | Threads, real-time delivery, system-message cards, unread counts |
| **Notifications** | Desktop/mobile/email fan-out honouring per-channel preferences |

**Core data flow — job post to payout**

```
Advertiser posts job
  → eligibility criteria stored on job_post
  → Discovery indexes post; creators filtered by §3.1
  → Creator expresses interest / sends proposal
  → Advertiser accepts
      → [atomic] contract created + escrow amount locked   (fails if funds short)
  → Milestone funded → creator delivers → advertiser releases
      → [atomic] ledger debit escrow / credit creator balance + timeline event + notification
  → Creator withdraws
      → BLOCKED unless valid tax form + KYC on file
      → payout via PSP → payout record + invoice
```

**Invariants (non-negotiable):**

- **INV-1** Contract creation and escrow lock are one atomic transaction. A contract can never exist unfunded.
- **INV-2** The money ledger is append-only and double-entry. Corrections are compensating entries, never mutations.
- **INV-3** Sum of all escrow holds equals the escrow account balance at the PSP, reconciled at least daily with an alert on any drift.
- **INV-4** Every state transition on job post, proposal, contract, and milestone is an explicit, validated edge in a documented state machine. No implicit status writes.
- **INV-5** All money-moving endpoints are idempotent under a client-supplied idempotency key.

### 4.2 Integration Points

| Integration | Purpose | Notes |
| --- | --- | --- |
| **YouTube Data API** | Channel ownership verification + subscriber count | OAuth; quota and rate-limit budget required |
| **TikTok API** | Channel verification + follower count | Approval process; regional availability varies |
| **Instagram Graph API** | Account verification + follower count | Requires Business/Creator account linkage |
| **X (Twitter) API** | Account verification + follower count | Paid tiers; cost must be modelled |
| **Payment Service Provider** | Card + PayPal capture, escrow-style holds, split payouts, multi-currency | **Selection TBD.** Must support marketplace payouts in every launch market |
| **Tax/compliance vendor** | W-9 / W-8BEN / W-8BEN-E collection & validation, VAT ID checks, 1099 / DAC7 reporting | **Selection TBD.** Building this in-house is not recommended |
| **KYC/AML vendor** | Identity verification + sanctions/PEP screening pre-payout | **[GAP]** Not in design; mandatory for global |
| **Maps & geocoding** | Discovery map, location pickers, distance calculations | Licence terms must permit commercial marketplace use |
| **Email delivery** | Verification, notifications, invoices | Deliverability/DMARC setup required |
| **SMS delivery** | 2FA OTP (`1:35061`) | Global coverage + per-country cost |
| **Object storage + CDN** | Avatars, intro videos, message attachments, tax documents | Tax documents on a separately-permissioned encrypted bucket |
| **Real-time transport** | Messaging and live contract-timeline updates | Mechanism TBD |

### 4.3 Security & Privacy

- **S-1 Authentication.** Password + mandatory 2FA for any account with escrow authority. TOTP/WebAuthn required (AC-11.7); SMS acceptable as a secondary; security question is recovery-only.
- **S-2 Authorization.** Per-account, per-role checks enforced server-side on every request (AC-12.6). Deny by default.
- **S-3 Payment data.** No raw card data ever touches CoWink infrastructure. PSP-hosted fields only; PCI-DSS SAQ-A scope.
- **S-4 Sensitive data classes.** TIN/SSN/EIN, tax documents, and government IDs are encrypted at rest under dedicated keys, masked in UI, access-logged, and excluded from logs, analytics, error reports, and non-production environments.
- **S-5 Transport.** TLS 1.3 everywhere; HSTS; secure, httpOnly, SameSite cookies.
- **S-6 Privacy regime.** GDPR/UK-GDPR (global launch makes this unavoidable), CCPA/CPRA, PIPEDA. Requires: lawful-basis mapping, data-subject access/erasure/portability workflows, consent records, DPAs with every subprocessor, cross-border transfer mechanism, breach notification within 72 hours, and a published retention schedule.
- **S-7 Retention.** Tax records retained per statutory minimum (typically 7 years) and **survive account closure** — closure anonymises the profile but preserves the financial and compliance record.
- **S-8 Content safety.** Reporting and moderation for profiles, job posts, proposals, and messages. **[GAP]** — no moderation UI exists anywhere in the design.
- **S-9 Fraud controls.** Velocity limits on account creation, job posting, and withdrawals; detection of collusive advertiser↔creator pairs cycling escrow to fabricate reputation.
- **S-10 Secrets & keys.** Managed secret store, documented rotation, no credentials in source.

---

## 5. Risks & Roadmap

### 5.1 Phased Rollout

**Phase 0 — Design & Definition (pre-build, blocking)**

Nothing below can be built safely until these are resolved:

1. Design the **dispute, cancellation, and refund** flows (AC-7.9).
2. Design **milestone delivery, approval, and revision** (AC-7.9).
3. Design **creator payout/withdrawal** screens (AC-8.12).
4. Define the **authoritative fee and tax model** and remove the contradictory hard-coded figures (AC-5.4).
5. Decide **mobile/responsive** scope (AC-X.3).
6. Sign off the **role permission matrix** (AC-12.2).
7. Full **copy pass** on all defects flagged in this document.
8. Define **launch markets** — this determines the entire compliance surface.

**Phase 1 — MVP**

Auth (EP-1) · Onboarding with channel verification (EP-2) · Discovery with map, filters, deterministic eligibility (EP-3) · Profiles & reputation display (EP-4) · Job posting, both deal types (EP-5) · Interest & acceptance (EP-6, advertiser-initiated only) · Contracts with milestones and timeline (EP-7) · Escrow fund/release **single currency, single launch market** (EP-8) · Messaging with system cards (EP-9) · Reviews (EP-10) · Account settings with real 2FA (EP-11) · **Dispute resolution** and **payout with tax gating** for the launch market only.

**Phase 2 — v1.1**

Creator-initiated proposals (EP-6 full) · Organizations, multi-account, roles, invitations (EP-12) · Full billing management and payment history (EP-8 AC-8.4–8.7) · Notification preferences across all three channels (EP-11 AC-11.4) · Accountant role (P-6) · Message attachments (EP-9 AC-9.8).

**Phase 3 — v2.0 (Global)**

Multi-currency and FX · Additional payout rails per market · Full tax matrix: VAT/GST/JCT, DAC7, local withholding (EP-13 AC-13.8) · KYC/AML with sanctions screening · Localization beyond English (EP-14) · Moderation and trust-and-safety tooling (S-8) · Responsive/mobile if not already in Phase 1.

> **Recommendation.** "Global from day one" is stated as the launch goal, but the design supports one currency and two tax jurisdictions. Building the *platform* to be currency- and jurisdiction-agnostic from the first line of code is correct and is assumed throughout §4; **launching** payouts in every market simultaneously is not — each new market adds tax, KYC, and payout-rail work that is largely independent of product code. Recommend architecting globally, launching payouts market-by-market, and treating Phase 3 as a sequence of market activations rather than one milestone.

### 5.2 Risks

| ID | Risk | Impact | Likelihood | Mitigation |
| --- | --- | --- | --- | --- |
| **R-1** | **Escrow with no dispute path.** The design funds, holds, and releases money but never handles disagreement. First contested delivery becomes a manual support crisis and a legal exposure. | Critical | Certain | Blocking Phase 0 item. Design dispute intake, evidence submission, resolution SLA, and partial-release before any escrow code ships. |
| **R-2** | **Global launch vs. US/CA-only compliance design.** Payout in an undesigned jurisdiction risks unlawful disbursement, unreported income, and sanctions violations. | Critical | High | Launch payouts market-by-market. Buy compliance (vendor) rather than build. Gate every payout on jurisdiction-specific checks that fail closed. |
| **R-3** | **Social platform API dependency.** Verification and reach data depend on four third-party APIs with changing terms, quotas, pricing, and approval regimes. Any one revoking access breaks eligibility. | High | High | Cache last-known-good values with the `Check-In` staleness stamp; degrade to "unverified" rather than failing; abstract behind one provider interface; monitor quota. |
| **R-4** | **Money-path correctness.** Multi-party escrow with milestones, fees, taxes, and FX is where marketplaces lose real money to rounding, race conditions, and double-release. | Critical | Medium | INV-1..INV-5; double-entry ledger; integer minor units; idempotency keys; daily PSP reconciliation with alerting; property-based tests on the ledger. |
| **R-5** | **Cold-start liquidity.** A two-sided marketplace with no creators is worthless to advertisers and vice versa. Ranking that favours completed contracts (§3.2) compounds this against new users. | High | High | Seed one side manually pre-launch; verify the new-creator-starvation test in §3.3; consider a launch-period ranking boost for new verified accounts. |
| **R-6** | **Off-platform disintermediation.** Once introduced, parties can transact directly and avoid fees — the structural failure mode of every marketplace. | High | High | Make escrow, dispute protection, and reputation genuinely valuable; keep contact-detail exchange inside messaging; monitor for contact-info leakage patterns. |
| **R-7** | **Fee model undefined and internally contradictory.** Three different fee structures appear across the design (AC-5.4, AC-8.2). | High | Certain | Blocking Phase 0 item. Single pricing service; UI renders server-returned values only. |
| **R-8** | **No mobile design despite a mobile-referencing product.** Creators skew mobile; a desktop-only marketplace loses supply. | High | Medium | Decide scope in Phase 0. If mobile is deferred, state it as an explicit non-goal and design for it in Phase 3. |
| **R-9** | **Weak second factor on money-controlling accounts.** Security question as 2FA (AC-11.7). | High | Medium | Ship TOTP/WebAuthn in Phase 1; demote the security question to recovery. |
| **R-10** | **Volume of undesigned states.** Empty, loading, error, and validation states are absent nearly everywhere (AC-3.12, AC-5.8, AC-X.5). Engineering will invent them inconsistently. | Medium | Certain | Define a shared state-pattern library in Phase 0; treat it as a design-system deliverable, not per-screen work. |
| **R-11** | **Copy quality.** `Exlopre`, `Funded In Escro`, `Ended & Payed`, `$5,00`, `Minimum Retirement to Communicate`, inconsistent step counts. | Medium | Certain | Full copy pass + a string-freeze checkpoint before i18n extraction (EP-14). |

---

## Appendix A — Screen Inventory (Figma → Requirement)

| Area | Section node | Frames | Epic |
| --- | --- | --- | --- |
| Advertiser — discovery & profile | `1:8225` | 7 | EP-3, EP-4 |
| Account/organization switcher | `1:11076` | 2 | EP-12 |
| Advertiser profile | `1:11165` | 2 | EP-4 |
| Job offers pipeline | `1:11887` | 10 | EP-6 |
| Post a Job — digital | `1:13582` | 15 | EP-5 |
| Post a Job — in-person | `1:17296` | 8 | EP-5 |
| Contract details (advertiser) | `1:19266` | 8 | EP-7, EP-9 |
| Messages (advertiser) | `1:20568` | 3 | EP-9 |
| Onboarding (advertiser) | `1:20991` | 15 | EP-2 |
| Login / Sign up (advertiser) | `1:21969` | 10 | EP-1 |
| Interested people | `1:22589` | 2 | EP-6 |
| Contact information | `1:33777` | 6 | EP-11 |
| Notification settings | `1:34435` | 2 | EP-11 |
| Password & security | `1:34664` | 6 | EP-11 |
| Organization settings | `1:35135` | 16 | EP-12, EP-8 |
| Accountant role | `1:40282` | 4 | EP-12, EP-13 |
| Canadian tax flow | `1:41054` | 14 | EP-13 |
| W-9 / W-8BEN | `1:42440` | 6 | EP-13 |
| Creator — discovery & profile | `1:51240` | 9 | EP-3, EP-4 |
| Contract details (creator) | `1:54866` | 6 | EP-7 |
| Onboarding (creator) | `1:55932` | 13 | EP-2 |
| Login / Sign up (creator) | `1:56539` | 10 | EP-1 |
| Offers & proposals (creator) | `1:57157` | 9 | EP-6 |
| Messages (creator) | `1:58345` | 3 | EP-9 |
| Creator profile | `1:58766` | 2 | EP-4 |
| Send proposals | `1:59339` | 22 | EP-6 |

**Total: 240 frames across 26 sections.**

---

## Appendix B — Open Questions

| # | Question | Owner | Blocks |
| --- | --- | --- | --- |
| Q-1 | What is the authoritative fee model — platform fee, escrow fee, processing fee, who pays each? | Product / Finance | EP-5, EP-8 |
| Q-2 | Which markets are in scope for payout at launch, and in what order thereafter? | Product / Legal | EP-13, Phase 3 |
| Q-3 | Is mobile web in scope for v1, or an explicit non-goal? | Product / Design | AC-X.3 |
| Q-4 | How are disputes resolved — platform arbitration, third-party, or advertiser-favoured default? | Product / Legal | R-1 |
| Q-5 | Is the permission matrix in AC-12.2 correct? | Product | EP-12 |
| Q-6 | What is the exact `Check-In` staleness threshold that marks channel data stale? | Product | §3.1 |
| Q-7 | Does "Creator Network" mean a broker representing several creators, or simply a hired creator slot? The design uses it both ways. | Product | EP-5, EP-6 |
| Q-8 | Can an advertiser hire the same creator on multiple slots of one `Many`-slot job post? | Product | AC-6.5 |
| Q-9 | What is `Share Feedback` sharing to — social, a link, or an internal action? | Product / Design | AC-10.2 |
| Q-10 | Is the intro video uploaded to CoWink or embedded from a connected channel? | Product / Design | EP-4 |
