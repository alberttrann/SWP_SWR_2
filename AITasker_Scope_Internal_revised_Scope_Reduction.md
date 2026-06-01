# AITasker MVP — Internal Scope Document
### SWP391 · Topic 7 · FPT University HCMC · 9-Week Build Plan

> **Purpose:** Internal reference document for the team. Not for submission as-is.
> **Last updated:** June 2026
> **Status:** Scope-reduced MVP — 28 tables, 9-week execution window
> **Scope reduction from:** 51-table full architecture (see reduction rationale in team Slack)

---

## Table of Contents

0. [Master Reference Sheet](#0-master-reference-sheet)
1. [System Overview](#1-system-overview)
2. [Actors & Roles](#2-actors--roles)
3. [Deliverable Features](#3-deliverable-features)
4. [Use Cases](#4-use-cases)
5. [Architecture & Tech Stack](#5-architecture--tech-stack)
6. [WBS & Milestones](#6-wbs--milestones)

---

## 0. Master Reference Sheet

> **Ground-truth lookup for all taxonomy codes, seam identifiers, scoring weights, and state machine definitions.** When any other section uses a code (e.g. A↔C, Archetype 2, Tier 2), refer here. Do not paraphrase.

---

### 0.1 The 6 Capability Domains

| Code | Name | What it covers |
|---|---|---|
| **A** | LLM Application Engineering | System prompt design, structured output, RAG orchestration, chain-of-thought |
| **B** | MLOps / LLMOps Infrastructure | Model serving, cost optimization, drift monitoring, deployment pipelines |
| **C** | AI Evaluation & Quality Systems | Metric design, ground truth creation, HITL workflow, benchmarking |
| **D** | Vector DB & Embeddings | HNSW indexing, chunking strategy, MRL truncation, similarity tuning |
| **E** | Data & Pipeline Engineering | Kafka, async processing, distributed locks, legacy ETL, high-volume ingestion |
| **F** | ML Modeling & Fine-Tuning | Cross-encoders, supervised fine-tuning, imbalanced data, feature engineering |

---

### 0.2 The 10 Seams

Seams are cross-domain competence boundaries. An expert who holds both adjacent domains may still lack the seam knowledge — the skill of diagnosing and resolving failures *at* the boundary.

| Seam | Name | What failure looks like without it |
|---|---|---|
| **A↔C** | Ground truth-driven iteration | Expert fine-tunes prompts with no baseline — can't tell if changes help or hurt |
| **A↔F** | Hybrid routing | LLM handles everything including cases it should defer to a classifier — high cost, low accuracy |
| **A↔D** | Retrieval-generation contract | Retrieved chunks are semantically right but structurally wrong — LLM produces hallucinated synthesis |
| **D↔E** | Distributed vector upsert safety | Two pipeline workers write the same embedding simultaneously — vector index corruption |
| **D↔F** | Embedding/index co-design | Fine-tuned model changes embedding space but index was built with base model — retrieval degrades silently |
| **C↔F** | Fine-tuning gating | Team fine-tunes before establishing evaluation baseline — no way to know if fine-tuning helped |
| **E↔F** | Training data as pipeline problem | ML model trained on clean data, deployed against dirty pipeline output — production accuracy collapse |
| **A↔B** | Prompt iteration under production constraints | Prompts optimized in playground, never tested under real latency/cost constraints — works in dev, fails in prod |
| **B↔E** | Cost management in streaming pipelines | Every Kafka message triggers an LLM call — cost explodes under load |
| **C↔E** | Evaluation data as pipeline concern | Ground truth dataset exists but isn't refreshed when pipeline schema changes — evaluation becomes stale |

---

### 0.3 The 6 Archetypes + 3 Tiers

**Archetypes** — what kind of AI project this is:

| # | Archetype | Core question | Primary domains | Load-bearing seam |
|---|---|---|---|---|
| **1** | Automated Decision System | *"Can AI replace a human decision my team makes repeatedly?"* | C, E, A | **A↔C** |
| **2** | RAG / Knowledge Retrieval | *"Can AI answer questions from our documents?"* | A, D, B | **A↔D** |
| **3** | Predictive / Forecasting | *"Can AI predict an outcome from our historical data?"* | F, C, E | **E↔F** |
| **4** | Data Pipeline + AI Integration | *"Can we connect AI into our high-volume data infrastructure?"* | E, B, D | **D↔E** |
| **5** | Model Fine-Tuning & Customization | *"Can we make a base model specialize on our domain?"* | F, A, C | **A↔F** |
| **6** | Evaluation Infrastructure Build | *"How do we know if our AI is actually working?"* | C, E, F | **A↔C** |

**Tiers** — how complex the project is:

| Tier | Label | Infrastructure signals |
|---|---|---|
| **1** | Simple / Greenfield | HTTP REST only, <100K records, no legacy |
| **2** | Moderate | One integration point, 100K–1M records, some legacy |
| **3** | Complex / Production | Kafka/async, >1M records, legacy migration, multi-system |

---

### 0.4 Expert Verification Tiers (MVP: 2-Tier System)

> **Scope reduction:** Tiers 3 (Scenario-verified) and 4 (Platform-demonstrated) are deferred to Phase 2. The MVP implements Tier 1 and Tier 2 only. The matching engine uses the two-tier confidence weights below.

| Tier | Label | How earned | Confidence factor |
|---|---|---|---|
| **1** | Claimed | Self-declared on profile | 0.20 |
| **2** | Evidence-backed | LLM extraction confidence ≥ 0.85 on portfolio submission; all required signal types found | 0.55 |

**What is deferred:**
- Tier 3 (Scenario-verified, 0.80): requires `scenario_assessments` + `scenario_responses` tables — Phase 2
- Tier 4 (Platform-demonstrated, 0.95): requires `expert_seam_outcome_signals` + signal accumulation rule — Phase 2

The composite matching formula works with two weights. Experts max out at Tier 2 in the MVP.

---

### 0.5 Composite Match Score Weights

| Component | Weight | Formula element |
|---|---|---|
| Seam alignment | **40%** | Per seam: criticality weight × expert's verification confidence (0.20 or 0.55). Missing load-bearing seam = negative contribution |
| Domain depth coverage | **25%** | Expert verified depth ≥ footprint required depth for each domain |
| Archetype-tier congruence | **20%** | Expert's self-declared or confirmed history with same archetype + tier |
| Engagement model fit | **10%** | Expert's declared model vs. footprint requirement |
| Stack tags & recency | **5%** | Stack tag overlap (stored as JSONB array on `expert_profiles`) |

**Match strength labels** (client sees label, not numeric score):
- **Strong** → composite score > 0.78
- **Qualified** → 0.58–0.78
- **Conditional** → 0.42–0.58

---

### 0.6 All State Machines

> Every state name in this document resolves to one of the machines below. State transitions that trigger ledger entries are marked **[LEDGER]**. Transitions that call external APIs are marked **[API]**.

---

**Milestone states:**
```
DEFINED           → milestone created; acceptance criteria + DoD items set
AWAITING_PAYMENT  → CEO clicked "Fund"; per-milestone VA generated; VietQR displayed
FUNDED            → SePay IPN confirmed [LEDGER: escrow lock]
IN_PROGRESS       → auto-advances from FUNDED; Expert begins work
SUBMITTED         → Expert submits deliverable; all required DoD items checked
IN_REVISION       → revision requested by reviewer; expert revises
APPROVED          → sign-off authority verifies all required acceptance criteria
                    [LEDGER: escrow release] [API: SePay chi hộ fires]
RELEASED          → chi hộ credit IPN confirmed; ledger settled
DISPUTED          → dispute filed; escrow frozen
  → AUTO_RESOLVED (LLM confidence ≥ 0.80)  [LEDGER: winner receives escrow]
  → MANUAL_REVIEW (LLM confidence < 0.80)  [admin resolves via dashboard button]
```

**Bid states (simplified mutable-row model):**
```
DRAFT
  → SUBMITTED (expert submits 3-component bid)
  → TECH_REVIEW (TECH_TEAM opens review)
     TECH_TEAM sets tech_status:
       REVISION_REQUESTED → expert edits bid row and resets tech_status → PENDING → TECH_REVIEW
       APPROVED → tech_status = APPROVED
  → CEO_REVIEW (unlocked only when tech_status = APPROVED)
     CEO may write negotiated_price_vnd (one counter-offer round)
  → SELECTED / DECLINED

Note: Bid revisions are mutable updates to the single row, not versioned snapshots.
      tech_feedback written by TECH_TEAM; expert reads and edits in place.
      No separate bid_versions, bid_revision_requests, price_negotiations tables in MVP.
```

**Spec states:**
```
DRAFT → [auto-publish quality gate] → PUBLISHED
                      ↓ (fail)
            RETURNED_TO_CLIENT (auto — LLM advisory note; re-enters elicitation at specific void)
PUBLISHED → SUSPENDED (admin emergency pull-back only)
```

**Engagement states:**
```
PENDING → CONNECTED (expert accepted; NDA click-through by both parties)
  → ACTIVE (first milestone funded)
  → CLOSED
       ↕
   DISPUTED
```

**Engagement type (immutable):**
```
PROJECT_BASED    — full elicitation + matching + mutable bid + multi-milestone escrow
SERVICE_PURCHASE — no elicitation; no bid; single fixed-price milestone; direct VA payment
TECH_DISCOVERY   — no elicitation; availability-based; single milestone; direct VA payment
```

**DoD checklist item states:**
```
PENDING → COMPLETED (note required for required items)
        → NOT_APPLICABLE (note required; only for non-required items)
DB CHECK: NOT (is_required = true AND status = 'NOT_APPLICABLE')
Required items: all must reach COMPLETED before milestone SUBMITTED is permitted.
```

**Wallet transaction types:**
```
TOP_UP          — SePay IPN (wallet VA) → available_balance += amount [LEDGER]
SUBSCRIPTION    — user activates Pro → available_balance -= price [LEDGER]
ESCROW_LOCK     — CEO funds milestone → available_balance -= amount; locked_balance += amount [LEDGER]
ESCROW_RELEASE  — milestone APPROVED → locked_balance -= amount; expert available_balance += net [LEDGER]
PLATFORM_FEE    — 5% deducted from expert on release [LEDGER]
ESCROW_REFUND   — dispute refund to client [LEDGER]
ESCROW_SPLIT    — admin-resolved split [LEDGER]
WITHDRAWAL      — expert cash-out → available_balance -= amount [LEDGER] [API: chi hộ fires]
```

**Withdrawal states:**
```
PENDING   → withdrawal_request created; wallet debited atomically; chi hộ API called [API]
COMPLETED → SePay credit IPN fires on expert's linked bank account
FAILED    → chi hộ error; wallet balance restored [LEDGER]; expert notified
```

**Subscription states:**
```
free    → pro (payment via wallet; IPN confirms; subscription_expires_at = now + 6 months)
pro     → EXPIRING_SOON (7 days before expiry)
        → free (grace: active engagements grandfathered)
free    → pro (renewal)
```

**Dispute resolution (simplified 2-layer):**
```
Dispute filed → escrow FROZEN
  → Layer 1: LLM criterion evaluation
      confidence ≥ 0.80 → AUTO_RESOLVED [LEDGER: escrow distributed per finding]
      confidence < 0.80 → MANUAL_REVIEW

  → MANUAL_REVIEW: Admin reviews in Dispute Monitor → clicks Resolve button
      → Admin chooses: Release to expert / Refund to client / 50-50 split
      → [LEDGER: per admin decision]
```

> **Scope reduction note:** The full 3-layer system (48h cooling window, automated mutual agreement form, automatic 50/50 Layer 3 split) is deferred. MVP Layer 2 is admin-resolved via a single dashboard button. This is honest about what can be built reliably in 9 weeks.

---

### 0.7 RBAC Roles

| JWT `active_role` | `client_subtype` | Human label | How created |
|---|---|---|---|
| `CLIENT` | `CEO` | Client — Business | Self-register |
| `CLIENT` | `TECH_TEAM` | Client — Technical | Via signed handoff link only |
| `EXPERT` | *(none)* | Expert | Self-register |
| `ADMIN` | *(none)* | Admin | DB-seeded only |

**Dual-role accounts:** `roles: ["CLIENT_CEO", "EXPERT"]` on one account. `active_role` determines context. Role switch reissues JWT without re-login. Self-exclusion always applies regardless of active role.

**Self-technical CEO:** `project.self_technical = true` → CEO account gains TECH_TEAM capabilities for that project only.

**Permission matrix (MVP — simplified from full 51-table version):**

| Action | CEO | TECH_TEAM | EXPERT | ADMIN |
|---|---|---|---|---|
| Register account | ✅ | ❌ (link only) | ✅ | ❌ (seeded) |
| Submit project via elicitation (Pro) | ✅ | ❌ | ❌ | ❌ |
| Complete tech handoff (Stage 4) | ❌ (unless self_technical) | ✅ | ❌ | ❌ |
| Top up wallet | ✅ | ❌ | ✅ | ❌ |
| Purchase subscription | ✅ (Client Pro) | ❌ | ✅ (Expert Pro) | ❌ |
| View Artifact A | ✅ | ✅ | ✅ (matched) | ✅ |
| View Artifact B (artifact_b_json) | ❌ (never) | ✅ (state ≥ CONNECTED) | ✅ (state ≥ CONNECTED) | ✅ |
| Review bids — seam analysis | ❌ | ✅ | ❌ | ❌ |
| Flag bid tech_status (APPROVED / REVISION_REQUESTED) | ❌ | ✅ | ❌ | ❌ |
| Approve expert selection (ceo_status → APPROVED) | ✅ | ❌ | ❌ | ❌ |
| Fund milestone (wallet or VietQR) | ✅ | ❌ | ❌ | ❌ |
| Create DoD checklist | ❌ | ❌ | ✅ | ❌ |
| Sign off technical milestones | ❌ | ✅ | ❌ | ❌ |
| Sign off business milestones | ✅ | ❌ | ❌ | ❌ |
| Submit milestone deliverable | ❌ | ❌ | ✅ | ❌ |
| Link bank account (Bank Hub) | ❌ | ❌ | ✅ | ❌ |
| Request withdrawal (chi hộ) | ❌ | ❌ | ✅ | ❌ |
| Browse Path B marketplace | ✅ | ✅ | ❌ | ❌ |
| Buy SERVICE_PURCHASE / TECH_DISCOVERY | ✅ | ❌ | ❌ | ❌ |
| Publish service listing | ❌ | ❌ | ✅ | ❌ |
| File a dispute | ✅ | ✅ | ✅ | ❌ |
| Post-engagement review | ✅ | ✅ | ✅ | ❌ |
| Real-time messaging | ✅ | ✅ | ✅ | ❌ (read-only) |
| Emergency spec pull-back / account suspension | ❌ | ❌ | ❌ | ✅ |
| Resolve dispute (manual) | ❌ | ❌ | ❌ | ✅ |

---

### 0.8 Payment Architecture

> **Zero manual choke points.** Every financial event is either a pure DB ledger entry or a SePay API call with webhook confirmation. Admin never touches money.

```
INBOUND — wallet top-up:
  User scans WALLET_TOPUP VA QR → bank transfer → SePay IPN → NestJS credits wallet

INBOUND — CEO funds milestone:
  CEO scans per-milestone VA QR (fixed_amount enforced by bank)
  → SePay IPN → NestJS atomic ledger [ESCROW_LOCK]
  → escrow_accounts HELD; milestone → FUNDED → IN_PROGRESS
  → paygated_documents for this milestone → release_state = RELEASED

INTERNAL — Milestone released:
  NestJS atomic ledger:
    [ESCROW_RELEASE]: client locked_balance -= amount
    [PLATFORM_FEE]:   platform wallet += amount * platform_fee_pct
    [CREDIT_EXPERT]:  expert available_balance += amount * (1 - platform_fee_pct)
  → milestone → APPROVED → chi hộ call fires [API]

OUTBOUND — Expert withdrawal:
  Expert requests withdrawal → NestJS [WITHDRAWAL] available_balance -= amount
  → SePay chi hộ API POST { amount, bank_account_xid }
  → SePay transfers to expert's verified linked account
  → SePay credit IPN → withdrawal → COMPLETED

INTERNAL — Subscription:
  Wallet balance checked → [SUBSCRIPTION] available_balance -= price
  → users.subscription_{role}_tier = 'pro'; expires_at = now + 6 months
```

**SePay integration points:**
1. **VietQR + per-VA IPN** — inbound payments (wallet top-up, milestone, subscription, service)
2. **Bank Hub Hosted Link** — expert links bank account via OTP (no password sharing); webhook stores `bank_account_xid`
3. **Chi hộ API** — automated outbound disbursement to expert's linked bank account

**Platform fee:** Stored in `platform_settings.platform_fee_pct` (default 0.05). PLATFORM_FEE ledger entry credits the seeded system wallet referenced by `platform_settings.platform_wallet_id`.

---

### 0.9 Subscription Tiers & Feature Gates

| Feature | Client Free | Client Pro (500K VND / 6 mo) | Expert Free | Expert Pro (300K VND / 6 mo) |
|---|---|---|---|---|
| Browse Path B marketplace | ✅ | ✅ | ❌ | ❌ |
| Buy SERVICE_PURCHASE / TECH_DISCOVERY | ✅ | ✅ | ❌ | ❌ |
| AI Elicitation Engine (F2) | ❌ | ✅ | ❌ | ❌ |
| Matching engine / shortlist (F4) | ❌ | ✅ | ❌ | ❌ |
| Artifact B access post-connection | ❌ | ✅ | ❌ | ❌ |
| Create expert profile + Tier 1 seam claims | ❌ | ❌ | ✅ | ✅ |
| LLM portfolio evidence verification (Tier 2) | ❌ | ❌ | ❌ | ✅ |
| Bid on Tier 2–3 projects | ❌ | ❌ | ❌ | ✅ |
| AI Service Generator | ❌ | ❌ | ❌ | ✅ |
| Earnings analytics dashboard | ❌ | ❌ | ❌ | ✅ |

**Subscription state is stored directly on `users`** — `subscription_client_tier`, `subscription_expert_tier`, `sub_client_expires_at`, `sub_expert_expires_at`. No separate `user_subscriptions` table in MVP.

---

## 1. System Overview

### 1.1 The Problem AITasker Solves

The AI services market has a double-sided failure:

**Client side:** Non-technical founders cannot accurately describe what they need. They post vague symptom descriptions that attract mismatched experts. Existing platforms have no mechanism to transform a symptom into a structured, matchable requirement.

**Expert side:** AI experts cannot demonstrate the specific cross-domain intersection capabilities that determine project success. Generic platforms match on discipline labels (keywords like "machine learning"), not on the seam knowledge that actually determines engagement success.

**Platform side:** No existing marketplace addresses the mutual IP deadlock (both sides withhold exactly what the other needs), the ground truth bootstrap problem (clients don't know they need evaluation data), or the proposal catch-22 (experts need architecture details to scope; clients share architecture only after hiring).

### 1.2 What AITasker Does Differently (MVP)

Three structural interventions:

**Point 1 — Before matching:** The AI Elicitation Engine transforms the client's symptom description into a taxonomy-grounded capability footprint before any expert sees the posting. Technical questions route to the client's technical team through a signed handoff link.

**Point 2 — At matching:** A composite scoring model ranks experts using seam knowledge (cross-domain competence at domain boundaries). The seam gap map makes capability gaps visible before any commitment.

**Point 3 — During engagement:** State-gated information release resolves the mutual IP deadlock. Architecture is disclosed only after expert connection acceptance. Expert reasoning documents release incrementally as milestone payments are confirmed via SePay IPN.

### 1.3 MVP Scope Boundaries

**In scope (building):**
- Complete 5-stage AI elicitation engine with Scenario A (no TECH_TEAM) and Scenario B (self-technical CEO) variants
- 2-tier expert verification (Claimed + Evidence-backed via LLM portfolio extraction)
- Composite-scoring matching engine with seam gap map
- JSONB-based dual artifact (artifact_a_json public; artifact_b_json state-gated on `projects` table)
- Simplified bid system: 3-component bid, mutable row with tech_status/ceo_status dual-approval, one-round counter-offer
- Subscription gate (Client Pro + Expert Pro); wallet top-up via SePay VA; subscription guard on all LLM routes
- Internal double-entry ledger wallet system; atomic escrow operations
- SePay VietQR per-VA payment; IPN webhook handler with all branch types
- SePay Bank Hub Hosted Link for expert bank account linking
- SePay chi hộ automated expert withdrawal (zero admin)
- Service-based hiring (Path B marketplace; SERVICE_PURCHASE and TECH_DISCOVERY types)
- 2-layer milestone tracking: acceptance criteria (payment contract) + DoD checklist (submission gate)
- Pay-gated documents (IPN-triggered IP release: STAGED → RELEASED on milestone FUNDED)
- Real-time messaging per engagement (one channel shared by all roles)
- Message read tracking (per-user unread badges)
- Post-engagement reviews with structured signals (informational only — no Tier 4 in MVP)
- 2-layer dispute resolution (LLM Layer 1 auto-resolve; admin-resolved Layer 2 via dashboard button)
- Admin dashboard: monitoring-only (Platform Integrity Monitor, dispute log, transaction ledger, analytics)
- AI service description generator
- Dual-role user accounts (CLIENT_CEO + EXPERT; role switcher; self-exclusion)
- AWS EC2 cloud deployment

**Out of scope (deferred to Phase 2):**
- Tier 3 verification (scenario_assessments, scenario_responses tables)
- Tier 4 verification (expert_seam_outcome_signals, signal accumulation rule)
- Layer 3 dispute resolution (48h cooling window, automated 50/50 split)
- Sprint tracking, sprint comments, DoD item comments (milestone_sprints, sprint_status_updates tables)
- Add-on Phase Protocol (addon_phase_requests table)
- Spec clarifications as a separate surface (pre-bid questions via messages channel in MVP)
- Bid version history (bid_versions, bid_revision_requests tables — revisions are mutable row updates)
- Price negotiation table (price_negotiations — replaced by negotiated_price_vnd column on capability_bids)
- CEO conflict override table (bid_conflict_overrides — tech_status/ceo_status replace this)
- Revision requests table (revision_requests — replaced by revision_note column on acceptance_criteria)
- Organizations and organization_members (schema-only Phase 2)
- Dispute resolution reports (dispute_resolution_reports)
- Outcome signals aggregate table (outcome_signals — absorbed into reviews.structured_signals_json)

---

## 2. Actors & Roles

> See Section 0.7 for the full permission matrix.

| Actor | JWT role | Description | Key MVP constraints |
|---|---|---|---|
| **CEO** | `CLIENT / CEO` | Non-technical client; initiates projects via elicitation; controls budget and expert selection | Cannot see artifact_b_json at any state; needs Client Pro subscription for elicitation and matching |
| **Tech Team** | `CLIENT / TECH_TEAM` | Client's technical lead; created only via signed handoff link | Cannot fund milestones; owns technical milestone sign-off; sees artifact_b_json post-connection |
| **Expert** | `EXPERT` | AI consultant; bids on projects (Path A) or sells services (Path B); must link bank account for withdrawal | Cannot see artifact_b_json until engagement CONNECTED + NDA accepted |
| **Admin** | `ADMIN` | Monitoring-only; seeded at DB level; no routine approvals; two emergency actions: spec pull-back and account suspension; resolves MANUAL_REVIEW disputes via dashboard button | |

**Dual-role:** `roles: ["CLIENT_CEO", "EXPERT"]` on one account. Self-exclusion prevents bidding on own projects regardless of active role.

**All-non-technical CEO (Scenario A):** Stage 4 unreachable → system offers TECH_DISCOVERY service purchase or injected Technical Discovery milestone as Milestone 0. Both use `engagement.type = TECH_DISCOVERY`.

**Self-technical CEO (Scenario B):** `project.self_technical = true` → CEO completes Stage 4 themselves. CEO gains TECH_TEAM capabilities for that project. No second account created.

---

## 3. Deliverable Features

### Feature Map (MVP)

```
F1    Authentication & Role Management (dual-role, Bank Hub field)
F1.5  Subscription System (wallet top-up via VA + subscription gate)
F2    AI-Guided Project Elicitation Engine          ← Primary AI innovation
F3    Expert Taxonomy Profile, Service Listings & Purchase (Path A + Path B)
F4    AI Expert Recommendation & Seam Matching      ← Primary AI innovation
F5    Scoped Information Release (JSONB Dual Artifact + Pay-gated Docs)
F6    Simplified Capability Bid System (mutable row; dual-status approval)
F7    Milestone Management + DoD (2-layer: criteria + checklist)
F9    Real-Time Messaging
F10   Internal Wallet + Escrow + SePay Full Integration
F11   Reviews & Reputation
F12   Admin Dashboard (monitoring-only + manual dispute resolution)
```

> **Note on numbering:** F8 (Add-On Phase Protocol) is deferred. Feature numbers are preserved from the full spec for traceability.

---

### F1 — Authentication & Role Management

**What it does:** JWT-based auth enforcing the RBAC structure on every protected route.

**JWT payload:**
```json
{
  "sub": "user_id",
  "active_role": "CLIENT | EXPERT | ADMIN",
  "client_subtype": "CEO | TECH_TEAM | null",
  "roles": ["CLIENT_CEO", "EXPERT"],
  "self_technical_projects": ["proj_abc"],
  "project_ids": ["proj_abc"]
}
```

**Account creation paths:**
1. **CEO:** Standard registration → `active_role = CLIENT, client_subtype = CEO`
2. **TECH_TEAM:** Via signed handoff link only → `client_subtype = TECH_TEAM, linked_project_id = project_id`
3. **EXPERT:** Standard registration → `active_role = EXPERT`
4. **Add second role:** Account Settings → role acquisition; `roles` array updated; role switcher appears
5. **ADMIN:** DB-seeded only

**Key `users` table fields:**
```sql
roles                    JSONB    DEFAULT '["CLIENT_CEO"]'
active_role              TEXT     NOT NULL
subscription_client_tier TEXT     DEFAULT 'free'
subscription_expert_tier TEXT     DEFAULT 'free'
sub_client_expires_at    TIMESTAMPTZ NULL
sub_expert_expires_at    TIMESTAMPTZ NULL
sepay_bank_account_xid   TEXT     NULL   -- set by Bank Hub webhook
bank_account_holder_name TEXT     NULL
bank_linked_at           TIMESTAMPTZ NULL
```

> **Scope change from full spec:** No `user_subscriptions` table. Subscription state is stored directly as columns on `users`. The subscription guard reads these columns directly.

**Owner:** TV2 (Chí Nhân)
**Effort:** ~5 days

---

### F1.5 — Subscription System

**What it does:** Gates all LLM-powered features behind a one-time activation payment. Satisfies the teacher's requirement that users subscribe to access AI features.

**Payment flow — subscription via internal wallet:**
```
POST /subscriptions/activate { role_type: "client" | "expert" }
Guard: wallet.available_balance >= subscription_price[role_type]

NestJS DB transaction:
  1. available_balance -= price
  2. wallet_transaction: { type: SUBSCRIPTION, amount: price, reference: "SUB-{user_id}:{role_type}" }
  3. users.subscription_{role_type}_tier = 'pro'
  4. users.sub_{role_type}_expires_at = now() + interval '6 months'
  5. Commit
  6. Notify: "Your Pro subscription is active until {date}"
```

**Wallet top-up:** Each user has one permanent `WALLET_TOPUP` virtual account created on registration. CEO opens "Top Up" → VietQR displayed → user transfers any amount → SePay IPN credits `wallet.available_balance`.

**Owner:** TV2 (backend + guard), TV3 (UI)
**Effort:** TV2: ~3 days, TV3: ~2 days

---

### F2 — AI-Guided Project Elicitation Engine

> **Primary AI innovation. This feature is the entire argument for AITasker's existence. TV1 treats Weeks 3–4 as a focused sprint. If F2 slips, everything slips.**

**What it does:** Transforms a symptom description into a taxonomy-grounded capability footprint and dual artifact through a 5-stage conversational diagnostic engine.

**Stage 1 — Symptom intake `[CEO]`:** CEO types a free-form pain description. LLM runs: intent separation (strips prescribed solutions from actual symptoms), scale signal extraction (volume/cost indicators for tier determination), void detection (checks for absent mandatory SDLC components: no evaluation ground truth → flag; no error handling strategy → flag).

**Stage 2 — Archetype confirmation + SDLC injection `[CEO]`:** System presents probable archetype in plain business language. CEO selects from three behavioral options. Missing SDLC components are injected one at a time as client protections:
- Ground truth void → *"To measure whether the AI is working, your expert will need a reference dataset."* If no: Phase 0 (Ground Truth Setup) injected as mandatory Milestone 1. CEO cannot remove it.

**Stage 3 — Architecture and scale probe `[CEO]`:** Four behavioral questions requiring no technical knowledge:
- Synchronous vs. asynchronous: *"When a user submits content, do they wait for an immediate result?"*
- Thundering Herd: *"Are there times when very large volumes arrive all at once?"*
- Stateful memory: *"Does the AI need to remember past items it has processed?"*
- HITL requirement: *"If the AI is uncertain, what should happen — reject, approve, or escalate?"*

**Stage 4 — Tech team handoff `[TECH_TEAM — account created here]`:** If probe reveals integration with an existing production system, async processing, stateful memory, or legacy data, the system hard-blocks the CEO and generates a signed handoff link. The TECH_TEAM inputs: stack tags, integration method, legacy data volume, deployment expectation, optional sensitive schema upload (goes to artifact_b_json). The link expires after 72 hours.

**Scenario A — No TECH_TEAM:** Stage 4 unreachable → system offers two options: inject TECH_DISCOVERY milestone as Milestone 0, or purchase a TECH_DISCOVERY service engagement. Both use `engagement.type = TECH_DISCOVERY`.

**Scenario B — Self-technical CEO:** CEO sets `project.self_technical = true` → Stage 4 form presented directly to CEO. CEO completes both sides in one session. No second account needed.

**Stage 5 — Synthesis `[System]`:** Resolves CEO/TECH_TEAM signal conflicts privately. Produces:
- `required_seams_json`, `required_domains_json`, `milestone_framework_json` (written to `projects` table)
- `artifact_a_json` (public spec: business intent, architecture labels, stack tags, escrow lock notice)
- `artifact_b_json` (technical vault: TECH_TEAM's uploads, schemas, integration contracts — state-gated)

**Auto-publish gate (replaces admin approval queue):**
- Footprint completeness score ≥ 0.7
- Matching pre-check: at least 1 expert above minimum threshold
- No unresolved hard-flagged voids

All pass → `projects.state = PUBLISHED`; matching engine fires; CEO notified.
Any fail → `projects.state = RETURNED_TO_CLIENT`; LLM generates targeted advisory note identifying the specific void; CEO re-enters elicitation at that exact void — not from the beginning. No admin queue.

**Owner:** TV1 (FastAPI + LLM extraction + synthesis), TV2 (elicitation UI + stage state machine)
**Effort:** TV1: ~10 days, TV2: ~4 days

---

### F3 — Expert Taxonomy Profile, Service Listings & Purchase

**Expert profile components:**

*Taxonomy capability map:*
- Six capability domains, each at one of three depth levels: Deep / Operational / Surface
- Ten seam claims: Tier 1 (self-declared) or Tier 2 (LLM-verified via portfolio submission)
- Stack tags: stored as `stack_tags_json JSONB` array on `expert_profiles` (no separate table)
- Engagement model: Advisory / Spec+review / Full implementation
- Archetype history: `archetype_history_json JSONB` on `expert_profiles` (self-declared for cold start; confirmed entries derived at query time from closed engagements)

*Portfolio evidence (Tier 2 upgrade):*
- Expert submits structured decision-point writeup
- FastAPI LLM extraction: if confidence ≥ 0.85 AND all required signal types found → auto-upgrade to Tier 2; `platform_decisions` row written
- If below threshold: LLM generates specific gap advisory; expert resubmits (5-attempt / 30-day lockout enforced via `expert_seam_claims.submission_count` and `locked_until`)

> **Scope change from full spec:** Tier 3 (scenario assessments) and Tier 4 (post-engagement signal accumulation) are deferred. Expert verification in MVP is Claimed (Tier 1) → Evidence-backed (Tier 2) via portfolio only.

*AI Service Generator:*
- Expert inputs key capabilities and target use cases
- LLM generates structured service description (title, description, scope, timeline, suggested price)
- Expert edits and publishes

*Service listings (Path B):*
```
POST /services/{id}/purchase [guard: CLIENT/CEO]
  1. Creates per-order VA (entity_type: SERVICE, fixed_amount: service.price_vnd)
  2. Returns VietQR
  3. User scans → SePay IPN fires
  4. IPN SERVICE branch:
     - Atomic: [ESCROW_LOCK] client.available_balance -= amount
     - escrow_accounts: { engagement_id, amount, status: HELD }
     - engagement created: { type: SERVICE_PURCHASE, state: ACTIVE }
     - Notify expert
```

**Owner:** TV3 (profile forms + Path B UI), TV1 (portfolio LLM extraction + AI service generator), TV2 (seam claim state machine + IPN SERVICE branch)
**Effort:** TV3: ~7 days, TV1: ~5 days, TV2: ~3 days

---

### F4 — AI Expert Recommendation & Seam Matching

**Five-component composite score (same formula as full spec; two-tier confidence only):**

| Component | Weight |
|---|---|
| Seam alignment | 40% |
| Domain depth coverage | 25% |
| Archetype-tier congruence | 20% |
| Engagement model fit | 10% |
| Stack tags & recency | 5% |

**Hard gate (eliminate before ranking):**
- Expert claimed-to-verified ratio > 4:1 → excluded from Tier 2+ projects

> **Scope change from full spec:** The negative outcome signal hard gate (`2+ negative expert_seam_outcome_signals`) is deferred with the cut of Tier 4. MVP uses only the ratio gate.

**Seam gap map:** For each required seam: Amber/Evidence-backed (Tier 2), Yellow/Claimed (Tier 1), Red/Absent. Green/Verified absent in MVP (no Tier 3/4).

**Shortlist:** 3–5 ranked candidates. Client sees strength label, not numeric score.

**Owner:** TV1 (scoring algorithm + gap map), TV2 (match card UI + shortlist display)
**Effort:** TV1: ~5 days, TV2: ~3 days

---

### F5 — Scoped Information Release (JSONB Dual Artifact + Pay-gated Docs)

**What it does:** Resolves the mutual IP deadlock.

**Artifact A (`artifact_a_json` on `projects` table):**
- Business intent, architecture category, stack tags, volume tier, SDLC milestone framework
- Escrow lock notice: *"Detailed backend schemas are held in information escrow until an expert is selected."*
- Always visible to matched experts

**Artifact B (`artifact_b_json` on `projects` table):**
- TECH_TEAM's schema uploads, payload samples, integration contracts
- FastAPI route guard: return `artifact_b_json` ONLY when `engagement.state >= CONNECTED AND client_nda_accepted_at IS NOT NULL AND expert_nda_accepted_at IS NOT NULL AND requester.active_role IN ('EXPERT', 'TECH_TEAM')`
- CEO permanently excluded at route level regardless of engagement state

> **Scope change from full spec:** `capability_footprints`, `artifact_a`, `artifact_b` are three separate tables in the full spec. MVP collapses all five fields into JSONB columns on `projects`. The trust gate is identical — it is a route guard, not a DB-level constraint. No architectural value is lost.

**Connection flow:**
1. CEO selects expert from shortlist
2. Platform sends connection request to Expert
3. Expert reviews Artifact A and accepts or declines
4. On acceptance: both CEO and Expert complete NDA click-through (checkboxes); `client_nda_accepted_at` and `expert_nda_accepted_at` set on `engagements`
5. `engagement.state → CONNECTED` → Artifact B becomes queryable by Expert and TECH_TEAM

**Pay-gated knowledge staging:**
- Expert uploads reasoning documents to staging area, tagged with a milestone release trigger
- When SePay IPN confirms milestone FUNDED: `paygated_documents.release_state → RELEASED` automatically — TECH_TEAM-only read; CEO permanently excluded
- The expert discloses full design rationale; the TECH_TEAM receives it only after real money has moved into escrow
- This is the core IP deadlock resolution. It is KEPT in MVP despite scope reduction — it is 3 columns and a status flip in the IPN handler.

**Owner:** TV1 (FastAPI Artifact B route guard + pay-gated doc serve), TV2 (connection flow + pay-gated staging UI)
**Effort:** TV1: ~2 days, TV2: ~5 days

---

### F6 — Simplified Capability Bid System

> **Scope change from full spec:** The full 4-surface system (spec clarifications, versioned revisions, price negotiation table, conflict override table) is replaced by a mutable single-row bid with dual status columns. Pre-bid technical questions use the `messages` channel (no Surface A table). The key innovation — TECH_TEAM approves before CEO selects — is preserved.

**Three bid components (same as full spec — 422 if any missing at SUBMITTED):**
1. Footprint Alignment Statement (domains at verified depth, seam claims)
2. Architectural Approach Summary (milestone framework and approach discrepancies)
3. Conditional Milestone Pricing (structured per-milestone pricing)

**Dual-approval flow:**
```
Expert submits bid → bid.state = SUBMITTED

TECH_TEAM reviews in Tech Dashboard:
  SET tech_status = 'REVISION_REQUESTED' + writes tech_feedback TEXT
    → Expert reads feedback, edits bid row (any component), resets tech_status = 'PENDING'
    → Loop until TECH_TEAM satisfied
  SET tech_status = 'APPROVED'
    → CEO_REVIEW unlocks

CEO reviews in CEO Dashboard (only after tech_status = APPROVED):
  CEO may write negotiated_price_vnd (optional counter-offer, one round only)
  SET ceo_status = 'APPROVED'   → bid.state = SELECTED; engagement → ACTIVE
  SET ceo_status = 'DECLINED'   → bid.state = DECLINED
```

**`capability_bids` key columns (scope-reduction additions):**
```sql
tech_status          TEXT  DEFAULT 'PENDING'
                           CHECK (tech_status IN ('PENDING','APPROVED','REVISION_REQUESTED'))
ceo_status           TEXT  DEFAULT 'PENDING'
                           CHECK (ceo_status IN ('PENDING','APPROVED','DECLINED'))
tech_feedback        TEXT  NULL   -- written when tech_status → REVISION_REQUESTED
negotiated_price_vnd BIGINT NULL  -- CEO counter-offer; one round only
```

**Route guard on ceo_status update:**
```
PUT /bids/{id}/ceo-decision
Guard: capability_bids.tech_status = 'APPROVED' → else 422 "Tech review not complete"
```

**Owner:** TV2 (bid state machine + routes), TV3 (bid submission form + TECH_TEAM review UI + CEO review UI)
**Effort:** TV2: ~3 days, TV3: ~4 days

---

### F7 — Milestone Management + DoD (2-Layer)

> **Scope change from full spec:** Layer 3 (sprint tracking: `milestone_sprints`, `sprint_status_updates`) and the Add-On Phase Protocol (F8: `addon_phase_requests`) are deferred. Sprint comments and DoD item comments are also deferred. The MVP implements the two-layer system: Layer 1 (acceptance criteria / payment contract) and Layer 2 (DoD checklist / submission gate). If scope changes occur mid-project, the CEO manually creates a new milestone — no automated scope evolution detection.

**Layer 1 — Milestones (Payment Contract):**
```sql
milestones:
  id, engagement_id, milestone_number
  deliverable_statement  TEXT
  sign_off_authority     TEXT  CHECK IN ('TECH_TEAM','CEO','JOINT')
  payment_amount_vnd     BIGINT
  state                  TEXT  (see §0.6 milestone state machine)
  va_number              TEXT NULL
  va_expires_at          TIMESTAMPTZ NULL
  funded_at, submitted_at, approved_at, released_at  TIMESTAMPTZ NULL

acceptance_criteria:
  id, milestone_id
  criterion_text         TEXT   -- LLM quality-gates on save (flags subjective language)
  is_required            BOOLEAN DEFAULT TRUE
  verified_by_role       TEXT   CHECK IN ('TECH_TEAM','CEO','JOINT')
  verified_at            TIMESTAMPTZ NULL
  revision_note          TEXT NULL  -- replaces cut revision_requests table
```

**Criterion quality gate (on milestone definition save):** LLM checks for subjective language ("client is satisfied"). Flags returned; expert acknowledges. Reduces disputes by forcing objective language upfront. Platform decision logged.

**State transition guards:**
- `SUBMITTED` transition: all `is_required = true` DoD items must have `completed_at` set → 422 with list of unchecked items
- `APPROVED` transition: all `is_required = true` criteria must have `verified_at` set → 422 with list of unverified criteria

**Sign-off authority routing:**
| Milestone type | Authority |
|---|---|
| Technical (model, pipeline) | TECH_TEAM |
| Business (deployment, reporting) | CEO |
| Joint | Both required; milestone waits until both `verified_at` set |

**Milestone funding (per-VA):**
```
CEO clicks "Fund Milestone N":
  1. NestJS creates per-milestone VA: { entity_type: MILESTONE, fixed_amount: amount, expires_at: now+24h }
  2. Returns VietQR with fixed amount
  3. CEO scans → SePay IPN fires
  4. IPN MILESTONE branch:
     - Validates amount == va.fixed_amount
     - Atomic: [ESCROW_LOCK] client.available_balance -= amount; locked_balance += amount
     - escrow_accounts: { milestone_id, amount, status: HELD }
     - milestone → FUNDED → IN_PROGRESS
     - paygated_docs staged for this milestone → release_state = RELEASED
```

**Milestone APPROVED → RELEASED:**
```
On final required criterion verified_at set:
NestJS atomic transaction:
  [ESCROW_RELEASE]: client.locked_balance -= amount
  [CREDIT_EXPERT]:  expert.available_balance += amount * (1 - fee_pct)
  [PLATFORM_FEE]:   platform_wallet.available_balance += amount * fee_pct
  escrow_accounts.status → RELEASED
  milestone → APPROVED
Commit → chi hộ API called (async, non-blocking)
```

**Layer 2 — DoD Checklist (Submission Gate):**
```sql
milestone_dod_items:
  id, milestone_id
  item_description   TEXT
  is_required        BOOLEAN DEFAULT TRUE
  status             TEXT CHECK IN ('PENDING','COMPLETED','NOT_APPLICABLE')
  completed_at       TIMESTAMPTZ NULL
  completion_note    TEXT NULL
  not_applicable_note TEXT NULL
  maps_to_criterion_id UUID NULL REFERENCES acceptance_criteria(id)

DB CHECK: NOT (is_required = TRUE AND status = 'NOT_APPLICABLE')
```

**Visibility:**
- Expert: full edit (COMPLETED / NOT_APPLICABLE with required notes)
- TECH_TEAM: read-only (no comment thread in MVP — uses messages channel)
- CEO: hidden

**Owner:** TV2 (milestone state machine, DoD guard, per-VA creation, IPN MILESTONE branch, criterion LLM check, escrow release + chi hộ call), TV3 (milestone management UI — CEO funding view, TECH_TEAM review interface, Expert DoD checklist editor)
**Effort:** TV2: ~9 days, TV3: ~5 days

---

### F9 — Real-Time Messaging

**What it does:** One chat channel per engagement, shared by CEO, TECH_TEAM, and Expert.

- Real-time via Socket.io (room per engagement_id)
- Message history persisted in PostgreSQL (`messages` table; `engagement_id` is the thread key — no separate thread table)
- File attachment: single `attachment_url TEXT NULL` per message (MVP — one file per message)
- Unread badge: per-user read tracking via `message_reads (message_id, user_id, read_at)`; unread count computed as `NOT EXISTS (SELECT 1 FROM message_reads WHERE message_id = ? AND user_id = ?)`
- ADMIN can view all threads read-only; cannot send messages
- Pre-bid technical questions use this channel (no separate spec_clarifications table in MVP)
- DoD and sprint discussions also use this channel (no DoD item comment thread or sprint comment thread in MVP)

**Owner:** TV2 (Socket.io + chat UI)
**Effort:** ~3 days

---

### F10 — Internal Wallet + Escrow + SePay Full Integration

**Architecture summary:** See Section 0.8 for full flow diagrams.

**IPN webhook handler branches:**
```
POST /webhooks/sepay/ipn
  1. Verify HMAC signature (SEPAY_SECRET_KEY)
  2. Idempotency: check wallet_transactions for existing reference_id
     CREATE UNIQUE INDEX wallet_tx_idempotency ON wallet_transactions(wallet_id, reference_id)
     WHERE reference_id IS NOT NULL;
  3. Look up virtual_accounts by va_number
  4. Branch on entity_type:

  WALLET_TOPUP:
    → wallet.available_balance += amount
    → wallet_transaction: { type: TOP_UP }

  MILESTONE:
    → validate: amount == va.fixed_amount
    → atomic: ESCROW_LOCK ledger + escrow_accounts HELD + milestone FUNDED
    → paygated_docs for this milestone → RELEASED

  SERVICE / TECH_DISCOVERY:
    → validate: amount == va.fixed_amount
    → atomic: ESCROW_LOCK + escrow_accounts HELD (engagement_id, not milestone_id)
    → engagement.state = ACTIVE; notify expert

  SUBSCRIPTION:
    → atomic: [SUBSCRIPTION] wallet deduct
    → users.subscription_{role}_tier = 'pro'
    → users.sub_{role}_expires_at = now + 6 months

  5. Return HTTP 200 synchronously (SePay retries on non-2xx; idempotency handles retries)
```

**Escrow dual-parent structure:**
```sql
escrow_accounts:
  milestone_id  UUID NULL REFERENCES milestones(id)   -- Path A: PROJECT_BASED
  engagement_id UUID NULL REFERENCES engagements(id)  -- Path B: SERVICE_PURCHASE / TECH_DISCOVERY

CONSTRAINT escrow_has_one_parent CHECK (
  (milestone_id IS NOT NULL AND engagement_id IS NULL) OR
  (milestone_id IS NULL AND engagement_id IS NOT NULL)
)
-- Two partial unique indexes replace old UNIQUE on milestone_id:
CREATE UNIQUE INDEX escrow_milestone_unique ON escrow_accounts(milestone_id)
  WHERE milestone_id IS NOT NULL;
CREATE UNIQUE INDEX escrow_engagement_unique ON escrow_accounts(engagement_id)
  WHERE engagement_id IS NOT NULL;
```

**Expert withdrawal:**
```
Expert clicks "Withdraw" [guard: bank_account_xid must be set]
NestJS atomic:
  1. wallet.available_balance -= amount
  2. wallet_transaction: { type: WITHDRAWAL }
  3. withdrawal_requests: { type: EXPERT_MANUAL, status: PENDING }
Commit → chi hộ API POST { amount, bank_account_xid, reference: "WD-{id}" }
SePay IPN → withdrawal_requests.status = COMPLETED
Failed → balance restored; expert notified
```

**Week 1 task:** Verify SePay chi hộ API tier availability via hotline 02873.059.589. Log result in Slack by EOD Week 1 Day 1. If chi hộ unavailable, evaluate TPBank personal Open API as fallback.

**Owner:** TV2 (all SePay integrations: IPN handler, Bank Hub Hosted Link, chi hộ, withdrawal flow, all tables), TV3 (wallet top-up UI, subscription UI, transaction history, earnings dashboard)
**Effort:** TV2: ~10 days, TV3: ~3 days

---

### F11 — Reviews & Reputation

**Three post-engagement review forms (role-gated):**

*TECH_TEAM reviews Expert (structured — feeds structured_signals_json on reviews):*
- Overall rating (1–5)
- Seam performance questions (e.g., "Did the expert raise the ground truth baseline proactively?")
- These responses stored in `reviews.structured_signals_json` — informational for admin dashboard
- No automatic Tier 4 accumulation in MVP (expert_seam_outcome_signals table cut)

*CEO reviews Expert (business-facing):*
- Overall rating, communication clarity, milestone structure effectiveness, open text

*Expert reviews engagement:*
- Overall rating, milestone approval responsiveness, Artifact B quality, CEO communication clarity

**Reputation display on expert profile:** Engagement count, completion rate, average rating.

> **Scope change from full spec:** Post-engagement signal processing does not update `expert_seam_claims.verification_tier` in MVP (Tier 4 is cut). `structured_signals_json` data is displayed in admin analytics only.

**Owner:** TV3 (review forms + reputation display), TV2 (review route + signal storage)
**Effort:** TV3: ~3 days, TV2: ~1 day

---

### F12 — Admin Dashboard (Monitoring-only + Manual Dispute Resolution)

**Design principle:** The Admin Dashboard is a monitoring surface, not an operational queue. The only write actions are three emergency interventions: spec pull-back, account suspension, and manual dispute resolution.

**Module 1 — Platform Integrity Monitor:** Reads `platform_decisions` log table. Displays:
- Spec auto-return history (void type + LLM advisory note sent)
- Seam verification log (every portfolio auto-upgrade / auto-return with confidence score)
- Dispute resolution log (Layer 1 LLM auto-resolve vs. MANUAL_REVIEW)

**Module 2 — Dispute Monitor:**
- All disputes with current state, LLM confidence score, filed date
- For `state = MANUAL_REVIEW`: admin sees both parties' positions and escrow amount
- **Write action:** Three buttons: "Release to Expert" / "Refund to Client" / "Split 50/50"
  - Each button triggers the corresponding atomic ledger operation [LEDGER]
  - Admin resolves disputes — the platform does not resolve automatically for confidence < 0.80

**Module 3 — Account Management:** View users by role, engagement count, seam claim tier history. Suspend / reactivate accounts.

**Module 4 — Transaction Ledger:** Full `wallet_transactions` + `escrow_accounts` + `withdrawal_requests` read-only view. Filter by date, type, expert, state.

**Module 5 — Analytics Dashboard:**
- Active projects by archetype and tier
- Elicitation completion rate and auto-publish pass rate
- Portfolio auto-upgrade rate and auto-return rate
- Dispute rate and LLM auto-resolution rate
- Milestone completion rate and average review cycle

**Owner:** TV2 (Platform Integrity Monitor data + account management + fraud detection + spec pull-back route + dispute resolution atomic operations), TV3 (all admin UI modules), TV1 (platform_decisions table writes from FastAPI)
**Effort:** TV2: ~5 days, TV3: ~3 days, TV1: ~1 day

---

## 4. Use Cases

```
CEO FLOWS
  UC01   Submit project via AI elicitation (Pro subscription required)
  UC01a  Proceed via Technical Discovery pathway — Scenario A (no TECH_TEAM)
  UC01b  Complete Stage 4 as self-technical CEO — Scenario B
  UC02   Top up wallet via WALLET_TOPUP VA VietQR
  UC03   Purchase Client Pro subscription from wallet
  UC04   View Artifact A and expert shortlist (Pro)
  UC05   Respond to pre-bid technical questions via messages channel
  UC06   Review TECH_TEAM bid analysis; set ceo_status (APPROVED / DECLINED)
  UC06n  Write counter-offer in negotiated_price_vnd (optional, one round)
  UC07   Send connection request; complete NDA click-through
  UC08   Fund milestone via per-milestone VA QR
  UC09   Approve business milestones; triggers escrow release + chi hộ
  UC10   Buy expert service directly — SERVICE_PURCHASE or TECH_DISCOVERY
  UC11   Complete post-engagement review (CEO form)

TECH_TEAM FLOWS
  UC01t  Complete tech handoff (Stage 4 — TECH_TEAM-only route)
  UC04t  View Artifact B post-connection
  UC05t  Answer pre-bid technical questions via messages channel
  UC06a  Review capability bids — seam analysis; set tech_status + tech_feedback
  UC07t  Access pay-gated reasoning documents (TECH_TEAM-only inbox)
  UC08t  Sign off technical milestones criterion by criterion
  UC09t  Complete post-engagement review (structured seam-signal form)

EXPERT FLOWS
  UC12   Create taxonomy-based profile (domains, seams, stack tags)
  UC13   Purchase Expert Pro subscription from wallet
  UC14   Submit portfolio evidence for LLM auto-verification (Tier 2)
  UC15   Link bank account via Bank Hub Hosted Link
  UC16   Publish AI service listing via AI generator (Path B)
  UC17   Browse project shortlist; ask pre-bid questions via messages channel
  UC18   Submit structured capability bid (3 required components)
  UC18r  Edit bid row after tech_feedback received; reset tech_status → PENDING
  UC19   Accept connection; view Artifact B
  UC20   Stage pay-gated reasoning documents with milestone release trigger
  UC21   Create DoD checklist for funded milestone
  UC22   Submit milestone deliverable (DoD gate enforced at route level)
  UC23   Request withdrawal (chi hộ fires automatically)
  UC24   Complete post-engagement review (Expert form)

DUAL-ROLE FLOWS
  UC-DR1 Add second role (Account Settings → role acquisition)
  UC-DR2 Switch active role via role switcher (JWT reissued)
  UC-DR3 Fund milestone from expert earnings (same wallet)

ADMIN FLOWS
  UC-A1  Emergency spec pull-back (state → SUSPENDED)
  UC-A2  Monitor Platform Integrity Monitor (auto-return log, verification log)
  UC-A3  Monitor transaction ledger (wallet_transactions, escrow_accounts)
  UC-A4  Monitor and manually resolve disputes (dispute_monitor → choose resolution)
  UC-A5  Monitor analytics and export research data

GENERAL FLOWS
  UC-G1  Browse Path B marketplace (Free tier accessible)
  UC-G2  Real-time messaging (all transactional roles; ADMIN read-only)
  UC-G3  View wallet, transaction history, subscription status
  UC-G4  File a dispute (any transactional role)
  UC-G5  Post-engagement review (all three role-specific forms)
```

---

## 5. Architecture & Tech Stack

### 5.1 System Architecture

Same two-service architecture as full spec:
- **NestJS (Main API):** Auth, all business logic, all SePay integrations, wallet ledger, bid state machine, milestone state machine, messaging, reviews, admin operations
- **FastAPI (AI Service):** Elicitation engine, LLM extraction, matching computation, seam gap map generation, portfolio evidence LLM evaluation, AI service generator, Artifact B route guard (passive DB read)
- **PostgreSQL (shared):** Single instance; both services connect via their respective ORMs

```
┌─────────────────────────────────────────────────────────┐
│                  PRESENTATION LAYER                     │
│     ReactJS + Tailwind CSS + Socket.io (client)         │
│  CEO Dashboard │ Tech Dashboard │ Expert App │ Admin    │
└────────────────────────┬────────────────────────────────┘
                         │ HTTPS / WebSocket
┌────────────────────────▼────────────────────────────────┐
│                APPLICATION LAYER                        │
│  ┌──────────────────────┐  ┌─────────────────────────┐  │
│  │  MAIN API (NestJS)   │  │  AI SERVICE (FastAPI)   │  │
│  │  • Auth + JWT        │  │  • Elicitation Engine   │  │
│  │  • User/Role CRUD    │  │  • LLM Extraction       │  │
│  │  • Project CRUD      │  │  • Matching + Scoring   │  │
│  │  • Bid state machine │  │  • Seam Gap Map         │  │
│  │  • Milestone life.   │  │  • Portfolio Eval LLM   │  │
│  │  • SePay IPN handler │  │  • AI Service Generator │  │
│  │  • Wallet ledger     │  │  • Artifact B guard     │  │
│  │  • Messaging Socket  │  │    (passive DB read)    │  │
│  │  • Admin routes      │  │  Pydantic + SQLAlchemy  │  │
│  │  Prisma ORM          │  │  Alembic migrations     │  │
│  └──────────┬───────────┘  └────────────┬────────────┘  │
│             └────────── Internal REST ──┘               │
└─────────────────────────│───────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────┐
│                    DATA LAYER                           │
│              PostgreSQL 16 (shared)                     │
│  28 tables — see §5.3 for complete schema               │
└─────────────────────────│───────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────┐
│               EXTERNAL SERVICES                         │
│  Anthropic Claude API  (elicitation + portfolio LLM)   │
│  SePay VietQR + VA     (all inbound payments)          │
│  SePay Bank Hub        (expert bank account linking)   │
│  SePay Chi Hộ API      (automated expert withdrawal)   │
│  SendGrid / Nodemailer (email notifications)           │
└─────────────────────────│───────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────┐
│           DEPLOYMENT (AWS EC2)                          │
│  EC2 t3.micro · Docker Compose · Elastic IP            │
│  GitHub Actions → SSH deploy on merge to main          │
└─────────────────────────────────────────────────────────┘
```

### 5.2 Tech Stack

| Layer | Technology | Notes |
|---|---|---|
| Frontend | ReactJS, React Router v6, Tailwind CSS, Socket.io-client | |
| Main API | NestJS, JWT, Zod validation | |
| Main ORM | Prisma ORM + Prisma Migrate | |
| AI Service | FastAPI, Pydantic v2, SQLAlchemy 2.0, Uvicorn | |
| AI Migrations | Alembic | |
| Database | PostgreSQL 16 (shared single instance) | |
| LLM API | Anthropic Claude claude-sonnet-4-6 | Elicitation + portfolio LLM + criterion quality check |
| Payment | SePay Node.js SDK in NestJS exclusively | FastAPI has zero involvement in payment flows |
| Real-time | Socket.io (NestJS server, React client) | |
| Email | SendGrid (free tier) or Nodemailer + SMTP | Handoff link delivery + milestone notifications |
| Auth | JWT (15min access, 7-day refresh), bcrypt | |
| File uploads | Multer + local disk on EC2 (MVP) | |
| Deployment | AWS EC2 t3.micro, Docker Compose, Elastic IP | GitHub Actions SSH deploy on merge to main |
| QA | Postman/Apidog, ASP.NET QA dashboard (TV5) | |

### 5.3 Database Schema (28 Tables)

```
users (id, email, password_hash, full_name, phone, roles JSONB, active_role,
       client_subtype, subscription_client_tier, subscription_expert_tier,
       sub_client_expires_at, sub_expert_expires_at,
       sepay_bank_account_xid, bank_account_holder_name, bank_linked_at,
       self_technical BOOLEAN, is_active, created_at)
  ├── client_profiles (user_id PK, company_name, industry, ceo_name)
  ├── expert_profiles (user_id PK, bio, engagement_model,
  │     stack_tags_json JSONB, archetype_history_json JSONB)
  │     ├── expert_domain_depths (expert_id, domain_code, depth_level, verification_tier)
  │     ├── expert_seam_claims (expert_id, seam_code, verification_tier,
  │     │                       submission_count, locked_until TIMESTAMPTZ NULL)
  │     └── portfolio_submissions (expert_id, seam_claim_id, project_description,
  │                                decision_points, status, llm_confidence,
  │                                submitted_at, evaluated_at)
  └── tech_team_profiles (user_id PK, linked_client_id, linked_project_id, role_title)

wallets (id, user_id UNIQUE, available_balance BIGINT DEFAULT 0,
         locked_balance BIGINT DEFAULT 0)
wallet_transactions (id, wallet_id, amount BIGINT, transaction_type TEXT,
                     reference_id TEXT, created_at)
  -- IDEMPOTENCY: CREATE UNIQUE INDEX wallet_tx_idempotency
  --   ON wallet_transactions(wallet_id, reference_id) WHERE reference_id IS NOT NULL

virtual_accounts (id, entity_type TEXT, entity_id TEXT, va_number TEXT UNIQUE,
                  fixed_amount BIGINT NULL, expires_at TIMESTAMPTZ NULL,
                  status TEXT DEFAULT 'ACTIVE')

withdrawal_requests (id, expert_id, type TEXT,  -- MILESTONE_RELEASE | EXPERT_MANUAL
                     amount BIGINT, bank_account_xid TEXT,
                     disbursement_id TEXT NULL, status TEXT,
                     requested_at, confirmed_at NULL)

platform_settings (id, platform_wallet_id UUID UNIQUE, platform_fee_pct FLOAT DEFAULT 0.05,
                   updated_at)  -- singleton; one row seeded at deploy

elicitation_sessions (id, user_id, current_stage INT, archetype, scenario_type,
                      void_list_json JSONB, state TEXT, created_at, updated_at)

projects (id, client_id, elicitation_session_id NULL,
          state TEXT, archetype, tier, self_technical BOOLEAN,
          required_seams_json JSONB,        -- replaces capability_footprints
          required_domains_json JSONB,      -- replaces capability_footprints
          milestone_framework_json JSONB,   -- replaces capability_footprints
          artifact_a_json JSONB NULL,       -- replaces artifact_a table
          artifact_b_json JSONB NULL,       -- replaces artifact_b table
          created_at)
  -- artifact_b_json served by FastAPI ONLY when:
  --   engagement.state >= CONNECTED AND both NDA timestamps set
  --   AND requester active_role IN ('EXPERT','TECH_TEAM')
  --   CEO permanently excluded regardless of engagement state

  └── tech_team_profiles.linked_project_id → projects.id

services (id, expert_id, title, description, domains_json JSONB,
          seams_json JSONB, price_vnd BIGINT, state TEXT,
          service_type TEXT)  -- AI_SERVICE | TECH_DISCOVERY

engagements (id, project_id NULL, expert_id, service_id NULL, type TEXT,
             state TEXT, connected_at NULL, client_nda_accepted_at NULL,
             expert_nda_accepted_at NULL)
  -- CONSTRAINT engagement_type_fk_consistency CHECK (
  --   (type='PROJECT_BASED' AND project_id IS NOT NULL AND service_id IS NULL) OR
  --   (type IN ('SERVICE_PURCHASE','TECH_DISCOVERY') AND project_id IS NULL AND service_id IS NOT NULL)
  -- )

  └── capability_bids (id, engagement_id UNIQUE,
        footprint_alignment_json JSONB, approach_summary TEXT,
        conditional_pricing_json JSONB, state TEXT,
        tech_status TEXT DEFAULT 'PENDING',   -- PENDING | APPROVED | REVISION_REQUESTED
        ceo_status  TEXT DEFAULT 'PENDING',   -- PENDING | APPROVED | DECLINED
        tech_feedback TEXT NULL,              -- replaces bid_revision_requests
        negotiated_price_vnd BIGINT NULL,     -- replaces price_negotiations
        version_number INT DEFAULT 1)

  └── milestones (id, engagement_id, milestone_number, deliverable_statement,
        sign_off_authority TEXT, payment_amount_vnd BIGINT, state TEXT,
        va_number TEXT NULL, va_expires_at TIMESTAMPTZ NULL,
        funded_at, submitted_at, approved_at, released_at)
        -- UNIQUE (engagement_id, milestone_number)

        ├── acceptance_criteria (id, milestone_id, criterion_text,
        │     is_required BOOLEAN, verified_by_role TEXT, verified_at NULL,
        │     revision_note TEXT NULL)   -- replaces revision_requests table

        ├── milestone_dod_items (id, milestone_id, item_description,
        │     is_required BOOLEAN, status TEXT, completed_at NULL,
        │     completion_note NULL, not_applicable_note NULL,
        │     maps_to_criterion_id NULL)
        │   -- CHECK: NOT (is_required = TRUE AND status = 'NOT_APPLICABLE')

        ├── milestone_submissions (id, milestone_id, expert_id,
        │     description, files_json JSONB, submitted_at)

        └── paygated_documents (id, milestone_id, document_url,
              release_state TEXT,  -- STAGED | RELEASED
              staged_at, released_at NULL)

  └── escrow_accounts (id,
        milestone_id UUID NULL,    -- Path A (PROJECT_BASED); exactly one of these two is set
        engagement_id UUID NULL,   -- Path B (SERVICE_PURCHASE / TECH_DISCOVERY)
        amount BIGINT, client_wallet_id, expert_wallet_id,
        status TEXT, held_at, released_at NULL)
      -- CHECK (escrow_has_one_parent): exactly one parent non-null
      -- Partial unique indexes: escrow_milestone_unique + escrow_engagement_unique

  └── disputes (id, engagement_id, milestone_id, criterion_id,
        escrow_account_id,  -- direct FK for O(1) freeze; type-agnostic
        filed_by, state TEXT, llm_confidence FLOAT NULL,
        filed_at, resolved_at NULL)

  └── messages (id, engagement_id, sender_id, content TEXT,
        attachment_url TEXT NULL, timestamp)
        -- No separate thread table; engagement_id IS the thread key
        -- Pre-bid technical questions and DoD discussions also use this channel

  └── message_reads (id, message_id, user_id, read_at)
      -- UNIQUE (message_id, user_id)

  └── reviews (id, engagement_id, reviewer_id, target_id, rating INT,
        comment TEXT NULL, structured_signals_json JSONB NULL,
        reviewer_role TEXT)  -- CEO | TECH_TEAM | EXPERT
      -- UNIQUE (engagement_id, reviewer_id)
      -- structured_signals_json absorbs cut outcome_signals table
      -- Informational only in MVP (no Tier 4 accumulation)

platform_decisions (id, decision_type TEXT, entity_type TEXT, entity_id TEXT,
                    llm_confidence FLOAT NULL, decision TEXT,
                    advisory_note TEXT NULL, created_at)
  -- Written by FastAPI only; never updated after insert
  -- decision_type: ELICITATION_SYNTHESIS | SPEC_AUTO_RETURN | SEAM_TIER_UPGRADE
  --              | PORTFOLIO_EVAL | DISPUTE_L1_EVAL | CRITERION_QUALITY_GATE
```

### 5.4 Service Communication

Same as full spec:
- Frontend → Main API: REST over HTTPS, JWT in Authorization header
- Frontend ↔ Main API (messaging): Socket.io with JWT handshake
- Main API → AI Service: Internal REST
- AI Service → Anthropic API: Standard HTTP (server-side only)
- SePay → Main API (IPN): `POST /webhooks/sepay/ipn` (NestJS public endpoint, HMAC-signed)
- Both services → PostgreSQL: Prisma (NestJS) + SQLAlchemy (FastAPI), same DB, separate connection pools

### 5.5 Deployment

AWS EC2 t3.micro, Docker Compose, Elastic IP. GitHub Actions SSH deploy on merge to `main`. Same configuration as full spec. TV5 owns from Week 8.

---

## 6. WBS & Milestones

### Team Assignments

| Member | Primary responsibilities |
|---|---|
| **TV1 (Minh Hùng)** | FastAPI AI service; LLM integration; elicitation engine; matching engine; portfolio LLM evaluation; criterion quality LLM check; dispute Layer 1 LLM evaluation; Artifact B route guard; `platform_decisions` log writes |
| **TV2 (Chí Nhân)** | NestJS main backend; auth (dual-role + role switcher); all SePay integrations (VA + IPN + Bank Hub + chi hộ); wallet ledger; subscription system; simplified bid state machine (tech_status/ceo_status); milestone state machine (DoD guard); escrow operations; dispute escrow freeze/release; admin routes |
| **TV3 (Cao Minh)** | All feature UI: Path B marketplace, bid review UI, milestone management UI (3 role variants), DoD checklist editor, wallet/subscription UI, reviews, admin dashboard |
| **TV4 (Tuấn Khang)** | SRS, UML diagrams, API documentation, elicitation UI copy, final report |
| **TV5 (Minh Thức)** | Test plan, Postman collections, QA dashboard, GitHub CI/CD, AWS EC2 deployment |

---

### WBS

#### Week 1 — Requirements, Design, Environment

| ID | Task | Owner | Days |
|---|---|---|---|
| 1.1 | Finalize functional requirements from Topic 7 + this scope doc | TV4 | 1 |
| 1.2 | Write SRS (actors, use cases, functional requirements) | TV4 + TV1 | 2 |
| 1.3 | Design PostgreSQL schema — all 28 tables, FKs, state fields | TV1 + TV2 | 2 |
| 1.4 | Design API contracts (all route signatures, request/response schemas) | TV1 + TV2 | 1 |
| 1.5 | Draw UML: Use Case diagram (4 actors), Activity diagrams (UC01 two-actor flow + UC18), Sequence diagrams (elicitation + connection flow) | TV4 | 2 |
| 1.6 | Set up GitHub repo, branching strategy, GitHub Actions CI | TV5 | 1 |
| 1.7 | Set up dev environment: Docker Compose, NestJS + FastAPI + React scaffolds | TV1 + TV2 | 1 |
| 1.8 | Prisma schema + initial migration; SQLAlchemy models + Alembic migration | TV2 (Prisma) + TV1 (SA) | 2 |
| 1.9 | Elicitation UI copy: all prompts, injection explanations, Scenario A/B option text | TV4 | 2 |
| 1.10 | Seed data scripts + Postman workspace setup | TV5 | 1 |

**Week 1 milestone:** Schema locked (28 tables); environment running; SRS drafted; all API contracts signed off; chi hộ tier decision logged

---

#### Week 2 — Auth, Expert Profile, Path B Marketplace, Wallet Foundation

| ID | Task | Owner | Days |
|---|---|---|---|
| 2.1 | F1: Auth — register/login/JWT/role middleware; `roles` JSONB; `active_role`; role switcher JWT reissue; self-exclusion guard | TV2 | 3 |
| 2.2 | F1: Auth frontend — login/register forms, role-based routing, role switcher toggle | TV2 + TV3 | 2 |
| 2.3 | F3: Expert profile CRUD — domain depths, seam claims, stack_tags_json, engagement model, archetype_history_json | TV2 | 3 |
| 2.3b | F1: TECH_TEAM handoff-link registration; `client_subtype` guard; Scenario B `self_technical` flag | TV2 | 2 |
| 2.4 | F3: Expert profile form UI | TV3 | 3 |
| 2.5 | F3: Portfolio evidence submission + LLM auto-verification (Tier 2 only) | TV2 (NestJS + auto-upgrade state machine) + TV1 (FastAPI LLM extraction) | 3 |
| 2.6 | F3: Service listing CRUD + Path B marketplace UI | TV3 | 3 |
| 2.7 | F3: AI Service Generator LLM call (FastAPI) | TV1 | 2 |
| 2.8 | F10/F1.5: SePay setup + wallet foundation — register SePay; verify chi hộ; wallets table; WALLET_TOPUP VA on registration; IPN handler WALLET_TOPUP branch; virtual_accounts table; wallet top-up UI | TV2 | 3 |
| 2.9 | F1.5: Subscription system — activation route (deduct wallet, update users cols); subscription guard middleware; subscription status UI | TV2 (backend + guard) + TV3 (UI) | 3 |
| 2.10 | Test plan for F1 + F3; Postman collections for auth, profile, wallet top-up | TV5 | 2 |

**Week 2 milestone:** Expert creates full taxonomy profile; Path B marketplace browsable; wallet top-up via SePay VA sandbox-tested; subscription activates and gates LLM routes; role switch works

---

#### Week 3 — Elicitation Engine + Milestone VA Funding + Bank Hub

| ID | Task | Owner | Days |
|---|---|---|---|
| 3.1 | F2: LLM extraction layer (FastAPI) — symptom intake → diagnostic state object | TV1 | 4 |
| 3.2 | F2: Guided dialogue state machine (Stages 1–3) | TV1 | 3 |
| 3.3 | F2: SDLC injection logic; Phase 0 hard gate; resistance handler | TV1 | 2 |
| 3.4 | F2: Tech handoff link + Scenario A/B branching — token-based link; TECH_TEAM input form; Scenario A discovery option; Scenario B self_technical form presented to CEO | TV2 (NestJS) + TV1 (FastAPI branch logic) | 3 |
| 3.5 | F2: Synthesis engine — conflict resolution; footprint + artifact JSONB construction; auto-publish gate | TV1 | 3 |
| 3.6 | F2: Elicitation conversational UI — chat-like interface; archetype selection; injection cards; Scenario A/B option screens | TV2 | 4 |
| 3.7 | F2: Tech team handoff UI — secure link landing page; categorical input form | TV2 | 2 |
| 3.8 | F10: Per-milestone VA creation + IPN MILESTONE branch | TV2 | 3 |
| 3.9 | F10: Bank Hub Hosted Link integration; BANK_ACCOUNT_LINKED webhook; bank_account_xid stored | TV2 | 2 |
| 3.10 | Test: AdTech CEO intake → Tier 3 Archetype 1 footprint output | TV5 | 2 |
| 3.11 | Test: VA funding end-to-end sandbox — create VA → scan QR → IPN → ESCROW_LOCK verified | TV5 | 1 |

**Week 3 milestone:** Complete elicitation flow functional with AdTech case; Scenario A/B implemented; milestone VA funding tested; Bank Hub expert linking functional

---

#### Week 4 — Matching Engine + LLM Verification

| ID | Task | Owner | Days |
|---|---|---|---|
| 4.1 | F4: Expert profile query from footprint (FastAPI) | TV1 | 2 |
| 4.2 | F4: Five-component composite scoring (2-tier confidence weights) | TV1 | 3 |
| 4.3 | F4: Hard gate — claimed-to-verified ratio > 4:1 exclusion | TV1 | 1 |
| 4.4 | F4: Seam gap map generation (Tier 2 = Amber, Tier 1 = Yellow, Absent = Red) | TV1 | 2 |
| 4.5 | F4: Shortlist generation — top 3–5 match cards with strength label + gap list | TV1 | 2 |
| 4.6 | F4: Match card UI — seam coverage visual, gap display, strength badge | TV2 | 3 |
| 4.7 | F2 + F12: Auto-return spec logic + Platform Integrity Monitor — NestJS auto-return handler; LLM advisory note; re-entry routing; FastAPI writes `platform_decisions` log | TV2 (NestJS state transitions) + TV1 (LLM advisory + log writes) | 2 |
| 4.8 | Test: Matching engine test matrix (5 synthetic experts × 3 footprints) | TV5 | 2 |

**Week 4 milestone:** Matching engine producing correct ranked shortlists; seam gap maps visible; auto-publish gate functional; portfolio LLM auto-upgrade (Tier 2) verified

---

#### Week 5 — Dual Artifact + Simplified Bid System

| ID | Task | Owner | Days |
|---|---|---|---|
| 5.1 | F5: Artifact A display UI (artifact_a_json rendered from projects row) | TV2 | 2 |
| 5.2 | F5: Connection flow — CEO selects expert; connection request; expert accepts; NDA click-through; Bank Hub prompt if bank not linked | TV2 | 3 |
| 5.3 | F5: Artifact B route guard (FastAPI) — checks engagement.state + NDA timestamps + role before returning artifact_b_json | TV1 | 2 |
| 5.4 | F5: Artifact B display UI (TECH_TEAM + Expert view, post-connection) | TV2 | 2 |
| 5.5 | F5: Pay-gated document staging — expert upload with milestone trigger selector; IPN MILESTONE branch releases docs on FUNDED | TV2 | 3 |
| 5.6 | F6: Capability bid submission form (3 required components; LLM conditional structure validator) | TV3 (form) + TV2 (validation route) | 3 |
| 5.7 | F6: TECH_TEAM bid review — set tech_status (APPROVED / REVISION_REQUESTED); write tech_feedback; bid state machine | TV2 (state machine) + TV3 (TECH_TEAM review UI) | 3 |
| 5.8 | F6: Expert bid revision (edit bid row on REVISION_REQUESTED); CEO review (ceo_status + optional negotiated_price_vnd); CEO review UI | TV2 (routes + guard) + TV3 (CEO review UI) | 2 |
| 5.9 | Test: Full bid flow — submit → TECH revision loop → CEO approval → SELECTED | TV5 | 2 |

**Week 5 milestone:** Complete elicitation-to-connection flow; Artifact B gating verified; simplified bid system with dual-status approval operational; pay-gated document staging functional

---

#### Week 6 — Milestone Management + DoD + Escrow Release

| ID | Task | Owner | Days |
|---|---|---|---|
| 6.1 | F7: Milestone creation form — deliverable statement, acceptance criteria (LLM criterion quality check on save), sign-off authority, payment amount | TV2 (route + LLM check) + TV3 (form UI) | 3 |
| 6.2 | F7: Milestone state transition guards — SUBMITTED guard (DoD check); APPROVED guard (criteria check); both return 422 with structured missing-items list | TV2 | 2 |
| 6.3 | F7: Criterion-referenced review interface — reviewer verifies criterion; revision_note written inline; JOINT milestone waits for both | TV2 (routes) + TV3 (review UI) | 3 |
| 6.4 | F7: Milestone APPROVED → RELEASED — atomic ledger (ESCROW_RELEASE + CREDIT_EXPERT + PLATFORM_FEE); pay-gated docs for next milestone released; chi hộ API called | TV2 | 3 |
| 6.5 | F7: DoD checklist — `milestone_dod_items` table; expert creates/manages after FUNDED; COMPLETED/NOT_APPLICABLE routes with required notes; TECH_TEAM read-only; CEO hidden | TV2 (routes + guard) + TV3 (DoD editor + read-only) | 3 |
| 6.6 | Test: Milestone lifecycle with DoD gate — submit blocked by incomplete DoD; criteria gate blocks APPROVED; JOINT milestone waits; RELEASED triggers chi hộ | TV5 | 2 |

**Week 6 milestone:** Full 2-layer milestone system operational; DoD gate enforced at SUBMITTED; criteria gate enforced at APPROVED; escrow release + chi hộ fires on APPROVED; pay-gated documents release correctly

---

#### Week 7 — Messaging, Service Purchase, Withdrawal, Reviews, Disputes

| ID | Task | Owner | Days |
|---|---|---|---|
| 7.1 | F9: Socket.io server — room per engagement; message persistence; ADMIN read-only join | TV2 | 2 |
| 7.2 | F9: Chat UI — message thread, file attachment, read status (message_reads), notification badge | TV2 | 3 |
| 7.3 | F3: SERVICE_PURCHASE + TECH_DISCOVERY payment flows — per-service-order VA; QR display; IPN SERVICE branch; engagement ACTIVE | TV2 (NestJS + IPN extension) + TV3 (buy service UI) | 3 |
| 7.4 | F10: Expert withdrawal — chi hộ API; atomic WITHDRAWAL ledger; withdrawal_requests PENDING; credit IPN → COMPLETED; FAILED balance restore | TV2 | 3 |
| 7.5 | F10: Transaction history + wallet dashboard — CEO, TECH_TEAM, Expert, Admin role-specific views | TV3 | 2 |
| 7.6 | F1: Dual-role UX — "Add Role" flow; role switcher top-nav; JWT reissue; dashboard re-render | TV2 (routes + JWT reissue) + TV3 (role switcher UI) | 2 |
| 7.7 | F11: Post-engagement review forms — TECH_TEAM structured form (structured_signals_json); CEO form; Expert form; reputation display | TV3 (forms + display) + TV2 (review route + signal storage) | 3 |
| 7.8 | F10 + F12: Dispute escrow — dispute filed → escrow FROZEN; FastAPI Layer 1 LLM evaluation; AUTO_RESOLVED (≥ 0.80) → ledger; MANUAL_REVIEW (< 0.80) → admin queue; Dispute Monitor admin UI with resolve buttons | TV2 (state machine + escrow freeze/release) + TV1 (Layer 1 LLM) + TV3 (Dispute Monitor UI) | 4 |
| 7.9 | Test: Withdrawal end-to-end; SERVICE_PURCHASE VA flow; dual-role switch; dispute auto-resolve and manual resolve paths | TV5 | 2 |

**Week 7 milestone:** Messaging live; SERVICE_PURCHASE payment functional; expert withdrawal automated; dual-role switch working; dispute 2-layer resolution verified (LLM auto-resolve + admin manual resolve)

---

#### Week 8 — Admin Dashboard, AWS Deployment, Integration, Bug Fix

| ID | Task | Owner | Days |
|---|---|---|---|
| 8.1 | F12: Admin account management — list, filter, suspend/reactivate | TV2 | 2 |
| 8.2 | F12: Platform Integrity Monitor UI — reads platform_decisions log; auto-return history; verification log; dispute resolution log | TV2 (data aggregation + spec pull-back route + fraud detection) + TV3 (UI) | 3 |
| 8.3 | F12: Analytics dashboard — active projects, elicitation rates, auto-upgrade rates, dispute rates, milestone completion | TV3 | 3 |
| 8.4 | F12: Transaction ledger UI — wallet_transactions + escrow_accounts + withdrawal_requests read-only; date/type filters | TV3 | 2 |
| 8.5 | AWS EC2 deployment — provision t3.micro; Elastic IP; Docker + Docker Compose; docker-compose.prod.yml; SePay IPN endpoint configured | TV5 | 2 |
| 8.6 | GitHub Actions CD — SSH deploy on merge to main; test deploy without downtime | TV5 | 1 |
| 8.7 | Full system integration test on EC2 — end-to-end: intake → elicitation → publish → shortlist → bid → connection → fund → submit → approve → RELEASED → review | TV5 | 3 |
| 8.8 | Bug fixing sprint — all P1 (blocking) and P2 (major) bugs from 8.7 | TV1 + TV2 + TV3 | 3 |
| 8.9 | Edge case hardening — duplicate IPN idempotency; amount mismatch handling; link expiry; Artifact B access before connection (blocked); submission without DoD (blocked); claim lockout re-enable (30-day timer) | TV1 + TV2 | 2 |
| 8.10 | Documentation update — UML sequence diagrams; API docs; EC2 deployment guide | TV4 | 3 |

**Week 8 milestone:** Admin dashboard complete; dispute manual resolution via button verified; AWS EC2 deployment live; full integration test on EC2 passed including real SePay sandbox payment; all P1 bugs resolved

---

#### Week 9 — Demo, Final Documentation, Submission

| ID | Task | Owner | Days |
|---|---|---|---|
| 9.1 | Demo script — step-by-step AdTech walkthrough; live SePay QR payment moment; 15-minute timing | TV4 + TV1 | 2 |
| 9.2 | Demo data seed on EC2 — realistic expert profiles (Tier 1 + Tier 2 seams); AdTech CEO intake text; expected footprint + shortlist | TV5 | 2 |
| 9.3 | Internal demo run on EC2 — full script against live cloud; identify blocking issues | All | 1 |
| 9.4 | Final bug fixes from internal demo | TV1 + TV2 + TV3 | 2 |
| 9.5 | Final report — system overview, research questions and findings, architecture, demo evidence, research contribution | TV4 | 3 |
| 9.6 | Slide deck preparation | TV4 | 2 |
| 9.7 | Switch SePay sandbox → production; update EC2 .env; verify IPN | TV2 + TV5 | 0.5 |
| 9.8 | Final submission — report, slides, code repository link, EC2 URL | All | 1 |

**Week 9 milestone:** Live demo on AWS EC2 with real SePay VietQR payment moment; demo rehearsed; final report submitted

---

### Milestone Summary

| Week | Key Deliverables | Success Condition |
|---|---|---|
| **1** | Schema (28 tables), API contracts, SRS, dev environment | Both services start; schema migrated; chi hộ tier confirmed |
| **2** | Expert profiles, Path B marketplace, wallet top-up VA→IPN→credited, subscription gate | Wallet sandbox verified; subscription guards LLM routes |
| **3** | Elicitation engine (Scenario A/B); milestone VA funding; Bank Hub linking | AdTech footprint + artifact JSONBs correct; VA fixed-amount enforced; Bank Hub linking functional |
| **4** | Matching engine, seam gap maps, 2-tier verification, auto-return spec | Test matrix rankings correct; LLM Tier 2 auto-upgrade at ≥0.85 |
| **5** | Connection flow, Artifact B gating, simplified bid dual-status approval, pay-gated docs | Artifact B blocked until CONNECTED + NDA; TECH_TEAM must approve before CEO selects |
| **6** | 2-layer milestone (criteria + DoD); escrow release + chi hộ fires on APPROVED | DoD 422 guard; criteria 422 guard; chi hộ API called; paygated docs release |
| **7** | Messaging; SERVICE_PURCHASE VA; chi hộ withdrawal; dispute 2-layer; reviews | Expert withdrawal COMPLETED; dispute auto-resolves or routes to admin queue |
| **8** | AWS EC2 live; admin dashboard; GitHub CD; full integration test | Platform on Elastic IP; sandbox end-to-end payment verified; P1/P2 bugs resolved |
| **9** | Production SePay; live demo; final report; slides | Demo shows real VND wallet top-up → milestone funded → expert withdrawal completed |

---

### Research Question Mapping

| Research Question | Evidence generated by | When collected |
|---|---|---|
| **RQ1:** Improving matching accuracy | Composite score distribution; seam gap map accuracy; Tier 1 vs Tier 2 match outcomes; bid revision frequency (fewer revisions = cleaner initial bids) | Admin Analytics from Week 8 |
| **RQ2:** AI-assisted scope definition | Elicitation completion rate; auto-publish pass rate; Phase 0 injection acceptance rate; DoD item count per milestone (more = better expert self-scoping) | `platform_decisions` + elicitation logs from Week 3 |
| **RQ3:** Trust factors in AI marketplaces | Connection acceptance rate per match strength; milestone completion rate; dispute frequency + Layer 1 auto-resolution rate; pay-gated doc release timing; chi hộ withdrawal confirmation speed (trust in automated payout) | Reviews + wallet_transactions + disputes from Week 7 |

---

### Critical Path

**F2 (Weeks 3–4) is the top of the critical path.** Matching (F4), Artifact B (F5), bid system (F6), milestones (F7) all depend on the elicitation engine producing a valid footprint and artifact JSONBs. TV1 treats this as a focused sprint. TV4 must deliver all elicitation UI copy by end of Week 1.

**F10 wallet/VA system (Weeks 2–3) is the second critical path.** Every financial operation depends on it. chi hộ tier verification must happen Week 1 Day 1.

**F6 simplified bid system (Week 5) is not on the critical path.** It can slip one week without blocking F7, because F7 can begin once any engagement reaches CONNECTED state.

**AWS EC2 deployment is TV5's responsibility from Week 8.** Do not defer to Week 9.

### Scope Reduction Trade-offs (for final report)

The following features were cut to achieve 9-week feasibility. Each can be reintroduced as Phase 2 with a single Alembic + Prisma migration:

| Cut | Trade-off | Phase 2 path |
|---|---|---|
| Tiers 3 + 4 (scenario assessments + signal accumulation) | Experts max out at Tier 2; matching confidence weights capped at 0.55 | Add `scenario_assessments` + `scenario_responses` + `expert_seam_outcome_signals` tables |
| Sprint tracking (Layer 3) | Scope evolution is handled manually (CEO creates new milestone) | Add `milestone_sprints` + `sprint_status_updates` tables |
| Add-On Phase Protocol | No automated scope evolution detection | Add `addon_phase_requests` table |
| Versioned bid history | Bid revisions are mutable row updates; no immutable audit trail | Add `bid_versions` + `bid_revision_requests` tables; revert `tech_status` col approach |
| Automated 48h dispute resolution | Admin manually resolves MANUAL_REVIEW disputes | Add Layer 2 mutual agreement form + automated 50/50 Layer 3 split |
| Organizations | No org billing or multi-member accounts | Add `organizations` + `organization_members` tables |
| Revision_requests table | Criterion revision feedback stored as `revision_note` col | Separate into `revision_requests` table for full per-revision audit trail |

