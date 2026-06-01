# AITasker — Internal Scope Document
### SWP391 · Topic 7 · FPT University HCMC · 9-Week Build Plan

> **Purpose:** Internal reference document for the team. Not for submission as-is.  
> **Last updated:** May 2026  
> **Status:** Scoped for 9-week execution

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

> **Purpose:** Ground-truth lookup for all taxonomy codes, seam identifiers, scoring weights, and state machine definitions referenced throughout this document. When any other section uses a code (e.g. A↔C, Archetype 2, Tier 3, verification Tier 4), this is the canonical definition. Do not paraphrase — refer back here.

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

Seams are cross-domain competence boundaries. An expert who holds both domains on either side of a seam may still lack the seam knowledge — the skill of diagnosing and resolving failures *at* the boundary. Seam coverage is the primary differentiator in the composite match score.

| Seam | Name | What failure looks like without it |
|---|---|---|
| **A↔C** | Ground truth-driven iteration | Expert fine-tunes prompts with no baseline — can't tell if changes help or hurt |
| **A↔F** | Hybrid routing | LLM handles everything including cases it should defer to a trained classifier — high cost, low accuracy |
| **A↔D** | Retrieval-generation contract | Retrieved chunks are semantically right but structurally wrong for the prompt — LLM produces hallucinated synthesis |
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

| # | Archetype | Core question the client is asking | Primary domains | Load-bearing seam | Signature seams |
|---|---|---|---|---|---|
| **1** | Automated Decision System | *"Can AI replace or augment a human decision my team makes repeatedly?"* | C, E, A | **A↔C** | C↔E, D↔E |
| **2** | RAG / Knowledge Retrieval | *"Can AI answer questions from our documents or knowledge base?"* | A, D, B | **A↔D** | A↔B, D↔E |
| **3** | Predictive / Forecasting | *"Can AI predict an outcome from our historical data?"* | F, C, E | **E↔F** | C↔F, D↔F |
| **4** | Data Pipeline + AI Integration | *"Can we connect AI into our existing high-volume data infrastructure?"* | E, B, D | **D↔E** | B↔E, C↔E |
| **5** | Model Fine-Tuning & Customization | *"Can we make a base model specialize on our domain's data?"* | F, A, C | **A↔F** | C↔F, E↔F |
| **6** | Evaluation Infrastructure Build | *"How do we know if our AI is actually working?"* | C, E, F | **A↔C** | C↔F, C↔E |

**Tiers** — how complex the project is:

| Tier | Label | What it means | Infrastructure signals |
|---|---|---|---|
| **1** | Simple / Greenfield | No existing system, low volume, fresh start | HTTP REST only, <100K records, no legacy |
| **2** | Moderate | One existing system integration, medium volume | One integration point, 100K–1M records, some legacy |
| **3** | Complex / Production | Deep existing infrastructure, high volume, multiple integrations | Kafka/async, >1M records, legacy migration required, multi-system |

**AdTech demo case = Tier 3, Archetype 1:** An automated ad compliance decision system running at ~600K ads/month, integrated into a Kafka pipeline with a Go Worker orchestrator and Milvus vector index.

---

### 0.4 Expert Verification Tiers

| Tier | Label | How earned | Confidence factor in composite score |
|---|---|---|---|
| **1** | Claimed | Self-declared on profile | 0.20 |
| **2** | Evidence-backed | LLM extraction confidence ≥ 0.85 on portfolio submission; all required signal types found | 0.55 |
| **3** | Scenario-verified | LLM rubric evaluation: all required dimensions individually pass the per-question rubric | 0.80 |
| **4** | Platform-demonstrated | Expert completed a real engagement where this seam was load-bearing / significant / contributing, with a positive outcome signal satisfying the signal accumulation rule | 0.95 |

**Signal accumulation rule for Tier 4 (DB-evaluated after every engagement close — no admin action):**
- 1× positive outcome signal from a **load-bearing** seam role → immediate Tier 4
- 2× positive outcome signals from **significant** seam roles (across any engagements) → Tier 4
- 3× positive outcome signals from **contributing** seam roles (across any engagements) → Tier 4

---

### 0.5 Composite Match Score Weights

| Component | Weight | Formula element |
|---|---|---|
| Seam alignment | **40%** | Per seam: criticality weight × expert's verification confidence factor. Missing load-bearing seam = negative contribution |
| Domain depth coverage | **25%** | Expert verified depth ≥ footprint required depth for each required domain |
| Archetype-tier congruence | **20%** | Has expert completed engagements of same archetype + tier before? |
| Engagement model fit | **10%** | Does expert's declared model (advisory / spec+review / full implementation) match footprint? |
| Stack tags & recency | **5%** | Stack tag overlap, weighted by recency |

**Match strength labels** (client sees label, not the numeric score):
- **Strong** → composite score > 0.78
- **Qualified** → 0.58–0.78
- **Conditional** → 0.42–0.58

---

### 0.6 All State Machines

> Every state name used anywhere in this document resolves to one of the machines below. State transitions that trigger ledger entries are marked **[LEDGER]**. State transitions that call external APIs are marked **[API]**.

---

**Elicitation session states:**
```
IN_PROGRESS → current_stage advances (1→2→3→4→5) as CEO and TECH_TEAM complete each stage
COMPLETED   → Stage 5 synthesis produced footprint + Artifact A; project row created
ABANDONED   → CEO navigated away; can resume from current_stage
RETURNED    → auto-publish quality gate failed; CEO re-enters at current_stage (not stage 1)
             spec.state = RETURNED_TO_CLIENT; LLM advisory note generated; same session row reused
```

**Milestone states:**
```
DEFINED           → milestone created, acceptance criteria + DoD items + sprint plan set
AWAITING_PAYMENT  → CEO clicked "Fund"; per-milestone VA generated (fixed_amount set); VietQR displayed
FUNDED            → SePay IPN confirmed (credit on milestone VA) [LEDGER: escrow lock]
IN_PROGRESS       → auto-advances from FUNDED; Expert begins sprint work
SUBMITTED         → Expert submits deliverable; all required DoD items COMPLETED; review clock starts
                    Guard: SELECT FROM milestone_dod_items WHERE is_required=true AND status!='COMPLETED'
                    → any rows → 422 with unchecked item list; submission blocked
IN_REVISION       → criterion-referenced revision requested; revision counter increments
APPROVED          → all required acceptance_criteria have verified_at set
                    Guard: SELECT FROM acceptance_criteria WHERE is_required=true AND verified_at IS NULL
                    → any rows → 422 with unverified criteria list
                    On transition: [LEDGER: escrow release at platform_settings.platform_fee_pct]
                    [API: chi hộ fires to expert's bank_account_xid]
RELEASED          → chi hộ credit IPN confirmed on expert's linked account; ledger settled
                    milestones.review_clock_paused_at = NULL; total_paused_seconds finalized
DISPUTED          → dispute filed (disputes table row created; escrow_accounts.status → FROZEN)
  Dispute resolved via:
  → disputes.state = AUTO_RESOLVED       (Layer 1 LLM ≥ 0.80) [LEDGER: winner receives escrow]
  → disputes.state = MUTUAL_RESOLVED     (Layer 2 party agreement) [LEDGER: per agreed split]
  → disputes.state = DISPUTED_AUTO_RESOLVED (Layer 3: 50/50) [LEDGER: equal split]
```

**Bid states (non-linear — supports push-back):**
```
DRAFT
  → SUBMITTED (expert submits; all 3 components required; conditional_pricing_json validated)
  → TECH_REVIEW (TECH_TEAM opens review)
  → REVISION_REQUESTED (TECH_TEAM flags component via bid_revision_requests row)
  → REVISED (expert edits flagged component only; bid_versions row written with revision_request_id FK)
  [TECH_REVIEW / REVISION_REQUESTED / REVISED loop — unlimited rounds]
  → TECH_APPROVED  (flag: Recommended)
  → TECH_DISAPPROVED (flag: Not Recommended; requires reason text in bid_revision_requests)
  → CONFLICT_PENDING (CEO wants to proceed despite TECH_DISAPPROVED)
  → CONFLICT_RESOLVED_CEO_OVERRIDE (bid_conflict_overrides row written; CEO_REVIEW unlocks)
  → CEO_REVIEW
  → PRICE_NEGOTIATION (optional; price_negotiations rows; max 2 rounds via round_number CHECK)
  → NEGOTIATION_COMPLETE
  → SELECTED / DECLINED

Bid revision versioning: every REVISED transition writes a bid_versions row with
  version_number (increments on capability_bids), snapshot_json, revision_request_id FK (links
  snapshot to the specific request that triggered it — NULL on version 1 = original submission).
```

**Spec clarification states (pre-bid push-back, Surface A):**
```
OPEN     → expert submits; target_component required; responded_by = NULL
ANSWERED → TECH_TEAM or CEO responds; responded_by FK set
RESOLVED → expert marks resolved
Multiple clarifications can be OPEN simultaneously from the same expert.
status CHECK: ('OPEN','ANSWERED','RESOLVED') — 'CLOSED' is NOT a valid value.
```

**Spec states (also tracked on artifact_a.state independently from projects.state):**
```
DRAFT → [auto-publish quality gate] → PUBLISHED
                      ↓ (fail)
            RETURNED_TO_CLIENT (auto — elicitation_sessions.state = RETURNED;
                                LLM advisory note generated; session resumes at failing stage)
PUBLISHED → SUSPENDED (admin emergency pull-back only — passive, reactive)
```

**Engagement states:**
```
PENDING   → connection request sent; both NDA timestamps NULL
CONNECTED → expert accepted; client_nda_accepted_at + expert_nda_accepted_at BOTH set
             Artifact B becomes queryable (DB guard checks both timestamps)
             Bank Hub link check: if expert.sepay_bank_account_xid IS NULL → prompt to link
ACTIVE    → first milestone FUNDED
CLOSED    → all milestones RELEASED or DISPUTED_AUTO_RESOLVED
           → NestJS evaluates expert_seam_outcome_signals Tier 4 accumulation rule
DISPUTED  → active dispute on any milestone (engagement continues for other milestones)

Note: SERVICE_PURCHASE and TECH_DISCOVERY engagements skip PENDING → created directly in ACTIVE.
```

**Engagement type (set at creation; immutable; enforced by table-level CHECK):**
```
PROJECT_BASED    — project_id NOT NULL; service_id NULL
SERVICE_PURCHASE — project_id NULL; service_id NOT NULL
TECH_DISCOVERY   — project_id NULL; service_id NOT NULL (TECH_DISCOVERY service type)
```

**Add-on phase request states:**
```
PENDING         → expert submits brief; trigger_sprint_status_id FK set (from SCOPE_EVOLUTION event)
TECH_APPROVED   → TECH_TEAM approves technical justification; tech_approved_at set
AWAITING_CEO    → TECH_TEAM approved; routing to CEO (UI label only — no DB state change)
FULLY_APPROVED  → CEO approves budget; ceo_approved_at set → NestJS creates new milestone row
REJECTED        → either party rejects; rejected_at + rejected_by + rejection_reason set
MILESTONE_CREATED → new milestone row written; created_milestone_id FK set

Unique constraint: one addon_phase_requests row per sprint_status_updates.id
  (prevents double-trigger if expert files multiple SCOPE_EVOLUTION updates)
```

**Sprint status (weekly — Expert submits; no payment consequence):**
```
ON_TRACK   → acknowledged; no action required
DEVIATION  → client notified; expert provides adjusted timeline; milestone clock unchanged
BLOCKER    → milestones.review_clock_paused_at = now(); milestones.state paused
  blocker_type: SCOPE_GAP | TECH_CONSTRAINT | EXTERNAL_DEPENDENCY
    → TECH_TEAM notified; TECH_TEAM marks resolved via PUT /sprint-status/{id}/resolve
    → NestJS: milestones.total_paused_seconds += (now - review_clock_paused_at);
              milestones.review_clock_paused_at = NULL; clock resumes
  blocker_type: SCOPE_EVOLUTION
    → scope_evolution_flag = true; F8 Add-On Phase triggered automatically
    → milestone clock paused until F8 FULLY_APPROVED or REJECTED
```

**DoD checklist item states:**
```
PENDING        → item created; not yet actioned
COMPLETED      → expert marks done; completion_note required
NOT_APPLICABLE → expert marks NA; not_applicable_note required
               → ONLY valid when is_required = FALSE (DB-level CHECK constraint:
                 NOT (is_required = TRUE AND status = 'NOT_APPLICABLE'))
Required items: all must reach COMPLETED before milestone SUBMITTED transition is permitted.
```

**Wallet transaction types (internal ledger — no external movement):**
```
TOP_UP          — SePay IPN (credit on WALLET_TOPUP VA) → available_balance += amount [LEDGER]
SUBSCRIPTION    — IPN on SUBSCRIPTION VA; user_subscriptions.status PENDING_PAYMENT→ACTIVE [LEDGER]
ESCROW_LOCK     — CEO funds milestone → available_balance -= amount; locked_balance += amount [LEDGER]
ESCROW_RELEASE  — milestone APPROVED → locked_balance -= amount; expert available_balance += net [LEDGER]
                  fee = amount * platform_settings.platform_fee_pct (read from DB, not hardcoded)
PLATFORM_FEE    — platform_settings.platform_wallet_id receives fee amount [LEDGER]
ESCROW_REFUND   — dispute → client available_balance += amount [LEDGER]
ESCROW_SPLIT    — Layer 3 50/50 → both parties available_balance += amount/2 [LEDGER]
WITHDRAWAL      — expert requests cash-out → available_balance -= amount [LEDGER]
                  type field on withdrawal_requests: MILESTONE_RELEASE (auto) | EXPERT_MANUAL (cash-out)
                  [API: SePay chi hộ fires to expert's bank_account_xid; IPN confirms]

Idempotency: wallet_transactions has UNIQUE INDEX on (wallet_id, reference_id) WHERE reference_id IS NOT NULL.
Two simultaneous SePay IPN retries cannot both INSERT — the second fails the unique constraint.
```

**Withdrawal states (fully automated via SePay Bank Hub + chi hộ):**
```
PENDING     → withdrawal_request created; wallet debited atomically; chi hộ API called [API]
PROCESSING  → chi hộ API accepted (disbursement_id set); awaiting SePay credit IPN
COMPLETED   → SePay credit IPN fires on expert's linked bank account; confirmed_at set; ledger settled
FAILED      → chi hộ API error; wallet balance restored [LEDGER]; expert notified
```

**Subscription states:**
```
PENDING_PAYMENT → user_subscriptions row created; SUBSCRIPTION VA generated with row id as entity_id;
                  IPN not yet received; subscription guard still blocks gated routes
ACTIVE          → SePay IPN confirms payment; activated_at set; expires_at = now + 6 months
EXPIRING_SOON   → 7 days before expires_at; notification sent
EXPIRED         → past expires_at; downgrade to free tier; active engagements grandfathered
                  → ACTIVE on renewal (new SUBSCRIPTION VA created; new IPN cycle)
```

**Dispute resolution layers:**
```
disputes row created → escrow_accounts.status → FROZEN (no balance move)
  → disputes.state = LAYER_1_EVAL
      LLM evaluates criterion vs. deliverable; disputes.llm_confidence set
      confidence ≥ 0.80 → disputes.state = AUTO_RESOLVED; resolution_layer = 1
                           [LEDGER: escrow distributed per LLM finding]
      confidence < 0.80 → disputes.state = LAYER_2_MUTUAL
                           disputes.layer2_deadline = now() + 48h

  → disputes.state = LAYER_2_MUTUAL (48h cooling + mutual agreement form)
      both parties agree same option → disputes.state = MUTUAL_RESOLVED; resolution_layer = 2
                                        [LEDGER: per agreed split]
      48h expire with no agreement → disputes.state = LAYER_3_FORCED

  → disputes.state = LAYER_3_FORCED → DISPUTED_AUTO_RESOLVED; resolution_layer = 3
      [LEDGER: 50/50 split]
      dispute_resolution_reports available for either party to submit (informational only)
```

---

### 0.7 RBAC Roles — The Definitive Constraint

> **Read this before building any route, guard, or UI panel.** Every access control decision traces back here.

**The platform has four JWT account types, with one account supporting multiple roles simultaneously.** Dual-role accounts are the designed path for AI practitioners who both seek and provide services.

| JWT `role` / `active_role` | `client_subtype` | Human label | How created |
|---|---|---|---|
| `CLIENT` (active) | `CEO` | Client — Business (CEO) | Self-register: "I need AI help" |
| `CLIENT` (active) | `TECH_TEAM` | Client — Technical | Via signed handoff link only — cannot self-register |
| `EXPERT` (active) | *(none)* | Expert | Self-register: "I provide AI services" |
| `ADMIN` (active) | *(none)* | Admin | DB-seeded only; no registration path |

**Dual-role accounts:** A single user account can hold `roles: ["CLIENT_CEO", "EXPERT"]` simultaneously. The `active_role` JWT field determines which dashboard and guards apply. Role-switching reissues JWT without re-login. Self-exclusion rule: matching engine always filters `expert.user_id NOT IN project.involved_parties` regardless of active role.

**Self-technical flag:** When a CEO indicates "I am also the technical decision-maker" during project setup, `project.self_technical = true`. For that project only: the CEO account gains TECH_TEAM capabilities (Artifact B access, bid technical review, technical milestone sign-off). No second account needed. Surface D conflict resolution is skipped for self-technical projects.

**Technical Discovery path:** When elicitation Stage 4 is unreachable (no TECH_TEAM available), a `TECH_DISCOVERY` engagement type is offered instead. The all-non-technical CEO still proceeds; experts include architecture discovery in their first milestone.

**Complete permission matrix:**

| Action | CLIENT / CEO | CLIENT / TECH_TEAM | EXPERT | ADMIN |
|---|---|---|---|---|
| Register account | ✅ (self-register) | ❌ (handoff link only) | ✅ (self-register) | ❌ (seeded only) |
| Add second role to existing account | ✅ (add EXPERT role) | ❌ | ✅ (add CLIENT_CEO role) | ❌ |
| Submit project via elicitation engine | ✅ | ❌ | ❌ | ❌ |
| Complete tech team architecture handoff | ❌ | ✅ | ❌ | ❌ |
| Complete Stage 4 as self-technical CEO | ✅ (if self_technical=true) | N/A | ❌ | ❌ |
| Top up platform wallet | ✅ | ❌ | ✅ | ❌ |
| Purchase subscription (Client Pro) | ✅ | ❌ | ❌ | ❌ |
| Purchase subscription (Expert Pro) | ❌ | ❌ | ✅ | ❌ |
| View Artifact A | ✅ | ✅ | ✅ (matched projects only) | ✅ |
| View Artifact B | ❌ (never) | ✅ (state ≥ CONNECTED) | ✅ (state ≥ CONNECTED) | ✅ |
| File spec clarification request (Surface A) | ❌ | ❌ | ✅ (pre-bid only) | ❌ |
| Respond to spec clarification | ❌ | ✅ (Tier 2+ projects) | ❌ | ❌ |
| Review bids — detailed seam analysis | ❌ | ✅ | ❌ | ❌ |
| Flag bid Recommended / Not Recommended | ❌ | ✅ | ❌ | ❌ |
| Request bid component revision (Surface B) | ❌ | ✅ | ❌ | ❌ |
| Revise bid (Surface B) | ❌ | ❌ | ✅ | ❌ |
| Approve expert selection (final) | ✅ | ❌ | ❌ | ❌ |
| Override TECH_TEAM disapproval with reason (Surface D) | ✅ | ❌ | ❌ | ❌ |
| Propose price adjustment (Surface C) | ✅ | ❌ | ❌ | ❌ |
| Accept / counter / decline price proposal (Surface C) | ❌ | ❌ | ✅ | ❌ |
| Send connection request to expert | ✅ | ❌ | ❌ | ❌ |
| Accept / decline connection request | ❌ | ❌ | ✅ | ❌ |
| Link bank account via Bank Hub Hosted Link | ❌ | ❌ | ✅ | ❌ |
| Fund milestone (wallet or VietQR) | ✅ | ❌ | ❌ | ❌ |
| Create sprint plan for milestone | ❌ | ❌ | ✅ | ❌ |
| Update DoD checklist items | ❌ | ❌ | ✅ | ❌ |
| Submit weekly sprint status update | ❌ | ❌ | ✅ | ❌ |
| Comment on sprint (TECH_TEAM feedback) | ❌ | ✅ | ❌ | ❌ |
| Submit milestone deliverable | ❌ | ❌ | ✅ | ❌ |
| Sign off technical milestones | ❌ | ✅ | ❌ | ❌ |
| Sign off business / budget milestones | ✅ | ❌ | ❌ | ❌ |
| Sign off joint milestones | ✅ | ✅ (both required) | ❌ | ❌ |
| Receive pay-gated reasoning documents | ❌ | ✅ | ❌ | ❌ |
| Approve add-on phase — technical justification | ❌ | ✅ | ❌ | ❌ |
| Approve add-on phase — budget & timeline | ✅ | ❌ | ❌ | ❌ |
| File a dispute | ✅ | ✅ | ✅ | ❌ |
| Browse expert service marketplace (Path B) | ✅ | ✅ | ❌ | ❌ |
| Buy expert service (SERVICE_PURCHASE / TECH_DISCOVERY) | ✅ | ❌ | ❌ | ❌ |
| Publish service listing (Path B) | ❌ | ❌ | ✅ | ❌ |
| Request withdrawal (triggers chi hộ API) | ❌ | ❌ | ✅ | ❌ |
| Post-engagement review | ✅ (CEO form) | ✅ (Tech Team form) | ✅ (Expert form) | ❌ |
| Real-time messaging | ✅ | ✅ | ✅ | ❌ (view-only) |
| Emergency spec pull-back | ❌ | ❌ | ❌ | ✅ |
| Account suspension | ❌ | ❌ | ❌ | ✅ |
| View Platform Integrity Monitor | ❌ | ❌ | ❌ | ✅ |


---

### 0.8 Payment Architecture — Internal Wallet + SePay VA + Bank Hub + Chi Hộ

> **Zero manual choke points.** Every financial event in the platform lifecycle is either a pure internal DB ledger entry (no external call) or a SePay API call with a webhook confirmation. Admin never touches money.

**Architecture overview:**

```
INBOUND  — User tops up wallet:
  User scans per-user VA QR → bank transfer → SePay IPN (credit on VA) → NestJS credits wallet

INBOUND  — CEO funds milestone:
  CEO scans per-milestone VA QR → bank transfer → SePay IPN (credit on milestone VA)
  → NestJS: atomic ledger [ESCROW_LOCK]: available_balance -= amount; locked_balance += amount
  → escrow_accounts record created; milestone → FUNDED

INTERNAL — Milestone released (sign-off authority approves):
  NestJS atomic ledger:
    [ESCROW_RELEASE]: client locked_balance -= amount
    [PLATFORM_FEE]:   platform_fees += amount * 0.05
    [CREDIT_EXPERT]:  expert available_balance += amount * 0.95
  → milestone → APPROVED → chi hộ call fires [API]

OUTBOUND — Expert withdrawal (fully automated, no admin):
  Expert requests withdrawal → NestJS: [WITHDRAWAL] available_balance -= amount
  → SePay chi hộ API POST: { amount, bank_account_xid, reference: "WD-{id}" }
  → SePay transfers to expert's verified linked account
  → SePay credit IPN fires on expert's linked account → withdrawal → COMPLETED

INTERNAL — Subscription payment:
  User wallet balance checked → [SUBSCRIPTION] available_balance -= price
  → subscription_tier = 'pro'; subscription_expires_at = now + 6 months
```

**Key tables (full schema in Section 5.3):**
- `wallets` — one per user; `available_balance`, `locked_balance` in VND
- `wallet_transactions` — immutable ledger entries; every balance change is a row here
- `virtual_accounts` — per-entity VA number; `entity_type` (WALLET_TOPUP | MILESTONE | SUBSCRIPTION | SERVICE); VA maps to platform's real bank account
- `expert_profiles.sepay_bank_account_xid` — set when expert completes Bank Hub Hosted Link; used as target for all chi hộ calls

**SePay integration points used:**
1. **VietQR + per-VA IPN** — inbound payments (wallet top-up, milestone funding, subscription, service purchase); personal bank account suffices; 30-min activation
2. **Bank Hub Hosted Link** — expert links their bank account via SePay's secure WebView (OTP-only, no password sharing); on `BANK_ACCOUNT_LINKED` webhook, `bank_account_xid` stored
3. **Chi hộ API** — programmatic outbound disbursement; SePay initiates transfer to expert's `bank_account_xid`; confirmation via credit IPN on expert's account

**Subscription payment (corrected flow):**
```
1. User clicks "Subscribe" → NestJS creates user_subscriptions row { status: PENDING_PAYMENT }
2. NestJS creates SUBSCRIPTION VA { entity_id: user_subscriptions.id, fixed_amount: price }
3. User scans VietQR → IPN fires → NestJS IPN SUBSCRIPTION branch:
   UPDATE user_subscriptions SET status='ACTIVE', activated_at=now(), expires_at=now()+6months
   UPDATE users SET subscription_{role}_tier='pro', sub_{role}_expires_at=...
   DEBIT wallet (SUBSCRIPTION ledger entry)
Note: subscription.id is used as entity_id (not user_id) so the VA can be matched to
the correct pending subscription row without ambiguity on IPN arrival.
```

**Platform fee:** Read from `platform_settings.platform_fee_pct` at transaction time — not hardcoded in application code. Default 0.05 (5%). Can be changed via DB without a code deploy.

---

### 0.9 Subscription Tiers & Feature Gates

> Every route calling FastAPI (LLM) or the matching engine checks subscription before processing. Guard returns HTTP 403 `{ code: "SUBSCRIPTION_REQUIRED", feature: "...", upgrade_url: "/subscribe" }` on failure.

| Feature | Client Free | Client Pro (500K VND / 6 mo) | Expert Free | Expert Pro (300K VND / 6 mo) |
|---|---|---|---|---|
| Browse expert marketplace (Path B) | ✅ | ✅ | ❌ | ❌ |
| Buy service directly (SERVICE_PURCHASE) | ✅ | ✅ | ❌ | ❌ |
| Buy TECH_DISCOVERY session | ✅ | ✅ | ❌ | ❌ |
| AI Elicitation Engine (F2, all stages) | ❌ | ✅ | ❌ | ❌ |
| Matching engine / shortlist (F4) | ❌ | ✅ | ❌ | ❌ |
| Artifact B access post-connection | ❌ | ✅ | ❌ | ❌ |
| Add-On Phase Protocol (F8) | ❌ | ✅ | ❌ | ❌ |
| Full seam gap maps on match cards | ❌ (domain-only) | ✅ | ❌ | ❌ |
| Create expert profile + Tier 1 seam claims | ❌ | ❌ | ✅ | ✅ |
| LLM portfolio evidence verification (Tier 2) | ❌ | ❌ | ❌ | ✅ |
| Scenario assessment access (Tier 3) | ❌ | ❌ | ❌ | ✅ |
| Bid on Tier 2–3 projects | ❌ | ❌ | ❌ | ✅ |
| AI Service Generator | ❌ | ❌ | ❌ | ✅ |
| Earnings analytics dashboard (full) | ❌ | ❌ | ❌ | ✅ |

**Dual-role users** hold two separate `user_subscriptions` rows (one per role type). Both deduct from the same wallet.

**Expiry handling:** 7-day advance notification → on expiry, downgrade to free tier → active in-progress engagements grandfathered (expert can still submit deliverables; CEO can still fund; payment-critical paths continue) → new gated features blocked until renewal.

---

## 1. System Overview


The AI services market has a double-sided failure:

**Client side:** Non-technical founders and businesses cannot accurately describe what they need. They post vague symptom descriptions ("our AI output is inconsistent") that attract mismatched experts. Existing platforms (Upwork, Fiverr) have no mechanism to transform a symptom into a structured, matchable requirement. The result is wrong expert selection, scope explosion mid-project, and wasted budget.

**Expert side:** AI experts cannot demonstrate the specific intersection capabilities that determine project success — knowing how to do ML fine-tuning is not the same as knowing how to fine-tune when there is no ground truth, when the data lives in a Kafka pipeline, and when the client's evaluation baseline does not yet exist. Generic platforms match on discipline labels (keywords like "machine learning"), not on the cross-domain seam knowledge that actually determines whether an engagement succeeds.

**Platform side:** No existing marketplace addresses the structural failure modes of AI consulting: the proposal catch-22 (experts need architecture details to scope; clients share architecture only after signing), the mutual IP deadlock (both sides withhold exactly what the other needs), the ground truth bootstrap problem (clients do not know they need labeled evaluation data), and the scope evolution gap (AI projects structurally generate new phases during execution that get treated as contract violations rather than designed lifecycle events).

### 1.2 What AITasker does differently

AITasker is not Fiverr with an AI badge. It is a purpose-built AI consulting marketplace that intervenes structurally at three points where all generic platforms fail:

**Point 1 — Before matching:** An AI-guided elicitation engine transforms the client's symptom description into a taxonomy-grounded capability footprint before any expert sees the posting. The engine injects missing AI SDLC components (ground truth baseline, evaluation infrastructure, architecture reality) and routes technical questions to the client's technical team through a secure information handoff — so the spec an expert responds to reflects the actual system, not the CEO's incomplete understanding of it.

**Point 2 — At matching:** A composite scoring model matches experts against footprints using capability intersection logic — specifically, the seam knowledge (cross-domain competence at domain boundaries) that determines project success. An expert who knows ML fine-tuning AND understands that fine-tuning requires pre-existing ground truth AND knows how to build that ground truth pipeline is a fundamentally different match than an expert who only knows fine-tuning. The seam gap map makes this visible to the client before any commitment.

**Point 3 — During engagement:** A state-gated information release system resolves the mutual IP deadlock. Client technical architecture is disclosed only after the expert's selection and connection acceptance. Expert design reasoning is released incrementally as milestone payments confirm commitment. The add-on phase protocol treats scope evolution as a designed lifecycle feature — with causal chain documentation — rather than a contract violation.

### 1.3 Scope boundaries for this 9-week build

**In scope (building):**
- Complete AI-guided elicitation engine (symptom → footprint → Artifact A); Scenario A (all-non-technical team → Technical Discovery pathway) and Scenario B (self-technical CEO) variants
- Taxonomy-based expert profiles with capability depth and seam declarations
- Composite-scoring matching engine with seam gap map output
- Simplified dual artifact with state-gated access control
- **Non-linear bid system:** pre-bid spec clarification (Surface A), bid revision loop with version history (Surface B), bounded price negotiation — max 2 rounds (Surface C), CEO/TECH_TEAM conflict resolution with mandatory override reason (Surface D)
- **Subscription gate system:** Client Pro + Expert Pro; all LLM and matching routes guarded; payment via internal wallet VA
- **Internal double-entry ledger wallet system:** one wallet per user; atomic ledger transactions for all financial events; no intermediate money movement during engagement lifecycle
- **SePay VA-based payment:** per-VA QR for wallet top-up, milestone funding, subscription, and service purchases; SePay IPN auto-confirms all inbound events
- **SePay Bank Hub Hosted Link:** expert bank account linking (OTP-verified, no password sharing); `bank_account_xid` stored for chi hộ
- **SePay chi hộ API (fully automated expert withdrawal):** NestJS calls chi hộ on withdrawal request; SePay disburses to expert's linked account; credit IPN confirms completion; zero admin involvement
- Service-based hiring: expert service listings (Path B / expert-first marketplace) with direct wallet payment flow
- **Milestone tracking with Definition of Done (DoD) + Sprint system:** 3-layer structure (milestone acceptance criteria as payment contract; DoD checklist as expert self-check; sprint plan + weekly status updates as progress tracking); SCOPE_EVOLUTION sprint status auto-triggers F8 add-on phase
- Project and milestone management with mandatory acceptance criteria + LLM criterion quality check
- Add-on phase request with causal chain documentation
- Real-time messaging per project
- Reviews and reputation signals
- Admin dashboard (monitoring-only: Platform Integrity Monitor, dispute log, transaction ledger, analytics)
- AI service description generator (for expert service listings)
- Dual-role user accounts (CLIENT_CEO + EXPERT on same account; role switcher; self-exclusion in matching)
- Organization schema (data model only — tables with nullable FKs; no UI; Phase 2)
- Cloud deployment on AWS EC2

**Out of scope (designed, not built):**
- Cryptographic key management / production-grade vault
- Pattern learning / monthly recalibration engine (no engagement history data)
- Multi-expert routing (single expert per engagement)
- Full automated scope drift detection (sprint status system captures deviation; escalation is manual)
- GitHub Projects integration (dropped to save effort — sprint/DoD system is native)
- Organization accounts UI (schema designed, no frontend or backend routes for org management)
- MoMo / ZaloPay as chi hộ fallback (SePay chi hộ is primary; verify availability Week 1)

---

## 2. Actors & Roles

### Primary Actors

> **See Section 0.7 for the full RBAC permission matrix.** This table describes who each actor is and their key constraints. All permissions are enforced by `users.active_role` + `users.client_subtype` JWT claims in NestJS guards.

| Actor | JWT `active_role` | Vietnamese label | Description | Key constraints |
|---|---|---|---|---|
| **Client — CEO** | `CLIENT / CEO` | Doanh nghiệp / Khách hàng | Non-technical founders, startup CEOs, product managers who initiate AI projects and handle business decisions. May also hold EXPERT role on the same account (dual-role). | Cannot evaluate expert technical depth; controls budget and final expert selection; cannot see Artifact B; can override TECH_TEAM bid recommendation with mandatory reason |
| **Client — Tech Team** | `CLIENT / TECH_TEAM` | Kỹ thuật viên khách hàng | The client's developer or technical lead. Account is created **only** via signed handoff link from F2 Stage 4. For self-technical CEOs (`project.self_technical = true`), the CEO account holds both CEO and TECH_TEAM capabilities for that specific project — no second account created. | Has architecture truth; cannot fund milestones or send connection requests; can respond to expert spec clarifications; owns technical milestone sign-offs |
| **Expert** | `EXPERT` | Chuyên gia AI | AI engineers, ML practitioners, AI consultants who provide AI implementation services via bidding (Path A / project-based) or service listings (Path B / expert-first). May also hold CLIENT_CEO role on the same account (dual-role). | Needs Artifact B to scope accurately; cannot access Artifact B until engagement CONNECTED; must link bank account via Bank Hub Hosted Link before requesting withdrawal |
| **Admin** | `ADMIN` | Quản trị viên | Platform integrity monitor. Cannot self-register — DB-seeded. Operational role is read-only monitoring; no routine approvals. Emergency write actions only. | Cannot file disputes; cannot fund milestones; no financial transactions; two emergency write actions: spec pull-back and account suspension |

**Dual-role users (Scenario C):** A single account can hold `roles: ["CLIENT_CEO", "EXPERT"]`. The `active_role` JWT field determines current context. Role switcher is a persistent top-bar toggle — reissues JWT without re-login. Self-exclusion guard: expert account cannot bid on any project where the same user_id appears as a project member.

**All-non-technical team (Scenario A):** When CEO has no technical team member, elicitation Stage 4 presents two options: (1) inject a Technical Discovery milestone as Milestone 0 — expert establishes architecture before main work; (2) purchase a TECH_DISCOVERY service engagement first. Both paths use `engagement.type = TECH_DISCOVERY`.

**Self-technical CEO (Scenario B):** CEO sets `project.self_technical = true`. Elicitation Stage 4 presents the TECH_TEAM form directly to the CEO. CEO completes both Stage 1-3 (business framing) and Stage 4 (technical details) in one session. CEO account gains TECH_TEAM capabilities for this project. Surface D conflict resolution gate skipped.

### Actor permission summary

> Abbreviated reference. See Section 0.7 for the complete matrix with all rows.

| Permission | CLIENT / CEO | CLIENT / TECH_TEAM | EXPERT | ADMIN |
|---|---|---|---|---|
| Top up platform wallet | ✅ | ❌ | ✅ | ❌ |
| Purchase subscription | ✅ (Client Pro) | ❌ | ✅ (Expert Pro) | ❌ |
| Submit project via elicitation engine | ✅ (Pro) | ❌ | ❌ | ❌ |
| Complete tech team handoff (Stage 4) | ❌ (unless self_technical) | ✅ | ❌ | ❌ |
| File spec clarification request (pre-bid) | ❌ | ❌ | ✅ | ❌ |
| Respond to spec clarification | ❌ | ✅ | ❌ | ❌ |
| Review bids — detailed seam analysis | ❌ | ✅ | ❌ | ❌ |
| Request bid revision (Surface B) | ❌ | ✅ | ❌ | ❌ |
| Override TECH_TEAM disapproval (Surface D) | ✅ | ❌ | ❌ | ❌ |
| Propose price adjustment (Surface C) | ✅ | ❌ | ❌ | ❌ |
| Approve expert selection (final) | ✅ | ❌ | ❌ | ❌ |
| Fund milestone (wallet or VietQR) | ✅ | ❌ | ❌ | ❌ |
| Create sprint plan + DoD checklist | ❌ | ❌ | ✅ | ❌ |
| Submit weekly sprint status | ❌ | ❌ | ✅ | ❌ |
| Comment on sprint (read delivery progress) | ❌ | ✅ | ❌ | ❌ |
| Sign off technical milestones | ❌ | ✅ | ❌ | ❌ |
| Sign off business / budget milestones | ✅ | ❌ | ❌ | ❌ |
| Link bank account via Bank Hub Hosted Link | ❌ | ❌ | ✅ | ❌ |
| Request withdrawal (triggers chi hộ API) | ❌ | ❌ | ✅ | ❌ |
| Browse Path B expert marketplace | ✅ | ✅ | ❌ | ❌ |
| Buy SERVICE_PURCHASE / TECH_DISCOVERY | ✅ | ❌ | ❌ | ❌ |
| Publish service listing (Path B) | ❌ | ❌ | ✅ | ❌ |
| File a dispute | ✅ | ✅ | ✅ | ❌ |
| Post-engagement review | ✅ (CEO form) | ✅ (Tech Team form) | ✅ (Expert form) | ❌ |
| Emergency spec pull-back / account suspension | ❌ | ❌ | ❌ | ✅ |

---

## 3. Deliverable Features

### Feature Map Overview

```
F1   Authentication & Role Management (incl. dual-role, Bank Hub field)
F1.5 Subscription System (wallet top-up via VA + subscription gate)
F2   AI-Guided Project Elicitation Engine          ← Primary AI innovation
F3   Expert Taxonomy Profile, Service Listings & Purchase (Path A + Path B)
F4   AI Expert Recommendation & Seam Matching      ← Primary AI innovation
F5   Scoped Information Release (Dual Artifact)     ← Trust mechanism innovation
F6   Structured Capability Bid System (Non-Linear: Surfaces A–D)
F7   Project & Milestone Management + DoD + Sprint Tracking
F8   Add-On Phase Protocol
F9   Real-Time Messaging
F10  Internal Wallet + Escrow + SePay Full Integration ← Zero-admin financial system
F11  Reviews & Reputation System
F12  Admin Dashboard (monitoring-only)
```

---

### F1 — Authentication & Role Management

**What it does:** JWT-based auth layer that enforces the RBAC role structure defined in Section 0.7 on every protected route.

**JWT payload structure:**
```json
{
  "sub": "user_id",
  "active_role": "CLIENT | EXPERT | ADMIN",
  "client_subtype": "CEO | TECH_TEAM | null",
  "roles": ["CLIENT_CEO", "EXPERT"],
  "self_technical_projects": ["proj_abc"],
  "project_ids": ["proj_abc", "proj_xyz"]
}
```
`client_subtype` is `null` for EXPERT and ADMIN. `roles` array enables dual-role accounts. `active_role` is the currently displayed context. All NestJS guards evaluate `active_role` + `client_subtype`, not `roles`.

**Account creation paths:**
1. **CLIENT / CEO:** Standard registration → "I need AI help" → `active_role = CLIENT, client_subtype = CEO, roles = ["CLIENT_CEO"]`.
2. **CLIENT / TECH_TEAM:** Only via signed handoff link. Link carries `{ project_id, client_subtype: TECH_TEAM }`. On registration: `active_role = CLIENT, client_subtype = TECH_TEAM, linked_project_id = project_id`.
3. **EXPERT:** Standard registration → "I provide AI services" → `active_role = EXPERT, roles = ["EXPERT"]`.
4. **Add second role:** Existing user opens "Account Settings → Add Role". CEO can add EXPERT role; Expert can add CLIENT_CEO role. After verification: `roles` array updated; role switcher appears in nav.
5. **ADMIN:** DB-seeded only. Registration endpoint returns 403 for `role = ADMIN` regardless of payload.

**Role switcher:** Persistent top-nav toggle visible when `roles.length > 1`. Click → NestJS reissues JWT with `active_role` updated → React re-renders dashboard for new role context. No re-login.

**Self-exclusion guard:** Every matching engine query appends `WHERE expert.user_id NOT IN (SELECT user_id FROM project_members WHERE project_id = ?)`. This fires regardless of `active_role` — switching to Expert role does not bypass the exclusion.

**Dashboard routing:**
- `CLIENT / CEO` → CEO Dashboard: wallet balance, project initiation, shortlist, selection, milestone funding, budget tracking, subscription status
- `CLIENT / TECH_TEAM` → Tech Dashboard: architecture handoff, Artifact B, bid review, sprint view, milestone sign-off, DoD read, pay-gated doc inbox
- `EXPERT` → Expert Dashboard: profile, service listings, bids, sprint plan editor, DoD checklist, milestone management, earnings, Bank Hub link status
- `ADMIN` → Admin Dashboard: read-only monitoring modules

**Key `users` table fields (additions for this iteration):**
```sql
users:
  roles                    JSONB     DEFAULT '["CLIENT_CEO"]'   -- array of held roles
  active_role              TEXT      NOT NULL                   -- currently active context
  subscription_client_tier TEXT      DEFAULT 'free'            -- 'free' | 'pro'
  subscription_expert_tier TEXT      DEFAULT 'free'
  sub_client_expires_at    TIMESTAMP NULL
  sub_expert_expires_at    TIMESTAMP NULL
  sepay_bank_account_xid   TEXT      NULL  -- set by Bank Hub BANK_ACCOUNT_LINKED webhook
  bank_account_holder_name TEXT      NULL  -- bank-verified name from Bank Hub
  bank_linked_at           TIMESTAMP NULL
```

**Owner:** TV2 (Chí Nhân)
**Effort:** ~6 days

---

### F1.5 — Subscription System

**What it does:** Gates all LLM-powered features (elicitation engine, matching engine, LLM verification) and advanced engagement features behind a one-time activation payment. Satisfies the teacher's explicit requirement that all users go through subscription to access AI features.

**Subscription guard (NestJS middleware):**
- Reads `user.subscription_client_tier` / `subscription_expert_tier` + `sub_client_expires_at` / `sub_expert_expires_at`
- If tier is `free` or expiry has passed: returns HTTP 403 `{ code: "SUBSCRIPTION_REQUIRED", feature: "elicitation_engine", upgrade_url: "/subscribe" }`
- Applied to: every FastAPI-calling route (all LLM features) + matching engine endpoint + Artifact B access route + add-on phase route + expert Tier 2+ bid routes

**Payment flow — subscription via SUBSCRIPTION VA (corrected two-step pattern):**
```
Step 1 — Create pending subscription row (POST /subscriptions/activate):
  Guard: user.wallet.available_balance >= subscription_price[role_type]
  (wallet balance check happens here before any VA is created)

  NestJS opens DB transaction:
    1. Creates user_subscriptions row: { status: PENDING_PAYMENT, role_type, tier: 'pro' }
    2. Creates SUBSCRIPTION VA: { entity_type: SUBSCRIPTION, entity_id: user_subscriptions.id,
                                   fixed_amount: price, expires_at: now + 24h }
    3. Returns VA VietQR to frontend
  Commits.

Step 2 — IPN arrives (async):
  NestJS IPN SUBSCRIPTION branch:
    1. Look up virtual_accounts by va_number; get entity_id = user_subscriptions.id
    2. Validate: transferAmount == va.fixed_amount
    3. Atomic DB transaction:
       wallet.available_balance -= amount
       wallet_transaction: { type: SUBSCRIPTION, amount, reference_id: 'SUB:'+user_subscriptions.id }
       UPDATE user_subscriptions SET status='ACTIVE', activated_at=now(),
                                      expires_at=now() + interval '6 months', price_paid=amount
       UPDATE users SET subscription_{role_type}_tier='pro',
                        sub_{role_type}_expires_at=user_subscriptions.expires_at
       VA.status = USED
    4. Notify user: "Pro subscription active until {expires_at}"
```

Note: if the user closes the browser before scanning the QR, the PENDING_PAYMENT row and VA exist but are harmless — VA expires in 24h; the row stays as an audit record. A new subscription attempt creates a new row and new VA.

**Wallet top-up via VA:**
- Each user has one `WALLET_TOPUP` virtual account (permanent, created on registration)
- CEO opens "Top Up Wallet" → displays VietQR for their `WALLET_TOPUP` VA
- User transfers any amount → SePay IPN fires → NestJS credits `wallet.available_balance`
- User can then purchase subscription or fund milestones directly from wallet balance

**Tables:**
```sql
wallets:
  id, user_id UNIQUE, available_balance BIGINT DEFAULT 0, locked_balance BIGINT DEFAULT 0

wallet_transactions:  -- IMMUTABLE; never updated after insert
  id, wallet_id, amount BIGINT (always positive; direction from transaction_type),
  transaction_type TEXT CHECK (IN 'TOP_UP','SUBSCRIPTION','ESCROW_LOCK','ESCROW_RELEASE',
                                'PLATFORM_FEE','ESCROW_REFUND','ESCROW_SPLIT','WITHDRAWAL'),
  reference_id TEXT NULL,  -- naming convention: 'SUB:{subscription_id}' | 'ESC_LOCK:{milestone_id}' | etc.
  created_at TIMESTAMPTZ

  -- Critical idempotency index (prevents SePay IPN double-credit on retry):
  UNIQUE INDEX wallet_tx_idempotency ON wallet_transactions(wallet_id, reference_id)
    WHERE reference_id IS NOT NULL;

virtual_accounts:
  id, entity_type TEXT CHECK (IN 'WALLET_TOPUP','MILESTONE','SERVICE','SUBSCRIPTION'),
  entity_id TEXT,  -- user_id for TOPUP; milestone_id for MILESTONE; user_subscriptions.id for SUBSCRIPTION
  va_number TEXT UNIQUE,  -- SePay-issued bank account number
  fixed_amount BIGINT NULL,  -- NULL for WALLET_TOPUP; set for others (bank rejects wrong amounts)
  expires_at TIMESTAMPTZ NULL,  -- NULL for WALLET_TOPUP; 24h for MILESTONE/SERVICE/SUBSCRIPTION
  status TEXT DEFAULT 'ACTIVE' CHECK (IN 'ACTIVE','EXPIRED','USED')

user_subscriptions:
  id, user_id, role_type TEXT CHECK (IN 'client','expert'),
  tier TEXT CHECK (IN 'free','pro'),
  status TEXT DEFAULT 'PENDING_PAYMENT' CHECK (IN 'PENDING_PAYMENT','ACTIVE','EXPIRING_SOON','EXPIRED'),
  price_paid BIGINT DEFAULT 0,
  activated_at TIMESTAMPTZ NULL,  -- NULL until IPN confirms payment
  expires_at TIMESTAMPTZ NULL
  UNIQUE (user_id, role_type)

platform_settings:  -- SINGLETON; exactly one row seeded at deploy
  id, platform_wallet_id UUID UNIQUE REFERENCES wallets(id),
  platform_fee_pct FLOAT DEFAULT 0.05 CHECK (BETWEEN 0 AND 1)
  -- Fee ledger uses: SELECT platform_fee_pct FROM platform_settings LIMIT 1
  -- Do not hardcode 0.05 in application code — always read from this table
```

**Owner:** TV2 (Chí Nhân) — wallet CRUD, VA creation via SePay API, IPN handler branch for WALLET_TOPUP, subscription activation route, subscription guard middleware
**TV3 (Cao Minh)** — wallet top-up UI, subscription purchase UI, subscription status badge in dashboard nav
**Effort:** TV2: ~5 days, TV3: ~2 days

---

### F2 — AI-Guided Project Elicitation Engine

**What it does:** Replaces and dramatically extends the Topic 7 "AI Job Assistant" requirement. Instead of form-based job posting with AI suggestion, this is a conversational diagnostic engine that transforms a symptom description into a taxonomy-grounded capability footprint. This is the platform's most important differentiator and the primary research contribution for RQ2.

**The problem it solves:** Clients describe what hurts, not what is broken. "Our AI output is inconsistent" is a symptom. The actual problem might be non-deterministic LLM output from an unstructured prompt, or it might be a race condition in a concurrent vector write pipeline, or it might be the absence of an evaluation baseline that makes "inconsistent" unmeasurable. These require completely different experts. A static form produces the symptom as a job posting. The elicitation engine produces a structured diagnosis.

**Stage-by-stage flow:**

**Stage 1 — Symptom intake `[Actor: CLIENT / CEO]`:** The CEO types a free-form paragraph describing their pain. The interface explicitly tells them NOT to write a technical spec: *"Tell us what hurts: what you're currently doing, what costs too much or doesn't work, and what business goal you're trying to reach."* The LLM extraction layer (FastAPI + Claude/OpenAI API) runs three extraction passes:
- Intent separation: strips prescribed solutions ("we want to fine-tune a model") from actual symptoms ("our compliance rate is inconsistent")
- Scale signal extraction: identifies volume, frequency, cost indicators that determine project tier
- Void detection: checks for absent mandatory AI SDLC components (no evaluation ground truth mentioned → flag; no error handling strategy mentioned → flag; no mention of existing system integration → probe required)

**Stage 2 — Archetype confirmation + SDLC injection `[Actor: CLIENT / CEO]`:** The system presents the probable project archetype in plain business language (not taxonomy codes) with three behavioral options for the CEO to select from. Upon confirmation, missing SDLC components are injected one at a time as client protections, not as technical requirements:
- Ground truth void → *"To measure whether the AI is working, your expert will need a reference dataset of past decisions. Do you currently have labeled examples of correct outcomes?"* → If no: Phase 0 (Ground Truth Setup) is injected as mandatory Milestone 1 with a plain explanation of why: *"Without this, there is no objective way to verify the AI is accurate — which means you cannot evaluate the expert's work."*
- The CEO cannot remove Phase 0. If they try, the system explains the contractual consequence: milestone acceptance criteria cannot be defined without an evaluation baseline, so payment cannot be triggered for subsequent milestones.

**Stage 3 — Architecture and scale probe `[Actor: CLIENT / CEO]`:** Four behavioral questions (not technical questions — the CEO can answer all of these without knowing their stack):
- *"When a user submits content for AI processing, do they wait for an immediate result, or do they come back later to check?"* → Determines synchronous vs. asynchronous architecture
- *"Are there times when a very large volume arrives all at once, like a bulk upload?"* → Determines Thundering Herd risk
- *"Does the AI need to remember past items it has processed — for example, to avoid re-processing something it already handled?"* → Determines vector database / semantic memory requirement (this single question would have surfaced Phase 2.1 of the AdTech case at intake)
- *"If the AI is uncertain or fails, what should happen — reject automatically, approve automatically, or escalate to a human?"* → Determines HITL workflow requirement

**Stage 4 — Tech team handoff `[Actor: CLIENT / TECH_TEAM — account created here]`:** If the probe reveals integration with an existing production system, async processing, stateful memory, or large legacy data, the system hard-blocks the CEO and generates a secure handoff link for their technical lead. **This link is the sole creation path for CLIENT / TECH_TEAM accounts** (see F1). The TECH_TEAM sees a completely different framing: *"Your CEO is scoping an AI integration. To protect your system's IP and ensure the hired expert understands your architecture before being selected, please confirm the following."* The TECH_TEAM inputs:
- Stack tags (multiple-choice: Go / Python / Node / Java / etc.)
- Integration method (HTTP REST / Kafka / RabbitMQ / Database polling)
- Legacy data volume range
- Deployment expectation (Docker container / serverless / monorepo commit)
- Sensitive schema upload (optional but incentivized — these go into the Artifact B vault, inaccessible until expert is selected)

The link carries a signed token with `{ project_id, client_subtype: TECH_TEAM }`. On registration, NestJS sets `users.role = CLIENT, users.client_subtype = TECH_TEAM, users.linked_project_id = project_id`. The link expires after 72 hours; the CEO can re-generate it. The CEO cannot complete this form — the Stage 4 route is guarded by `client_subtype = TECH_TEAM`.

**Stage 5 — Synthesis and spec generation `[Actor: System]`:** The system resolves CEO vs. TECH_TEAM signal conflicts (private reconciliation — neither party is told the other contradicted them) and produces:
- **Capability Footprint**: structured internal record of required domains, seams, tier, and SDLC milestone framework
- **Artifact A**: the public-facing sanitized spec visible to matched experts — no proprietary content, but full architecture category labels, stack tags, volume tier, and escrow lock notice for Artifact B
- **Artifact B structure**: populated with tech team's sensitive uploads, state-gated until expert is connected

The entire elicitation lifecycle is persisted in the `elicitation_sessions` table — one row per attempt, tracking `current_stage` (1-5), `archetype`, `scenario_type` (STANDARD / SCENARIO_A / SCENARIO_B), `void_list_json`, and `state` (IN_PROGRESS / COMPLETED / ABANDONED / RETURNED). The FK direction is `projects.elicitation_session_id → elicitation_sessions(id)` — not the reverse. This means a session can exist without a project (incomplete sessions) but a project always points to the session that produced it. If the auto-publish gate fails, `elicitation_sessions.state = RETURNED` and the CEO re-enters at `current_stage` without losing their prior inputs.

**Auto-publish gate:** After synthesis, the system runs an automated quality check before publishing Artifact A. This replaces a mandatory admin approval step — routing every spec through admin before any expert can see it would create a bottleneck for a small-team operation and is not how real marketplaces work.

The quality check runs three automated validations:
- **Footprint completeness score** ≥ 0.7 (computed by the synthesis engine — mandatory injections present, required seam count meets archetype minimum, SDLC milestone framework populated)
- **Matching pre-check** passes: at least 1 expert in the pool scores above the minimum match threshold
- **No hard-flagged voids** remain unresolved (e.g., ground truth void acknowledged but Phase 0 not injected)

If all three pass → Artifact A is auto-published immediately. The client is notified; the matching engine generates the shortlist.

If any check fails → the spec state is set to `RETURNED_TO_CLIENT` automatically. The LLM generates a targeted advisory note based on the specific failure (e.g., "Your spec could not be published because the Ground Truth Baseline milestone has not been injected. Please complete the following step in the elicitation flow: [dynamically injected missing question]"). The client re-enters the elicitation engine **at the specific void that caused the failure** — not from the beginning. There is no admin exception queue and no manual override path. A spec that cannot pass the automated gate is not ready; the correct resolution is to complete the missing information, not to shortcut the gate.

Admin retains a single passive power over published specs: the ability to **pull back any published spec** at any time (`spec.state → SUSPENDED`), for example if an expert reports a factual error in Artifact A after shortlisting. This is an emergency brake — it requires no routine admin attention. See Section 0.6 for the full spec state machine.

**Owner:** TV1 (Minh Hùng) — FastAPI + LLM extraction + synthesis logic  
**TV2 (Chí Nhân)** — Frontend conversational UI, guided dialogue state machine  
**Effort:** TV1: ~12 days, TV2: ~5 days  
**AI dependency:** Anthropic Claude API (claude-sonnet-4-6) for extraction layers

---

### F3 — Expert Taxonomy Profile, Service Listings & Purchase

**What it does:** Covers Topic 7's "Expert profile & portfolio," "Publish AI services," and the service-based hiring model requirement. Extended with taxonomy-based capability declarations (enabling F4's matching) and a real SePay payment flow for direct service purchases — satisfying Topic 7's explicit "supports both service-based and project-based hiring model" requirement.

**Expert profile components:**

*Taxonomy capability map:*
- Six capability domains, each declared at one of three depth levels (Deep / Working / Aware):
  - [A] LLM Application Engineering (system prompt design, structured output, RAG orchestration)
  - [B] MLOps / LLMOps Infrastructure (model serving, cost optimization, drift monitoring)
  - [C] AI Evaluation & Quality Systems (metric design, ground truth creation, human-in-the-loop)
  - [D] Vector DB & Embeddings (HNSW indexing, chunking, MRL truncation, similarity tuning)
  - [E] Data & Pipeline Engineering (Kafka, async processing, distributed locks, legacy ETL)
  - [F] ML Modeling & Fine-Tuning (cross-encoders, supervised fine-tuning, imbalanced data)
- Ten seam claims (self-declared at Tier 1; LLM auto-upgrades to Tier 2 or 3 on portfolio/scenario evidence):
  - A↔C (Ground truth-driven iteration), A↔F (Hybrid routing), A↔D (Retrieval-generation contract)
  - D↔E (Distributed vector upsert safety), D↔F (Embedding/index co-design)
  - C↔F (Fine-tuning gating), E↔F (Training data as pipeline problem)
  - A↔B (Prompt iteration under production constraints), B↔E (Cost management in streaming pipelines)
  - C↔E (Evaluation data as pipeline concern)
- Stack tags (free multi-select with search)
- Engagement model: Advisory only / Spec + review / Full implementation
- **Archetype history (`expert_profiles.archetype_history_json`):** Expert self-declares which project archetypes they have worked on prior to joining the platform (e.g., `[{archetype_code: "ARCHETYPE_1", tier: "TIER_3", self_declared: true}]`). This seeds the **20% Archetype-tier congruence** component of the composite match score for cold-start experts who have no closed platform engagements yet. As engagements close, confirmed archetype history is derived at query time from `engagements → projects` and overrides the self-declared JSON — the JSON is never updated after verified history exists.

*Portfolio evidence:*
- Past project submissions using structured decision-point questions (not a resume upload):
  - What was the system's architecture when you entered the engagement?
  - What were the 2–3 most consequential technical decisions you made?
  - What would have happened if you had made a different choice at each point?
- Each submission targets a **specific seam claim** via `portfolio_submissions.seam_claim_id FK → expert_seam_claims(id)`. This FK enforces that every LLM evaluation result updates the correct seam claim row directly — no ambiguity about which claim a confidence score applies to.
- Sanitization note: experts are explicitly told not to include client names, proprietary code, or NDA-protected details — the reasoning is the evidence, not the artifact

*Verification tiers (visible on profile):*

All tier upgrades are automated — no admin reviews portfolio evidence or grades scenario responses. See Section 0.4 for the full tier definitions, confidence factors, and Tier 4 signal accumulation rule.

- **Claimed (Tier 1):** Self-declared on profile, no supporting evidence. Confidence factor: 0.20.
- **Evidence-backed (Tier 2):** Expert submits portfolio evidence using the structured decision-point questions. FastAPI LLM extraction runs against the submission. If extraction confidence ≥ 0.85 AND all required signal types found for the seam → auto-upgrade to Tier 2; expert notified immediately. If confidence is below threshold or any required signal type is missing → LLM generates a specific gap advisory ("Your response demonstrated [signal A] and [signal B]. To reach Evidence-backed status for this seam, please also provide evidence of [signal C: description]") and auto-returns the submission for resubmission. Experts may resubmit as many times as needed; a 5-attempt limit per seam triggers a 30-day lockout (DB flag) to prevent infinite retry loops. Confidence factor: 0.55.
- **Scenario-verified (Tier 3):** Expert requests Tier 3 verification for a seam. The system auto-selects a question from the question bank (lowest `last_used_at` timestamp for that seam, to prevent gaming). The expert responds. FastAPI LLM evaluates the response against the question's stored rubric, which requires all specified dimensions to individually pass (not an average). Pass → auto-upgrade to Tier 3. Any dimension fails → LLM-generated specific failure note; new question auto-selected for resubmission. Same 5-attempt / 30-day cooldown rule applies. Confidence factor: 0.80.
- **Platform-demonstrated (Tier 4):** Populated automatically after any engagement closes with a positive outcome signal. Signal accumulation rule evaluated by NestJS against the `expert_seam_outcome_signals` table — no admin action required. Confidence factor: 0.95.

*AI Service Generator:*
- Expert inputs their key capabilities and target use cases in plain text
- LLM generates a structured service description (title, description, what's included, engagement model, estimated timeline, suggested price range)
- Expert edits and publishes
- This satisfies Topic 7's "AI Service Generator" requirement

*Service listings (Path B — Expert-First Marketplace):*
The expert-first marketplace is a **co-equal discovery path** alongside the project-based flow. Clients can arrive here directly (browsing experts → buying packaged services) or be recommended here by the elicitation engine when Stage 4 is unreachable (Scenario A Technical Discovery pathway).

- Expert publishes fixed-price AI service packages (e.g., "RAG System Setup — 2 weeks, 8,000,000 VND")
- Service cards: title, domains, seam tags, engagement model, price, expert reputation
- Filterable marketplace: by domain, seam, archetype, price range, availability
- Clients can browse without a subscription (Free tier) — discovery is open
- Purchasing requires wallet balance (top-up first if needed)

**Three engagement types on the platform:**

| Type | Elicitation | Matching | Bid system | Subscription required |
|---|---|---|---|---|
| `PROJECT_BASED` | Full F2 engine | Composite score + shortlist | Full structured bid + non-linear surfaces | CLIENT PRO |
| `SERVICE_PURCHASE` | None | Browse + filter | None (fixed scope) | Free (no LLM in purchase path) |
| `TECH_DISCOVERY` | None | Availability-based | None (fixed scope) | Free |

*Service purchase flow — wallet-based, fully automated:*
```
Input: { service_id } from POST /services/{id}/purchase [guard: CLIENT / CEO active]
Guard: user.wallet.available_balance >= service.price_vnd

NestJS:
  1. Creates per-order VA (entity_type: SERVICE, entity_id: purchase_id,
     fixed_amount: service.price_vnd, expires_at: now + 24h)
  2. Returns VietQR for that VA to frontend
  3. User scans QR → bank transfer exact amount → SePay IPN fires (credit on service VA)

NestJS IPN handler (service purchase branch):
  4. Identifies entity_type = SERVICE from va_number
  5. Atomic ledger: [ESCROW_LOCK] client.available_balance -= amount; escrow_accounts created
  6. Creates service_engagements record { state: FUNDED }
  7. Notifies expert: "New service order received"
```

*TECH_DISCOVERY service purchase:*
Same flow as SERVICE_PURCHASE. When engagement CLOSED: CEO receives architecture summary document from expert → CEO can optionally complete Stage 4 using this document → main PROJECT_BASED elicitation continues.

**Owner:** TV3 (Cao Minh) — Profile forms, service listing UI, Path B marketplace UI, purchase flow UI, TECH_DISCOVERY service listing category
**TV1 (Minh Hùng)** — Portfolio evidence LLM extraction, scenario assessment LLM rubric evaluation, AI service generator LLM call
**TV2 (Chí Nhân)** — Service purchase NestJS route, per-order VA creation, IPN handler branch for SERVICE type, service_engagements state machine, seam claim auto-upgrade state machine, Tier 4 signal accumulation on engagement close
**Effort:** TV3: ~9 days, TV1: ~6 days, TV2: ~4 days

---

### F4 — AI Expert Recommendation & Seam Matching

**What it does:** Covers Topic 7's "AI Expert Recommendation" requirement, extended from simple collaborative filtering to a composite scoring model that makes capability intersection legible. Primary contribution to RQ1.

**The five-component composite score:**

| Component | Weight | What it measures |
|---|---|---|
| Seam alignment | 40% | For each seam required by the footprint: (criticality weight × expert's verification confidence). Missing load-bearing seam = negative contribution |
| Domain depth coverage | 25% | Whether expert's verified depth level meets or exceeds the footprint's requirement for each domain |
| Archetype-tier congruence | 20% | Expert's direct engagement history with the same archetype-tier combination |
| Engagement model fit | 10% | Whether expert's declared model matches what the footprint requires |
| Stack tags & recency | 5% | Stack tag overlap, weighted by recency of relevant engagement |

**Verification confidence factors (used in seam alignment):**
- Claimed: 0.20
- Evidence-backed: 0.55
- Scenario-verified: 0.80
- Platform-demonstrated: 0.95

**Hard gates (eliminate before ranking):**
- Expert has a claimed-to-verified ratio above 4:1 → excluded from Tier 2+ projects
- Expert has two or more negative outcome signals on a seam that is load-bearing in the footprint → excluded from that project

**Seam gap map:** For each required seam in the footprint, the match card shows:
- Green / Verified: Tier 3 or 4 verification — platform has genuine confidence
- Amber / Evidence-backed: Tier 2 — positive signals but no scenario confirmation
- Yellow / Claimed: Tier 1 — self-declared only, highest risk
- Red / Absent: Expert has no claim on this seam — named explicitly as a gap

**Match card components (visible to client):**
1. Match strength: Strong (>0.78) / Qualified (0.58–0.78) / Conditional (0.42–0.58)
2. Verified capability summary: 2–3 sentence project-specific description from verification record
3. Seam coverage visual: color-coded grid of all required seams
4. Known capability gaps: absent seams named as explicitly as strengths (not hidden)
5. Recommended next step: Invite to bid / Suggest scenario assessment completion first / Schedule scoping call

**Shortlist:** 3–5 ranked candidates presented as match cards. Score numbers are internal — clients see the three-tier strength label, not the numeric composite.

**Owner:** TV1 (Minh Hùng) — Composite scoring algorithm, seam gap map generation  
**TV2 (Chí Nhân)** — Match card UI, shortlist display  
**Effort:** TV1: ~6 days, TV2: ~4 days

---

### F5 — Scoped Information Release (Simplified Dual Artifact)

**What it does:** The structural mechanism that resolves the mutual IP deadlock — clients withholding architecture, experts withholding reasoning. Contributes directly to RQ3 (trust factors). Simplified from the full design: state-based database access control rather than cryptographic key management.

**Artifact A (Public Spec — always visible to matched experts):**
- Project archetype and complexity tier
- Business intent statement (translated from symptom)
- Capability footprint: required domains with depth levels, required seams with criticality
- System physics: architecture category, stack tags, throughput tier, deployment expectation — no schemas, no topic names, no proprietary content
- SDLC milestone framework: injected milestones locked as non-negotiable
- Escrow lock notice: *"Detailed backend schemas, payload structures, and legacy database details are held in information escrow. They will be disclosed to the selected expert upon connection acceptance."*

**Artifact B (Technical Blueprint — state-gated):**
- Tech team's uploaded schemas, payload samples, integration contracts
- Proprietary rule taxonomies and current production prompt/config (if uploaded)
- Access control: DB-level row visibility check — `EXPERT` and `CLIENT / TECH_TEAM` can only query Artifact B fields when `engagement.state ≥ CONNECTED`. `CLIENT / CEO` **cannot access Artifact B at any engagement state** — this is a hard guard, not a UI hide. The CEO sees only the Artifact A public spec.
- `ADMIN` can view Artifact B at all times for dispute / spec review purposes

**Connection flow (the trust gate):**
1. `CLIENT / CEO` selects an expert from the shortlist → connection request sent
2. `EXPERT` reviews Artifact A and accepts or declines — the Expert cannot see Artifact B before accepting
3. Upon acceptance: `engagement.state → CONNECTED`; `engagement.connected_at = now()`
4. CEO completes NDA click-through → `engagement.client_nda_accepted_at = now()`; Expert completes NDA → `engagement.expert_nda_accepted_at = now()`
5. Artifact B access gate activates: the FastAPI route checks **both NDA timestamps** before returning data:
   ```sql
   SELECT ab.* FROM artifact_b ab
   WHERE ab.project_id = :project_id
     AND EXISTS (
       SELECT 1 FROM engagements e
       WHERE e.project_id = ab.project_id
         AND e.state >= 'CONNECTED'
         AND e.client_nda_accepted_at IS NOT NULL
         AND e.expert_nda_accepted_at IS NOT NULL
         AND (e.expert_id = :current_user_id   -- EXPERT path
              OR (e.project_id IN (             -- TECH_TEAM path
                SELECT linked_project_id FROM tech_team_profiles
                WHERE user_id = :current_user_id)))
     )
   -- CEO permanently excluded regardless of state — no CEO branch exists
   ```
6. Bank Hub link check: if `expert.sepay_bank_account_xid IS NULL` on CONNECTED → platform prompts expert to complete Bank Hub Hosted Link in their dashboard. Engagement proceeds — bank linking is not a blocker for connection, but is required before withdrawal.

`CLIENT / TECH_TEAM` NDA: the TECH_TEAM account is bound to the NDA by the `client_nda_accepted_at` timestamp from the CEO's click-through on the same engagement — no separate TECH_TEAM click required. Their Artifact B access is gated on `client_nda_accepted_at IS NOT NULL` (already set by CEO) + `expert_nda_accepted_at IS NOT NULL` + engagement being in CONNECTED state.

**Pay-gated knowledge staging `[Actor: EXPERT stages → CLIENT / TECH_TEAM receives]`:**
- `EXPERT` uploads reasoning documents to a staging area, tagged with a milestone release trigger: *"Release this document to the client's tech team when Milestone 1 payment is confirmed"*
- When SePay IPN webhook confirms Milestone 1 payment (milestone state advances to `FUNDED`), the staged document becomes visible to `CLIENT / TECH_TEAM` automatically — not to `CLIENT / CEO`. Pay-gated reasoning is technical architecture content; it belongs in the TECH_TEAM's panel, not the CEO's.
- The expert writes their full design rationale without sanitizing it — the disclosure is gated on demonstrated commitment, not assumed trust
- This directly resolves the mutual IP deadlock: the TECH_TEAM receives the unsanitized "why" behind every architectural decision, but only after real money has moved into the platform escrow account

**Owner:** TV1 (Minh Hùng) — FastAPI Artifact B serving route (reads `engagement.state` from DB; returns elicitation-generated technical blueprint data only when state ≥ CONNECTED — passive DB read, no business logic written here)
**TV2 (Chí Nhân)** — Artifact A display, connection flow UI + NestJS connection state transition, pay-gated staging interface
**Effort:** TV1: ~2 days, TV2: ~6 days

---

### F6 — Structured Capability Bid System (Non-Linear)

**What it does:** Replaces free-form proposals with a structured bid format and adds four explicit push-back surfaces that handle legitimate pre-connection negotiation without leaking into unstructured chat. The bid flow is intentionally non-linear — multiple revision loops are supported before expert selection is finalized.

---

**Surface A — Pre-bid Spec Clarification `[Actor: EXPERT requests → TECH_TEAM or CEO responds]`**

Before submitting a bid, an Expert can file structured clarification requests against specific components of Artifact A.

```
Input (POST /specs/{spec_id}/clarifications):
{
  target_component: "footprint" | "milestone_framework" | "acceptance_criteria" | "system_physics",
  question_text: string
}
State: spec_clarifications.status: OPEN → ANSWERED → RESOLVED
  CHECK: status IN ('OPEN','ANSWERED','RESOLVED') — 'CLOSED' is NOT valid
Multiple clarifications can be OPEN simultaneously from the same expert.
Responder: spec_clarifications.responded_by FK (NULL until ANSWERED)
  → TECH_TEAM on Tier 2+ projects, CEO on Tier 1 projects
  → Set when TECH_TEAM/CEO writes the response; provides audit trail of who answered
Every response is stored — immutable audit trail.
```

Expert marks each clarification RESOLVED before submitting a bid (can proceed with unresolved ones; system flags them in bid form with a warning).

---

**Surface B — Bid Revision Loop `[Actor: TECH_TEAM flags → EXPERT revises]`**

After bid is SUBMITTED, TECH_TEAM can flag specific components for revision with a structured reason. Only flagged components are editable in the revision. Every revision creates a new version record.

```
Bid state transitions:
DRAFT → SUBMITTED → TECH_REVIEW
  → REVISION_REQUESTED  [TECH_TEAM: { flagged_component, reason }]
  → REVISED             [EXPERT: edits only flagged component; bid_versions row written]
  [TECH_REVIEW / REVISION_REQUESTED / REVISED loop — unlimited rounds]
  → TECH_APPROVED (flag: Recommended)
  → TECH_DISAPPROVED (flag: Not Recommended + required reason text)
  [If TECH_DISAPPROVED → Surface D triggers before CEO_REVIEW unlocks]
  → CEO_REVIEW
  [Surface C optional negotiation window here]
  → SELECTED | DECLINED

DB additions:
  bid_versions: (id, bid_id, version_number, snapshot_json, revised_at, reviser_id)
  bid_revision_requests: (id, bid_id, flagged_component, reason, requested_by, requested_at)
```

---

**Surface C — Bounded Price Negotiation `[Actor: CLIENT / CEO proposes → EXPERT accepts/counters/declines]`**

After TECH_APPROVED and before formal selection. Maximum 2 rounds. On round 2 expiry, CEO must accept the expert's last offer or decline.

```
Input (POST /bids/{bid_id}/negotiate):
{
  proposing_party: "CEO",
  proposed_milestone_pricing: [{ milestone_id, proposed_amount_vnd }],
  round_number: 1,    -- server increments; returns 422 if round_number > 2
  message: string
}

Expert response (PUT /bids/{bid_id}/negotiate/{neg_id}/respond):
{
  response: "ACCEPT" | "COUNTER" | "DECLINE",
  counter_pricing: [...],    -- required if COUNTER
  message: string
}

If round_number = 2 and expert COUNTERs:
  CEO must ACCEPT or DECLINE (no further counter).

DB: price_negotiations (id, bid_id, round_number, proposer, proposed_pricing_json,
                         response, counter_pricing_json, message, created_at)
```

---

**Surface D — CEO / TECH_TEAM Conflict Resolution `[Actor: CLIENT / CEO overrides Not Recommended]`**

When TECH_DISAPPROVED and CEO wants to proceed.

```
POST /bids/{bid_id}/override-disapproval
Guard: bid.state = TECH_DISAPPROVED; active_role = CLIENT/CEO
Input: { override_reason: string (min 50 chars), acknowledged_risks: string }
On success: bid.state → CONFLICT_RESOLVED_CEO_OVERRIDE
             bid_conflict_overrides row written:
             { bid_id, ceo_user_id: current_user_id, override_reason, acknowledged_risks,
               overridden_at: now() }
             -- ceo_user_id is an explicit FK; used for RQ3 correlation
             -- char_length(override_reason) >= 50 enforced at DB level via CHECK constraint
             CEO_REVIEW unlocks
DB: bid_conflict_overrides (id, bid_id UNIQUE, ceo_user_id FK users, override_reason,
                             acknowledged_risks, overridden_at)
```

Override records surface in Platform Integrity Monitor. Post-engagement negative outcomes correlate against override records → RQ3 evidence.

---

**Bid components (three required — 422 returned if any missing at SUBMITTED state):**

1. **Footprint Alignment Statement** — Domains at verified depth; seam claims; any discrepancies. Pre-populated from expert profile; expert confirms or flags. Load-bearing gap warning: *"This seam is load-bearing. Your Tier 1 claim will be visible to the client."*

2. **Architectural Approach Summary** — Engagement structure. Must address (a) SDLC milestone framework from Artifact A, (b) any stated-approach discrepancy. No WBS or proprietary design — expert doesn't have Artifact B yet.

3. **Conditional Milestone Pricing** — Costs against the milestone framework. Each conditional must be structured: `{ condition, price_range_min_vnd, price_range_max_vnd, trigger_description }`. Free-text price ("TBD") returns 422 listing unstructured conditionals.

**Owner:** TV3 (Cao Minh) — Bid submission form (3 components), bid comparison UI (TECH_TEAM view), Surface A clarification thread UI, Surface C negotiation UI, Surface D override form
**TV2 (Chí Nhân)** — Bid state machine; bid_versions writes on every REVISED; Surface B revision flagging route; Surface C negotiation state machine (round counter + expiry guard); Surface D conflict override route + guard; `bid_versions`, `bid_revision_requests`, `price_negotiations`, `bid_conflict_overrides` tables
**Effort:** TV3: ~7 days, TV2: ~5 days

---

### F7 — Project & Milestone Management + Definition of Done + Sprint Tracking

**What it does:** Covers Topic 7's "Track milestones & approve deliverables" and "Manage projects & deliverables" requirements, extended with a three-layer tracking system (milestone acceptance criteria as payment contract; DoD checklist as expert self-check; sprint plan + weekly status as progress tracking). Layers serve different audiences and have different decision authorities.

---

**Layer 1 — Milestones (The Payment Contract)**

Every milestone is a payment-triggering delivery contract. No milestone can reach APPROVED without every required acceptance criterion being explicitly verified by the designated sign-off authority.

```
milestone schema:
  id, engagement_id, milestone_number (UNIQUE per engagement)
  deliverable_statement     TEXT        -- what the expert produces (artifact type + scope)
  sign_off_authority        TEXT        -- TECH_TEAM | CEO | JOINT
  payment_amount_vnd        BIGINT      -- locked at bid acceptance; drives escrow lock
  state                     TEXT        -- see milestone state machine in Section 0.6
  va_number                 TEXT NULL   -- per-milestone VA; generated on AWAITING_PAYMENT
  va_expires_at             TIMESTAMPTZ NULL  -- 24h from generation; SePay rejects wrong amounts
  funded_at, submitted_at, approved_at, released_at  TIMESTAMPTZ NULL
  review_clock_paused_at    TIMESTAMPTZ NULL  -- set when BLOCKER sprint status is filed
                                              -- NULL = clock is running
  total_paused_seconds      INT DEFAULT 0     -- accumulates total pause duration for SLA tracking
                                              -- on BLOCKER resolved: += (now - review_clock_paused_at);
                                              -- then review_clock_paused_at = NULL
```

```
acceptance_criteria schema:
  id, milestone_id
  criterion_text            TEXT        -- specific, measurable, objective
  is_required               BOOLEAN     -- required = must be verified for APPROVED transition
  verified_by_role          TEXT        -- TECH_TEAM | CEO | JOINT
  verified_at               TIMESTAMP NULL
  
revision_requests schema (per criterion):
  id, criterion_id, requested_by, criterion_verdict, feedback_text, requested_at
```

**Criterion quality gate (on milestone definition save):** LLM validation pass checks for subjective language ("client is satisfied", "looks good"). Criteria containing subjective language are flagged with a suggested rewrite. Expert can override the flag but must acknowledge it. This reduces later disputes by forcing objective language up front.

**Milestone state transition guards (NestJS route-level):**
- `SUBMITTED` transition: all `is_required = true` DoD items must have `status = 'COMPLETED'` → else 422 with list of unchecked required items
- `APPROVED` transition: all `is_required = true` acceptance criteria must have `verified_at` set → else 422 with list of unverified criteria
- `APPROVED → RELEASED`: automatic on verification of final required criterion → triggers:
  ```sql
  -- Read fee rate from DB (never hardcode):
  SELECT platform_fee_pct FROM platform_settings LIMIT 1;
  net_expert = milestone.payment_amount_vnd * (1 - platform_fee_pct)
  fee_amount = milestone.payment_amount_vnd * platform_fee_pct

  -- Two ledger entries (both in same transaction):
  INSERT wallet_transactions (wallet_id=expert_wallet, amount=net_expert, type='ESCROW_RELEASE',
                               reference_id='ESC_RELEASE:'+milestone.id)
  INSERT wallet_transactions (wallet_id=platform_wallet_id, amount=fee_amount, type='PLATFORM_FEE',
                               reference_id='PLATFORM_FEE:'+milestone.id)
  ```
  Then: chi hộ API called for net_expert amount to expert's `bank_account_xid`

**Sign-off authority routing:**
| Milestone type | Authority | Route guard |
|---|---|---|
| Technical (model, pipeline, integration) | `CLIENT / TECH_TEAM` | `client_subtype = TECH_TEAM` |
| Business / budget (deployment, reporting, handover) | `CLIENT / CEO` | `client_subtype = CEO` |
| Joint / architecture-defining | Both CEO and TECH_TEAM | Both guards; milestone waits until both `verified_at` are set |

**Funding flow (per-milestone VA):**
```
CEO clicks "Fund Milestone N":
  1. NestJS creates per-milestone virtual account via SePay Bank Hub API:
     { entity_type: MILESTONE, entity_id: milestone_id,
       fixed_amount: milestone.payment_amount_vnd, expires_at: now + 24h }
  2. Returns VA number → frontend generates VietQR displaying fixed amount
  3. CEO scans with banking app; bank rejects if amount differs from fixed_amount
  4. SePay IPN fires: { va: milestone_va_number, transfer_type: credit, amount: X }
  5. NestJS IPN handler (MILESTONE branch):
     - Validates va_number matches a MILESTONE VA record
     - Validates amount == milestone.payment_amount_vnd
     - Atomic DB transaction:
         wallet_transactions: { type: ESCROW_LOCK, amount, reference: milestone_id }
         wallets: client.available_balance -= amount; client.locked_balance += amount
         escrow_accounts: { milestone_id, amount, status: HELD, held_at: now() }
         milestone.state → FUNDED
         milestone.state → IN_PROGRESS (auto-advance)
     - Socket.io notify both parties
```

---

**Layer 2 — Definition of Done (Expert's Self-Check)**

DoD checklist is the bridge between the expert's internal task decomposition and the public acceptance criteria. Created by the expert at milestone start (after FUNDED state). Visible to Expert (edit) and TECH_TEAM (read, comment). CEO does not see DoD — it is technical detail belonging in the TECH_TEAM relationship.

```
milestone_dod_items schema:
  id, milestone_id
  item_description          TEXT
  is_required               BOOLEAN DEFAULT TRUE
  status                    TEXT DEFAULT 'PENDING' CHECK (IN 'PENDING','COMPLETED','NOT_APPLICABLE')
  completed_at              TIMESTAMPTZ NULL
  completion_note           TEXT NULL   -- required when status → COMPLETED
  not_applicable_note       TEXT NULL   -- required when status → NOT_APPLICABLE
  maps_to_criterion_id      UUID NULL FK acceptance_criteria(id)  -- optional upward link to Layer 1

  -- DB-level guard (prevents required items from bypassing submission gate via NOT_APPLICABLE):
  CONSTRAINT dod_required_cannot_be_na
    CHECK (NOT (is_required = TRUE AND status = 'NOT_APPLICABLE'))
```

**DoD visibility:**
- Expert: full edit (mark COMPLETED / NOT_APPLICABLE; write notes)
- TECH_TEAM: read + comment thread per DoD item (lightweight thread, separate from main messaging)
- CEO: hidden entirely
- Admin: read only

**Submission gate:** On `POST /milestones/{id}/submit`, NestJS checks: `WHERE milestone_id = ? AND is_required = true AND status != 'COMPLETED'`. If any rows returned → 422 `{ code: "DOD_INCOMPLETE", unchecked_items: [...] }`. Expert cannot submit without completing all required DoD items.

---

**Layer 3 — Sprint Plan + Weekly Status (Progress Tracking)**

Sprints are the temporal decomposition of a milestone. They have no direct payment consequence — their value is early-warning of deviation and automated escalation of scope evolution.

```
milestone_sprints schema:
  id, milestone_id
  sprint_number             INT         -- UNIQUE per milestone
  created_by                UUID FK users(id)  -- always EXPERT; route guard enforces active_role=EXPERT
  week_range                TEXT        -- e.g. "Week 5–6"
  deliverable_checkpoint    TEXT
  tasks                     JSONB DEFAULT '[]'  -- [{id, description, status: 'TODO'|'IN_PROGRESS'|'DONE'}]
  UNIQUE (milestone_id, sprint_number)

sprint_status_updates schema:
  id, milestone_id
  milestone_sprint_id       UUID FK milestone_sprints(id)  -- direct FK (not implicit via sprint_number)
  sprint_number             INT   -- denormalized for display only
  submitted_by              UUID FK users(id)  -- EXPERT only; route guard enforces
  submitted_at              TIMESTAMPTZ
  status                    TEXT CHECK (IN 'ON_TRACK','DEVIATION','BLOCKER')
  deviation_note            TEXT NULL   -- required when status = DEVIATION
  blocker_type              TEXT NULL   -- required when BLOCKER: 'SCOPE_GAP'|'TECH_CONSTRAINT'|
                                        --   'EXTERNAL_DEPENDENCY'|'SCOPE_EVOLUTION'
  blocker_description       TEXT NULL
  estimated_resolution      TEXT NULL
  scope_evolution_flag      BOOLEAN DEFAULT FALSE  -- auto-set when blocker_type='SCOPE_EVOLUTION'
  resolved_at               TIMESTAMPTZ NULL
  resolved_by               UUID NULL FK users(id)
```

**Sprint comment tables (separate from main engagement messages channel):**
```
sprint_comments:
  id, milestone_sprint_id FK, commenter_id FK users
  comment_text TEXT, created_at TIMESTAMPTZ
  Route guard: WRITE = TECH_TEAM only (client_subtype='TECH_TEAM')
               READ  = TECH_TEAM + EXPERT; CEO hidden

dod_item_comments:
  id, dod_item_id FK milestone_dod_items, commenter_id FK users
  comment_text TEXT, created_at TIMESTAMPTZ
  Route guard: WRITE = TECH_TEAM only
               READ  = TECH_TEAM + EXPERT; CEO hidden
```

These comment threads are separate from `messages` (the main engagement channel). Sprint and DoD comments are technical coordination between TECH_TEAM and EXPERT; they never appear in the CEO's view.

**Sprint creation:** Expert creates sprint plan at the start of each milestone (after FUNDED state, before marking IN_PROGRESS as actively started). TECH_TEAM can comment per sprint but cannot modify the plan.

**Weekly status update cadence:** Status update due on a configurable day (default: Monday). If overdue by 3 days → reminder notification to Expert. If overdue by 5 days → both CEO and TECH_TEAM notified: *"Expert has not submitted this week's status update. You may contact them via messaging."* This is not a dispute trigger.

**Status response logic (NestJS on POST /milestones/{id}/sprint-status):**
```
ON_TRACK:
  → acknowledged; no further action; sprint dashboard updated

DEVIATION:
  → CEO and TECH_TEAM notified (deviation_note included in notification)
  → milestone clock unchanged; engagement continues

BLOCKER (type: SCOPE_GAP | TECH_CONSTRAINT | EXTERNAL_DEPENDENCY):
  → TECH_TEAM notified; milestone review clock paused
  → TECH_TEAM marks resolved via PUT /sprint-status/{id}/resolve
  → milestone clock resumes

BLOCKER (type: SCOPE_EVOLUTION):
  → scope_evolution_flag = true on sprint_status_updates record
  → F8 Add-On Phase Protocol triggered automatically
  → milestone clock paused until F8 resolves
```

**Dashboard visibility by role:**
- CEO sees: sprint number, week range, checkpoint description, status badge (ON_TRACK / DEVIATION / BLOCKER), plain-language summary of any deviation/blocker. No task list. No DoD.
- TECH_TEAM sees: full sprint task list with statuses, DoD checklist (read-only), sprint status updates (all fields), blocker descriptions, comment thread per sprint.
- Expert sees: sprint plan in edit mode (can update task statuses), submitted status update history, TECH_TEAM comment thread.

**Owner:** TV2 (Chí Nhân) — Milestone state machine (all state transitions + both guards: DOD_INCOMPLETE check on SUBMITTED, criteria verification check on APPROVED); per-milestone VA creation via SePay Bank Hub API; IPN handler MILESTONE branch (atomic ledger: escrow lock + wallet debit); criterion quality LLM check on save; sprint status logic (clock pause/resume, SCOPE_EVOLUTION → F8 auto-trigger); `milestone_dod_items`, `milestone_sprints`, `sprint_status_updates` tables
**TV3 (Cao Minh)** — Milestone management UI (CEO view: funding, status badges; TECH_TEAM view: criterion review interface, DoD read, sprint comments; Expert view: DoD checklist editor, sprint plan editor, status update form)
**Effort:** TV2: ~12 days, TV3: ~6 days

---

### F8 — Add-On Phase Protocol

**What it does:** Provides the structural mechanism for handling the inevitable scope evolution of AI projects — the feature that transforms Phase 2.1 from a conflict into a designed lifecycle event. Directly addresses the fifth structural failure mode.

**Trigger:** Expert's status update contains a Blocker flag AND the blocker description references work that requires a capability domain or seam not present in the original footprint. The expert manually classifies the blocker as one of:
- Scope adjustment (same capabilities, more time)
- Timeline extension (same scope, more time)
- Add-on phase event (genuinely new work, new capabilities)

**Add-on phase event flow:**

The add-on phase request is created automatically when a `sprint_status_updates` row with `scope_evolution_flag = true` is written. NestJS reads `sprint_status_updates.id` and sets it as `addon_phase_requests.trigger_sprint_status_id` — this FK links every add-on request to the specific sprint event that caused it, providing an unambiguous causal chain in the audit trail.

```
addon_phase_requests schema:
  id, engagement_id FK, submitted_by FK (EXPERT who filed)
  trigger_sprint_status_id  UUID FK sprint_status_updates(id) UNIQUE NULL
    -- UNIQUE partial index: one add-on per SCOPE_EVOLUTION event; prevents double-trigger
    -- NULL for any manually created add-ons (not triggered by sprint system)
  created_milestone_id      UUID FK milestones(id) NULL  -- set when MILESTONE_CREATED
  causal_chain              TEXT NOT NULL
  new_capability_requirements TEXT NOT NULL
  why_invisible             TEXT NOT NULL
  status TEXT CHECK (IN 'PENDING','TECH_APPROVED','AWAITING_CEO','FULLY_APPROVED',
                      'REJECTED','MILESTONE_CREATED')
  tech_approved_at          TIMESTAMPTZ NULL  -- set independently by TECH_TEAM
  tech_approved_by          UUID NULL FK users(id)
  ceo_approved_at           TIMESTAMPTZ NULL  -- set independently by CEO
  ceo_approved_by           UUID NULL FK users(id)
  rejected_at               TIMESTAMPTZ NULL  -- either party can reject
  rejected_by               UUID NULL FK users(id)
  rejection_reason          TEXT NULL         -- required when rejected_at is set
```

**Approval flow:**
1. Expert submits brief → `addon_phase_requests` row created (`PENDING`); `trigger_sprint_status_id` set
2. Platform routes to both sign-off authorities **independently** (not sequentially):
   - `CLIENT / TECH_TEAM` approves technical justification → `tech_approved_at` set; `status → TECH_APPROVED`
   - `CLIENT / CEO` approves budget and timeline → `ceo_approved_at` set; `status → AWAITING_CEO` then `FULLY_APPROVED` when CEO acts
3. **Either party can reject at any time** → `rejected_at` + `rejected_by` + `rejection_reason` set; engagement continues with original scope; expert must deliver within original spec or negotiate via messaging
4. Both approved → NestJS auto-creates new milestone row; `created_milestone_id` set; `status → MILESTONE_CREATED`; no admin notification
5. `CLIENT / CEO` funds the new milestone via per-milestone VA QR → IPN confirms → work proceeds

**Why this matters for the demo:** The AdTech Phase 2.1 case (LP semantic deduplication emerging after Phase 1) is the perfect demo scenario. On AITasker: the causal chain document explains exactly why Phase 2.1 is a consequence of Phase 1, the tech team confirms the LP reuse rate from their own analytics, and the amendment is processed within days. Off AITasker: the expert sends a new price quote, the client feels blindsided, both sides blame each other.

**Owner:** TV3 (Cao Minh) — Add-on phase brief form, approval workflow UI
**TV2 (Chí Nhân)** — Scope evolution log data model (Prisma), amendment record creation, add-on phase NestJS routing logic
**Effort:** TV3: ~4 days, TV2: ~2 days

---

### F9 — Real-Time Messaging

**What it does:** Covers Topic 7's "Messaging system (real-time chat)" requirement.

**Specifics:**
- One chat channel per active engagement — all three transactional roles (`CLIENT / CEO`, `CLIENT / TECH_TEAM`, `EXPERT`) share the same thread. This is intentional: the CEO needs to see the technical conversation context without having a separate side-channel.
- `ADMIN` can view all message threads (read-only) for dispute audit purposes. Admin cannot send messages.
- Real-time via Socket.io (room keyed by `engagement_id`)
- Message history persisted in PostgreSQL

**Messages schema:**
```sql
messages:
  id, engagement_id FK, sender_id FK users
  content TEXT NOT NULL
  attachments_json JSONB NULL  -- supports multiple files per message
                               -- format: [{url, filename, size_bytes}]
                               -- NULL when no attachment
  timestamp TIMESTAMPTZ DEFAULT now()
```
Note: `attachments_json` replaces the earlier single `attachment_url TEXT` field. Multiple files per message require JSONB; the array format supports display of filename and size in the UI without a separate API call.

**Read tracking — `message_reads` table:**
```sql
message_reads:
  id, message_id FK messages, user_id FK users
  read_at TIMESTAMPTZ DEFAULT now()
  UNIQUE (message_id, user_id)  -- one read record per user per message

-- Unread count query (used for notification badge):
SELECT COUNT(*) FROM messages m
WHERE m.engagement_id = :engagement_id
  AND NOT EXISTS (
    SELECT 1 FROM message_reads r
    WHERE r.message_id = m.id AND r.user_id = :current_user_id
  );
```

When a user opens a chat thread, NestJS bulk-inserts `message_reads` rows for all unread messages in that thread. Socket.io emits a `messages.read` event to other participants so their unread counts update in real time.

**Owner:** TV2 (Chí Nhân) — Socket.io server, message persistence NestJS route, `message_reads` upsert logic, unread count query, real-time `messages.read` broadcast
**TV3 (Cao Minh)** — Chat UI, attachment display, unread badge
**Effort:** TV2: ~4 days, TV3: ~2 days

---

### F10 — Internal Wallet + Escrow + SePay Full Integration

**What it does:** Covers Topic 7's "Secure payment via escrow," "Transaction history & reviews," and "Track earnings & withdraw money" requirements with a fully automated zero-admin financial architecture. Real money enters via SePay VietQR (personal bank account suffices); internal lifecycle transactions are pure DB ledger entries; expert withdrawals are automated via SePay Bank Hub + chi hộ API.

---

**Architecture summary (see Section 0.8 for full details):**

```
INBOUND  — wallet top-up, milestone funding, subscription, service purchase
  → per-entity VA QR displayed → user scans → SePay IPN (credit) → NestJS ledger

INTERNAL — escrow lock, release, fee, dispute split, subscription deduct
  → pure atomic DB transactions; no external API calls

OUTBOUND — expert withdrawal
  → NestJS calls SePay chi hộ API → SePay transfers to expert's linked bank account
  → SePay credit IPN on expert's account → withdrawal → COMPLETED
  → Zero admin involvement at any step
```

---

**SePay integration components (all owned by TV2 in NestJS):**

| Component | Purpose | Endpoint |
|---|---|---|
| VietQR payment gateway | Inbound payment QR generation | SePay Node.js SDK `POST /v1/orders` |
| Balance change IPN webhook | All inbound payment confirmations | `POST /webhooks/sepay/ipn` |
| Bank Hub Link Token API | Generate Hosted Link for expert bank account linking | `POST /bankhub/v1/link-tokens` |
| Bank Hub Hosted Link (WebView) | Expert completes bank linking via OTP (no password) | Embedded in Expert Dashboard |
| Bank Hub BANK_ACCOUNT_LINKED webhook | Capture verified `bank_account_xid` | `POST /webhooks/bankhub/events` |
| Chi hộ (disbursement) API | Programmatic outbound transfer to expert's `bank_account_xid` | `POST /chiho/v1/disburse` |

**Week 1 task (TV2):** Verify SePay chi hộ API tier requirement via hotline 02873.059.589. If chi hộ requires a tier above available, evaluate TPBank personal Open API as fallback (same Bank Hub linking, different chi hộ endpoint). Log the finding in Slack with a decision by Week 2 Day 1.

---

**IPN webhook handler (NestJS — branches by entity_type of matched VA):**

```
POST /webhooks/sepay/ipn
  1. Verify HMAC signature using SEPAY_SECRET_KEY
  2. Idempotency: UNIQUE INDEX on wallet_transactions(wallet_id, reference_id) — second retry fails
     at DB level; no application-layer check needed (but application should still catch unique violation)
  3. Look up virtual_accounts by va_number from payload → get entity_type, entity_id
  4. Branch on entity_type:

  WALLET_TOPUP:
    entity_id = user_id
    → atomic: wallet.available_balance += transferAmount
    → wallet_transaction: { type: TOP_UP, reference_id: 'TOP_UP:'+va_number+':'+sepay_ref }
    → VA.status = USED (WALLET_TOPUP VAs stay ACTIVE — user may top up again to same VA)
    → notify user: "Wallet topped up: +{amount} VND"

  MILESTONE:
    entity_id = milestone_id
    → validate: transferAmount == va.fixed_amount (exact match required)
    → atomic DB transaction:
        wallet.available_balance -= amount; wallet.locked_balance += amount
        wallet_transaction: { type: ESCROW_LOCK, reference_id: 'ESC_LOCK:'+milestone_id }
        escrow_accounts: { milestone_id, engagement_id: NULL, client_wallet_id, expert_wallet_id,
                           amount, status: HELD }
        milestone.state → FUNDED → IN_PROGRESS (auto-advance)
        paygated_docs staged for this milestone: release_state = RELEASED
    → VA.status = USED; milestone.va_number referenced for audit
    → Socket.io notify all engagement participants

  SERVICE / TECH_DISCOVERY:
    entity_id = engagement_id (the service engagement created at purchase-click time)
    → validate: transferAmount == va.fixed_amount
    → atomic:
        client.locked_balance += amount; client.available_balance -= amount
        wallet_transaction: { type: ESCROW_LOCK, reference_id: 'SVC_LOCK:'+engagement_id }
        escrow_accounts: { milestone_id: NULL, engagement_id, client_wallet_id, expert_wallet_id,
                           amount, status: HELD }
        engagements.state: PENDING → ACTIVE (SERVICE_PURCHASE created directly in ACTIVE)
    → VA.status = USED
    → notify expert: "New service order received"

  SUBSCRIPTION:
    entity_id = user_subscriptions.id (the PENDING_PAYMENT row created before VA)
    → validate: transferAmount == va.fixed_amount
    → look up user_subscriptions by id; get user_id and role_type
    → atomic:
        wallet.available_balance -= amount
        wallet_transaction: { type: SUBSCRIPTION, reference_id: 'SUB:'+user_subscriptions.id }
        UPDATE user_subscriptions SET status='ACTIVE', activated_at=now(),
                                       expires_at=now()+6months, price_paid=amount
        UPDATE users SET subscription_{role_type}_tier='pro', sub_{role_type}_expires_at=...
        VA.status = USED
    → notify user: "Your Pro subscription is active until {expires_at}"

  5. Return HTTP 200 synchronously (SePay retries on non-2xx)
```

---

**Milestone approval and payment release (atomic ledger + chi hộ):**

```
When final required acceptance criterion is verified (milestone → APPROVED):

NestJS opens DB transaction:
  fee_pct = SELECT platform_fee_pct FROM platform_settings LIMIT 1  -- read from DB; never hardcode
  platform_wallet_id = SELECT platform_wallet_id FROM platform_settings LIMIT 1
  fee = milestone.payment_amount_vnd * fee_pct
  net = milestone.payment_amount_vnd - fee

  1. client.locked_balance -= milestone.payment_amount_vnd
  2. INSERT wallet_transaction: { wallet_id: expert_wallet, type: ESCROW_RELEASE, amount: net,
                                   reference_id: 'ESC_RELEASE:'+milestone.id }
  3. INSERT wallet_transaction: { wallet_id: platform_wallet_id, type: PLATFORM_FEE, amount: fee,
                                   reference_id: 'PLATFORM_FEE:'+milestone.id }
  4. expert.available_balance += net
  5. escrow_accounts.status → RELEASED; released_at = now()
  6. milestone.state → RELEASED
  7. paygated_documents for next milestone → RELEASED
Commits.

After commit (async, non-blocking):
  → NestJS calls chi hộ API:
    POST /chiho/v1/disburse {
      amount: (milestone.payment_amount_vnd - fee),
      bank_account_xid: expert.sepay_bank_account_xid,
      reference: "MS-{milestone_id}"
    }
  → chi hộ returns { disbursement_id, status: PENDING }
  → withdrawal_requests: { type: MILESTONE_RELEASE, status: PENDING, disbursement_id }
```

---

**Bank Hub Hosted Link — Expert bank account linking:**

When expert first requests withdrawal (or prompted on registration):

```
NestJS POST /bankhub/v1/link-tokens { user_id, purpose: "LINK_BANK_ACCOUNT" }
  → returns { hosted_link_url }
  → Expert Dashboard embeds hosted_link_url in iframe/WebView
  → Expert selects bank, enters account number, enters OTP (bank-verified; no password)
  → Bank Hub fires BANK_ACCOUNT_LINKED webhook:
    { bank_account_xid, account_number, account_holder_name, brand_name }
  → NestJS webhook handler (POST /webhooks/bankhub/events):
    users.sepay_bank_account_xid = bank_account_xid
    users.bank_account_holder_name = account_holder_name  -- bank-verified
    users.bank_linked_at = now()
    → notify expert: "Bank account linked: {bank_name} | {account_number}"
```

No manual bank detail entry. Account number and holder name are bank-verified, not self-reported.

---

**Expert withdrawal (fully automated, zero admin):**

```
Expert clicks "Withdraw" [guard: EXPERT active_role; bank_account_xid must be set]
Input: { amount_vnd } (min 100,000 VND; must be ≤ available_balance)

NestJS opens DB transaction:
  1. Validate: wallet.available_balance >= amount
  2. wallet.available_balance -= amount
  3. wallet_transaction: { type: WITHDRAWAL, amount, reference: withdrawal_id }
  4. withdrawal_requests: { status: PENDING, amount, bank_account_xid, requested_at: now() }
Commits.

After commit (async):
  5. chi hộ API POST {
       amount,
       bank_account_xid: users.sepay_bank_account_xid,
       reference: "WD-{withdrawal_id}"
     }
  6. SePay processes (seconds to minutes)
  7. SePay credit IPN fires on expert's linked bank account
     → NestJS IPN handler (or Bank Hub IPN): matches reference to withdrawal_id
     → withdrawal_requests.status → COMPLETED; confirmed_at = now()
     → notify expert: "Withdrawal of {amount} VND completed to {bank_name}"

If chi hộ API returns error:
  → withdrawal_requests.status → FAILED
  → wallet.available_balance += amount  (restore balance; atomic rollback)
  → notify expert: "Withdrawal failed. Your balance has been restored."
```

---

**Dispute escrow operations (all automatic — no admin):**

Disputes are tracked in the `disputes` table with `escrow_account_id FK escrow_accounts(id)`. This direct FK makes the escrow freeze O(1) and works identically for PROJECT_BASED and SERVICE_PURCHASE engagements without branching logic.

```
disputes schema:
  id, engagement_id FK, milestone_id FK, criterion_id FK acceptance_criteria,
  escrow_account_id FK escrow_accounts(id),  -- direct FK; freeze/release is type-agnostic
  filed_by FK users,  -- CEO | TECH_TEAM | EXPERT (all three can file)
  state TEXT CHECK (IN 'PENDING','LAYER_1_EVAL','AUTO_RESOLVED','LAYER_2_MUTUAL',
                       'MUTUAL_RESOLVED','LAYER_3_FORCED','DISPUTED_AUTO_RESOLVED')
  resolution_layer INT NULL CHECK (IN 1,2,3)
  llm_confidence FLOAT NULL CHECK (BETWEEN 0 AND 1)
  layer2_deadline TIMESTAMPTZ NULL  -- now() + 48h; set when Layer 1 fails
  filed_at, resolved_at TIMESTAMPTZ

dispute_resolution_reports:  -- "report this resolution" — informational only
  id, dispute_id FK, reported_by FK users, reason_text TEXT, created_at TIMESTAMPTZ
```

```
Dispute filed:
  → INSERT disputes { state: 'LAYER_1_EVAL' }
  → escrow_accounts.status → FROZEN (no balance move)

Layer 1 LLM evaluates (disputes.llm_confidence set):
  confidence ≥ 0.80:
    → disputes.state = AUTO_RESOLVED; resolution_layer = 1
    → [LEDGER: winner receives escrow per LLM finding]
    → escrow_accounts.status → RELEASED or REFUNDED
  confidence < 0.80:
    → disputes.state = LAYER_2_MUTUAL
    → disputes.layer2_deadline = now() + 48h
    → structured mutual agreement form sent to all parties

Layer 2 both agree:
  → disputes.state = MUTUAL_RESOLVED; resolution_layer = 2
  → [LEDGER: per agreed split or full to one party]

Layer 2 48h expires:
  → disputes.state = LAYER_3_FORCED → DISPUTED_AUTO_RESOLVED; resolution_layer = 3
  → [LEDGER: 50/50 split; chi hộ for expert share; client locked_balance → available]
  → dispute_resolution_reports available for either party (informational)
```

---

**Transaction history (role-specific views):**
- `CLIENT / CEO`: funded milestones with VA numbers, amounts, dates; wallet balance and top-up history; total project spend; subscription status
- `CLIENT / TECH_TEAM`: milestone status and release dates; pay-gated document inbox; no financial amounts
- `EXPERT`: per-milestone earned amounts; wallet available balance; withdrawal history with bank confirmation timestamps; subscription status
- `ADMIN`: full ledger read — `wallet_transactions`, `escrow_accounts`, `withdrawal_requests`; no write actions

**Key DB tables for this architecture (see Section 5.3 for full schema):**
```sql
wallets           (id, user_id UNIQUE, available_balance BIGINT, locked_balance BIGINT)
wallet_transactions  -- IMMUTABLE; UNIQUE INDEX (wallet_id, reference_id) for idempotency
virtual_accounts  (entity_type, entity_id, va_number UNIQUE, fixed_amount NULL,
                   expires_at NULL, status)
platform_settings -- SINGLETON; platform_wallet_id FK wallets; platform_fee_pct FLOAT
user_subscriptions (status: PENDING_PAYMENT|ACTIVE|EXPIRING_SOON|EXPIRED)
escrow_accounts   (milestone_id NULL, engagement_id NULL; exactly one non-null; dual partial unique indexes)
disputes          (engagement_id, milestone_id, criterion_id, escrow_account_id FK,
                   filed_by, state, llm_confidence, layer2_deadline)
dispute_resolution_reports (dispute_id, reported_by, reason_text)
withdrawal_requests (type: MILESTONE_RELEASE|EXPERT_MANUAL; status: PENDING|PROCESSING|COMPLETED|FAILED;
                     bank_account_xid, disbursement_id, telegram_notified_at)
```

**Owner:** TV2 (Chí Nhân) — All SePay integrations (IPN handler with all branches; Bank Hub Link Token API; Hosted Link WebView embed; BANK_ACCOUNT_LINKED webhook handler; chi hộ API calls; chi hộ confirmation IPN handler); full internal ledger atomic transactions; withdrawal flow; dispute escrow operations; all tables above
**TV3 (Cao Minh)** — Wallet top-up UI; subscription purchase UI; transaction history UI; expert earnings dashboard; Bank Hub link prompt in Expert Dashboard
**Effort:** TV2: ~12 days, TV3: ~4 days

**SePay setup (Week 1-2, TV2):**
- Register at my.sepay.vn (personal bank account sufficient; VietQR activates in 30 min)
- Activate Bank Hub → get Link Token API credentials
- Verify chi hộ tier availability (hotline call Week 1)
- Configure IPN endpoints: `POST /webhooks/sepay/ipn` (inbound) + `POST /webhooks/bankhub/events` (Bank Hub)
- Store: `SEPAY_MERCHANT_ID`, `SEPAY_SECRET_KEY`, `BANKHUB_CLIENT_ID`, `BANKHUB_CLIENT_SECRET`, `CHIHO_API_KEY` in EC2 `.env`

---

### F11 — Reviews & Reputation System

**What it does:** Covers Topic 7's "reviews" requirement, extended with structured seam-specific outcome signals that feed back into the verification system.

**Post-engagement review (three separate forms, role-gated):**

*`CLIENT / TECH_TEAM` reviews the Expert (structured — feeds seam signals):*
- Overall rating (1–5 stars)
- Seam-specific performance questions (subset of the outcome signal questionnaire):
  - "Did the expert raise the ground truth baseline requirement proactively or after problems emerged?" (A↔C seam signal)
  - "Did the expert's technical proposal require significant revision after seeing the actual system?" (scope accuracy signal)
  - "When scope evolution occurred, was the causal chain explanation convincing?" (add-on phase quality signal)
**Signals feed into expert's verification tier updates and Tier 4 signal accumulation:**

The TECH_TEAM's structured review produces two rows: one `outcome_signals` aggregate row (keyed `outcome_signals.review_id FK reviews(id) UNIQUE` — enforcing 1:1 between a review and its aggregate), and N `expert_seam_outcome_signals` rows — one per seam mentioned in `seam_signals_json`. NestJS extracts the per-seam rows after each review save:

```
expert_seam_outcome_signals schema:
  id, engagement_id FK, expert_id FK, reviewer_id FK (TECH_TEAM user)
  outcome_signal_id FK outcome_signals(id)  -- aggregate that generated this row
  seam_claim_id FK expert_seam_claims(id)   -- direct FK for Tier 4 UPDATE write-back
  seam_code TEXT, signal_type TEXT (LOAD_BEARING|SIGNIFICANT|CONTRIBUTING)
  valence TEXT (POSITIVE|NEGATIVE)
  seam_role TEXT  -- how central this seam was to the actual engagement

Indexes: (expert_id, seam_code, valence, signal_type)
  -- powers both Tier 4 accumulation query and matching hard gate query
```

**Tier 4 accumulation rule (evaluated by NestJS on every engagement CLOSED):**
```sql
-- Load-bearing: 1 positive signal → immediate Tier 4
SELECT COUNT(*) FROM expert_seam_outcome_signals
WHERE expert_id = :expert_id AND seam_code = :seam_code
  AND signal_type = 'LOAD_BEARING' AND valence = 'POSITIVE' → if >= 1 → Tier 4

-- Significant: 2 positive signals → Tier 4
SELECT COUNT(*) FROM expert_seam_outcome_signals
WHERE expert_id = :expert_id AND seam_code = :seam_code
  AND signal_type = 'SIGNIFICANT' AND valence = 'POSITIVE' → if >= 2 → Tier 4

-- Contributing: 3 positive signals → Tier 4
SELECT COUNT(*) FROM expert_seam_outcome_signals
WHERE expert_id = :expert_id AND seam_code = :seam_code
  AND signal_type = 'CONTRIBUTING' AND valence = 'POSITIVE' → if >= 3 → Tier 4
```

**Matching hard gate:** `SELECT COUNT(*) ... WHERE valence='NEGATIVE' AND signal_type='LOAD_BEARING'` → if >= 2 → expert excluded from shortlist for any project where that seam is load-bearing.

*`CLIENT / CEO` reviews the Expert (business-facing):*
- Overall rating (1–5 stars)
- Communication clarity rating
- Milestone structure effectiveness rating
- Open text

*`EXPERT` reviews the engagement (both client sub-roles are assessed together — the Expert experienced them as a unit):*
- Overall rating (1–5 stars)
- Milestone approval responsiveness: "Did the designated sign-off authority review within the agreed window?" (reflects on TECH_TEAM for technical milestones, CEO for business milestones)
- Technical information availability: "Was Artifact B complete and accurate when you first accessed it?" (reflects on TECH_TEAM's handoff quality)
- CEO communication: "Were budget and timeline decisions made promptly and clearly?"
- Open text

**Reputation display (on expert profile):**
- Engagement count and completion rate
- Average rating (overall)
- Seam performance indicators: which seams have produced proactive signals vs. reactive signals vs. absent signals across all platform engagements
- CEO and TECH_TEAM approval cycle time averages (tracked separately — CEO for business milestones, TECH_TEAM for technical milestones)

**Owner:** TV3 (Cao Minh) — Review forms, reputation display
**TV2 (Chí Nhân)** — Review signal processing (reads structured questionnaire responses, updates `expert_seam_claims.verification_tier` and `expert_profiles.earned_balance` via Prisma)
**Effort:** TV3: ~4 days, TV2: ~2 days

---

### F12 — Admin Dashboard

**What it does:** Covers Topic 7's Admin requirements. In the zero-admin operational model, the Admin Dashboard is a **monitoring and exception surface**, not an operational queue. Every routine decision (spec publish/reject, seam verification upgrades, scenario grading, dispute resolution, payout confirmation) is handled by automated systems. Admin's role is: emergency intervention, platform integrity monitoring, and research data access. Admin should never need to log in to the platform to keep it running day-to-day.

**Design principle:** Every module in this dashboard is read-first. The only write actions available to admin are emergency interventions (spec pull-back, account suspension) — not routine approvals.

---

**Module 1 — Platform Integrity Monitor:**

The primary admin view. Shows the automated system's full decision history:

- *Spec auto-return log:* Every spec that failed the automated quality gate, with the specific void that caused failure, the LLM-generated advisory note sent to the client, and the client's subsequent re-entry point in the elicitation engine. `platform_decisions.entity_type = 'project'`; `entity_id = projects.id`.
- *Seam verification log:* Every auto-upgrade (Tier 1→2, 2→3) with the LLM confidence score and the signal types found. Every auto-return with the gap advisory sent. Every submission locked due to the 5-attempt limit. `entity_type = 'portfolio_submission'` or `'scenario_response'`; `entity_id = respective table UUID`. Admin reads this to audit LLM decision quality.
- *Criterion quality gate log:* Every acceptance_criterion that triggered the LLM subjective-language check on save, with the original text and the suggested rewrite. `entity_type = 'acceptance_criterion'`.
- *Bid conflict override log:* Every Surface D override. `entity_type = 'capability_bid'`. Used for RQ3 CEO/TECH_TEAM conflict → engagement failure correlation.
- *Tier 4 elevation log:* Every automatic Tier 4 elevation with seam, engagement, signal type, and outcome score. Derived from `expert_seam_outcome_signals` evaluation.
- *Dispute resolution log:* Every dispute filed, with resolution layer, `llm_confidence`, and any `dispute_resolution_reports` submitted. `entity_type = 'acceptance_criterion'` (disputes are always against a criterion).

`platform_decisions` is written by FastAPI only — never by NestJS. The `entity_type` column is a required companion to the polymorphic `entity_id`, allowing the Platform Integrity Monitor to JOIN the correct source table for detail views without a union query.

**Emergency write action:** Admin can **pull back any published spec** (`spec.state → SUSPENDED`) with a reason note visible to the client but not to experts. This is the only write action in this module.

---

**Module 2 — Account Management:**

- View all users with role, registration date, engagement count, and verification tier history
- View any user's full profile, engagement history, and seam verification log
- Suspend / reactivate accounts (for genuine abuse cases — rare, manual by design)
- Automated behavioral flags surfaced here for review (read-only):
  - Claim-to-verified ratio > 4:1 → system already excludes from Tier 2+ projects; admin can see who is flagged
  - 5-attempt submission lockout hits — expert notified; admin sees in log
  - Premature dispute pattern (client disputes without review clock expiring 3+ times) — system has restricted dispute access; admin sees the flag
  - LLM content moderation rejections (portfolio submissions, bid proposals, service descriptions rejected by the auto-filter)
- *Fraud signal alert (rare — Telegram notification, no platform action required):* If the same bank account number appears on two different expert withdrawal requests, NestJS auto-blocks both withdrawals and sends an informational Telegram notification. Admin investigates externally and manually reactivates once resolved.

---

**Module 3 — Spec Emergency Queue:**

This queue should be **empty under normal operation.** It contains only specs that admin has manually suspended post-publication. A non-empty queue means either a factual error was reported in a published spec or a pattern of elicitation failures needs investigation. Admin reviews the suspension reason for each entry and either restores the spec (if the issue is resolved) or sends a manual advisory to the client via the platform.

There is no queue for failed specs — those are handled entirely by the auto-return system and never surface here.

---

**Module 4 — Dispute Monitor:**

- View all disputes filed, current resolution layer, and the automated determination
- View Layer 2 mutual agreement form state: which options each party has selected and how much time remains in the cooling window
- View "Report this resolution" submissions — logged text entries, informational only, no required action
- **No write actions.** Disputes resolve through the 3-layer automated system (see Section 0.6). Admin does not issue determinations, does not manually freeze or release escrow.
- *Edge case flag:* If a Type 2 dispute (process violation) generates a "Report this resolution" flag AND the audit trail shows a clear protocol deviation, the system surfaces a special "Requires external review" flag. Admin may contact the parties outside the platform. This is expected to be extremely rare at SWP scale.

---

**Module 5 — Transaction Ledger & Research Data:**

- Full ledger view: all `escrow_records` with SePay transaction IDs, milestone references, funded/released/disputed timestamps
- Withdrawal audit trail: all `withdrawal_requests` with type (MILESTONE_RELEASE / EXPERT_MANUAL), status, `disbursement_id` (SePay chi hộ reference), `confirmed_at` (set when SePay credit IPN fires on expert's linked account), chi hộ timeline from request to confirmation
- Research data exports for final report: match score distributions, elicitation completion rates, seam verification upgrade rates by confidence band, dispute layer distribution, milestone completion rates, review ratings — direct evidence for RQ1, RQ2, RQ3
- Filter by date range, archetype, tier, expert, engagement state

---

**Module 6 — Analytics Dashboard:**

- Active projects count, by archetype and tier
- Average composite matching score for connected engagements vs. declined bids (RQ1 evidence)
- Elicitation completion rate — % of intake sessions that reach spec publication (RQ2 evidence)
- Auto-publish pass rate — % of specs that pass the quality gate on first synthesis
- Seam verification throughput: submissions per day, auto-upgrade rate, auto-return rate, average resubmissions before upgrade (RQ1 evidence on LLM verification quality)
- Milestone completion rate, average revision cycles
- Dispute rate, Layer 1 auto-resolution rate, average time-to-resolution by layer (RQ3 evidence — trust indicator)
- Review completion rate, average ratings

**Owner:** TV2 (Chí Nhân) — Admin dashboard framework, account management module, Platform Integrity Monitor data aggregation (reads `platform_decisions` log), spec pull-back route, fraud signal detection (bank account deduplication: same `bank_account_xid` on two withdrawal requests → auto-block both + flag in admin dashboard), dispute monitor read interface, subscription guard middleware
**TV3 (Cao Minh)** — Analytics dashboard UI, transaction ledger UI, dispute monitor UI, research data export interface
**TV1 (Minh Hùng)** — FastAPI `platform_decisions` log writes: every LLM auto-upgrade, auto-return, dispute Layer 1 assessment, criterion quality gate, and content moderation decision is written with `entity_type` + `entity_id` so the Platform Integrity Monitor can JOIN the correct source table
**Effort:** TV2: ~8 days, TV3: ~4 days, TV1: ~2 days

---

## 4. Use Cases

### Use Case Map

```
CLIENT FLOWS — CLIENT / CEO
  UC01   Submit project via AI elicitation engine (Pro subscription required)
  UC01a  Proceed via Technical Discovery pathway — Scenario A (no TECH_TEAM)
  UC01b  Complete Stage 4 as self-technical CEO — Scenario B
  UC02   Top up platform wallet via WALLET_TOPUP VA VietQR
  UC03   Purchase Client Pro subscription from wallet balance
  UC04   View Artifact A and expert shortlist (Pro)
  UC05   Respond to expert spec clarification request (Surface A — Tier 1 projects)
  UC06b  Approve expert selection (after reviewing TECH_TEAM bid analysis + flags)
  UC06c  Override TECH_TEAM Not Recommended flag with mandatory reason (Surface D)
  UC06d  Propose price adjustment in negotiation window (Surface C — max 2 rounds)
  UC07   Send connection request to expert; complete NDA click-through
  UC08   Fund milestone via per-milestone VA QR (from wallet or direct bank scan)
  UC09   Approve business / budget milestones — triggers escrow release + chi hộ
  UC10   Approve add-on phase — budget & timeline dimension
  UC11   Buy expert service directly — SERVICE_PURCHASE or TECH_DISCOVERY (Free tier OK)
  UC12a  Complete post-engagement review (CEO form)

CLIENT FLOWS — CLIENT / TECH_TEAM (account created via signed handoff link in UC01)
  UC01t  Complete tech team architecture handoff (TECH_TEAM-only route — CEO guarded out)
  UC04   View Artifact A (shared read access with CEO)
  UC05t  Respond to expert spec clarification (Surface A — Tier 2+ projects)
  UC06a  Review capability bids — detailed seam analysis; flag Recommended / Not Recommended
  UC06r  Request bid component revision (Surface B)
  UC07b  Access Artifact B post-connection (TECH_TEAM-only; CEO guarded out at DB level)
  UC08t  Sign off technical milestones criterion by criterion
  UC13t  Read sprint plan and DoD checklist; comment on sprint thread
  UC10t  Approve add-on phase — technical justification dimension
  UC12b  Complete post-engagement review (Tech Team structured seam signal form)

EXPERT FLOWS
  UC13   Create taxonomy-based profile and seam declarations
  UC14   Purchase Expert Pro subscription from wallet balance
  UC15   Submit portfolio evidence for LLM auto-verification
  UC16   Request and complete scenario assessment (Tier 3 auto-delivery + LLM auto-grading)
  UC17   Link bank account via Bank Hub Hosted Link (Bank Hub WebView; OTP-only)
  UC18   Publish AI service listing via AI generator (Path B marketplace)
  UC19   Browse project shortlist → file spec clarification request before bidding (Surface A)
  UC20   Submit structured capability bid (3 required components)
  UC20r  Revise specific bid component on REVISION_REQUESTED (Surface B)
  UC20c  Accept / counter / decline price negotiation proposal (Surface C)
  UC21   Accept connection and access Artifact B
  UC22   Stage pay-gated reasoning documents with milestone release trigger
  UC23   Create sprint plan + DoD checklist for funded milestone
  UC24   Submit weekly sprint status update (ON_TRACK / DEVIATION / BLOCKER)
  UC25   Submit milestone deliverable (DoD gate enforced at route level)
  UC26   Submit add-on phase brief with causal chain (auto-triggered from SCOPE_EVOLUTION status)
  UC27   Request withdrawal (chi hộ fires automatically; bank_account_xid required)
  UC28   Complete post-engagement review (Expert form — CEO + TECH_TEAM dimensions separately)

DUAL-ROLE FLOWS (accounts with roles: ["CLIENT_CEO", "EXPERT"])
  UC-DR1 Add second role to account (account settings → role acquisition)
  UC-DR2 Switch active role via role switcher — JWT reissued; dashboard changes
  UC-DR3 Fund project milestone from expert earnings (same wallet; no external top-up needed)

ADMIN FLOWS (monitoring only — zero routine approvals)
  UC-A1  Emergency spec pull-back (published spec only — reactive)
  UC-A2  Monitor Platform Integrity Monitor (seam verification log, auto-return log, dispute log)
  UC-A3  Monitor transaction ledger (wallet_transactions, escrow_accounts, withdrawal_requests)
  UC-A4  Monitor bid conflict override log (Surface D decisions)
  UC-A5  Monitor platform analytics and export research data

GENERAL FLOWS
  UC30  Browse Path B expert marketplace [CEO, TECH_TEAM — Free tier accessible]
  UC31  Real-time messaging within engagement [all transactional roles; ADMIN read-only]
  UC32  View wallet, transaction history, subscription status [all roles — role-specific panels]
```

### Critical Use Case Detail — UC01: AI Elicitation Engine

> **Two actors, one flow.** The CEO drives Stages 1–3; the TECH_TEAM drives Stage 4 via a separate account created by the handoff link. They never share a screen during elicitation.

| Step | Actor | Action | System response |
|---|---|---|---|
| 1 | `CLIENT / CEO` | Opens "Post a Project" — sees behavioral intake prompt, not a form | Intake interface displayed; CEO dashboard only |
| 2 | `CLIENT / CEO` | Types free-form symptom description | LLM extraction runs: intent separation, scale signals, void detection |
| 3 | System | — | Presents archetype options in behavioral language |
| 4 | `CLIENT / CEO` | Selects archetype that matches their goal | Archetype locked. SDLC injection sequence begins |
| 5 | System | — | Presents first injection: ground truth void detected → explains why Phase 0 is mandatory |
| 6 | `CLIENT / CEO` | Accepts Phase 0 milestone | Phase 0 injected as Milestone 1. Subsequent injections follow |
| 7 | System | — | Runs 4 behavioral architecture probes |
| 8 | `CLIENT / CEO` | Answers all 4 probes | Infrastructure threshold evaluation runs |
| 9 | System | — | Infrastructure threshold crossed → generates signed tech handoff link (`client_subtype: TECH_TEAM` encoded in token) |
| 10 | `CLIENT / CEO` | Sends link to their technical lead | Tech handoff link active, 72-hour expiry. CEO cannot complete Stage 4 themselves. |
| 11 | `CLIENT / TECH_TEAM` | Opens link → registers account (or logs in if already registered under a different project) → sees security framing → inputs stack/data/deployment tags → optionally uploads sensitive schema | TECH_TEAM account linked to `project_id`. Inputs written to Artifact B staging. Stack tags go to Artifact A |
| 12 | System | — | Synthesis engine resolves CEO vs. TECH_TEAM signal conflicts. Footprint and Artifact A generated |
| 13 | System | — | Automated quality gate runs: completeness score, matching pre-check, void resolution check |
| 13a | System | **All checks pass (expected path)** | Artifact A auto-published. Matching engine generates shortlist. `CLIENT / CEO` notified. |
| 13b | System | **Any check fails (exception path)** | Spec state set to `RETURNED_TO_CLIENT`. LLM generates targeted advisory note identifying the specific void. `CLIENT / CEO` re-enters elicitation at that exact point. No admin action. |
| *(N/A)* | *(no admin step)* | *(exception path fully automated — no admin involvement)* | CEO receives advisory note and resumes elicitation from the specific void that failed |

### Critical Use Case Detail — UC15: Expert Submits Capability Bid

| Step | Actor | Action | System response |
|---|---|---|---|
| 1 | Expert | Receives bid invitation, views Artifact A and seam gap map for their profile | Match card displayed with their specific gap analysis |
| 2 | Expert | Opens bid form | Footprint alignment section pre-populated from their profile |
| 3 | Expert | Confirms domain depth alignment, seam claims | Any claimed-only seams on load-bearing positions are flagged: *"This seam is load-bearing and you have no verification. You can proceed with your bid, but the client will see this gap. Would you like to complete the seam scenario assessment first?"* |
| 4 | Expert | Writes architectural approach summary | Must address the SDLC milestone framework and any stated-approach discrepancy flagged in Artifact A |
| 5 | Expert | Enters conditional milestone pricing | Conditionals validated: must have resolution condition and price range |
| 6 | Expert | Submits bid | Bid enters review queue. Expert can no longer see other experts' bids |

---

## 5. Architecture & Tech Stack

### 5.1 System Architecture

The system follows a Service-Oriented Architecture with two backend services — a Main API (NodeJS/NestJS) for business logic and marketplace operations, and an AI Service (FastAPI) for the elicitation engine, matching computation, and LLM calls — both sharing a PostgreSQL database.

```
┌──────────────────────────────────────────────────────────┐
│                    PRESENTATION LAYER                      │
│         ReactJS + Tailwind CSS + Socket.io (client)       │
│  CEO Dashboard │ Tech Dashboard │ Expert App │ Admin Panel │
└────────────────────────┬─────────────────────────────────┘
                         │ HTTPS / WebSocket
┌────────────────────────▼─────────────────────────────────┐
│                  APPLICATION LAYER                         │
│                                                            │
│  ┌─────────────────────────┐  ┌─────────────────────────┐ │
│  │   MAIN API              │  │   AI SERVICE             │ │
│  │   NodeJS / NestJS       │  │   FastAPI / Python       │ │
│  │                         │  │                          │ │
│  │  • Auth (JWT)           │  │  • Elicitation Engine    │ │
│  │  • User / Role CRUD     │  │  • LLM Extraction Layer  │ │
│  │  • Project Management   │  │  • Matching Computation  │ │
│  │  • Milestone Lifecycle  │  │  • Seam Gap Map Gen      │ │
│  │  • Messaging (Socket)   │  │  • Footprint Synthesis   │ │
│  │  • SePay Escrow         │  │  • AI Service Generator  │ │
│  │  • SePay IPN Handler    │  │  • Portfolio Evidence    │ │
│  │  • Reviews              │  │    Extraction            │ │
│  │  • Admin Operations     │  │  • Artifact B access     │ │
│  │                         │  │    control (DB read)     │ │
│  │  Validation: Zod / DTO  │  │  • Pay-gated doc serve   │ │
│  │  ORM: Prisma            │  │    (DB read only)        │ │
│  │  Migrations: Prisma Mg  │  │  Validation: Pydantic    │ │
│  └─────────────┬───────────┘  │  ORM: SQLAlchemy         │ │
│                │ Internal REST │  Migrations: Alembic     │ │
│                └──────────────│──────────────┬───────────┘ │
│                               └──────────────┘             │
└───────────────────────────────│───────────────────────────┘
                                │
┌───────────────────────────────▼──────────────────────────┐
│                      DATA LAYER                            │
│                PostgreSQL (shared)                         │
│   Users │ Profiles │ Projects │ Footprints │ Artifacts    │
│   Milestones │ Bids │ Messages │ escrow_records │ Reviews  │
│   service_purchases │ withdrawal_requests                  │
└──────────────────────────────────────────────────────────┘
                                │
┌───────────────────────────────▼──────────────────────────┐
│                  EXTERNAL SERVICES                         │
│  Anthropic Claude API  (Elicitation + Service Generator   │
│                         + Criterion quality check)         │
│  SePay VietQR + VA     (Inbound: wallet, milestone,       │
│                          subscription, service payments)   │
│  SePay Bank Hub        (Expert bank account linking via   │
│                          Hosted Link; WebView embed)       │
│  SePay Chi Hộ API      (Expert withdrawal disbursement;   │
│                          credit IPN confirms completion)   │
│  Email service  (tech handoff link, notifications)        │
└──────────────────────────────────────────────────────────┘
                                │
┌───────────────────────────────▼──────────────────────────┐
│              DEPLOYMENT (AWS EC2 — cloud-based)            │
│  EC2 t3.micro · Docker Compose · Elastic IP               │
│  GitHub Actions → SSH deploy on merge to main             │
└──────────────────────────────────────────────────────────┘
                                │
┌───────────────────────────────▼──────────────────────────┐
│                    QA LAYER (Internal)                     │
│  Postman / Apidog collections  │  ASP.NET QA Dashboard    │
│  GitHub Actions (lint + basic CI)                         │
└──────────────────────────────────────────────────────────┘
```

### 5.2 Tech Stack by Layer

| Layer | Technology | Owner | Rationale |
|---|---|---|---|
| **Frontend** | ReactJS, React Router v6, Axios, Tailwind CSS, Socket.io-client | TV2, TV3 | Team's existing strength; fast iteration with component libraries |
| **Main API** | NodeJS, NestJS framework, JWT (access + refresh), Zod validation | TV2 | Mirrored from team's existing SWP experience; NestJS provides structure at scale |
| **Main ORM** | Prisma ORM, Prisma Migrate | TV2 | Type-safe queries, versioned schema, familiar to TV2 |
| **AI Service** | FastAPI, Pydantic v2, SQLAlchemy 2.0, Uvicorn | TV1 | TV1's core stack; excellent LLM API integration; async-native |
| **AI Migrations** | Alembic | TV1 | Standard with SQLAlchemy, familiar to TV1 |
| **Database** | PostgreSQL 16 (single instance, shared by both services) | TV1 (schema), TV2 (Prisma models) | Single source of truth; both services query same DB via their respective ORMs |
| **LLM API** | Anthropic Claude claude-sonnet-4-6 | TV1 | Available via Anthropic API; strong structured output; used for elicitation extraction, synthesis, service generator |
| **Payment** | SePay VietQR — Node.js SDK integrated into NestJS. MERCHANT_ID + SECRET_KEY from my.sepay.vn. All payment order creation, IPN webhook handling, escrow state machine, and withdrawal logic live exclusively in NestJS. FastAPI has zero involvement in payment flows. Sandbox available immediately; VietQR production activated in ~30 min | TV2 | No enterprise account required; VietQR activates in 30 minutes; Node.js SDK officially supported; covers both service purchases and milestone escrow |
| **Real-time** | Socket.io (server on NestJS, client on React) | TV2 | Straightforward integration with NestJS; well-documented |
| **State-gated Artifact B** | FastAPI serves elicitation-generated Artifact B data with a passive DB state read (`SELECT engagement.state FROM engagements WHERE id = ?`) — if state ≥ CONNECTED, return data; else return 403. No writes, no business logic. All engagement state transitions are owned by NestJS | TV1 | Artifact B lives in the AI service because it was produced by the elicitation engine; serving it there avoids a cross-service call on every request |
| **Email** | SendGrid (free tier) or Nodemailer + SMTP | TV2 | Tech handoff link delivery, milestone notifications |
| **Auth** | JWT (access token 15min, refresh token 7 days), bcrypt password hashing | TV2 | Standard, team-familiar |
| **File uploads** | Multer (NestJS) + local disk storage on EC2 (MVP) | TV2 | Simple for demo; can be replaced with S3 later |
| **Deployment** | AWS EC2 t3.micro (free tier), Docker Compose, Elastic IP, GitHub Actions SSH deploy on merge to `main` | TV5 | Satisfies Topic 7 explicit cloud-based scalable architecture requirement; free tier covers the SWP window; Docker Compose keeps service orchestration simple |
| **QA** | Postman/Apidog, ASP.NET Core QA dashboard (TV5's tool) | TV5 | Existing tool investment |
| **CI/CD** | GitHub Actions — lint + build check on PR; SSH deploy to EC2 on merge to `main` | TV5 | Lightweight CI; automated deploy to cloud removes "works on my machine" demo risk |
| **Dev workflow** | GitHub, `main` / `dev` / `feature/*` branching, PR required for `dev` merge | TV5 | Established from prior SWP |

### 5.3 Database Schema Overview

> **Canonical source:** The Physical ER Plan (AITasker Physical ER v2) is the authoritative field-level specification. This section provides an annotated structural overview with all 51 tables, critical fields, and constraints. Migration must follow the FK dependency order at the end of this section.

**Global conventions:** PKs = `UUID DEFAULT gen_random_uuid()` (requires pgcrypto); timestamps = `TIMESTAMPTZ`; money = `BIGINT` (VND integer); JSON = `JSONB`; enums via `TEXT CHECK`; all columns `NOT NULL` unless marked `NULL`.

**Seeded at deploy (run before first user registers):**
```sql
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
INSERT INTO users (id, email, password_hash, full_name, roles, active_role, is_active)
  VALUES ('00000000-0000-0000-0000-000000000001', 'platform@aitasker.internal',
          'SEEDED_NO_LOGIN', 'AITasker Platform', '["ADMIN"]', 'ADMIN', true);
INSERT INTO wallets (user_id) VALUES ('00000000-0000-0000-0000-000000000001');
-- Store returned wallet id as env var PLATFORM_FEE_WALLET_ID
INSERT INTO platform_settings (platform_wallet_id, platform_fee_pct)
  VALUES (:platform_fee_wallet_id, 0.05);
```

**Section 1 — Users & Role Profiles**
```sql
users:
  id UUID PK, email TEXT UNIQUE, password_hash TEXT, full_name TEXT, phone TEXT NULL
  roles JSONB DEFAULT '[]'                     -- ["CLIENT_CEO","EXPERT"] for dual-role
  active_role TEXT CHECK (IN 'CLIENT','EXPERT','ADMIN')
  client_subtype TEXT NULL CHECK (IN 'CEO','TECH_TEAM')
  subscription_client_tier TEXT DEFAULT 'free' CHECK (IN 'free','pro')
  subscription_expert_tier TEXT DEFAULT 'free' CHECK (IN 'free','pro')
  sub_client_expires_at TIMESTAMPTZ NULL
  sub_expert_expires_at TIMESTAMPTZ NULL
  sepay_bank_account_xid TEXT NULL             -- set by BANK_ACCOUNT_LINKED webhook
  bank_account_holder_name TEXT NULL           -- bank-verified; from Bank Hub
  bank_linked_at TIMESTAMPTZ NULL
  self_technical BOOLEAN DEFAULT FALSE         -- Scenario B; grants TECH_TEAM capabilities on project
  is_active BOOLEAN DEFAULT TRUE, created_at TIMESTAMPTZ DEFAULT now()

client_profiles:  user_id PK FK users; company_name, industry, ceo_name TEXT NULL
expert_profiles:  user_id PK FK users; bio TEXT NULL; engagement_model TEXT NULL;
  archetype_history_json JSONB NULL  -- cold-start seed for 20% Archetype-tier congruence;
                                     -- overridden at query time by confirmed engagements
tech_team_profiles:
  user_id PK FK users
  linked_client_id FK users        -- CEO who issued the handoff link
  linked_project_id FK projects    -- scope guard: account locked to one project
  role_title TEXT NULL
```

**Section 2 — Expert Capability**
```sql
expert_domain_depths:
  expert_id FK, domain_code TEXT CHECK (IN 'DOMAIN_A'..'DOMAIN_F')
  depth_level TEXT CHECK (IN 'SURFACE','OPERATIONAL','DEEP')
  verification_tier TEXT DEFAULT 'CLAIMED' CHECK (IN 'CLAIMED','EVIDENCE_BACKED',
                                                    'SCENARIO_VERIFIED','PLATFORM_DEMONSTRATED')
  UNIQUE (expert_id, domain_code)

expert_seam_claims:
  expert_id FK, seam_code TEXT  -- 'SEAM_AC'|'SEAM_AF'|'SEAM_AD'|'SEAM_DE'|'SEAM_DF'|
                                 -- 'SEAM_CF'|'SEAM_EF'|'SEAM_AB'|'SEAM_BE'|'SEAM_CE'
  verification_tier TEXT DEFAULT 'CLAIMED'; submission_count INT DEFAULT 0
  locked_until TIMESTAMPTZ NULL  -- 30-day lockout after 5th failed submission
  UNIQUE (expert_id, seam_code)

expert_stack_tags:  expert_id FK; tag TEXT; UNIQUE (expert_id, tag)

portfolio_submissions:
  expert_id FK, seam_claim_id FK expert_seam_claims  -- targets specific claim (not ambiguous)
  project_description TEXT, decision_points TEXT
  status TEXT DEFAULT 'PENDING' CHECK (IN 'PENDING','APPROVED','REJECTED')
  llm_confidence FLOAT NULL     -- >=0.85 triggers auto-upgrade to Tier 2
  submitted_at TIMESTAMPTZ, evaluated_at TIMESTAMPTZ NULL

scenario_assessments:
  seam_code TEXT, question_text TEXT, rubric_json JSONB
  last_used_at TIMESTAMPTZ NULL  -- NestJS selects lowest to prevent gaming

scenario_responses:
  assessment_id FK, expert_id FK, seam_claim_id FK expert_seam_claims
  response_text TEXT; pass_fail TEXT NULL CHECK (IN 'PASS','FAIL','PENDING')
  llm_rubric_scores_json JSONB NULL  -- per-dimension; all must pass independently

expert_seam_outcome_signals:
  engagement_id FK, expert_id FK, reviewer_id FK users (TECH_TEAM)
  outcome_signal_id FK outcome_signals
  seam_claim_id FK expert_seam_claims    -- direct FK for Tier 4 UPDATE write-back
  seam_code TEXT
  signal_type TEXT CHECK (IN 'LOAD_BEARING','SIGNIFICANT','CONTRIBUTING')
  valence TEXT CHECK (IN 'POSITIVE','NEGATIVE')  -- NEGATIVE feeds matching hard gate
  seam_role TEXT, created_at TIMESTAMPTZ
  INDEX: (expert_id, seam_code, valence, signal_type)  -- Tier 4 + hard gate queries
```

**Section 3 — Wallet, Finance & Subscriptions**
```sql
wallets:
  user_id UUID UNIQUE FK users
  available_balance BIGINT DEFAULT 0 CHECK (>= 0)
  locked_balance BIGINT DEFAULT 0 CHECK (>= 0)

wallet_transactions:  -- IMMUTABLE; never UPDATE after INSERT
  wallet_id FK wallets, amount BIGINT CHECK (> 0)  -- always positive; direction from type
  transaction_type TEXT CHECK (IN 'TOP_UP','SUBSCRIPTION','ESCROW_LOCK','ESCROW_RELEASE',
                                 'PLATFORM_FEE','ESCROW_REFUND','ESCROW_SPLIT','WITHDRAWAL')
  reference_id TEXT NULL  -- 'TOP_UP:{va}:{ref}' | 'ESC_LOCK:{milestone_id}' |
                          --  'ESC_RELEASE:{milestone_id}' | 'SUB:{subscription_id}' |
                          --  'WITHDRAWAL:{withdrawal_id}' | 'PLATFORM_FEE:{milestone_id}'
  created_at TIMESTAMPTZ
  -- CRITICAL idempotency index (prevents SePay IPN double-credit on retry):
  UNIQUE INDEX wallet_tx_idempotency ON wallet_transactions(wallet_id, reference_id)
    WHERE reference_id IS NOT NULL;

virtual_accounts:
  entity_type TEXT CHECK (IN 'WALLET_TOPUP','MILESTONE','SERVICE','SUBSCRIPTION')
  entity_id TEXT   -- user_id | milestone_id | engagement_id | user_subscriptions.id
  va_number TEXT UNIQUE, fixed_amount BIGINT NULL
  expires_at TIMESTAMPTZ NULL  -- NULL for WALLET_TOPUP; 24h for others
  status TEXT DEFAULT 'ACTIVE' CHECK (IN 'ACTIVE','EXPIRED','USED')

user_subscriptions:
  user_id FK, role_type TEXT CHECK (IN 'client','expert')
  tier TEXT CHECK (IN 'free','pro')
  status TEXT DEFAULT 'PENDING_PAYMENT'
    CHECK (IN 'PENDING_PAYMENT','ACTIVE','EXPIRING_SOON','EXPIRED')
  price_paid BIGINT DEFAULT 0
  activated_at TIMESTAMPTZ NULL  -- set by IPN handler
  expires_at TIMESTAMPTZ NULL
  UNIQUE (user_id, role_type)

withdrawal_requests:
  expert_id FK expert_profiles
  type TEXT DEFAULT 'EXPERT_MANUAL' CHECK (IN 'MILESTONE_RELEASE','EXPERT_MANUAL')
  amount BIGINT CHECK (> 0), bank_account_xid TEXT
  disbursement_id TEXT NULL      -- SePay chi ho response ID
  status TEXT DEFAULT 'PENDING' CHECK (IN 'PENDING','PROCESSING','COMPLETED','FAILED')
  requested_at TIMESTAMPTZ, confirmed_at TIMESTAMPTZ NULL
  telegram_notified_at TIMESTAMPTZ NULL  -- fraud alert audit trail

platform_settings:  -- SINGLETON; seed at deploy; never INSERT again
  platform_wallet_id UUID UNIQUE FK wallets
  platform_fee_pct FLOAT DEFAULT 0.05 CHECK (BETWEEN 0 AND 1)
  -- ALWAYS read from this table at transaction time; never hardcode 0.05 in app code
```

**Section 4 — Organizations (Phase 2 schema — tables created; no routes/UI in SWP)**
```sql
organizations: owner_user_id FK users; payout_designated_user_id UUID NULL FK users;
               sepay_bank_account_xid TEXT NULL; subscription_tier TEXT DEFAULT 'free'
organization_members: org_id FK, user_id FK; org_role TEXT CHECK (IN 'OWNER','MEMBER','BILLING');
                      platform_roles JSONB DEFAULT '[]'; UNIQUE (org_id, user_id)
```

**Section 5 — Elicitation Engine (new)**
```sql
elicitation_sessions:
  user_id FK users (CEO who started)
  current_stage INT DEFAULT 1 CHECK (BETWEEN 1 AND 5)
  archetype TEXT NULL          -- locked after Stage 2 confirmation
  scenario_type TEXT NULL CHECK (IN 'STANDARD','SCENARIO_A','SCENARIO_B')
  void_list_json JSONB DEFAULT '[]'  -- [{void_code, injected: boolean}]
  state TEXT DEFAULT 'IN_PROGRESS' CHECK (IN 'IN_PROGRESS','COMPLETED','ABANDONED','RETURNED')
  -- FK direction: projects.elicitation_session_id -> elicitation_sessions(id); NOT reverse
  -- Sessions can exist without a project (incomplete attempts)
```

**Section 6 — Projects & Dual Artifacts**
```sql
projects:
  client_id FK users
  elicitation_session_id UUID NULL FK elicitation_sessions  -- NULL for non-elicitation paths
  state TEXT DEFAULT 'DRAFT' CHECK (IN 'DRAFT','PUBLISHED','RETURNED_TO_CLIENT','SUSPENDED')
  archetype TEXT NULL, tier TEXT NULL CHECK (IN 'TIER_1','TIER_2','TIER_3')
  self_technical BOOLEAN DEFAULT FALSE

capability_footprints: project_id PK FK projects
  required_domains_json, required_seams_json, milestone_framework_json JSONB; technical_architecture TEXT NULL

artifact_a: project_id PK FK projects
  state TEXT DEFAULT 'DRAFT' CHECK (IN 'DRAFT','PUBLISHED','RETURNED_TO_CLIENT','SUSPENDED')
  -- state managed independently from projects.state
  business_intent TEXT NULL, physics_constraints_json JSONB NULL, escrow_lock_notice TEXT NULL

artifact_b: project_id PK FK projects
  tech_stack_json, schema_uploads_json, integration_contracts_json JSONB NULL
  -- Access guard in FastAPI SQL: checks engagement.state >= CONNECTED + both NDA timestamps
  -- CEO path permanently excluded (no CEO branch in the guard query)
```

**Section 7 — Services**
```sql
services: expert_id FK; title, description TEXT; domains_json, seams_json JSONB
  price_vnd BIGINT CHECK (> 0)
  state TEXT DEFAULT 'DRAFT' CHECK (IN 'DRAFT','PUBLISHED','SUSPENDED')
  service_type TEXT CHECK (IN 'AI_SERVICE','TECH_DISCOVERY')
```

**Section 8 — Engagements & Bid System**
```sql
engagements:
  project_id UUID NULL FK projects      -- NULL for SERVICE/TECH_DISCOVERY
  expert_id FK expert_profiles
  service_id UUID NULL FK services      -- NULL for PROJECT_BASED
  type TEXT CHECK (IN 'PROJECT_BASED','SERVICE_PURCHASE','TECH_DISCOVERY')  -- immutable
  state TEXT DEFAULT 'PENDING' CHECK (IN 'PENDING','CONNECTED','ACTIVE','CLOSED','DISPUTED')
  connected_at TIMESTAMPTZ NULL
  client_nda_accepted_at TIMESTAMPTZ NULL   -- CEO NDA click-through
  expert_nda_accepted_at TIMESTAMPTZ NULL   -- Expert NDA acceptance
  -- BOTH timestamps required for Artifact B access (checked in FastAPI guard SQL)
  organization_id UUID NULL FK organizations
  CONSTRAINT engagement_type_fk_consistency CHECK (
    (type='PROJECT_BASED' AND project_id IS NOT NULL AND service_id IS NULL) OR
    (type IN ('SERVICE_PURCHASE','TECH_DISCOVERY') AND project_id IS NULL AND service_id IS NOT NULL)
  )

spec_clarifications:
  spec_id FK artifact_a, expert_id FK, responded_by UUID NULL FK users
  target_component TEXT, question_text TEXT, response_text TEXT NULL
  status TEXT DEFAULT 'OPEN' CHECK (IN 'OPEN','ANSWERED','RESOLVED')  -- 'CLOSED' NOT valid

capability_bids:
  engagement_id UUID UNIQUE FK engagements  -- 1:1
  footprint_alignment_json, approach_summary, conditional_pricing_json  -- all required at SUBMITTED
  state TEXT DEFAULT 'DRAFT' CHECK (IN 'DRAFT','SUBMITTED','TECH_REVIEW','REVISION_REQUESTED',
    'REVISED','TECH_APPROVED','TECH_DISAPPROVED','CONFLICT_PENDING',
    'CONFLICT_RESOLVED_CEO_OVERRIDE','PRICE_NEGOTIATION','NEGOTIATION_COMPLETE',
    'CEO_REVIEW','SELECTED','DECLINED')
  version_number INT DEFAULT 1

bid_versions:  -- IMMUTABLE snapshot; one row per REVISED transition
  bid_id FK, revision_request_id UUID NULL FK bid_revision_requests  -- NULL on version 1
  version_number INT, snapshot_json JSONB, revised_at TIMESTAMPTZ, reviser_id FK users
  UNIQUE (bid_id, version_number)

bid_revision_requests:  -- Surface B
  bid_id FK; flagged_component TEXT CHECK (IN 'FOOTPRINT','APPROACH','PRICING')
  reason TEXT; requested_by FK users

price_negotiations:  -- Surface C; max 2 rounds
  bid_id FK; round_number INT CHECK (IN 1,2)  -- DB-level hard cap
  proposer TEXT CHECK (IN 'CEO','EXPERT')
  response TEXT NULL CHECK (IN 'ACCEPTED','COUNTERED','DECLINED','PENDING')
  UNIQUE (bid_id, round_number)

bid_conflict_overrides:  -- Surface D
  bid_id UUID UNIQUE FK capability_bids
  ceo_user_id FK users  -- explicit FK for RQ3 correlation
  override_reason TEXT CHECK (char_length(override_reason) >= 50)  -- DB-level minimum
  acknowledged_risks TEXT, overridden_at TIMESTAMPTZ
```

**Section 9 — Milestones (3-Layer System)**
```sql
milestones:
  engagement_id FK, milestone_number INT
  sign_off_authority TEXT CHECK (IN 'TECH_TEAM','CEO','JOINT')
  payment_amount_vnd BIGINT CHECK (> 0)
  state TEXT DEFAULT 'DEFINED' CHECK (IN 'DEFINED','AWAITING_PAYMENT','FUNDED','IN_PROGRESS',
                                       'SUBMITTED','IN_REVISION','APPROVED','RELEASED','DISPUTED')
  va_number TEXT NULL, va_expires_at TIMESTAMPTZ NULL
  review_clock_paused_at TIMESTAMPTZ NULL  -- set on BLOCKER sprint status
  total_paused_seconds INT DEFAULT 0       -- accumulates for SLA tracking
  UNIQUE (engagement_id, milestone_number)

acceptance_criteria:  -- Layer 1: payment contract
  milestone_id FK; criterion_text TEXT; is_required BOOLEAN DEFAULT TRUE
  verified_by_role TEXT CHECK (IN 'TECH_TEAM','CEO','JOINT')
  verified_at TIMESTAMPTZ NULL  -- all required=true rows non-NULL required for APPROVED

revision_requests: criterion_id FK; requested_by FK users; feedback_text TEXT

milestone_dod_items:  -- Layer 2: expert self-check; gates SUBMITTED
  milestone_id FK; item_description TEXT; is_required BOOLEAN DEFAULT TRUE
  status TEXT DEFAULT 'PENDING' CHECK (IN 'PENDING','COMPLETED','NOT_APPLICABLE')
  completed_at TIMESTAMPTZ NULL, completion_note TEXT NULL, not_applicable_note TEXT NULL
  maps_to_criterion_id UUID NULL FK acceptance_criteria
  CONSTRAINT dod_required_cannot_be_na
    CHECK (NOT (is_required = TRUE AND status = 'NOT_APPLICABLE'))

milestone_sprints:  -- Layer 3: temporal decomposition; no payment consequence
  milestone_id FK; sprint_number INT; created_by FK users (EXPERT only; route guard enforces)
  week_range TEXT, deliverable_checkpoint TEXT
  tasks JSONB DEFAULT '[]'  -- [{id, description, status: 'TODO'|'IN_PROGRESS'|'DONE'}]
  UNIQUE (milestone_id, sprint_number)

sprint_status_updates:
  milestone_id FK
  milestone_sprint_id FK milestone_sprints  -- direct FK; replaces implicit sprint_number lookup
  sprint_number INT (denormalized display), submitted_by FK users (EXPERT only)
  status TEXT CHECK (IN 'ON_TRACK','DEVIATION','BLOCKER')
  blocker_type TEXT NULL CHECK (IN 'SCOPE_GAP','TECH_CONSTRAINT','EXTERNAL_DEPENDENCY','SCOPE_EVOLUTION')
  scope_evolution_flag BOOLEAN DEFAULT FALSE  -- auto-set; triggers addon_phase_requests

sprint_comments:  -- TECH_TEAM per-sprint thread; separate from messages
  milestone_sprint_id FK; commenter_id FK users; comment_text TEXT
  ROUTE: WRITE=TECH_TEAM; READ=TECH_TEAM+EXPERT; CEO hidden

dod_item_comments:  -- TECH_TEAM per-DoD-item thread; separate from messages
  dod_item_id FK milestone_dod_items; commenter_id FK users; comment_text TEXT
  ROUTE: WRITE=TECH_TEAM; READ=TECH_TEAM+EXPERT; CEO hidden

milestone_submissions: milestone_id FK, expert_id FK; description TEXT; files_json JSONB

paygated_documents:
  milestone_id FK  -- releases when THIS milestone reaches FUNDED via IPN
  document_url TEXT
  release_state TEXT DEFAULT 'STAGED' CHECK (IN 'STAGED','RELEASED')

escrow_accounts:  -- dual-parent; exactly one of milestone_id or engagement_id must be set
  milestone_id UUID NULL FK milestones     -- PROJECT_BASED
  engagement_id UUID NULL FK engagements   -- SERVICE/TECH_DISCOVERY
  amount BIGINT; client_wallet_id FK wallets; expert_wallet_id FK wallets
  status TEXT DEFAULT 'HELD' CHECK (IN 'HELD','RELEASED','FROZEN','REFUNDED','SPLIT')
  CONSTRAINT escrow_has_one_parent CHECK (
    (milestone_id IS NOT NULL AND engagement_id IS NULL) OR
    (milestone_id IS NULL AND engagement_id IS NOT NULL)
  )
  CREATE UNIQUE INDEX escrow_milestone_unique ON escrow_accounts(milestone_id)
    WHERE milestone_id IS NOT NULL;
  CREATE UNIQUE INDEX escrow_engagement_unique ON escrow_accounts(engagement_id)
    WHERE engagement_id IS NOT NULL;
```

**Section 10 — Disputes**
```sql
disputes:
  engagement_id FK, milestone_id FK, criterion_id FK acceptance_criteria
  escrow_account_id FK escrow_accounts  -- direct FK; freeze is O(1) and type-agnostic
  filed_by FK users
  state TEXT DEFAULT 'PENDING' CHECK (IN 'PENDING','LAYER_1_EVAL','AUTO_RESOLVED',
    'LAYER_2_MUTUAL','MUTUAL_RESOLVED','LAYER_3_FORCED','DISPUTED_AUTO_RESOLVED')
  resolution_layer INT NULL CHECK (IN 1,2,3)
  llm_confidence FLOAT NULL CHECK (BETWEEN 0 AND 1)
  layer2_deadline TIMESTAMPTZ NULL  -- now()+48h when Layer 1 fails
  filed_at TIMESTAMPTZ, resolved_at TIMESTAMPTZ NULL

dispute_resolution_reports:  -- informational only; no admin action
  dispute_id FK; reported_by FK users; reason_text TEXT
```

**Section 11 — Add-On, Messaging, Reviews, Platform**
```sql
addon_phase_requests:
  engagement_id FK, submitted_by FK users (EXPERT)
  trigger_sprint_status_id UUID NULL FK sprint_status_updates
  CREATE UNIQUE INDEX addon_per_sprint_event ON addon_phase_requests(trigger_sprint_status_id)
    WHERE trigger_sprint_status_id IS NOT NULL;  -- one add-on per SCOPE_EVOLUTION event
  created_milestone_id UUID NULL FK milestones   -- set when MILESTONE_CREATED
  causal_chain, new_capability_requirements, why_invisible TEXT (all required)
  status TEXT DEFAULT 'PENDING' CHECK (IN 'PENDING','TECH_APPROVED','AWAITING_CEO',
                                        'FULLY_APPROVED','REJECTED','MILESTONE_CREATED')
  tech_approved_at TIMESTAMPTZ NULL, tech_approved_by UUID NULL FK users
  ceo_approved_at TIMESTAMPTZ NULL, ceo_approved_by UUID NULL FK users
  rejected_at TIMESTAMPTZ NULL, rejected_by UUID NULL FK users, rejection_reason TEXT NULL

messages:
  engagement_id FK, sender_id FK users, content TEXT
  attachments_json JSONB NULL  -- [{url, filename, size_bytes}]; NULL = no attachment

message_reads:  -- unread badge tracking
  message_id FK messages, user_id FK users, read_at TIMESTAMPTZ
  UNIQUE (message_id, user_id)

reviews:
  engagement_id FK, reviewer_id FK users, target_id FK users
  rating INT CHECK (BETWEEN 1 AND 5), comment TEXT NULL, structured_signals_json JSONB NULL
  reviewer_role TEXT CHECK (IN 'CEO','TECH_TEAM','EXPERT')
  UNIQUE (engagement_id, reviewer_id)

outcome_signals:
  engagement_id FK, reviewer_id FK users
  review_id UUID UNIQUE FK reviews  -- 1:1 enforcement; one aggregate per review
  seam_signals_json JSONB, ratings_json JSONB
  UNIQUE (engagement_id, reviewer_id)

platform_decisions:  -- FastAPI ONLY; never NestJS; never UPDATE after INSERT
  decision_type TEXT CHECK (IN 'ELICITATION_SYNTHESIS','SPEC_AUTO_RETURN','SEAM_TIER_UPGRADE',
    'PORTFOLIO_EVAL','SCENARIO_EVAL','DISPUTE_L1_EVAL','CRITERION_QUALITY_GATE','CONTENT_MODERATION')
  entity_type TEXT CHECK (IN 'project','capability_bid','acceptance_criterion',
    'portfolio_submission','scenario_response','spec_clarification','service')
    -- required companion to entity_id; Platform Integrity Monitor JOINs on this to find details
  entity_id TEXT  -- UUID of affected row (polymorphic; intentionally no DB FK)
  llm_confidence FLOAT NULL, decision TEXT, advisory_note TEXT NULL
```

**FK Migration Order — 51 tables (migrate in this exact sequence):**
```
 1  users                    17  services
 2  client_profiles          18  engagements
 3  expert_profiles          19  virtual_accounts (standalone)
 4  wallets                  20  expert_domain_depths
 5  organizations            21  expert_seam_claims
 6  organization_members     22  expert_stack_tags
 7  user_subscriptions       23  portfolio_submissions
 8  withdrawal_requests      24  scenario_assessments (standalone)
 9  wallet_transactions      25  scenario_responses
10  elicitation_sessions     26  spec_clarifications
11  projects                 27  capability_bids
12  tech_team_profiles       28  bid_revision_requests
13  capability_footprints    29  bid_versions
14  artifact_a               30  price_negotiations
15  artifact_b               31  bid_conflict_overrides
16  platform_settings        32  milestones

33  acceptance_criteria      43  addon_phase_requests
34  revision_requests        44  sprint_comments
35  milestone_dod_items      45  dod_item_comments
36  milestone_sprints        46  messages
37  sprint_status_updates    47  message_reads
38  milestone_submissions    48  reviews
39  paygated_documents       49  outcome_signals
40  escrow_accounts          50  expert_seam_outcome_signals
41  disputes                 51  platform_decisions (standalone)
42  dispute_resolution_reports
```

### 5.4 Service Communication

- **Frontend → Main API:** REST over HTTPS, JWT in Authorization header
- **Frontend → Main API (messaging):** Socket.io with JWT handshake
- **Main API → AI Service:** Internal REST calls (Main API orchestrates; AI Service is called for elicitation processing, matching computation, LLM calls, criterion quality check, and portfolio/scenario LLM evaluation)
- **AI Service → Anthropic API:** Standard HTTP with API key (server-side only)
- **Main API → SePay VA API:** NestJS creates Virtual Accounts for each payment entity (WALLET_TOPUP, MILESTONE, SERVICE, SUBSCRIPTION) via SePay Bank Hub API. Returns VA number; frontend displays VietQR. No legacy `/v1/orders` QR creation — all payments go through per-entity VAs with fixed amounts.
- **SePay → Main API (IPN):** SePay fires `POST /webhooks/sepay/ipn` on every VA credit — identified by `va_number` + entity_type. Idempotency: unique index on `wallet_transactions(wallet_id, reference_id)` prevents double-credit on IPN retry.
- **Main API → Bank Hub Hosted Link API:** NestJS generates Link Token for expert bank account linking. Expert completes OTP flow in embedded WebView. `BANK_ACCOUNT_LINKED` webhook fires to `POST /webhooks/bankhub/events` — stores `bank_account_xid`.
- **Main API → SePay Chi Hộ API:** NestJS calls chi hộ on milestone APPROVED (MILESTONE_RELEASE) and on expert withdrawal request (EXPERT_MANUAL). Fires to expert's `bank_account_xid`. Completion confirmed via SePay credit IPN.
- **Both services → PostgreSQL:** Direct DB connection (Prisma from NestJS, SQLAlchemy from FastAPI) — same PostgreSQL instance, different connection pools
- **GitHub Actions → EC2:** SSH deploy on merge to `main`

### 5.5 Deployment (AWS EC2 — satisfies Topic 7 cloud-based requirement)

All services are deployed to AWS EC2 using Docker Compose. This is not optional — Topic 7 explicitly requires cloud-based scalable architecture. TV5 owns deployment setup in Week 8.

**AWS setup:**
- EC2 t3.micro (free tier eligible for 12 months) — single instance running all Docker Compose services
- Elastic IP attached (free when instance is running) — stable public IP for SePay IPN endpoint and frontend access
- Security groups: port 80/443 (frontend via nginx reverse proxy), port 3000 (NestJS internal), port 8000 (FastAPI internal, internal only), port 5432 (PostgreSQL, internal only)
- PostgreSQL runs in Docker on the same EC2 (MVP) — no RDS needed for SWP scale

**Docker Compose services:** `nginx` (reverse proxy + static React build), `nestjs` (main API), `fastapi` (AI service), `postgres` (database)

**CI/CD pipeline:**
- GitHub Actions: lint + build check on every PR to `dev`
- On merge to `main`: GitHub Actions SSH into EC2 → `git pull` → `docker compose up -d --build`
- SePay IPN endpoint must be publicly accessible on the Elastic IP — configure this URL in SePay dashboard during Week 5 sandbox setup

**Environment variables on EC2:** `ANTHROPIC_API_KEY`, `SEPAY_MERCHANT_ID`, `SEPAY_SECRET_KEY`, `DATABASE_URL`, `JWT_SECRET`, `SENDGRID_API_KEY` — stored in `.env` on EC2, never committed to repo

**Domain name:** Optional for SWP (bare Elastic IP works for demo). If the team wants a domain, Freenom or a ~$1/year `.xyz` domain is sufficient.

---

## 6. WBS & Milestones

### Team Assignments Summary

| Member | Primary responsibilities | Features owned |
|---|---|---|
| **Minh Hùng (TV1)** | FastAPI AI service, LLM integration, matching engine, elicitation engine, state-gated DB access, DB schema, chi hộ LLM dispute evaluation | F2 (engine + Scenario A/B branching), F4 (scoring + self-exclusion filter), F5 (Artifact B DB access), F7 (criterion quality LLM check + Layer 1 dispute LLM), F8 (add-on logic), F11 (signal processing), F12 (platform_decisions log) |
| **Chí Nhân (TV2)** | NestJS main backend, auth (dual-role + role switcher), all SePay integrations (VA + IPN + Bank Hub + chi hộ), internal wallet ledger, subscription system, full bid state machine (all 4 surfaces), milestone state machine (DoD + sprint guards), dispute escrow operations, admin ops | F1 (incl. dual-role + Bank Hub field), F1.5, F2 (UI + Scenario A/B branches), F5 (connection + pay-gated staging), F6 (bid state machine + all 4 surfaces), F7 (all state transitions + DoD guard + sprint logic + F8 trigger), F10 (entire financial system), F12 (user mgmt + Platform Integrity Monitor) |
| **Cao Minh (TV3)** | Feature UI modules: Path B marketplace, bid surfaces UI, DoD/sprint UI (3 role variants), dispute UI, wallet/subscription UI, reviews | F3 (listings + Path B marketplace), F6 (bid form + Surface A/C/D UI), F7 (DoD editor + sprint plan — 3 role views), F8 (add-on UI), F10 (transaction history + earnings + withdrawal UI), F11 (review forms), F12 (analytics + dispute monitor) |
| **Tuấn Khang (TV4)** | SRS, UML diagrams (updated for dual-role + non-linear bid + DoD/sprint), API documentation, elicitation UI copy, Scenario A/B option copy, Surface A/B/C/D UI copy, final report | All documentation; elicitation + bid + sprint UI copy |
| **Minh Thức (TV5)** | Test plan, Postman collections, QA dashboard, GitHub CI/CD, AWS EC2 deployment, bug tracking | All testing (incl. atomic ledger test cases, chi hộ sandbox test, dual-role switch, bid revision loop, DoD gate enforcement), QA infrastructure, cloud deployment |

---

### WBS

#### Week 1 — Requirements, Design, Environment Setup

| ID | Task | Owner | Days | Depends on |
|---|---|---|---|---|
| 1.1 | Finalize functional requirements from Topic 7 + this scope doc | TV4 | 1 | — |
| 1.2 | Write SRS (aligned with this scope — actors, use cases, functional requirements) | TV4 + TV1 | 2 | 1.1 |
| 1.3 | Design PostgreSQL schema (all tables, relationships, state fields) | TV1 + TV2 | 2 | 1.2 |
| 1.4 | Design API contracts (all route signatures, request/response schemas) | TV1 + TV2 | 1 | 1.3 |
| 1.5 | Draw UML: Use Case diagram (all 4 roles — CLIENT/CEO, CLIENT/TECH_TEAM, EXPERT, ADMIN — as distinct actors), Activity diagrams for UC01 (two-actor flow: CEO drives Stages 1–3, TECH_TEAM drives Stage 4 via handoff link) + UC15, Sequence diagrams for elicitation flow and connection flow | TV4 | 2 | 1.2, 1.4 |
| 1.6 | Set up GitHub repo, branching strategy, PR template, GitHub Actions CI | TV5 | 1 | — |
| 1.7 | Set up dev environment: Docker Compose (PostgreSQL), NestJS scaffold, FastAPI scaffold, React scaffold | TV1 + TV2 | 1 | 1.6 |
| 1.8 | Write Prisma schema + initial migration; write SQLAlchemy models + Alembic migration | TV2 (Prisma) + TV1 (SA) | 2 | 1.3, 1.7 |
| 1.9 | Write elicitation UI copy: all prompts, injection explanations, resistance handling text, tech team framing | TV4 | 2 | 1.2 |
| 1.10 | Set up Postman workspace, seed data scripts | TV5 | 1 | 1.8 |

**Week 1 milestone:** Schema locked, environment running, SRS drafted, all API contracts signed off by team

---

#### Week 2 — Authentication, Expert Profile, Service Marketplace, Wallet Foundation

| ID | Task | Owner | Days | Depends on |
|---|---|---|---|---|
| 2.1 | F1: Auth — register/login/JWT/role middleware (NestJS); `users.roles` JSONB array; `active_role` field; role switcher JWT reissue endpoint; self-exclusion guard in matching queries | TV2 | 3 | 1.8 |
| 2.2 | F1: Auth frontend — login/register forms, role-based routing, role switcher toggle in top nav | TV2 + TV3 | 2 | 2.1 |
| 2.3 | F3: Expert profile CRUD — domain depth map, seam claims, stack tags, engagement model (NestJS + Prisma) | TV2 | 3 | 2.1 |
| 2.3b | F1: TECH_TEAM handoff-link registration path; `client_subtype` guard for TECH_TEAM-only routes; Scenario B: `self_technical` flag handling on project + elicitation branch | TV2 | 2 | 2.1 |
| 2.4 | F3: Expert profile form UI — taxonomy-based capability declaration interface | TV3 | 3 | 2.3 |
| 2.5 | F3: Portfolio evidence submission + LLM auto-verification | TV2 (NestJS + auto-upgrade state machine) + TV1 (FastAPI LLM extraction + gap advisory) | 3 | 2.1 |
| 2.6 | F3: Service listing CRUD + Path B marketplace UI (browse, filter by domain/seam/archetype) | TV3 | 3 | 2.3 |
| 2.7 | F3: AI Service Generator LLM call (FastAPI) | TV1 | 2 | 1.7 |
| 2.8 | **F10/F1.5: SePay setup + wallet foundation** — Register SePay (personal account; VietQR 30 min); verify chi hộ tier availability (hotline call — log result by EOD Week 2 Day 1); create `wallets` table + Prisma model; WALLET_TOPUP VA creation logic on user registration; wallet top-up IPN handler branch; `virtual_accounts` table; wallet top-up UI | TV2 | 3 | 1.8 |
| 2.9 | **F1.5: Subscription system** — `user_subscriptions` table; subscription activation route (deduct from wallet, set tier + expiry); subscription guard middleware (gates all FastAPI-calling routes + matching); subscription status UI badge; expiry notification job | TV2 (backend + guard) + TV3 (UI) | 3 | 2.8 |
| 2.10 | Test plan for F1 + F3; Postman collections for auth, profile, wallet top-up routes | TV5 | 2 | 2.1, 2.3, 2.8 |

**Week 2 milestone:** Expert can create full taxonomy profile; Path B service marketplace browsable; auth and role switcher working for all roles; wallet top-up via SePay VA tested in sandbox (IPN confirms → wallet credited); subscription purchase from wallet working; chi hộ tier decision logged

---

#### Week 3 — Elicitation Engine (Core AI Innovation) + Milestone VA Funding

| ID | Task | Owner | Days | Depends on |
|---|---|---|---|---|
| 3.1 | F2: LLM extraction layer — FastAPI endpoint accepting intake text, returning diagnostic state object (archetype, tier, scale signals, void list) | TV1 | 4 | 1.7 |
| 3.2 | F2: Guided dialogue state machine — FastAPI state manager for elicitation stages 1–3 | TV1 | 3 | 3.1 |
| 3.3 | F2: SDLC injection logic — archetype-specific injection checklists, Phase 0 hard gate, resistance handler | TV1 | 2 | 3.2 |
| 3.4 | F2: Tech handoff link generator + Scenario A/B branching — Scenario A: presents Technical Discovery option when no TECH_TEAM; Scenario B: `self_technical = true` presents Stage 4 form directly to CEO; standard: token-based URL, tech team input form, Artifact B vault write | TV2 (NestJS) + TV1 (FastAPI vault logic + branch logic) | 3 | 3.2 |
| 3.5 | F2: Synthesis engine — conflict resolution, footprint construction, Artifact A generation | TV1 | 3 | 3.4 |
| 3.6 | F2: Elicitation conversational UI — chat-like interface, archetype selection, injection cards, probe flow, Scenario A option screen, Scenario B self-technical toggle | TV2 | 4 | 3.2 |
| 3.7 | F2: Tech team handoff UI — secure link landing page, categorical input form, upload interface | TV2 | 2 | 3.4 |
| 3.8 | **F10: Per-milestone VA creation + IPN MILESTONE branch** — NestJS "Fund Milestone" endpoint: creates per-milestone VA via SePay Bank Hub API (`fixed_amount = milestone.amount_vnd`; `expires_at = now + 24h`); returns VA number → frontend displays VietQR; IPN handler MILESTONE branch: validates VA match + exact amount → atomic ledger ([ESCROW_LOCK]: wallet.available_balance -= amount; locked_balance += amount; escrow_accounts HELD; milestone → FUNDED → IN_PROGRESS) | TV2 | 3 | 2.8 (parallel with elicitation) |
| 3.9 | **F10: Bank Hub Hosted Link integration** — NestJS Link Token API call; Hosted Link embed in Expert Dashboard; BANK_ACCOUNT_LINKED webhook handler (`POST /webhooks/bankhub/events`): stores `sepay_bank_account_xid`, `bank_account_holder_name`, `bank_linked_at` on user | TV2 | 2 | 2.8 |
| 3.10 | Test: Run AdTech CEO intake through elicitation, validate Tier 3 Archetype 1 footprint output | TV5 | 2 | 3.5 |
| 3.11 | Test: End-to-end sandbox VA funding — create milestone VA → scan QR → IPN fires → verify ESCROW_LOCK ledger entry + milestone FUNDED state | TV5 | 1 | 3.8 |

**Week 3 milestone:** Complete elicitation flow functional with AdTech case (Scenario A/B branches implemented); per-milestone VA funding tested in sandbox with atomic ledger verified; Bank Hub Hosted Link expert bank linking functional

---

#### Week 4 — Matching Engine (Core AI Innovation)

| ID | Task | Owner | Days | Depends on |
|---|---|---|---|---|
| 4.1 | F4: Expert profile query from footprint — FastAPI service that reads all active expert profiles from DB and extracts verification confidence scores per domain and seam | TV1 | 2 | 2.3 |
| 4.2 | F4: Five-component composite scoring model implementation | TV1 | 3 | 4.1 |
| 4.3 | F4: Hard gate logic (claimed-to-verified ratio, negative outcome signal check) | TV1 | 1 | 4.2 |
| 4.4 | F4: Seam gap map generation (for each required seam: expert's tier → color-coded status) | TV1 | 2 | 4.2 |
| 4.5 | F4: Shortlist generation — top 3–5 with match cards (strength label, verified summary, seam grid, gap list, recommended next step) | TV1 | 2 | 4.4 |
| 4.6 | F4: Match card UI — seam coverage visual, gap display, match strength badge | TV2 | 3 | 4.5 |
| 4.7 | F2 + F12: Auto-return spec logic + Platform Integrity Monitor — NestJS/FastAPI auto-return handler: on quality gate failure, sets `spec.state = RETURNED_TO_CLIENT`, generates LLM advisory note with the specific void, re-opens elicitation at the failure point; FastAPI writes structured decision record to `platform_decisions` log table; admin Platform Integrity Monitor reads this table to display auto-return history with void type and advisory text | TV2 (NestJS state transitions + re-entry routing) + TV1 (LLM advisory generation + decision log writes) | 2 | 3.5, 4.5 |
| 4.8 | Test: Matching engine test matrix (5 synthetic expert profiles × 3 synthetic footprints, pre-computed expected rankings) | TV5 | 2 | 4.5 |
| 4.9 | F3: Scenario assessment auto-delivery engine — NestJS selects question from question bank (lowest `last_used_at` for the requested seam); delivers to expert immediately on Tier 3 request; FastAPI LLM rubric evaluation: evaluates expert response against all required dimensions in the question's stored rubric; all dimensions pass → auto-upgrade to Tier 3; any dimension fails → LLM generates specific failure note + auto-selects new question for resubmission; `platform_decisions` log write for each grading event | TV2 (question bank selection + delivery + state machine) + TV1 (rubric evaluation LLM call + failure note generation) | 3 | 4.1 |

**Week 4 milestone:** Matching engine producing correct ranked shortlists; seam gap maps visible in match cards; automated quality gate auto-publishes passing specs and auto-returns failing specs with targeted LLM advisory (no admin exception queue); portfolio evidence LLM auto-upgrade verified; scenario assessment auto-delivery and LLM rubric grading functional

---

#### Week 5 — Dual Artifact + Non-Linear Bid System

| ID | Task | Owner | Days | Depends on |
|---|---|---|---|---|
| 5.1 | F5: Artifact A full display UI — all 6 sections with escrow lock notice | TV2 | 2 | 3.5 |
| 5.2 | F5: Connection flow — CEO selects expert, connection request sent, expert accepts/declines; Bank Hub link check on CONNECTED (prompt expert to link if `bank_account_xid` is null) | TV2 | 2 | 4.5 |
| 5.3 | F5: State-gated Artifact B — FastAPI route checks `engagement.state >= CONNECTED` before returning Artifact B fields | TV1 | 2 | 5.2 |
| 5.4 | F5: Artifact B display UI (TECH_TEAM + expert view, post-connection) | TV2 | 2 | 5.3 |
| 5.5 | F5: Pay-gated document staging — expert upload with milestone trigger selector; NestJS stores in DB; release fires inside IPN MILESTONE handler on FUNDED | TV2 | 3 | 5.2 |
| 5.6 | **F6: Surface A — Spec clarification requests** — `spec_clarifications` table; expert POST route; TECH_TEAM/CEO response route; clarification thread UI; resolved flag; version logging | TV2 (routes + state) + TV3 (UI) | 2 | 5.1 |
| 5.7 | **F6: Capability bid submission form** — 3 required components; LLM conditional structure validator (returns 422 on unstructured conditionals); `bid_versions` first write (version 1 = original submission) | TV3 (form) + TV2 (validation route) | 3 | 5.6 |
| 5.8 | **F6: Surface B — Bid revision loop** — TECH_TEAM revision request route (flags specific component + reason); `bid_revision_requests` table; expert revision route (edits only flagged component; `bid_versions` new row written); bid state machine (SUBMITTED → TECH_REVIEW → REVISION_REQUESTED → REVISED → TECH_APPROVED / TECH_DISAPPROVED) | TV2 (state machine) + TV3 (bid comparison UI with TECH_TEAM annotation panel) | 3 | 5.7 |
| 5.9 | **F6: Surface C — Price negotiation** — `price_negotiations` table; CEO propose route (round counter guard — max 2); Expert respond route (ACCEPT / COUNTER / DECLINE); negotiation history display | TV2 (routes + round guard) + TV3 (negotiation UI) | 2 | 5.8 |
| 5.10 | **F6: Surface D — CEO/TECH_TEAM conflict resolution** — override route guard (blocks CEO_REVIEW if bid is TECH_DISAPPROVED); override form (mandatory reason + risks fields min 50 chars); `bid_conflict_overrides` table; Platform Integrity Monitor logging | TV2 (guard + route + table) + TV3 (override form UI) | 2 | 5.8 |
| 5.11 | Test: Full bid flow — UC19 (spec clarification) → UC20 (bid submit) → revision loop → override → negotiation → selection | TV5 | 2 | 5.10 |

**Week 5 milestone:** Complete elicitation-to-connection flow; Artifact B gating verified; non-linear bid flow with all four surfaces (A, B, C, D) operational; bid revision history written correctly; override logged in Platform Integrity Monitor

---

#### Week 6 — Project & Milestone Management + DoD + Sprint Tracking

| ID | Task | Owner | Days | Depends on |
|---|---|---|---|---|
| 6.1 | F7 Layer 1: Milestone creation form — deliverable statement, acceptance criteria (LLM criterion quality check on save — flags subjective language), sign-off authority selector, payment amount | TV2 (route + LLM check via FastAPI) + TV3 (form UI) | 3 | 5.2 |
| 6.2 | F7 Layer 1: Milestone state transition guards — `SUBMITTED` guard: checks all required DoD items COMPLETED; `APPROVED` guard: checks all required criteria have `verified_at`; both return 422 with structured missing-items list | TV2 | 2 | 6.1 |
| 6.3 | F7 Layer 1: Criterion-referenced review interface — reviewer must select criterion before free-text; revision requests require criterion reference; JOINT milestone waits for both TECH_TEAM and CEO `verified_at`; revision cycle counter + "Elevated revision cycle" flag at > 2 | TV2 (NestJS routes) + TV3 (review UI) | 3 | 6.2 |
| 6.4 | F7 Layer 1: Milestone APPROVED → RELEASED — atomic ledger: [ESCROW_RELEASE]: locked_balance -= amount; [CREDIT_EXPERT]: expert.available_balance += amount * 0.95; [PLATFORM_FEE]: += amount * 0.05; pay-gated docs released; next milestone VA generated + CEO notified; chi hộ API called for expert share | TV2 | 3 | 6.3, 3.8, 3.9 |
| 6.5 | **F7 Layer 2: DoD checklist** — `milestone_dod_items` table; expert creates/manages checklist after FUNDED; COMPLETED/NOT_APPLICABLE routes with required notes; TECH_TEAM read + sprint comment thread; CEO hidden (route guard); DoD UI in Expert Dashboard (editor) and Tech Dashboard (read-only) | TV2 (routes + guard + table) + TV3 (DoD editor UI + read-only view) | 3 | 6.1 |
| 6.6 | **F7 Layer 3: Sprint plan** — `milestone_sprints` table; expert creates sprint plan after FUNDED; task JSONB updates; TECH_TEAM comment thread per sprint; sprint plan UI (expert editor, TECH_TEAM read + comment, CEO: checkpoint summary only) | TV2 (routes + table) + TV3 (sprint plan UI — 3 role variants) | 3 | 6.1 |
| 6.7 | **F7 Layer 3: Weekly sprint status updates** — `sprint_status_updates` table; expert status form (ON_TRACK/DEVIATION/BLOCKER with typed fields); overdue reminder job (3-day notice → 5-day client notification); ON_TRACK: acknowledge; DEVIATION: notify parties; BLOCKER (non-SCOPE_EVOLUTION): pause milestone clock, TECH_TEAM notified, resolve route; BLOCKER (SCOPE_EVOLUTION): auto-trigger F8 Add-On Phase, pause clock | TV2 (routes + status logic + clock pause/resume + F8 trigger) + TV3 (status form UI) | 3 | 6.6 |
| 6.8 | F8: Add-on phase brief form + approval workflow — causal chain, new capability requirement, why-invisible fields; routes to both sign-off authorities; dual-approval auto-generates amendment record; amendment record display | TV3 (form + UI) + TV2 (routing + auto-amendment NestJS logic) | 3 | 6.7 |
| 6.9 | Test: Milestone lifecycle with DoD + sprint — submit → DoD gate enforced → revision → JOINT approval → RELEASED → chi hộ confirmed; sprint status SCOPE_EVOLUTION → F8 auto-triggered | TV5 | 2 | 6.7, 6.4 |

**Week 6 milestone:** Full three-layer milestone tracking operational; DoD gate enforced at SUBMITTED state; sprint SCOPE_EVOLUTION auto-triggers add-on phase; escrow release + chi hộ call fires on APPROVED; pay-gated documents release correctly

---

#### Week 7 — Messaging, Service Purchase, Expert Withdrawal (Chi Hộ), Reviews, Dual-Role UX

| ID | Task | Owner | Days | Depends on |
|---|---|---|---|---|
| 7.1 | F9: Socket.io server setup in NestJS — room per engagement, message persistence; ADMIN read-only join | TV2 | 2 | 5.2 |
| 7.2 | F9: Chat UI — message thread, file attachment, read status, notification badge | TV2 | 3 | 7.1 |
| 7.3 | **F3: SERVICE_PURCHASE + TECH_DISCOVERY payment flows** — client "Buy Service" → NestJS creates per-service-order VA (`fixed_amount = service.price_vnd`); QR displayed; IPN SERVICE branch: atomic ledger ESCROW_LOCK + creates service_engagements FUNDED; expert notified; TECH_DISCOVERY path follows same flow | TV2 (NestJS + IPN extension) + TV3 (buy service UI) | 3 | 3.8, 2.6 |
| 7.4 | **F10: Expert withdrawal — chi hộ API integration (zero admin)** — withdrawal request form (requires `bank_account_xid`; blocked if null with prompt to complete Bank Hub linking); NestJS: atomic [WITHDRAWAL] ledger debit + `withdrawal_requests` PENDING; async chi hộ API call `{ amount, bank_account_xid, reference: "WD-{id}" }`; chi hộ IPN / credit IPN handler: matches reference → `withdrawal_requests` → COMPLETED → notify expert; FAILED path: wallet balance restore + notify | TV2 | 4 | 3.9, 6.4 |
| 7.5 | **F10: Transaction history + wallet dashboard** — CEO: VA numbers, funded milestones, wallet balance, subscription status; TECH_TEAM: milestone status, pay-gated inbox; Expert: earnings, withdrawal history with bank confirmation timestamps, available balance; Admin: full wallet_transactions + escrow_accounts ledger | TV3 | 3 | 7.4 |
| 7.6 | **F1: Dual-role UX** — "Add Role" flow in account settings (CEO adds EXPERT; Expert adds CEO); role switcher top-nav toggle; JWT reissue on switch; dashboard re-render for new active_role; self-exclusion guard verified in matching test | TV2 (role acquisition route + JWT reissue) + TV3 (role switcher UI) | 3 | 2.1 |
| 7.7 | F11: Post-engagement review forms — TECH_TEAM structured seam-signal review; CEO business review; Expert dual review (CEO + TECH_TEAM dimensions separately); review signal processing → `expert_seam_claims.verification_tier` update + Tier 4 signal accumulation check | TV3 (forms) + TV2 (signal processing) | 3 | 6.4 |
| 7.8 | **F10: Dispute escrow automation** — dispute state transitions (DISPUTED → escrow FROZEN); Layer 1 LLM criterion evaluation (FastAPI) + LEDGER on ≥ 0.80 confidence; Layer 2 mutual agreement form + auto-execute on match; Layer 3 50/50 ledger split + chi hộ for expert share + client balance restore | TV2 (state machine + ledger operations) + TV1 (Layer 1 LLM evaluation) + TV3 (dispute UI) | 4 | 6.4 |
| 7.9 | Test: Expert withdrawal end-to-end — request → chi hộ fires → credit IPN confirms → withdrawal COMPLETED; SERVICE_PURCHASE VA flow tested; dual-role switch verified | TV5 | 2 | 7.4, 7.6 |

**Week 7 milestone:** Messaging live; SERVICE_PURCHASE flow functional via wallet VA; expert withdrawal fully automated via chi hộ (zero admin); dual-role account switch working; dispute 3-layer resolution with automatic ledger operations verified
| 7.7 | F11: Review signal processing — NestJS reads structured review responses, updates `expert_seam_claims.verification_tier` and appends outcome signal record via Prisma | TV2 | 2 | 7.6 |
| 7.8 | F11: Reputation display on expert profile — completion rate, ratings, seam performance indicators | TV3 | 2 | 7.7 |
| 7.9 | Test: Messaging flow; service purchase SePay payment flow; withdrawal request cycle; review submission | TV5 | 2 | 7.2, 7.3, 7.5, 7.6 |

**Week 7 milestone:** Real-time messaging operational; service-based purchase flow live with real SePay payment; expert withdrawal cycle operational; post-engagement reviews and reputation signals working

---

#### Week 8 — Admin Dashboard, AWS Deployment, Integration, Bug Fixing

| ID | Task | Owner | Days | Depends on |
|---|---|---|---|---|
| 8.1 | F12: Admin user management — list, filter, suspend/reactivate | TV2 | 2 | 2.1 |
| 8.2 | F12: Dispute 3-layer resolution engine — NestJS dispute handler: on dispute filed, auto-freezes escrow (`escrow_records.state = FROZEN`) + triggers Layer 1 FastAPI LLM criterion evaluation; if confidence ≥ 0.80 → auto-resolves, auto-releases/refunds escrow per determination; if confidence < 0.80 → starts 48-hour cooling timer, sends structured mutual agreement form to both parties; scheduled job monitors cooling timer expiry → Layer 3 50/50 auto-split if no mutual agreement reached; full dispute log written to `platform_decisions` table; Dispute Monitor UI reads this log (read-only for admin) | TV2 (NestJS dispute handler + escrow freeze/release automation + cooling timer + Layer 3 auto-split) + TV1 (FastAPI Layer 1 LLM criterion evaluation) + TV3 (Dispute Monitor UI in admin dashboard) | 4 | 6.4, 7.4 |
| 8.3 | F12: Platform Integrity Monitor UI + Account Management + Fraud Detection — admin Platform Integrity Monitor: reads `platform_decisions` log table, displays auto-return history, auto-upgrade history, dispute layer history; account management: user list + suspend/reactivate; fraud signal: bank account deduplication check on withdrawal creation (if same account on 2+ experts → auto-block both + Telegram informational alert to admin) | TV2 (fraud detection logic + Telegram alert) + TV3 (Platform Integrity Monitor UI + account management UI) | 3 | 7.4 |
| 8.4 | F12: Analytics dashboard — active projects by archetype/tier, average match score, elicitation completion rate, auto-publish pass rate, seam verification throughput (auto-upgrade rate / auto-return rate), dispute layer distribution, milestone completion rate, SePay transaction volume, review completion rate | TV3 | 3 | all features |
| 8.5 | **AWS EC2 deployment** — provision t3.micro, assign Elastic IP, configure security groups, install Docker + Docker Compose, write docker-compose.prod.yml, deploy all services, configure IPN endpoint URL in SePay dashboard, verify end-to-end on live EC2 | TV5 | 2 | all features deployable |
| 8.6 | **GitHub Actions CD** — SSH deploy action on merge to `main`: pull → rebuild → `docker compose up -d`; test that deploy completes without downtime | TV5 | 1 | 8.5 |
| 8.7 | Full system integration test on EC2 — end-to-end: intake → elicitation → spec publish → shortlist → bid → connection → milestone AWAITING_PAYMENT → real SePay sandbox QR scan → FUNDED → work → submit → approve → RELEASED → expert balance updated → review | TV5 | 3 | 8.5 |
| 8.8 | Bug fixing sprint — all P1 (blocking) and P2 (major) bugs from 8.7 | TV1 + TV2 + TV3 | 3 | 8.7 |
| 8.9 | Edge case hardening — duplicate IPN detection (idempotency on inbound + out-webhook), amount mismatch handling, tech handoff link expiry, Artifact B access before connection, milestone without acceptance criteria (blocked), dispute re-filing after Layer 3 resolution (blocked), submission-limit lockout expiry re-enable (30-day timer) | TV1 + TV2 | 2 | 8.8 |
| 8.10 | Documentation update — UML sequence diagrams updated (include SePay IPN flow), API docs current, EC2 deployment guide | TV4 | 3 | 8.7 |

**Week 8 milestone:** Admin dashboard complete (monitoring-only, zero admin approval clicks); 3-layer dispute resolution engine with automatic ledger operations functional; expert withdrawal fully automated via SePay chi hộ (chi hộ API call → SePay credit IPN → COMPLETED, zero admin); AWS EC2 deployment live and stable; full integration test on EC2 passed; all P1 bugs resolved

---

#### Week 9 — Demo Preparation, Final Documentation, Submission

| ID | Task | Owner | Days | Depends on |
|---|---|---|---|---|
| 9.1 | Demo script finalization — step-by-step walkthrough of AdTech case study through the platform, scripted for a 15-minute live demo. Include a live SePay payment moment: presenter scans QR with phone → milestone FUNDED state appears on screen in real time | TV4 + TV1 | 2 | 8.8 |
| 9.2 | Demo data setup on EC2 — seed realistic expert profiles with different verification tiers, seed the AdTech CEO intake text, prepare expected footprint and shortlist outputs | TV5 | 2 | 8.8 |
| 9.3 | Internal demo run on EC2 — team runs the full demo script against live cloud environment, identifies any blocking issues | All | 1 | 9.1, 9.2 |
| 9.4 | Final bug fixes from internal demo | TV1 + TV2 + TV3 | 2 | 9.3 |
| 9.5 | Final report writing — system overview, research questions and findings, architecture, demo evidence (screenshots), research contribution statement | TV4 | 3 | 9.3 |
| 9.6 | Slide deck preparation | TV4 | 2 | 9.5 |
| 9.7 | Switch SePay from sandbox to production (VietQR already activated from Week 2); update MERCHANT_ID + SECRET_KEY on EC2 `.env`; verify IPN endpoint receives production webhooks | TV2 + TV5 | 0.5 | 9.4 |
| 9.8 | Final submission — report, slides, code repository link (EC2 URL for live demo) | All | 1 | 9.5, 9.7 |

**Week 9 milestone:** Live demo on AWS EC2 with real SePay VietQR payment moment, demo script rehearsed, final report submitted

---

### Milestone Summary

| Week | Milestone | Key Deliverables | Success Condition |
|---|---|---|---|
| **1** | Foundation locked | Schema (incl. wallets, virtual_accounts, bid_versions, dod_items, sprints, subscriptions, organizations stub), API contracts, SRS, dev environment | Both services start; schema migrated; team aligned; chi hộ tier decision confirmed |
| **2** | Supply side + wallet foundation | Expert taxonomy profiles, Path B marketplace, role switcher auth, wallet top-up VA → IPN → wallet credited, subscription purchase from wallet working | Wallet top-up sandbox verified; subscription activates and gates LLM routes; role switch reissues JWT |
| **3** | Elicitation engine + milestone VA funding | F2 complete (Scenario A/B implemented); per-milestone VA funding atomic ledger verified; Bank Hub Hosted Link expert linking working | AdTech footprint correct; VA fixed-amount enforced by bank; escrow_lock ledger entry verified; expert bank linked via Hosted Link |
| **4** | Matching engine + LLM verification | Composite scorer, seam gap maps, auto-upgrade (Tier 2+3), auto-return spec, Platform Integrity Monitor | Test matrix rankings correct; LLM auto-upgrade at ≥0.85 confidence; scenario LLM grading works |
| **5** | Non-linear bid system + trust gates | Connection flow, Artifact B gating, Surfaces A/B/C/D all functional, bid version history written, override logged | Bid revision loop creates versioned history; Surface D override requires reason field and logs to Platform Integrity Monitor |
| **6** | Full milestone management | 3-layer system: acceptance criteria + DoD gate + sprint tracking; SCOPE_EVOLUTION → F8 auto-trigger; APPROVED → RELEASED → chi hộ call fires | DOD gate returns 422 with missing items; criteria gate returns 422 with unverified criteria; chi hộ API called after approval; dispute Layer 1-3 all resolve atomically |
| **7** | Full marketplace operational | Messaging; SERVICE_PURCHASE VA flow; chi hộ withdrawal (zero admin); dual-role switch; 3-layer dispute escrow; reviews | Expert withdrawal COMPLETED via chi hộ IPN (no admin); dual-role switch changes dashboard; dispute auto-splits 50/50 if Layer 3 |
| **8** | Cloud deployment + admin dashboard | AWS EC2 live, GitHub Actions CD, Platform Integrity Monitor, bid conflict override log, transaction ledger, analytics | Platform accessible via Elastic IP; sandbox payment end-to-end on cloud; all P1/P2 bugs resolved |
| **9** | Delivered | Production SePay, live demo on EC2, final report, slides | Demo shows real VND wallet top-up → milestone funded → expert withdrawal completed automatically |

---

### Research Question Mapping (SWP Evidence Collection)

| Research Question | Evidence generated by | When collected |
|---|---|---|
| **RQ1:** How to improve matching accuracy between AI experts and client requirements? | Composite score distribution across shortlisted experts; pre-platform vs. on-platform spec quality (elicitation output vs. raw intake); expert seam coverage for connected engagements vs. declined bids; LLM verification confidence distribution (auto-upgrade vs. auto-return rates); bid revision history (Surface B — how many rounds before TECH_APPROVED, correlation with engagement success); Scenario A vs. B vs. standard footprint quality comparison | Admin Analytics dashboard + Platform Integrity Monitor — from Week 8 onwards |
| **RQ2:** How effective are AI-assisted tools in helping non-technical users define project scope? | Elicitation completion rate; auto-publish pass rate (first-attempt vs. resubmission); footprint completeness scores; Phase 0 injection acceptance rate; average DoD checklist items per milestone (more items = better expert self-scoping); sprint DEVIATION / BLOCKER rate (fewer = better initial scoping); Surface A clarification request rate (fewer = better Artifact A quality); bid revision request rate (Surface B — fewer revisions = cleaner initial bids) | Elicitation logs + `platform_decisions` log + `sprint_status_updates` + `bid_revision_requests` — from Week 3 onwards |
| **RQ3:** What factors influence trust and successful transactions in AI service marketplaces? | Connection acceptance rate per match strength tier; milestone completion rate; dispute frequency and Layer 1 auto-resolution rate; DoD completion rate before dispute (complete DoD → fewer disputes); Surface D override rate and correlation with negative outcomes; review completion rate; wallet top-up to first milestone funding time (measures trust-to-action latency); chi hộ withdrawal confirmation time (trust in automated payout); price negotiation (Surface C) — how often it's used and whether it improves or delays engagement formation | Review + reputation system + wallet_transactions + withdrawal_requests + bid_conflict_overrides — from Week 7 onwards |

---

### Critical Path

The critical path runs through F2 (Weeks 3–4) because everything downstream depends on the elicitation engine producing a valid footprint, and through F10's wallet/VA system (Weeks 2–3) because every financial operation depends on it:

- F4 (matching) depends on F2's footprint output
- F5 (dual artifact) depends on F2's Artifact A/B structure
- F6 non-linear bid surfaces depend on F5's connection flow
- F7 (milestone management + DoD + sprint) depends on F5's connection + F6's bid selection
- F10 wallet top-up + VA funding underpins all milestone escrow operations
- F1.5 (subscription) must gate F2 — subscription guard must exist before elicitation is buildable end-to-end
- F8, F9, F11 all depend on an active engagement
- chi hộ withdrawal (F10 outbound) depends on Bank Hub Hosted Link working (Week 3)

**If F2 slips, the entire project slips.** TV1 treats Weeks 3–4 as a focused sprint. TV4 must deliver elicitation UI copy (task 1.9) + Scenario A/B option copy by end of Week 1.

**If the wallet/VA system slips (task 2.8), milestone funding is blocked.** TV2 must set up SePay and implement WALLET_TOPUP VA + IPN branch before elicitation is integrated with payments (Week 3). The chi hộ verification (hotline call) must happen Week 1 Day 1 — this is the first task gating the entire outbound payment architecture.

**Non-linear bid system (Week 5) is high-complexity but not on the critical path.** It can slip one week without blocking Weeks 6–7, because F7 milestone management can begin with any connection being CONNECTED — it does not require the full bid revision loop to be functional.

**AWS EC2 deployment is TV5's responsibility from Week 8.** Do not defer to Week 9.

---
