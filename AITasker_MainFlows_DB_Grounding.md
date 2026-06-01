# AITasker — 18 Main Flows: Full DB Grounding & Cross-Table CRUD Analysis

> **Purpose of this document:** For each of the 18 main flows, this document explicitly traces every state change, event trigger, and database operation to the actual physical schema (51 tables, Physical ER v2). Every SQL block uses exact column names. Every state transition references Section 0.6 of the Master Reference Sheet. No swimlane ASCII art is repeated here — this document is the data-layer companion to the flow diagrams.

> **Schema note:** All money amounts are `BIGINT` (VND integer). All IDs are `UUID`. All timestamps are `TIMESTAMPTZ`. The canonical field for account access control is `users.is_active` (not `is_suspended` — use `UPDATE users SET is_active = false` for suspension). The `users.roles` is `JSONB`, e.g. `["CLIENT_CEO"]`.

---

## How to Read Each Flow Entry

Each flow entry contains:
- **Master Reference Anchors** — which sections of 0.1–0.6 this flow exercises
- **Pre-conditions** — state that must already exist in the DB
- **Step-by-step operations** — actor, event, exact SQL, state transitions
- **Cross-table dependency map** — which FKs are read/written and why
- **End-state trace** — final state of all touched tables

---

# GROUP 1 — Onboarding & Account Setup

---

## MF-1: Client (CEO) Registration & Subscription

### Master Reference Anchors
- **0.7 RBAC:** Creates `CLIENT / CEO` account type; `active_role = 'CLIENT'`, `client_subtype = 'CEO'`
- **0.9 Subscription Tiers:** Registration starts at `free`; subscription purchase unlocks elicitation engine
- **0.8 Payment Architecture:** Per-user WALLET_TOPUP VA created at registration; subscription uses PENDING_PAYMENT → ACTIVE flow

### Pre-conditions
- No existing `users` row with matching email

### Phase A — Registration

**Step [R1] — INSERT users (atomic with wallet + VA)**
```sql
BEGIN;

INSERT INTO users (
  email, password_hash, full_name, phone,
  roles,               -- JSONB: '["CLIENT_CEO"]'
  active_role,         -- 'CLIENT'
  client_subtype,      -- 'CEO'
  subscription_client_tier,  -- 'free'
  subscription_expert_tier,  -- 'free'
  sub_client_expires_at,     -- NULL
  sub_expert_expires_at,     -- NULL
  sepay_bank_account_xid,    -- NULL (CEO never links bank; only experts do)
  self_technical,            -- FALSE
  is_active                  -- TRUE
) VALUES (?, bcrypt(?), ?, ?, '["CLIENT_CEO"]', 'CLIENT', 'CEO',
          'free', 'free', NULL, NULL, NULL, FALSE, TRUE)
RETURNING id AS new_user_id;

-- Every user gets exactly one wallet; one wallet serves all roles
INSERT INTO wallets (user_id, available_balance, locked_balance)
VALUES (new_user_id, 0, 0);

COMMIT;
```

**Step [R2] — Create WALLET_TOPUP Virtual Account via SePay Bank Hub API**
```
POST SePay Bank Hub API → returns { va_number: "VN..." }
```
```sql
INSERT INTO virtual_accounts (
  entity_type,   -- 'WALLET_TOPUP'
  entity_id,     -- new_user_id (IPN resolution: va_number → user)
  va_number,     -- from SePay response
  fixed_amount,  -- NULL (any top-up amount accepted)
  expires_at,    -- NULL (permanent VA for this user)
  status         -- 'ACTIVE'
) VALUES ('WALLET_TOPUP', new_user_id, ?, NULL, NULL, 'ACTIVE');
```

**Cross-table dependency:** `virtual_accounts.entity_id → users.id` (no DB-level FK by design; polymorphic — the IPN handler reads `entity_type` first to know which table `entity_id` refers to)

**State after Phase A:**
| Table | New rows/changes |
|---|---|
| `users` | 1 row: `is_active=true`, `sub_client_tier='free'` |
| `wallets` | 1 row: `available_balance=0`, `locked_balance=0` |
| `virtual_accounts` | 1 row: `entity_type='WALLET_TOPUP'`, `status='ACTIVE'` |

### Phase B — Wallet Top-Up (SePay IPN Flow)

**Step [T1] — SePay fires IPN on VA credit**
```sql
-- Step 1: Resolve VA to user
SELECT entity_id, entity_type FROM virtual_accounts
WHERE va_number = ? AND entity_type = 'WALLET_TOPUP';
-- Returns: entity_id = user_id

-- Step 2: Idempotency check (unique index prevents double-credit)
-- UNIQUE INDEX wallet_tx_idempotency ON wallet_transactions(wallet_id, reference_id)
-- WHERE reference_id IS NOT NULL;
-- If reference_id already exists → return 200, no action

-- Step 3: Atomic credit
BEGIN;
UPDATE wallets
  SET available_balance = available_balance + ?  -- amount from IPN
  WHERE user_id = (SELECT entity_id FROM virtual_accounts WHERE va_number = ?);

INSERT INTO wallet_transactions (wallet_id, amount, transaction_type, reference_id, created_at)
  VALUES (?, ?, 'TOP_UP', 'TOP_UP:' || va_number || ':' || transfer_ref, now());
COMMIT;
```

**State transition (Section 0.6 Wallet Types):** `TOP_UP` ledger entry written → `wallets.available_balance` increases.

### Phase C — Subscription Purchase (PENDING_PAYMENT → ACTIVE)

**Step [S1] — Create subscription row + SUBSCRIPTION VA**
```sql
BEGIN;
-- Create pending subscription row FIRST so its id can be used as VA entity_id
INSERT INTO user_subscriptions (user_id, role_type, tier, status, price_paid)
  VALUES (?, 'client', 'pro', 'PENDING_PAYMENT', 500000)
RETURNING id AS sub_id;

-- Create SUBSCRIPTION VA with the subscription row's id as entity_id
-- (NOT user_id — this lets IPN handler resolve to exact subscription row)
INSERT INTO virtual_accounts (entity_type, entity_id, va_number, fixed_amount, expires_at, status)
  VALUES ('SUBSCRIPTION', sub_id, ?, 500000, now() + INTERVAL '24 hours', 'ACTIVE');
COMMIT;
-- Return VietQR to frontend for user to scan
```

**Step [S2] — SePay IPN fires on SUBSCRIPTION VA credit**
```sql
-- IPN SUBSCRIPTION branch: entity_type='SUBSCRIPTION', entity_id=user_subscriptions.id
BEGIN;
-- Debit wallet
UPDATE wallets
  SET available_balance = available_balance - 500000
  WHERE user_id = ?;

-- Immutable ledger entry
INSERT INTO wallet_transactions (wallet_id, amount, transaction_type, reference_id, created_at)
  VALUES (?, 500000, 'SUBSCRIPTION', 'SUB:' || sub_id, now());

-- Activate subscription
UPDATE user_subscriptions
  SET status = 'ACTIVE',
      activated_at = now(),
      expires_at = now() + INTERVAL '6 months',
      price_paid = 500000
  WHERE id = sub_id;

-- Mirror to users row for JWT guard performance
UPDATE users
  SET subscription_client_tier = 'pro',
      sub_client_expires_at = now() + INTERVAL '6 months'
  WHERE id = ?;

-- Mark VA as used
UPDATE virtual_accounts SET status = 'USED' WHERE entity_id = sub_id::text AND entity_type = 'SUBSCRIPTION';
COMMIT;
```

**State transition (Section 0.6 Subscription States):** `PENDING_PAYMENT → ACTIVE`

**Subscription guard:** Every FastAPI-calling route checks `users.subscription_client_tier = 'pro'` AND `users.sub_client_expires_at > now()`. Guard returns HTTP 403 `{ code: "SUBSCRIPTION_REQUIRED" }` on failure.

### End-State Trace — MF-1

| Table | Field | Value after completion |
|---|---|---|
| `users` | `subscription_client_tier` | `'pro'` |
| `users` | `sub_client_expires_at` | `now() + 6 months` |
| `wallets` | `available_balance` | top-up amount − 500,000 VND |
| `wallet_transactions` | types | `TOP_UP` + `SUBSCRIPTION` rows |
| `user_subscriptions` | `status` | `ACTIVE` |
| `virtual_accounts` | WALLET_TOPUP | `status='ACTIVE'` (permanent) |
| `virtual_accounts` | SUBSCRIPTION | `status='USED'` |

---

## MF-2: Expert Registration, Profile & Verification (Tier 1 → 2 → 3)

### Master Reference Anchors
- **0.4 Expert Verification Tiers:** Full progression from Claimed (0.20) → Evidence-backed (0.55) → Scenario-verified (0.80)
- **0.1 Six Domains / 0.2 Ten Seams:** `expert_domain_depths` stores domain codes A–F; `expert_seam_claims` stores seam codes SEAM_AC, etc.
- **0.7 RBAC:** `active_role = 'EXPERT'`, `client_subtype = NULL`

### Phase A — Registration (same pattern as MF-1, different role fields)

```sql
BEGIN;
INSERT INTO users (email, password_hash, full_name, phone,
  roles,        -- '["EXPERT"]'
  active_role,  -- 'EXPERT'
  client_subtype, -- NULL
  subscription_client_tier, -- 'free'
  subscription_expert_tier, -- 'free'
  is_active) VALUES (?, bcrypt(?), ?, ?, '["EXPERT"]', 'EXPERT', NULL, 'free', 'free', TRUE)
RETURNING id AS expert_user_id;

INSERT INTO wallets (user_id, available_balance, locked_balance)
  VALUES (expert_user_id, 0, 0);
COMMIT;
-- Then: WALLET_TOPUP VA created via SePay Bank Hub (same as MF-1 Step R2)
```

### Phase B — Tier 1 Profile Creation (Free Tier — no subscription required)

**Step [P1] — INSERT expert_profiles**
```sql
INSERT INTO expert_profiles (user_id, bio, engagement_model, archetype_history_json)
  VALUES (?, 'Experienced ML practitioner...', 'MILESTONE',
          '[{"archetype_code":"ARCHETYPE_1","tier":"TIER_3","self_declared":true}]');
-- archetype_history_json seeds the 20% Archetype-tier congruence score for cold-start experts.
-- Once real engagements close, this is overridden at query time from engagements→projects.
```

**Step [P2] — INSERT expert_domain_depths (one row per declared domain)**
```sql
-- Expert claims Deep knowledge of Domain A, Working of Domain C, Surface of Domain E:
INSERT INTO expert_domain_depths (expert_id, domain_code, depth_level, verification_tier)
  VALUES
  (expert_user_id, 'DOMAIN_A', 'DEEP',    'CLAIMED'),  -- confidence_factor = 0.20
  (expert_user_id, 'DOMAIN_C', 'OPERATIONAL', 'CLAIMED'),
  (expert_user_id, 'DOMAIN_E', 'SURFACE', 'CLAIMED');
-- UNIQUE (expert_id, domain_code) — prevents duplicate domain declarations
```

**Step [P3] — INSERT expert_seam_claims (one row per claimed seam)**
```sql
-- Expert claims Seam A↔C and Seam D↔E:
INSERT INTO expert_seam_claims (expert_id, seam_code, verification_tier, submission_count, locked_until)
  VALUES
  (expert_user_id, 'SEAM_AC', 'CLAIMED', 0, NULL),  -- Section 0.2: A↔C seam
  (expert_user_id, 'SEAM_DE', 'CLAIMED', 0, NULL);  -- Section 0.2: D↔E seam
-- UNIQUE (expert_id, seam_code)
-- submission_count: tracks 5-attempt throttle
-- locked_until: 30-day lockout after 5 failed submissions
```

**Step [P4] — INSERT expert_stack_tags**
```sql
INSERT INTO expert_stack_tags (expert_id, tag) VALUES
  (expert_user_id, 'Python'),
  (expert_user_id, 'Kafka');
-- UNIQUE (expert_id, tag)
```

### Phase C — Expert Pro Subscription (required for Tier 2 and 3)

Same pattern as MF-1 Phase C, but `role_type = 'expert'`, `price_paid = 300000`:
```sql
-- user_subscriptions: role_type='expert', tier='pro', status='PENDING_PAYMENT' → 'ACTIVE'
-- users: subscription_expert_tier='pro', sub_expert_expires_at=now()+6months
```

### Phase D — Tier 2 Upgrade via LLM Portfolio Evidence

**Step [V1] — Expert submits portfolio evidence**
```sql
INSERT INTO portfolio_submissions (
  expert_id,
  seam_claim_id,  -- FK to expert_seam_claims.id — exactly which seam is being upgraded
  project_description,
  decision_points,
  status,         -- 'PENDING'
  submitted_at
) VALUES (expert_user_id, seam_claim_id_ac, 'Built A/B testing...', 'Decision 1...', 'PENDING', now());
```

**Critical FK:** `portfolio_submissions.seam_claim_id → expert_seam_claims(id)` — this direct FK eliminates any ambiguity about which seam claim the LLM evaluation result should update. Without this, a confidence score returning from FastAPI has no guaranteed target.

**Step [V2] — FastAPI LLM extraction runs**
FastAPI evaluates the submission text against the seam taxonomy. Returns:
```json
{ "seam_code": "SEAM_AC", "confidence": 0.91, "signals_found": ["cross_domain_diagnosis", "evaluation_metric_selection", "failure_mode_awareness"], "all_required": true }
```

**Step [V3] — Auto-upgrade branch (confidence ≥ 0.85 AND all required signals)**
```sql
BEGIN;
-- Upgrade the seam claim
UPDATE expert_seam_claims
  SET verification_tier = 'EVIDENCE_BACKED'  -- confidence_factor → 0.55
  WHERE id = seam_claim_id_ac;

-- Update the submission
UPDATE portfolio_submissions
  SET status = 'APPROVED',
      llm_confidence = 0.91,
      evaluated_at = now()
  WHERE id = ?;

-- Write audit record (FastAPI writes to platform_decisions; NestJS does NOT)
INSERT INTO platform_decisions (
  decision_type,   -- 'PORTFOLIO_EVAL'
  entity_type,     -- 'portfolio_submission'
  entity_id,       -- portfolio_submissions.id
  llm_confidence,  -- 0.91
  decision,        -- 'APPROVED'
  advisory_note    -- NULL (no gap note on success)
) VALUES ('PORTFOLIO_EVAL', 'portfolio_submission', ?, 0.91, 'APPROVED', NULL);
COMMIT;
-- Notify expert: "Seam A↔C is now Evidence-backed"
```

**Step [V3b] — Auto-return branch (confidence < 0.85 OR missing signals)**
```sql
BEGIN;
UPDATE expert_seam_claims
  SET submission_count = submission_count + 1,  -- increment throttle counter
      locked_until = CASE WHEN submission_count + 1 >= 5
                     THEN now() + INTERVAL '30 days' ELSE NULL END
  WHERE id = seam_claim_id_ac;

UPDATE portfolio_submissions SET status = 'REJECTED', llm_confidence = 0.61, evaluated_at = now()
  WHERE id = ?;

INSERT INTO platform_decisions (decision_type, entity_type, entity_id, llm_confidence, decision, advisory_note)
  VALUES ('PORTFOLIO_EVAL', 'portfolio_submission', ?, 0.61, 'RETURNED',
    'Your response demonstrated cross_domain_diagnosis. Missing: production_constraint_consideration.');
COMMIT;
```

### Phase E — Tier 3 Upgrade via Scenario Assessment

**Step [S1] — Auto-select question from question bank**
```sql
-- Select lowest last_used_at to prevent gaming
SELECT id, question_text, rubric_json
  FROM scenario_assessments
  WHERE seam_code = 'SEAM_AC'
  ORDER BY last_used_at ASC NULLS FIRST
  LIMIT 1;

-- Mark as delivered
UPDATE scenario_assessments SET last_used_at = now() WHERE id = ?;
```

**Step [S2] — Expert submits response**
```sql
INSERT INTO scenario_responses (
  assessment_id,   -- FK to scenario_assessments.id
  expert_id,
  seam_claim_id,   -- FK to expert_seam_claims.id (direct — no ambiguity)
  response_text,
  pass_fail,       -- 'PENDING' until LLM evaluates
  submitted_at
) VALUES (assessment_id, expert_user_id, seam_claim_id_ac, 'My approach...', 'PENDING', now());
```

**Step [S3] — FastAPI LLM rubric evaluation**
```sql
-- All required dimensions must individually pass (NOT average)
-- rubric_json example: { "dimensions": ["cross_domain_diagnosis", "production_constraint_consideration"], "each_threshold": 0.70 }

-- PASS: all dimensions pass
BEGIN;
UPDATE expert_seam_claims SET verification_tier = 'SCENARIO_VERIFIED'  -- confidence → 0.80
  WHERE id = seam_claim_id_ac;
UPDATE scenario_responses
  SET pass_fail = 'PASS',
      llm_rubric_scores_json = '{"cross_domain_diagnosis": 0.91, "production_constraint_consideration": 0.78}'
  WHERE id = ?;
INSERT INTO platform_decisions (decision_type, entity_type, entity_id, llm_confidence, decision)
  VALUES ('SCENARIO_EVAL', 'scenario_response', ?, 0.84, 'PASS');
COMMIT;
```

### Phase F — Bank Account Linking via Bank Hub Hosted Link (required for withdrawal)

**Step [B1] — SePay Bank Hub BANK_ACCOUNT_LINKED webhook fires**
```sql
-- Expert completes OTP flow in embedded WebView; SePay fires webhook
UPDATE users
  SET sepay_bank_account_xid = ?,      -- SePay-verified bank_account_xid
      bank_account_holder_name = ?,    -- bank-verified name (not self-reported)
      bank_linked_at = now()
  WHERE id = expert_user_id;
```

**No manual bank detail entry.** The account number and holder name are bank-verified, not self-reported — this prevents routing errors in chi hộ disbursements.

### End-State Trace — MF-2

| Table | Key field values after completion |
|---|---|
| `users` | `active_role='EXPERT'`, `subscription_expert_tier='pro'`, `sepay_bank_account_xid` set |
| `expert_profiles` | `archetype_history_json` seeded |
| `expert_domain_depths` | Rows for DOMAIN_A, DOMAIN_C, DOMAIN_E: `verification_tier='CLAIMED'` initially |
| `expert_seam_claims` | SEAM_AC: `verification_tier='SCENARIO_VERIFIED'`; SEAM_DE: `'CLAIMED'` |
| `portfolio_submissions` | SEAM_AC submission: `status='APPROVED'`, `llm_confidence=0.91` |
| `scenario_responses` | SEAM_AC response: `pass_fail='PASS'` |
| `platform_decisions` | `PORTFOLIO_EVAL` + `SCENARIO_EVAL` rows (written by FastAPI) |

---

## MF-3: Tech Team Account Creation via Handoff Link

### Master Reference Anchors
- **0.7 RBAC:** `CLIENT / TECH_TEAM` is the **only** role that cannot self-register; must use signed handoff link
- **Section F1:** `client_subtype = 'TECH_TEAM'`, `linked_project_id` on `tech_team_profiles` scopes the account to one project

### Pre-conditions
- `projects` row exists (DRAFT state); CEO's `elicitation_sessions` row has `current_stage = 4` pending

### Step [H1] — CEO generates signed handoff link (NestJS)

The signed link encodes `{ project_id, client_subtype: 'TECH_TEAM' }` in a JWT with 72-hour expiry. The link URL carries the JWT as a query parameter. There is no `project_handoff_links` table in the physical ER — the link's validity is encoded in the JWT signature itself.

```
Handoff URL: https://aitasker.vn/handoff?token={signed_jwt}
JWT payload: { project_id: "proj_abc", client_subtype: "TECH_TEAM", exp: now+72h }
```

### Step [H2] — TECH_TEAM clicks link and registers

```sql
BEGIN;
INSERT INTO users (
  email, password_hash, full_name, phone,
  roles,          -- '["CLIENT"]'
  active_role,    -- 'CLIENT'
  client_subtype, -- 'TECH_TEAM'  ← set from signed token; cannot be self-selected
  is_active
) VALUES (?, bcrypt(?), ?, ?, '["CLIENT"]', 'CLIENT', 'TECH_TEAM', TRUE)
RETURNING id AS tech_user_id;

-- Wallet + WALLET_TOPUP VA (same as other users)
INSERT INTO wallets (user_id, available_balance, locked_balance) VALUES (tech_user_id, 0, 0);
COMMIT;
```

### Step [H3] — INSERT tech_team_profiles (scopes account to one project)

```sql
INSERT INTO tech_team_profiles (
  user_id,            -- tech_user_id
  linked_client_id,   -- CEO's user_id (who generated the link)
  linked_project_id,  -- project_id from the signed token
  role_title          -- optional: "CTO", "Lead Backend"
) VALUES (tech_user_id, ceo_user_id, 'proj_abc', 'Lead Backend');
```

**Critical FK chain:** `tech_team_profiles.linked_project_id → projects(id)` — this scope guard prevents the TECH_TEAM account from accessing any other project's Artifact B, milestone sign-offs, or bid reviews.

**TECH_TEAM-specific guards in NestJS:**
- Artifact B access: route checks `tech_team_profiles.linked_project_id = requested_project_id`
- Bid review: `client_subtype = 'TECH_TEAM'` required
- Stage 4 form submission: `client_subtype = 'TECH_TEAM'` required (CEO cannot complete this route)

### Step [H4] — TECH_TEAM completes Stage 4 → populates artifact_b

```sql
UPDATE artifact_b
  SET tech_stack_json = '{"language":"Go","broker":"Kafka"}',
      schema_uploads_json = '{"s3_paths":["s3://aitasker-vault/proj_abc/schema.sql"]}',
      integration_contracts_json = '{"kafka_topic":"ad_submissions","payload_format":"Avro"}'
  WHERE project_id = 'proj_abc';

-- Update elicitation session stage
UPDATE elicitation_sessions
  SET current_stage = 5,  -- ready for synthesis
      scenario_type = 'STANDARD'
  WHERE user_id = ceo_user_id AND state = 'IN_PROGRESS';
```

### End-State Trace — MF-3

| Table | Field | Value |
|---|---|---|
| `users` | `client_subtype` | `'TECH_TEAM'` |
| `tech_team_profiles` | `linked_project_id` | `'proj_abc'` |
| `artifact_b` | `tech_stack_json` | populated with real architecture |
| `elicitation_sessions` | `current_stage` | `5` (synthesis ready) |

---

# GROUP 2 — Path A: Project-Based Flow

---

## MF-4: AI Elicitation Engine (Project Intake)

### Master Reference Anchors
- **0.3 Archetypes + Tiers:** Elicitation engine classifies project into one of 6 archetypes + 3 tiers
- **0.2 Ten Seams:** Void detection flags missing SDLC components; seam requirements written to `capability_footprints.required_seams_json`
- **0.6 Elicitation Session States:** `IN_PROGRESS → COMPLETED` (or `RETURNED` if quality gate fails)
- **0.9 Subscription:** Elicitation engine is Client Pro-gated

### Pre-conditions
- `users.subscription_client_tier = 'pro'` AND `sub_client_expires_at > now()` (subscription guard passes)

### Stage 1 — Session Creation + Symptom Intake

**Step [E1] — INSERT elicitation_sessions and projects**
```sql
BEGIN;
INSERT INTO elicitation_sessions (user_id, current_stage, state, void_list_json)
  VALUES (ceo_user_id, 1, 'IN_PROGRESS', '[]')
RETURNING id AS session_id;

INSERT INTO projects (client_id, elicitation_session_id, state, self_technical)
  VALUES (ceo_user_id, session_id, 'DRAFT', FALSE)
RETURNING id AS project_id;

INSERT INTO artifact_a (project_id, state) VALUES (project_id, 'DRAFT');
INSERT INTO artifact_b (project_id) VALUES (project_id);
INSERT INTO capability_footprints (project_id, required_domains_json, required_seams_json,
  milestone_framework_json) VALUES (project_id, '[]', '[]', '[]');
COMMIT;
```

**FK chain:** `projects.elicitation_session_id → elicitation_sessions(id)` — direction is always project → session (not reverse). A session can exist without a project (abandoned attempt); a project always points to the session that produced it.

**Step [E2] — CEO submits free-form symptom text → FastAPI 3-layer extraction**

FastAPI runs three extraction passes on the raw text:
1. **Intent separation** — strips "we want to fine-tune a model" (prescribed solution) from "our compliance rate is inconsistent" (actual symptom)
2. **Scale signal extraction** — identifies volume/throughput indicators (e.g., "600K ads/month" → Tier 3 signal)
3. **Void detection** — detects absent SDLC components (no evaluation ground truth → flag `GROUND_TRUTH_VOID`)

FastAPI writes to `platform_decisions`:
```sql
INSERT INTO platform_decisions (decision_type, entity_type, entity_id, decision, advisory_note)
  VALUES ('ELICITATION_SYNTHESIS', 'project', project_id,
    'ARCHETYPE_1_TIER_3', 'Detected voids: GROUND_TRUTH_VOID, ASYNC_PIPELINE_VOID');
```

**Step [E3] — Update session with Stage 1 results**
```sql
UPDATE elicitation_sessions
  SET current_stage = 2,
      archetype = 'ARCHETYPE_1',
      void_list_json = '[{"void_code":"GROUND_TRUTH_VOID","injected":false},
                         {"void_code":"ASYNC_PIPELINE_VOID","injected":false}]',
      updated_at = now()
  WHERE id = session_id;
```

### Stage 2 — Archetype Confirmation + SDLC Injection

**Step [E4] — CEO confirms Archetype 1 → Phase 0 injected (cannot be removed)**
```sql
UPDATE elicitation_sessions
  SET current_stage = 3,
      void_list_json = '[{"void_code":"GROUND_TRUTH_VOID","injected":true},
                         {"void_code":"ASYNC_PIPELINE_VOID","injected":false}]',
      updated_at = now()
  WHERE id = session_id;

-- Phase 0 milestone pre-seeded into footprint
UPDATE capability_footprints
  SET milestone_framework_json = '[{"phase":0,"name":"Ground Truth Setup","mandatory":true,
    "rationale":"No evaluation baseline = no way to verify AI accuracy"}]'
  WHERE project_id = project_id;
```

### Stage 3 — Architecture Probe → Infrastructure Threshold

**Step [E5] — CEO answers 4 behavioral questions**

If answers indicate async pipeline + >1M records → `requires_tech_handoff = true` → MF-3 triggered.

**Step [E6] — Update session for Scenario B (self_technical)**
```sql
-- If CEO is self-technical:
UPDATE projects SET self_technical = TRUE WHERE id = project_id;
UPDATE elicitation_sessions
  SET current_stage = 4,
      scenario_type = 'SCENARIO_B',
      updated_at = now()
  WHERE id = session_id;
```

### Stage 5 — Synthesis + Auto-Publish Quality Gate

**Step [E7] — FastAPI synthesis produces footprint and Artifact A**
```sql
BEGIN;
UPDATE capability_footprints
  SET required_domains_json = '[{"code":"DOMAIN_A","depth":"DEEP"},
                                 {"code":"DOMAIN_C","depth":"OPERATIONAL"},
                                 {"code":"DOMAIN_E","depth":"SURFACE"}]',
      required_seams_json = '[{"code":"SEAM_AC","criticality":"LOAD_BEARING"},
                               {"code":"SEAM_CE","criticality":"SIGNIFICANT"},
                               {"code":"SEAM_DE","criticality":"SIGNIFICANT"}]',
      milestone_framework_json = '[{"phase":0,"name":"Ground Truth Setup"},
                                    {"phase":1,"name":"Core Decision Logic"},
                                    {"phase":2,"name":"Pipeline Integration"}]',
      technical_architecture = 'ASYNC_KAFKA_STATEFUL'
  WHERE project_id = project_id;

UPDATE artifact_a
  SET business_intent = 'Automate ad compliance decisions at 600K/month scale',
      physics_constraints_json = '{"throughput":"600K/month","latency":"<2s p95","volume_tier":"TIER_3"}',
      escrow_lock_notice = 'Detailed schemas, payload structures are held in information escrow...'
  WHERE project_id = project_id;

-- Synthesis session complete
UPDATE elicitation_sessions
  SET current_stage = 5, state = 'COMPLETED', updated_at = now()
  WHERE id = session_id;
COMMIT;
```

**Step [E8] — Automated quality gate (three checks)**

The quality gate runs automatically after synthesis:
1. `capability_footprints.required_seams_json` must contain ≥ archetype minimum seam count
2. Matching pre-check: `SELECT COUNT(*) FROM expert_seam_claims WHERE seam_code = ANY(required_seams_array)` ≥ 1 qualifying expert
3. No `void_list_json` entries with `injected = false` remaining

**PASS path:**
```sql
UPDATE projects SET state = 'PUBLISHED' WHERE id = project_id;
UPDATE artifact_a SET state = 'PUBLISHED' WHERE project_id = project_id;
-- Matching engine triggered (MF-5)
```

**FAIL path (e.g., incomplete footprint):**
```sql
UPDATE projects SET state = 'RETURNED_TO_CLIENT' WHERE id = project_id;
UPDATE artifact_a SET state = 'RETURNED_TO_CLIENT' WHERE project_id = project_id;
UPDATE elicitation_sessions
  SET state = 'RETURNED',
      current_stage = 2,  -- re-enter at the failing stage, not stage 1
      updated_at = now()
  WHERE id = session_id;
-- LLM generates advisory note → INSERT platform_decisions with advisory_note
INSERT INTO platform_decisions (decision_type, entity_type, entity_id, decision, advisory_note)
  VALUES ('SPEC_AUTO_RETURN', 'project', project_id, 'RETURNED',
          'Ground Truth Baseline void not resolved. Please answer: Do you have labeled examples?');
```

**State transition (Section 0.6 Spec States):** `DRAFT → PUBLISHED` or `DRAFT → RETURNED_TO_CLIENT`

### End-State Trace — MF-4

| Table | Key state |
|---|---|
| `elicitation_sessions` | `state='COMPLETED'`, `current_stage=5`, `archetype='ARCHETYPE_1'` |
| `projects` | `state='PUBLISHED'`, `tier='TIER_3'` |
| `capability_footprints` | All JSON fields populated with seams + milestone framework |
| `artifact_a` | `state='PUBLISHED'` |
| `artifact_b` | Populated by TECH_TEAM |
| `platform_decisions` | `ELICITATION_SYNTHESIS` + possibly `SPEC_AUTO_RETURN` rows |

---

## MF-5: AI Matching & Shortlisting

### Master Reference Anchors
- **0.5 Composite Match Score Weights:** 40% seam alignment, 25% domain depth, 20% archetype-tier, 10% engagement model, 5% stack tags
- **0.4 Verification Tiers → Confidence Factors:** CLAIMED=0.20, EVIDENCE_BACKED=0.55, SCENARIO_VERIFIED=0.80, PLATFORM_DEMONSTRATED=0.95
- **0.3 Hard Gates:** claimed-to-verified ratio >4:1 excludes from Tier 2+ projects; 2+ NEGATIVE seam outcome signals on load-bearing seam excludes from that project

### Pre-conditions
- `projects.state = 'PUBLISHED'`
- `capability_footprints` populated with `required_seams_json` and `required_domains_json`
- Expert profiles exist in DB with `expert_seam_claims` and `expert_domain_depths`

### Step [M1] — FastAPI reads footprint and candidate experts

```sql
-- Read the published footprint
SELECT cf.required_seams_json, cf.required_domains_json, cf.technical_architecture, p.tier, p.archetype
  FROM capability_footprints cf
  JOIN projects p ON p.id = cf.project_id
  WHERE p.id = ?;

-- Read all candidate expert profiles
SELECT ep.user_id, ep.engagement_model, ep.archetype_history_json,
       u.subscription_expert_tier
  FROM expert_profiles ep
  JOIN users u ON u.id = ep.user_id
  WHERE u.is_active = TRUE
    AND u.subscription_expert_tier = 'pro';  -- Free experts cannot bid on Tier 2+ projects
```

### Step [M2] — Apply hard gate filters BEFORE scoring

```sql
-- Hard gate 1: claimed-to-verified ratio > 4:1 → exclude from Tier 2+ projects
-- (Tier 2+ projects have required_seams that need verified coverage)
SELECT esc.expert_id,
       COUNT(*) FILTER (WHERE esc.verification_tier = 'CLAIMED') AS claimed_count,
       COUNT(*) FILTER (WHERE esc.verification_tier != 'CLAIMED') AS verified_count
  FROM expert_seam_claims esc
  GROUP BY esc.expert_id
  HAVING (COUNT(*) FILTER (WHERE esc.verification_tier = 'CLAIMED')::FLOAT /
         NULLIF(COUNT(*) FILTER (WHERE esc.verification_tier != 'CLAIMED'), 0)) > 4;
-- → exclude these expert_ids from shortlist candidates

-- Hard gate 2: 2+ NEGATIVE outcome signals on load-bearing seam of THIS project
SELECT esos.expert_id
  FROM expert_seam_outcome_signals esos
  WHERE esos.seam_code = ANY(load_bearing_seam_codes)  -- from required_seams_json
    AND esos.valence = 'NEGATIVE'
  GROUP BY esos.expert_id
  HAVING COUNT(*) >= 2;
-- → exclude these expert_ids from this specific project's shortlist
```

### Step [M3] — Five-component composite score computation (FastAPI)

For each non-excluded expert, FastAPI computes:

```
Component 1 (40%): Seam Alignment
  For each required seam in footprint:
    Find expert's expert_seam_claims.verification_tier for that seam
    Map to confidence_factor: CLAIMED→0.20, EVIDENCE_BACKED→0.55, SCENARIO_VERIFIED→0.80, PLATFORM_DEMONSTRATED→0.95
    Multiply by seam criticality weight: LOAD_BEARING→1.0, SIGNIFICANT→0.7, CONTRIBUTING→0.4
    Missing seam on LOAD_BEARING position → NEGATIVE contribution (-0.1)

Component 2 (25%): Domain Depth Coverage
  For each required domain in footprint:
    Expert depth ≥ required depth? → 1.0 per domain; else 0.0

Component 3 (20%): Archetype-Tier Congruence
  Check engagements→projects for same archetype AND tier as current project
  Cold start: read expert_profiles.archetype_history_json
  1 match → 1.0; none → 0.0

Component 4 (10%): Engagement Model Fit
  footprint.engagement_model_required == expert_profiles.engagement_model ? 1.0 : 0.0

Component 5 (5%): Stack Tag Overlap
  COUNT(matching tags) / COUNT(required tags), weighted by recency of engagements using those tags
```

### Step [M4] — Shortlist materialization and seam gap map generation

```sql
-- Store match results (no explicit expert_job_matches table in schema;
-- shortlist is returned as API response and rendered client-side)
-- Seam gap map for each expert is computed in FastAPI and returned in the API response:
-- { seam_code: "SEAM_AC", coverage: "SCENARIO_VERIFIED", color: "GREEN" }
-- { seam_code: "SEAM_DE", coverage: "CLAIMED", color: "YELLOW" }
-- { seam_code: "SEAM_EF", coverage: "ABSENT", color: "RED" }
```

**Match strength labels (Section 0.5):**
- composite_score > 0.78 → `STRONG`
- 0.58–0.78 → `QUALIFIED`
- 0.42–0.58 → `CONDITIONAL`

CEO sees: label + seam gap map + verified capability summary. CEO never sees the numeric score.

### Self-Exclusion Guard for Dual-Role Users

```sql
-- Before including any expert in shortlist:
SELECT 1 FROM tech_team_profiles WHERE user_id = candidate_expert_id AND linked_project_id = ?;
UNION
SELECT 1 FROM projects WHERE client_id = candidate_expert_id AND id = ?;
-- If any row found → exclude this candidate (they are involved in this project as client)
```

---

## MF-6: Bid & Connection Flow (Non-Linear — Surfaces A, B, C, D)

### Master Reference Anchors
- **0.6 Bid States:** `DRAFT → SUBMITTED → TECH_REVIEW → REVISION_REQUESTED → REVISED → TECH_APPROVED/TECH_DISAPPROVED → [CONFLICT_PENDING] → CEO_REVIEW → [PRICE_NEGOTIATION → NEGOTIATION_COMPLETE] → SELECTED/DECLINED`
- **0.6 Engagement States:** `PENDING → CONNECTED → ACTIVE`
- **0.7 RBAC:** TECH_TEAM reviews bids (route guard: `client_subtype = 'TECH_TEAM'`); CEO selects (route guard: `client_subtype = 'CEO'`)

### Pre-conditions
- Shortlist exists (MF-5 completed); expert is subscribed (`subscription_expert_tier = 'pro'` for Tier 2+ projects)

### Surface A — Pre-Bid Spec Clarification

**Step [A1] — Expert files clarification request**
```sql
INSERT INTO spec_clarifications (
  spec_id,         -- FK to artifact_a.project_id
  expert_id,
  responded_by,    -- NULL until ANSWERED
  target_component, -- 'MILESTONE_FRAMEWORK' | 'FOOTPRINT' | 'ACCEPTANCE_CRITERIA' | 'SYSTEM_PHYSICS'
  question_text,
  status           -- 'OPEN'  (CHECK: NOT 'CLOSED' — 'CLOSED' is an invalid status value)
) VALUES (project_id, expert_user_id, NULL, 'MILESTONE_FRAMEWORK', 'Phase 0 — is ground truth already labeled?', 'OPEN');
```

**Step [A2] — TECH_TEAM responds (Tier 2+ projects; CEO responds for Tier 1)**
```sql
-- Guard: client_subtype = 'TECH_TEAM' for Tier 2+ projects
UPDATE spec_clarifications
  SET response_text = 'Yes, 5,000 labeled examples from 2024 audit cycle.',
      responded_by = tech_user_id,  -- FK to users.id; records who answered
      status = 'ANSWERED'
  WHERE id = ? AND status = 'OPEN';
```

**Step [A3] — Expert marks resolved**
```sql
UPDATE spec_clarifications SET status = 'RESOLVED', resolved_at = now() WHERE id = ?;
```

### Bid Submission + Version 1

**Step [B1] — INSERT engagements (creates the bid context)**
```sql
INSERT INTO engagements (
  project_id,
  expert_id,
  service_id,   -- NULL for PROJECT_BASED
  type,         -- 'PROJECT_BASED'
  state         -- 'PENDING' (connection not yet accepted)
) VALUES (project_id, expert_user_id, NULL, 'PROJECT_BASED', 'PENDING')
RETURNING id AS engagement_id;
-- CONSTRAINT engagement_type_fk_consistency enforced: PROJECT_BASED → project_id NOT NULL, service_id NULL
```

**Step [B2] — INSERT capability_bids (version 1 = original submission)**
```sql
INSERT INTO capability_bids (
  engagement_id,   -- UNIQUE: 1:1 with engagement
  footprint_alignment_json,  -- Component 1
  approach_summary,           -- Component 2
  conditional_pricing_json,   -- Component 3: must be structured { condition, price_range_min_vnd, price_range_max_vnd, trigger_description }
  state,            -- 'SUBMITTED'
  version_number    -- 1
) VALUES (engagement_id, '{"domains":...}', 'My approach...', '[{"condition":"Avro payload","price_range_min_vnd":500000,...}]', 'SUBMITTED', 1);

-- Write version 1 snapshot (revision_request_id = NULL for original)
INSERT INTO bid_versions (bid_id, revision_request_id, version_number, snapshot_json, reviser_id)
  VALUES (bid_id, NULL, 1, '{"footprint_alignment_json":...}', expert_user_id);
```

### Surface B — TECH_TEAM Bid Revision Loop

**Step [B3] — TECH_TEAM flags component for revision**
```sql
UPDATE capability_bids SET state = 'REVISION_REQUESTED' WHERE id = bid_id;

INSERT INTO bid_revision_requests (
  bid_id,
  flagged_component,  -- 'APPROACH' | 'FOOTPRINT' | 'PRICING'
  reason,
  requested_by,       -- tech_user_id
  requested_at
) VALUES (bid_id, 'APPROACH', 'Please address Phase 0 milestone explicitly.', tech_user_id, now())
RETURNING id AS revision_request_id;
```

**Step [B4] — Expert revises flagged component only → new bid_versions row**
```sql
BEGIN;
UPDATE capability_bids
  SET approach_summary = 'Updated: Phase 0 covered by dedicating first sprint to ground truth audit...',
      state = 'REVISED',
      version_number = version_number + 1  -- increment version counter
  WHERE id = bid_id;

-- Immutable snapshot linking to the request that caused it
INSERT INTO bid_versions (bid_id, revision_request_id, version_number, snapshot_json, reviser_id, revised_at)
  VALUES (bid_id, revision_request_id, 2,
          '{"approach_summary":"Updated: Phase 0 covered...","footprint_alignment_json":...}',
          expert_user_id, now());
COMMIT;
```

**bid_versions.revision_request_id FK is essential for audit:** Links each version snapshot to the specific `bid_revision_requests` row that caused it. Without this FK, you cannot answer "which revision request produced version N?" from the DB alone.

### Surface C — Price Negotiation (max 2 rounds)

```sql
-- CEO proposes (round 1):
INSERT INTO price_negotiations (bid_id, round_number, proposer, proposed_pricing_json, response, message)
  VALUES (bid_id, 1, 'CEO', '[{"milestone":0,"proposed_amount_vnd":8000000}]', 'PENDING', 'Can we reduce Phase 0?');
-- UNIQUE (bid_id, round_number) — prevents >2 rounds at DB level
-- CHECK (round_number IN (1,2)) — hard DB cap

-- Expert responds (COUNTER):
UPDATE price_negotiations
  SET response = 'COUNTERED',
      counter_pricing_json = '[{"milestone":0,"counter_amount_vnd":9000000}]',
      message = 'Phase 0 requires 3 days minimum; 9M is final offer.'
  WHERE bid_id = bid_id AND round_number = 1;

UPDATE capability_bids SET state = 'NEGOTIATION_COMPLETE' WHERE id = bid_id;
```

### Surface D — CEO Override of TECH_DISAPPROVED

```sql
-- Guard: bid.state = 'TECH_DISAPPROVED' AND active_role = 'CLIENT' AND client_subtype = 'CEO'
INSERT INTO bid_conflict_overrides (
  bid_id,           -- UNIQUE: one override per bid
  ceo_user_id,      -- explicit FK; used for RQ3 correlation (CEO override → engagement failure rate)
  override_reason,  -- CHECK: char_length >= 50
  acknowledged_risks,
  overridden_at
) VALUES (bid_id, ceo_user_id, 'TECH_TEAM flagged pricing concern but expert credentials verified externally by CEO.', 'Project may require extra budget for Phase 0.', now());

UPDATE capability_bids SET state = 'CONFLICT_RESOLVED_CEO_OVERRIDE' WHERE id = bid_id;
-- CEO_REVIEW unlocks
```

```sql
INSERT INTO platform_decisions (decision_type, entity_type, entity_id, decision, advisory_note)
  VALUES ('CRITERION_QUALITY_GATE', 'capability_bid', bid_id, 'CEO_OVERRIDE',
          'CEO overrode TECH_DISAPPROVED: see bid_conflict_overrides for reason');
```

### CEO Selection → Connection

**Step [C1] — CEO selects expert → connection request sent**
```sql
UPDATE capability_bids SET state = 'SELECTED' WHERE id = bid_id;
-- engagement already exists from B1; just update state
UPDATE engagements SET state = 'PENDING' WHERE id = engagement_id;  -- connection request sent
```

**Step [C2] — Expert accepts → CONNECTED; both NDA timestamps set**
```sql
-- Expert NDA click-through:
UPDATE engagements SET expert_nda_accepted_at = now() WHERE id = engagement_id;
-- CEO NDA click-through:
UPDATE engagements SET client_nda_accepted_at = now() WHERE id = engagement_id;

-- Both timestamps set → advance to CONNECTED
UPDATE engagements
  SET state = 'CONNECTED',
      connected_at = now()
  WHERE id = engagement_id
    AND client_nda_accepted_at IS NOT NULL
    AND expert_nda_accepted_at IS NOT NULL;
```

**Artifact B access gate (FastAPI SQL):**
```sql
-- Checked on every Artifact B request from EXPERT or TECH_TEAM:
SELECT ab.* FROM artifact_b ab
  WHERE ab.project_id = ?
    AND EXISTS (
      SELECT 1 FROM engagements e
      WHERE e.project_id = ab.project_id
        AND e.state >= 'CONNECTED'  -- text comparison works because states are ordered by lifecycle
        AND e.client_nda_accepted_at IS NOT NULL
        AND e.expert_nda_accepted_at IS NOT NULL
        AND (e.expert_id = :current_user_id
             OR EXISTS (SELECT 1 FROM tech_team_profiles ttp
                        WHERE ttp.user_id = :current_user_id AND ttp.linked_project_id = ab.project_id))
    );
-- CEO path: no branch exists → CEO can never see Artifact B
```

**State transition:** `PENDING → CONNECTED` (both NDA timestamps required)

### End-State Trace — MF-6

| Table | Key state |
|---|---|
| `engagements` | `state='CONNECTED'`, both NDA timestamps set |
| `capability_bids` | `state='SELECTED'`, `version_number=2` |
| `bid_versions` | 2 rows: v1 (revision_request_id=NULL), v2 (revision_request_id=req_id) |
| `bid_revision_requests` | 1 row: flagged_component='APPROACH' |
| `price_negotiations` | 1 row: round=1, response='COUNTERED' |
| `spec_clarifications` | `status='RESOLVED'`, `responded_by=tech_user_id` |
| `bid_conflict_overrides` | 0 or 1 row (if Surface D was triggered) |

---

## MF-7: Milestone Management & Escrow Payment

### Master Reference Anchors
- **0.6 Milestone States:** `DEFINED → AWAITING_PAYMENT → FUNDED → IN_PROGRESS → SUBMITTED → IN_REVISION → APPROVED → RELEASED`
- **0.6 DoD Checklist States:** `PENDING → COMPLETED` (required items must be COMPLETED before SUBMITTED)
- **0.6 Sprint Status:** `ON_TRACK | DEVIATION | BLOCKER` (BLOCKER pauses review clock via `review_clock_paused_at`)
- **0.8 Payment Architecture:** VA-per-milestone with fixed_amount; escrow_accounts holds locked funds

### Pre-conditions
- `engagements.state = 'CONNECTED'`
- `wallets.available_balance ≥ milestone.payment_amount_vnd` (CEO wallet)
- `users.sepay_bank_account_xid IS NOT NULL` for expert (Bank Hub linking complete)

### Layer 1 — Milestone (Payment Contract)

**Step [ML1] — INSERT milestones + acceptance_criteria**
```sql
INSERT INTO milestones (
  engagement_id, milestone_number,
  deliverable_statement,  -- "Production-ready LLM decision pipeline..."
  sign_off_authority,     -- 'JOINT' (both CEO and TECH_TEAM must approve)
  payment_amount_vnd,     -- 10000000 (10M VND; locked from bid pricing)
  state                   -- 'DEFINED'
) VALUES (engagement_id, 1, 'Production-ready LLM decision pipeline...', 'JOINT', 10000000, 'DEFINED')
RETURNING id AS milestone_id;

-- LLM criterion quality check runs on each criterion_text
-- (FastAPI flags subjective language like "client is satisfied")
INSERT INTO acceptance_criteria (milestone_id, criterion_text, is_required, verified_by_role)
  VALUES
  (milestone_id, 'Pipeline processes 600K ads/month at p95 latency < 2 seconds', TRUE, 'TECH_TEAM'),
  (milestone_id, 'Ground truth precision ≥ 0.85 on holdout set', TRUE, 'TECH_TEAM'),
  (milestone_id, 'Documentation delivered to client repository', TRUE, 'CEO');
```

### CEO Funds Milestone — Per-Milestone VA Flow

**Step [ML2] — Generate MILESTONE Virtual Account (fixed_amount)**
```sql
BEGIN;
-- Create per-milestone VA via SePay Bank Hub API
-- fixed_amount = payment_amount_vnd (bank rejects transfers of wrong amount)
INSERT INTO virtual_accounts (
  entity_type,   -- 'MILESTONE'
  entity_id,     -- milestone_id
  va_number,     -- from SePay API response
  fixed_amount,  -- 10000000 (bank rejects if scan amount differs)
  expires_at,    -- now() + INTERVAL '24 hours'
  status         -- 'ACTIVE'
) VALUES ('MILESTONE', milestone_id, ?, 10000000, now() + INTERVAL '24 hours', 'ACTIVE');

UPDATE milestones SET state = 'AWAITING_PAYMENT', va_number = ?, va_expires_at = now() + INTERVAL '24 hours'
  WHERE id = milestone_id;
COMMIT;
```

**Step [ML3] — SePay IPN fires on MILESTONE VA credit**
```sql
-- IPN payload: { va_number: "...", amount: 10000000, transfer_type: "in" }
-- IPN handler: look up virtual_accounts by va_number; entity_type = 'MILESTONE'

BEGIN;
-- Validate: IPN amount must exactly match va.fixed_amount
-- Idempotency: unique index on wallet_transactions(wallet_id, reference_id) prevents double-credit

-- Lock client wallet funds
UPDATE wallets
  SET available_balance = available_balance - 10000000,
      locked_balance = locked_balance + 10000000
  WHERE user_id = ceo_user_id;

INSERT INTO wallet_transactions (wallet_id, amount, transaction_type, reference_id, created_at)
  VALUES (ceo_wallet_id, 10000000, 'ESCROW_LOCK', 'ESC_LOCK:' || milestone_id::text, now());

-- Create escrow record (PROJECT_BASED path: milestone_id set, engagement_id NULL)
INSERT INTO escrow_accounts (
  milestone_id,     -- set (PROJECT_BASED)
  engagement_id,    -- NULL
  amount,           -- 10000000
  client_wallet_id,
  expert_wallet_id,
  status,           -- 'HELD'
  held_at
) VALUES (milestone_id, NULL, 10000000, ceo_wallet_id, expert_wallet_id, 'HELD', now());

UPDATE milestones SET state = 'FUNDED', funded_at = now(), va_number = NULL WHERE id = milestone_id;
UPDATE virtual_accounts SET status = 'USED' WHERE entity_id = milestone_id::text AND entity_type = 'MILESTONE';

-- Release pay-gated documents staged for THIS milestone
UPDATE paygated_documents
  SET release_state = 'RELEASED', released_at = now()
  WHERE milestone_id = milestone_id AND release_state = 'STAGED';

COMMIT;
-- Auto-advance to IN_PROGRESS
UPDATE milestones SET state = 'IN_PROGRESS' WHERE id = milestone_id;
```

**State transition:** `AWAITING_PAYMENT → FUNDED → IN_PROGRESS`

### Layer 2 — DoD Checklist (Expert Self-Check)

**Step [ML4] — Expert creates DoD items after milestone reaches FUNDED**
```sql
INSERT INTO milestone_dod_items (
  milestone_id,
  item_description,
  is_required,
  status,       -- 'PENDING'
  maps_to_criterion_id  -- optional upward link to acceptance_criteria
) VALUES
  (milestone_id, 'Ground truth dataset validated (no nulls, no duplicates)', TRUE, 'PENDING', criterion_id_gt),
  (milestone_id, 'Evaluation script tested on 100-item holdout sample', TRUE, 'PENDING', criterion_id_gt),
  (milestone_id, 'Load test completed at 600K/month simulation', TRUE, 'PENDING', criterion_id_latency),
  (milestone_id, 'Code coverage > 80%', FALSE, 'PENDING', NULL);  -- non-required, can be NOT_APPLICABLE
-- CONSTRAINT dod_required_cannot_be_na: is_required=TRUE AND status='NOT_APPLICABLE' → DB REJECT
```

**Step [ML5] — TECH_TEAM leaves comments per DoD item (separate from messages channel)**
```sql
INSERT INTO dod_item_comments (dod_item_id, commenter_id, comment_text, created_at)
  VALUES (dod_item_id_gt, tech_user_id, 'Use the anonymised dataset, not the raw export.', now());
-- Route guard: WRITE = TECH_TEAM only (client_subtype='TECH_TEAM'); READ = TECH_TEAM + EXPERT; CEO hidden
```

**Step [ML6] — Expert marks DoD items COMPLETED**
```sql
UPDATE milestone_dod_items
  SET status = 'COMPLETED',
      completed_at = now(),
      completion_note = 'Ground truth: 5,423 rows validated with zero nulls via script output attached.'
  WHERE id = dod_item_id_gt AND is_required = TRUE;
-- NOT_APPLICABLE only valid when is_required = FALSE (enforced by CHECK constraint)
```

### Layer 3 — Sprint Tracking (Progress + Scope Evolution)

**Step [ML7] — Expert creates sprint plan**
```sql
INSERT INTO milestone_sprints (
  milestone_id, sprint_number, created_by, week_range, deliverable_checkpoint, tasks
) VALUES
  (milestone_id, 1, expert_user_id, 'Week 5-6', 'Ground truth import + evaluation script',
   '[{"id":"t1","description":"Import ground truth","status":"TODO"},{"id":"t2","description":"Write eval script","status":"TODO"}]');
-- Route guard: created_by must have active_role = 'EXPERT'
-- UNIQUE (milestone_id, sprint_number) — prevents duplicate sprints
```

**Step [ML8] — TECH_TEAM leaves sprint comments**
```sql
INSERT INTO sprint_comments (milestone_sprint_id, commenter_id, comment_text, created_at)
  VALUES (sprint_id, tech_user_id, 'Confirm which data pipeline version to target for Sprint 2.', now());
-- Route guard: WRITE = TECH_TEAM only; READ = TECH_TEAM + EXPERT; CEO shows summary only
```

**Step [ML9] — Expert submits weekly sprint status (BLOCKER example)**
```sql
INSERT INTO sprint_status_updates (
  milestone_id,
  milestone_sprint_id,   -- direct FK (not implicit via sprint_number — prevents orphan updates)
  sprint_number,         -- denormalized for display
  submitted_by,
  status,                -- 'BLOCKER'
  blocker_type,          -- 'TECH_CONSTRAINT'
  blocker_description,   -- "Ground truth has 30% label inconsistency..."
  estimated_resolution,
  scope_evolution_flag   -- FALSE (not a scope evolution; just a technical blocker)
) VALUES (milestone_id, sprint_id, 1, expert_user_id, 'BLOCKER',
          'TECH_CONSTRAINT', 'Ground truth labels have 30% inconsistency. Need corrected labels from TECH_TEAM.', '3 business days', FALSE);

-- Pause review clock
UPDATE milestones
  SET review_clock_paused_at = now()
  WHERE id = milestone_id AND review_clock_paused_at IS NULL;
```

**Step [ML10] — TECH_TEAM resolves blocker**
```sql
UPDATE sprint_status_updates SET resolved_at = now(), resolved_by = tech_user_id WHERE id = ?;

-- Accumulate paused time and resume clock
UPDATE milestones
  SET total_paused_seconds = total_paused_seconds + EXTRACT(EPOCH FROM (now() - review_clock_paused_at))::INT,
      review_clock_paused_at = NULL
  WHERE id = milestone_id;
```

### Milestone Submission + Sign-Off

**Step [ML11] — Expert submits deliverable (DoD gate enforced)**
```sql
-- Gate check: all is_required=TRUE DoD items must have status='COMPLETED'
-- Returns 422 if any required items are still PENDING
SELECT COUNT(*) FROM milestone_dod_items
  WHERE milestone_id = ? AND is_required = TRUE AND status != 'COMPLETED';
-- → if count > 0: 422 { code: "DOD_INCOMPLETE", unchecked_items: [...] }

INSERT INTO milestone_submissions (milestone_id, expert_id, description, files_json, submitted_at)
  VALUES (milestone_id, expert_user_id, 'All criteria met. Evaluation script output attached.', '["s3://...eval_output.json"]', now());

UPDATE milestones SET state = 'SUBMITTED', submitted_at = now() WHERE id = milestone_id;
```

**Step [ML12] — Sign-off authority verifies criteria one by one**
```sql
-- JOINT milestone: TECH_TEAM verifies technical criteria
-- Route guard: verified_by_role matches caller's role
UPDATE acceptance_criteria
  SET verified_at = now()
  WHERE id = criterion_id_latency AND verified_by_role = 'TECH_TEAM';

-- CEO verifies business criteria
UPDATE acceptance_criteria
  SET verified_at = now()
  WHERE id = criterion_id_docs AND verified_by_role = 'CEO';
```

**Step [ML13] — APPROVED transition (criteria gate enforced + atomic ledger)**
```sql
-- Gate check: all is_required=TRUE acceptance criteria must have verified_at set
SELECT COUNT(*) FROM acceptance_criteria
  WHERE milestone_id = ? AND is_required = TRUE AND verified_at IS NULL;
-- → if count > 0: 422 { code: "CRITERIA_UNVERIFIED", unverified_criteria: [...] }

-- Read platform fee from DB (never hardcoded):
SELECT platform_fee_pct, platform_wallet_id FROM platform_settings LIMIT 1;
-- platform_fee_pct = 0.05, platform_wallet_id = 'uuid-of-platform-wallet'

BEGIN;
-- Release escrow: client locked_balance decremented
UPDATE wallets SET locked_balance = locked_balance - 10000000
  WHERE user_id = ceo_user_id;

UPDATE escrow_accounts SET status = 'RELEASED', released_at = now()
  WHERE milestone_id = milestone_id;

-- Credit expert: net of 5% platform fee
-- Fee reads from platform_settings — never hardcoded 0.05
INSERT INTO wallet_transactions (wallet_id, amount, transaction_type, reference_id)
  VALUES (expert_wallet_id, 9500000, 'ESCROW_RELEASE', 'ESC_RELEASE:' || milestone_id::text);
UPDATE wallets SET available_balance = available_balance + 9500000 WHERE user_id = expert_user_id;

-- Platform fee to seeded platform wallet
INSERT INTO wallet_transactions (wallet_id, amount, transaction_type, reference_id)
  VALUES (platform_wallet_id, 500000, 'PLATFORM_FEE', 'PLATFORM_FEE:' || milestone_id::text);
UPDATE wallets SET available_balance = available_balance + 500000 WHERE user_id = platform_system_user_id;

UPDATE milestones SET state = 'APPROVED', approved_at = now() WHERE id = milestone_id;
COMMIT;

-- chi hộ API called for expert (MILESTONE_RELEASE type withdrawal):
INSERT INTO withdrawal_requests (expert_id, type, amount, bank_account_xid, status, requested_at)
  VALUES (expert_user_id, 'MILESTONE_RELEASE', 9500000, expert_bank_account_xid, 'PENDING', now());
-- POST SePay chi hộ API → SePay credit IPN fires → withdrawal.status = 'COMPLETED'
-- UPDATE milestones SET state = 'RELEASED', released_at = now() on chi hộ confirmation
```

### End-State Trace — MF-7

| Table | Field | Final value |
|---|---|---|
| `milestones` | `state` | `RELEASED` |
| `milestones` | `total_paused_seconds` | accumulated BLOCKER pause duration |
| `wallets` (CEO) | `locked_balance` | 0 (escrow released) |
| `wallets` (Expert) | `available_balance` | `+9,500,000` |
| `escrow_accounts` | `status` | `RELEASED` |
| `withdrawal_requests` | `status` | `COMPLETED` |
| `wallet_transactions` | types | `ESCROW_LOCK` + `ESCROW_RELEASE` + `PLATFORM_FEE` |
| `milestone_dod_items` | `status` | all `COMPLETED` |
| `acceptance_criteria` | `verified_at` | all set |

---

## MF-8: Dispute Resolution (3 Layers)

### Master Reference Anchors
- **0.6 Dispute States:** `disputes.state`: `PENDING → LAYER_1_EVAL → AUTO_RESOLVED | LAYER_2_MUTUAL → MUTUAL_RESOLVED | LAYER_3_FORCED → DISPUTED_AUTO_RESOLVED`
- **0.6 Escrow freeze:** `escrow_accounts.status → 'FROZEN'` (no balance move on dispute filing)
- **0.7 RBAC:** CEO, TECH_TEAM, or EXPERT can all file disputes

### Pre-conditions
- `milestones.state IN ('SUBMITTED', 'IN_REVISION')`
- Active `escrow_accounts` row for this milestone with `status = 'HELD'`

### Step [D1] — Dispute filed → escrow frozen

```sql
BEGIN;
-- Freeze escrow immediately (no balance movement — just status change)
UPDATE escrow_accounts SET status = 'FROZEN'
  WHERE milestone_id = ? AND status = 'HELD';

-- Create dispute record
INSERT INTO disputes (
  engagement_id,
  milestone_id,
  criterion_id,        -- dispute always against a specific criterion (the objective contract element)
  escrow_account_id,   -- direct FK — makes freeze/release O(1) and type-agnostic
  filed_by,            -- any of: CEO, TECH_TEAM, EXPERT
  state                -- 'LAYER_1_EVAL'
) VALUES (engagement_id, milestone_id, criterion_id, escrow_id, filer_user_id, 'LAYER_1_EVAL');
COMMIT;
```

**`disputes.escrow_account_id` FK is critical:** Without this direct FK, the dispute handler would need to JOIN through milestone → escrow to find and freeze the correct escrow record. With the FK, it's a single-row UPDATE by escrow_account_id — and it works identically for `PROJECT_BASED` (milestone-linked escrow) and `SERVICE_PURCHASE` (engagement-linked escrow) without any branching logic.

### Step [D2] — Layer 1: LLM criterion evaluation (FastAPI)

FastAPI evaluates the deliverable against the specific criterion that was disputed. Returns confidence score.

**Branch A — confidence ≥ 0.80: AUTO_RESOLVED**
```sql
-- Expert wins example:
BEGIN;
UPDATE escrow_accounts SET status = 'RELEASED', released_at = now() WHERE id = escrow_id;

-- Release expert's share
UPDATE wallets SET available_balance = available_balance + 9500000 WHERE user_id = expert_user_id;
INSERT INTO wallet_transactions (wallet_id, amount, transaction_type, reference_id)
  VALUES (expert_wallet_id, 9500000, 'ESCROW_RELEASE', 'DISPUTE_RELEASE:' || dispute_id::text);

-- Return client's locked funds
UPDATE wallets SET locked_balance = locked_balance - 10000000 WHERE user_id = ceo_user_id;

-- Platform fee still applies
UPDATE wallets SET available_balance = available_balance + 500000 WHERE user_id = platform_user_id;
INSERT INTO wallet_transactions (wallet_id, amount, transaction_type, reference_id)
  VALUES (platform_wallet_id, 500000, 'PLATFORM_FEE', 'DISPUTE_FEE:' || dispute_id::text);

UPDATE disputes SET state = 'AUTO_RESOLVED', resolution_layer = 1, llm_confidence = 0.85, resolved_at = now()
  WHERE id = dispute_id;
UPDATE milestones SET state = 'RELEASED', released_at = now() WHERE id = milestone_id;
COMMIT;

-- FastAPI writes audit:
INSERT INTO platform_decisions (decision_type, entity_type, entity_id, llm_confidence, decision)
  VALUES ('DISPUTE_L1_EVAL', 'acceptance_criterion', criterion_id, 0.85, 'AUTO_RESOLVED_EXPERT_WINS');
```

**Branch B — confidence < 0.80: move to Layer 2**
```sql
UPDATE disputes
  SET state = 'LAYER_2_MUTUAL',
      llm_confidence = 0.61,
      layer2_deadline = now() + INTERVAL '48 hours'  -- cooling window
  WHERE id = dispute_id;
```

### Step [D3] — Layer 2: Mutual Agreement (48-hour window)

```sql
-- If both parties agree same option (e.g., partial payment 70/30 split):
BEGIN;
UPDATE escrow_accounts SET status = 'RELEASED', released_at = now() WHERE id = escrow_id;
-- Expert receives 70%: 7,000,000
UPDATE wallets SET available_balance = available_balance + 7000000 WHERE user_id = expert_user_id;
INSERT INTO wallet_transactions (wallet_id, amount, transaction_type, reference_id)
  VALUES (expert_wallet_id, 7000000, 'ESCROW_RELEASE', 'MUTUAL_RELEASE:' || dispute_id::text);
-- Client refunded 30% back to available: 3,000,000
UPDATE wallets
  SET locked_balance = locked_balance - 10000000,
      available_balance = available_balance + 3000000
  WHERE user_id = ceo_user_id;
INSERT INTO wallet_transactions (wallet_id, amount, transaction_type, reference_id)
  VALUES (ceo_wallet_id, 3000000, 'ESCROW_REFUND', 'MUTUAL_REFUND:' || dispute_id::text);

UPDATE disputes SET state = 'MUTUAL_RESOLVED', resolution_layer = 2, resolved_at = now() WHERE id = dispute_id;
COMMIT;
```

### Step [D4] — Layer 3: Forced 50/50 Split (48h window expires)

```sql
BEGIN;
UPDATE escrow_accounts SET status = 'SPLIT', released_at = now() WHERE id = escrow_id;

-- Client: locked → available (50%)
UPDATE wallets
  SET locked_balance = locked_balance - 10000000,
      available_balance = available_balance + 5000000
  WHERE user_id = ceo_user_id;
INSERT INTO wallet_transactions (wallet_id, amount, transaction_type, reference_id)
  VALUES (ceo_wallet_id, 5000000, 'ESCROW_SPLIT', 'SPLIT:' || dispute_id::text);

-- Expert: 50% (minus platform fee)
UPDATE wallets SET available_balance = available_balance + 4750000 WHERE user_id = expert_user_id;
INSERT INTO wallet_transactions (wallet_id, amount, transaction_type, reference_id)
  VALUES (expert_wallet_id, 4750000, 'ESCROW_SPLIT', 'SPLIT:' || dispute_id::text);

-- Platform fee on expert's share
UPDATE wallets SET available_balance = available_balance + 250000 WHERE user_id = platform_user_id;
INSERT INTO wallet_transactions (wallet_id, amount, transaction_type, reference_id)
  VALUES (platform_wallet_id, 250000, 'PLATFORM_FEE', 'SPLIT_FEE:' || dispute_id::text);

UPDATE disputes SET state = 'DISPUTED_AUTO_RESOLVED', resolution_layer = 3, resolved_at = now()
  WHERE id = dispute_id;
UPDATE milestones SET state = 'RELEASED', released_at = now() WHERE id = milestone_id;
COMMIT;
-- chi hộ API fires for expert's 4,750,000 VND share
```

**Step [D5] — "Report this resolution" (informational only)**
```sql
INSERT INTO dispute_resolution_reports (dispute_id, reported_by, reason_text, created_at)
  VALUES (dispute_id, filer_user_id, 'LLM evaluated incorrectly — deliverable clearly met criterion 2.', now());
-- No action required from admin. Logged in Platform Integrity Monitor for product improvement.
```

---

# GROUP 3 — Path B: Service Marketplace Flow

---

## MF-9: Expert Service Publishing (Path B)

### Master Reference Anchors
- **0.7 RBAC:** `EXPERT` with `subscription_expert_tier = 'pro'` required for AI Service Generator
- **0.6 Engagement Types:** `SERVICE_PURCHASE` and `TECH_DISCOVERY` — no elicitation, no bid

### Step [SP1] — AI Service Generator (FastAPI LLM)

FastAPI generates service listing draft from expert's raw description + profile context:
```sql
-- Expert must be Pro-subscribed for AI Service Generator (subscription guard applies)
-- After LLM generates draft:
INSERT INTO services (
  expert_id,
  title,         -- "End-to-End Legal RAG System — 14 days"
  description,   -- LLM-generated
  domains_json,  -- '[{"code":"DOMAIN_A"},{"code":"DOMAIN_D"}]'
  seams_json,    -- '[{"code":"SEAM_AD"}]'
  price_vnd,     -- 15000000
  state,         -- 'DRAFT'
  service_type   -- 'AI_SERVICE' | 'TECH_DISCOVERY'
) VALUES (expert_user_id, ?, ?, '...', '...', 15000000, 'DRAFT', 'AI_SERVICE')
RETURNING id AS service_id;
```

### Step [SP2] — Expert edits and publishes

```sql
-- Pre-publish validation:
-- 1. title non-empty, price > 0, description non-empty (application-level checks)
-- 2. Expert's bank account linked (required for chi hộ payout):
SELECT sepay_bank_account_xid FROM users WHERE id = expert_user_id;
-- → NULL: return 422 { code: "BANK_ACCOUNT_REQUIRED", link_url: "/settings/bank" }

UPDATE services SET state = 'PUBLISHED' WHERE id = service_id AND expert_id = expert_user_id;
```

---

## MF-10: Client Buys Service / Tech Discovery (Path B)

### Master Reference Anchors
- **0.6 Engagement Type:** `SERVICE_PURCHASE` — `project_id NULL`, `service_id NOT NULL`, created in `ACTIVE` state
- **0.6 Escrow:** Per-service-order VA with fixed_amount; `escrow_accounts.engagement_id` set (not milestone_id)
- **0.9 Subscription:** SERVICE_PURCHASE available to FREE-tier clients (no LLM in purchase path)

### Step [SB1] — CEO clicks "Buy Service"

```sql
BEGIN;
-- Create the engagement FIRST (needed as escrow entity_id)
INSERT INTO engagements (project_id, expert_id, service_id, type, state)
  VALUES (NULL, expert_user_id, service_id, 'SERVICE_PURCHASE', 'PENDING')
  -- CONSTRAINT: type='SERVICE_PURCHASE' → project_id must be NULL, service_id must NOT NULL
RETURNING id AS engagement_id;

-- Create per-service-order VA (24h expiry, fixed amount = service.price_vnd)
INSERT INTO virtual_accounts (entity_type, entity_id, va_number, fixed_amount, expires_at, status)
  VALUES ('SERVICE', engagement_id, ?, services.price_vnd, now() + INTERVAL '24 hours', 'ACTIVE');
COMMIT;
-- Return VietQR to CEO
```

### Step [SB2] — SePay IPN fires on SERVICE VA credit

```sql
-- IPN: entity_type='SERVICE', entity_id=engagement_id

BEGIN;
-- Lock client wallet
UPDATE wallets
  SET available_balance = available_balance - 15000000,
      locked_balance = locked_balance + 15000000
  WHERE user_id = ceo_user_id;
INSERT INTO wallet_transactions (wallet_id, amount, transaction_type, reference_id)
  VALUES (ceo_wallet_id, 15000000, 'ESCROW_LOCK', 'SVC_LOCK:' || engagement_id::text);

-- Service escrow uses engagement_id (not milestone_id)
INSERT INTO escrow_accounts (
  milestone_id,     -- NULL (SERVICE_PURCHASE path)
  engagement_id,    -- engagement_id (set)
  amount, client_wallet_id, expert_wallet_id, status
) VALUES (NULL, engagement_id, 15000000, ceo_wallet_id, expert_wallet_id, 'HELD');
-- CONSTRAINT escrow_has_one_parent: (NULL, engagement_id) passes; (NULL, NULL) would fail

UPDATE engagements SET state = 'ACTIVE' WHERE id = engagement_id;
-- SERVICE_PURCHASE skips PENDING → created directly in ACTIVE
UPDATE virtual_accounts SET status = 'USED' WHERE entity_id = engagement_id::text AND entity_type = 'SERVICE';
COMMIT;
```

**The `escrow_has_one_parent` CHECK constraint is what makes MF-10 safe:** it prevents an escrow record from being created with both `milestone_id` and `engagement_id` set, or both NULL. Without it, the dispute freeze logic (which joins `disputes.escrow_account_id → escrow_accounts`) would need to know which path it was on to find the right record.

---

# GROUP 4 — Financial Flows

---

## MF-11: Wallet Top-Up (SePay IPN)

### Master Reference Anchors
- **0.8 Payment Architecture:** Inbound leg; WALLET_TOPUP VA; IPN idempotency via unique index
- **0.6 Wallet Transaction Types:** `TOP_UP` ledger entry

This flow is fully detailed in MF-1 Phase B above. Key points:

**Idempotency constraint (prevents double-credit on SePay retry):**
```sql
-- This unique partial index at the DB level — not application logic alone:
CREATE UNIQUE INDEX wallet_tx_idempotency
  ON wallet_transactions(wallet_id, reference_id)
  WHERE reference_id IS NOT NULL;
-- reference_id = 'TOP_UP:{va_number}:{transfer_ref}'
-- Second IPN retry with same reference_id → unique constraint violation → caught and ignored safely
```

**Tables touched:** `virtual_accounts` (SELECT by va_number), `wallets` (UPDATE), `wallet_transactions` (INSERT)

---

## MF-12: Expert Withdrawal (Automated Chi Hộ)

### Master Reference Anchors
- **0.8 Payment Architecture:** Outbound leg; chi hộ API; bank_account_xid from Bank Hub
- **0.6 Withdrawal States:** `PENDING → PROCESSING → COMPLETED | FAILED`
- **0.7 RBAC:** Only `EXPERT` active_role can request withdrawal

### Pre-conditions
- `users.sepay_bank_account_xid IS NOT NULL` (Bank Hub Hosted Link completed in MF-2 Phase F)
- `wallets.available_balance ≥ requested_amount`

**Step [W1] — Atomic debit + withdrawal record (debit-first pattern prevents double-spend)**
```sql
BEGIN;
-- Guard: available_balance >= amount
SELECT available_balance FROM wallets WHERE user_id = expert_user_id FOR UPDATE;
-- → if insufficient: ROLLBACK; return 422

UPDATE wallets
  SET available_balance = available_balance - 9500000
  WHERE user_id = expert_user_id;

INSERT INTO withdrawal_requests (
  expert_id,
  type,              -- 'EXPERT_MANUAL' (user-initiated) vs 'MILESTONE_RELEASE' (auto on approval)
  amount,            -- 9500000
  bank_account_xid,  -- from users.sepay_bank_account_xid (bank-verified, not self-reported)
  disbursement_id,   -- NULL until chi hộ API responds
  status,            -- 'PENDING'
  requested_at
) VALUES (expert_user_id, 'EXPERT_MANUAL', 9500000, expert_bank_account_xid, NULL, 'PENDING', now())
RETURNING id AS withdrawal_id;

INSERT INTO wallet_transactions (wallet_id, amount, transaction_type, reference_id)
  VALUES (expert_wallet_id, 9500000, 'WITHDRAWAL', 'WITHDRAWAL:' || withdrawal_id::text);
COMMIT;
```

**Step [W2] — Call SePay chi hộ API (async, after commit)**
```
POST SePay chi hộ API: { amount: 9500000, bank_account_xid: "...", reference: "WD-{withdrawal_id}" }
→ Response: { disbursement_id: "SEPAY_DISB_XXX", status: "PROCESSING" }
```
```sql
UPDATE withdrawal_requests
  SET disbursement_id = 'SEPAY_DISB_XXX', status = 'PROCESSING'
  WHERE id = withdrawal_id;
```

**Step [W3] — SePay credit IPN fires (expert's bank receives funds)**
```sql
UPDATE withdrawal_requests
  SET status = 'COMPLETED', confirmed_at = now()
  WHERE id = withdrawal_id;
-- Notify expert: "Withdrawal of 9,500,000 VND completed"
```

**Step [W4] — chi hộ API failure → atomic wallet restore**
```sql
BEGIN;
UPDATE wallets SET available_balance = available_balance + 9500000 WHERE user_id = expert_user_id;
UPDATE withdrawal_requests SET status = 'FAILED' WHERE id = withdrawal_id;
-- Reverse the WITHDRAWAL ledger entry:
INSERT INTO wallet_transactions (wallet_id, amount, transaction_type, reference_id)
  VALUES (expert_wallet_id, 9500000, 'WITHDRAWAL', 'WITHDRAWAL_FAILED:' || withdrawal_id::text);
COMMIT;
-- Notify expert: "Withdrawal failed. Balance restored."
```

**`withdrawal_requests.type` distinction is essential for earnings dashboard:** MILESTONE_RELEASE rows (auto-created on milestone APPROVED) are shown separately from EXPERT_MANUAL rows (user-initiated cash-outs), giving the expert a clear audit trail of what was earned vs what was manually withdrawn.

---

## MF-13: Subscription Purchase from Wallet

This flow is fully covered in MF-1 Phase C. Key differences for Expert Pro:

```sql
-- Same two-step PENDING_PAYMENT → ACTIVE pattern, different values:
INSERT INTO user_subscriptions (user_id, role_type, tier, status, price_paid)
  VALUES (expert_user_id, 'expert', 'pro', 'PENDING_PAYMENT', 300000);
-- VA: entity_type='SUBSCRIPTION', fixed_amount=300000
-- On IPN: users.subscription_expert_tier='pro', sub_expert_expires_at=now()+6months
```

**Tables touched:** `user_subscriptions` (INSERT + UPDATE), `wallets` (UPDATE), `wallet_transactions` (INSERT), `virtual_accounts` (INSERT + UPDATE status='USED'), `users` (UPDATE subscription fields)

---

# GROUP 5 — Advanced / Edge Flows

---

## MF-14: Add-On Phase Protocol (Scope Evolution)

### Master Reference Anchors
- **0.6 Sprint Status:** `SCOPE_EVOLUTION` blocker_type → `scope_evolution_flag = TRUE` → F8 auto-trigger
- **0.6 Addon Phase States:** `PENDING → TECH_APPROVED → AWAITING_CEO → FULLY_APPROVED → MILESTONE_CREATED`
- **0.7 RBAC:** EXPERT submits brief; TECH_TEAM approves technical justification; CEO approves budget

### Pre-conditions
- `milestones.state = 'IN_PROGRESS'`
- `sprint_status_updates.scope_evolution_flag = TRUE` (set by expert's BLOCKER submission)

### Step [F8-1] — SCOPE_EVOLUTION sprint status triggers auto-creation

```sql
-- When expert submits sprint_status_update with blocker_type='SCOPE_EVOLUTION':
UPDATE sprint_status_updates SET scope_evolution_flag = TRUE WHERE id = sprint_update_id;

-- Pause milestone review clock
UPDATE milestones SET review_clock_paused_at = now()
  WHERE id = milestone_id AND review_clock_paused_at IS NULL;

-- NestJS auto-creates addon_phase_requests:
INSERT INTO addon_phase_requests (
  engagement_id,
  submitted_by,                -- expert_user_id
  trigger_sprint_status_id,    -- FK to the SCOPE_EVOLUTION sprint_status_updates row
                               -- UNIQUE partial index: prevents double-trigger on same event
  causal_chain,               -- "Phase 1 established embedding pipeline. Analysis revealed 34% LP near-duplicates..."
  new_capability_requirements, -- "Semantic deduplication seam (A↔D) not in original footprint"
  why_invisible,              -- "This was invisible at intake: LP reuse rate only computable post-Phase 1 data"
  status                      -- 'PENDING'
) VALUES (engagement_id, expert_user_id, sprint_update_id,
          'Phase 1 established...', 'Semantic deduplication...', 'Invisible at intake...', 'PENDING');
```

**`trigger_sprint_status_id` unique partial index:**
```sql
CREATE UNIQUE INDEX addon_per_sprint_event ON addon_phase_requests(trigger_sprint_status_id)
  WHERE trigger_sprint_status_id IS NOT NULL;
-- Prevents the same SCOPE_EVOLUTION event from generating two add-on requests
-- (e.g., if NestJS retries the handler)
```

### Step [F8-2] — TECH_TEAM approves technical justification (independent of CEO)

```sql
UPDATE addon_phase_requests
  SET status = 'TECH_APPROVED',
      tech_approved_at = now(),
      tech_approved_by = tech_user_id
  WHERE id = ? AND status = 'PENDING';
```

### Step [F8-3] — CEO approves budget and timeline

```sql
UPDATE addon_phase_requests
  SET status = 'FULLY_APPROVED',
      ceo_approved_at = now(),
      ceo_approved_by = ceo_user_id
  WHERE id = ? AND status = 'TECH_APPROVED';
```

### Step [F8-4] — System auto-creates new milestone + resumes clock

```sql
BEGIN;
INSERT INTO milestones (engagement_id, milestone_number, deliverable_statement,
  sign_off_authority, payment_amount_vnd, state)
VALUES (engagement_id, next_milestone_number, 'LP Semantic Deduplication Pipeline',
        'JOINT', 5000000, 'DEFINED')
RETURNING id AS new_milestone_id;

UPDATE addon_phase_requests
  SET status = 'MILESTONE_CREATED',
      created_milestone_id = new_milestone_id  -- FK back to the newly created milestone
  WHERE id = addon_id;

-- Resume review clock on original milestone
UPDATE milestones
  SET total_paused_seconds = total_paused_seconds + EXTRACT(EPOCH FROM (now() - review_clock_paused_at))::INT,
      review_clock_paused_at = NULL
  WHERE id = original_milestone_id;
COMMIT;
```

**`addon_phase_requests.created_milestone_id` FK is essential:** Without this back-reference, you cannot answer "which add-on request created this milestone?" from the DB, which is needed for the scope evolution audit trail in the Platform Integrity Monitor.

---

## MF-15: Dual-Role Account Switch

### Master Reference Anchors
- **0.7 RBAC:** `users.roles` JSONB array; `users.active_role` controls current dashboard and guards
- **Self-exclusion:** matching engine always checks `expert.user_id NOT IN project.involved_parties`

### Pre-conditions
- `users.roles` contains `["CLIENT_CEO", "EXPERT"]` (second role added via Account Settings)
- Both `subscription_client_tier` and `subscription_expert_tier` may each be `'free'` or `'pro'`

### Step [DR1] — User adds second role (one-time setup)

```sql
UPDATE users
  SET roles = roles || '["EXPERT"]'::jsonb  -- append EXPERT role
  WHERE id = user_id AND roles @> '["CLIENT_CEO"]'::jsonb;
-- After this, wallet is still the same wallet (one wallet per user regardless of roles)
-- Bank Hub Hosted Link prompted if switching to Expert and sepay_bank_account_xid IS NULL
```

### Step [DR2] — User clicks role switcher → JWT reissued

```sql
-- NestJS reissues JWT with new active_role
-- No DB change needed — just new JWT payload:
-- { active_role: "EXPERT", roles: ["CLIENT_CEO", "EXPERT"], ... }
-- Dashboard re-renders for Expert context
-- Subscription guard now checks subscription_expert_tier (not subscription_client_tier)
```

### Step [DR3] — Self-exclusion in matching engine

```sql
-- When matching engine builds expert shortlist for any project:
-- Check if candidate expert_user_id is involved in the project being matched
SELECT 1 FROM projects WHERE client_id = candidate_user_id AND id = ?
UNION
SELECT 1 FROM tech_team_profiles WHERE user_id = candidate_user_id AND linked_project_id = ?;
-- → if any row: exclude from shortlist regardless of active_role
```

### Step [DR4] — Expert earnings fund client projects (same wallet)

```sql
-- Expert earns 9,500,000 VND from MF-7 → wallet.available_balance += 9,500,000
-- Expert switches to CLIENT role
-- CEO funds their own new project milestone from the same wallet:
UPDATE wallets SET available_balance = available_balance - 10000000, locked_balance = locked_balance + 10000000
  WHERE user_id = dual_role_user_id;  -- same user_id, same wallet row
-- No top-up needed — earnings fund the next project directly
```

---

## MF-16: Expert Verification — Auto Tier 4 Upgrade (Post-Engagement)

### Master Reference Anchors
- **0.4 Signal Accumulation Rule:** 1× LOAD_BEARING positive → Tier 4; 2× SIGNIFICANT → Tier 4; 3× CONTRIBUTING → Tier 4
- **0.6 Engagement States:** Rule evaluated on `CLOSED` transition; no admin action

### Step [T4-1] — TECH_TEAM submits structured review at engagement close

```sql
-- TECH_TEAM review triggers outcome_signals and expert_seam_outcome_signals extraction
INSERT INTO reviews (engagement_id, reviewer_id, target_id, rating, structured_signals_json, reviewer_role)
  VALUES (engagement_id, tech_user_id, expert_user_id, 5,
          '{"seam_signals":[{"seam_code":"SEAM_AC","signal_type":"LOAD_BEARING","valence":"POSITIVE","seam_role":"Architect"}]}',
          'TECH_TEAM')
RETURNING id AS review_id;

INSERT INTO outcome_signals (engagement_id, reviewer_id, review_id, seam_signals_json, ratings_json)
  VALUES (engagement_id, tech_user_id, review_id,
          '[{"seam_code":"SEAM_AC","signal_type":"LOAD_BEARING","seam_role":"Architect"}]',
          '{"technical_clarity":5,"scope_accuracy":4}');
-- review_id UNIQUE: enforces 1:1 between a review and its outcome_signals aggregate
```

### Step [T4-2] — NestJS extracts per-seam rows into expert_seam_outcome_signals

```sql
INSERT INTO expert_seam_outcome_signals (
  engagement_id,
  expert_id,
  reviewer_id,
  outcome_signal_id,   -- FK to outcome_signals.id (the aggregate that generated this row)
  seam_claim_id,       -- DIRECT FK to expert_seam_claims.id → Tier 4 UPDATE writes to correct row
  seam_code,
  signal_type,         -- 'LOAD_BEARING'
  valence,             -- 'POSITIVE'
  seam_role
) VALUES (engagement_id, expert_user_id, tech_user_id, outcome_signal_id, seam_claim_id_ac,
          'SEAM_AC', 'LOAD_BEARING', 'POSITIVE', 'Architect');
```

**`seam_claim_id` FK is essential for Tier 4 write-back:** NestJS needs to UPDATE the exact `expert_seam_claims` row. Without this FK, it would need to SELECT by `(expert_id, seam_code)` first — which works but means the write-back depends on the uniqueness constraint holding, and it's less precise.

### Step [T4-3] — Tier 4 accumulation rule evaluation (on engagement CLOSED)

```sql
-- Rule: 1× LOAD_BEARING + POSITIVE → immediate Tier 4
SELECT COUNT(*) FROM expert_seam_outcome_signals
  WHERE expert_id = expert_user_id
    AND seam_code = 'SEAM_AC'
    AND signal_type = 'LOAD_BEARING'
    AND valence = 'POSITIVE';
-- Returns 1 → condition met

-- Upgrade the seam claim
BEGIN;
UPDATE expert_seam_claims
  SET verification_tier = 'PLATFORM_DEMONSTRATED'  -- confidence_factor → 0.95
  WHERE id = seam_claim_id_ac AND expert_id = expert_user_id;

INSERT INTO platform_decisions (decision_type, entity_type, entity_id, decision, advisory_note)
  VALUES ('SEAM_TIER_UPGRADE', 'portfolio_submission', seam_claim_id_ac,
          'PLATFORM_DEMONSTRATED', 'Tier 4 via signal accumulation: 1× LOAD_BEARING POSITIVE on SEAM_AC');
COMMIT;

-- Update engagement state
UPDATE engagements SET state = 'CLOSED' WHERE id = engagement_id;
```

**Matching hard gate update:** From this point, future match queries will use `confidence_factor = 0.95` for SEAM_AC — the highest possible, giving this expert the strongest composite score for any future project requiring SEAM_AC.

---

# GROUP 6 — Admin Flows

---

## MF-17: Platform Integrity Monitor & Analytics

### Master Reference Anchors
- **0.7 RBAC:** Admin has read-only access to most tables; two write actions only (spec pull-back, account suspension)
- **`platform_decisions` written by FastAPI only** — NestJS never writes to this table
- **`platform_decisions.entity_type`** is required companion to polymorphic `entity_id` — allows JOIN to correct source table

### Integrity Monitor Reads

**Spec auto-return log:**
```sql
SELECT pd.entity_id AS project_id, pd.advisory_note, pd.created_at, pd.decision,
       p.state, es.current_stage
  FROM platform_decisions pd
  JOIN projects p ON p.id = pd.entity_id::uuid
  JOIN elicitation_sessions es ON es.id = p.elicitation_session_id
  WHERE pd.decision_type = 'SPEC_AUTO_RETURN'
  ORDER BY pd.created_at DESC;
-- entity_type='project' tells the monitor to JOIN projects, not capability_bid or other tables
```

**Seam verification log:**
```sql
SELECT pd.entity_id AS submission_id, pd.llm_confidence, pd.decision, pd.advisory_note, pd.created_at,
       esc.seam_code, esc.verification_tier, esc.submission_count
  FROM platform_decisions pd
  JOIN portfolio_submissions ps ON ps.id = pd.entity_id::uuid
  JOIN expert_seam_claims esc ON esc.id = ps.seam_claim_id
  WHERE pd.decision_type = 'PORTFOLIO_EVAL'
  ORDER BY pd.created_at DESC;
```

**Bid conflict override log (CEO overrode TECH_DISAPPROVED):**
```sql
SELECT bco.bid_id, bco.ceo_user_id, bco.override_reason, bco.overridden_at,
       u.full_name AS ceo_name, cb.state AS current_bid_state
  FROM bid_conflict_overrides bco
  JOIN users u ON u.id = bco.ceo_user_id
  JOIN capability_bids cb ON cb.id = bco.bid_id
  ORDER BY bco.overridden_at DESC;
```

**Dispute resolution log:**
```sql
SELECT d.id, d.state, d.resolution_layer, d.llm_confidence, d.filed_at, d.resolved_at,
       ac.criterion_text, e.type AS engagement_type
  FROM disputes d
  JOIN acceptance_criteria ac ON ac.id = d.criterion_id
  JOIN engagements e ON e.id = d.engagement_id
  ORDER BY d.filed_at DESC;
```

**Analytics: seam verification throughput (for RQ1 evidence)**
```sql
SELECT decision, COUNT(*) AS count,
       AVG(llm_confidence) AS avg_confidence,
       DATE_TRUNC('week', created_at) AS week
  FROM platform_decisions
  WHERE decision_type = 'PORTFOLIO_EVAL'
  GROUP BY decision, week
  ORDER BY week DESC;
```

### Admin Emergency Write — Spec Pull-Back

```sql
-- Only admin can do this; only applies to PUBLISHED specs
UPDATE projects SET state = 'SUSPENDED' WHERE id = ? AND state = 'PUBLISHED';
UPDATE artifact_a SET state = 'SUSPENDED' WHERE project_id = ? AND state = 'PUBLISHED';
```

### Wallet Ledger Audit

```sql
SELECT wt.transaction_type, wt.amount, wt.reference_id, wt.created_at,
       u.full_name, u.active_role
  FROM wallet_transactions wt
  JOIN wallets w ON w.id = wt.wallet_id
  JOIN users u ON u.id = w.user_id
  WHERE wt.created_at BETWEEN ? AND ?
  ORDER BY wt.created_at DESC;
```

---

## MF-18: Account Management & Fraud Detection

### Master Reference Anchors
- **0.7 RBAC:** Admin's `is_active` write is the only account state mechanism; no `is_suspended` field exists in schema
- **`platform_decisions`** receives `ADMIN_SUSPENSION` and `ADMIN_REACTIVATION` entries

### Phase A — Automated Fraud Detection (Bank Account Deduplication)

**Step [FR1] — Expert links bank account via Bank Hub**

Before storing the `bank_account_xid`, NestJS runs deduplication:
```sql
-- Check if this bank_account_xid is already linked to another expert
SELECT u.id, u.full_name, u.email
  FROM users u
  WHERE u.sepay_bank_account_xid = ?  -- incoming bank_account_xid from BANK_ACCOUNT_LINKED webhook
    AND u.id != current_expert_user_id;  -- exclude self
```

**Step [FR2] — If duplicate found → auto-block both accounts**
```sql
BEGIN;
-- Block the new expert (do not store the xid — reject the linking)
-- Nothing stored on new expert; bank_account_xid remains NULL

-- Block existing expert immediately
UPDATE users SET is_active = FALSE WHERE id = existing_expert_id;

-- Both get audit records
INSERT INTO platform_decisions (decision_type, entity_type, entity_id, decision, advisory_note)
  VALUES
  ('CONTENT_MODERATION', 'project', new_expert_id::text, 'FRAUD_AUTO_BLOCK',
   'Bank account xid already linked to another expert account'),
  ('CONTENT_MODERATION', 'project', existing_expert_id::text, 'FRAUD_AUTO_BLOCK',
   'Duplicate bank account detected; account suspended pending investigation');
COMMIT;
```

**Note on schema:** The physical ER uses `users.is_active = FALSE` for suspension (not an `is_suspended` field). All guards check `users.is_active = TRUE`; setting it FALSE immediately blocks all JWT validation for that user.

### Phase B — Manual Suspension

**Step [MS1] — Admin submits suspension**
```sql
BEGIN;
UPDATE users
  SET is_active = FALSE
  WHERE id = target_user_id;

INSERT INTO platform_decisions (decision_type, entity_type, entity_id, decision, advisory_note)
  VALUES ('CONTENT_MODERATION', 'project', target_user_id::text, 'ADMIN_SUSPENSION',
          'Spam bids / Harassment');  -- admin-provided reason stored in advisory_note
COMMIT;
-- Suspended user's existing JWT → auth guard returns 401 on next request
-- (Guard reads users.is_active on every protected request)
```

### Phase C — Reactivation

```sql
BEGIN;
UPDATE users SET is_active = TRUE WHERE id = target_user_id;

INSERT INTO platform_decisions (decision_type, entity_type, entity_id, decision, advisory_note)
  VALUES ('CONTENT_MODERATION', 'project', target_user_id::text, 'ADMIN_REACTIVATION',
          'Suspension resolved; re-link bank account required if fraud case');
COMMIT;
-- Note: if suspended for duplicate bank account, expert must complete Bank Hub Hosted Link again
-- (sepay_bank_account_xid was never stored; remains NULL)
-- Active engagements and wallet balances are untouched — only route access is revoked by is_active
```

### End-State Trace — MF-18

| Table | Step | Change |
|---|---|---|
| `users` | FR2 auto-block | `is_active=FALSE` for existing expert |
| `platform_decisions` | FR2 | `decision_type='CONTENT_MODERATION'`, `decision='FRAUD_AUTO_BLOCK'` |
| `users` | MS1 manual suspend | `is_active=FALSE` |
| `platform_decisions` | MS1 | `decision='ADMIN_SUSPENSION'`, `advisory_note=reason` |
| `users` | PC reactivate | `is_active=TRUE` |
| `platform_decisions` | PC | `decision='ADMIN_REACTIVATION'` |
| `wallets` | all steps | **unchanged** — escrow never touched by suspension |
| `engagements` | all steps | **unchanged** — active engagements continue after reactivation |

---

# Cross-Table FK Map: Critical Relationships for Teacher Review

The following FK relationships are the most frequently queried in the system and the most important for understanding the data model's correctness:

| FK | From | To | Why it exists |
|---|---|---|---|
| `projects.elicitation_session_id` | `projects` | `elicitation_sessions` | One-way: project always points to session; session can exist without project (abandoned attempts) |
| `tech_team_profiles.linked_project_id` | `tech_team_profiles` | `projects` | Scope guard: TECH_TEAM account locked to exactly one project |
| `portfolio_submissions.seam_claim_id` | `portfolio_submissions` | `expert_seam_claims` | LLM evaluation result updates the exact seam claim (no ambiguity) |
| `scenario_responses.seam_claim_id` | `scenario_responses` | `expert_seam_claims` | Same: Tier 3 rubric score updates the exact seam claim |
| `expert_seam_outcome_signals.seam_claim_id` | `expert_seam_outcome_signals` | `expert_seam_claims` | Tier 4 accumulation write-back targets exact claim row |
| `expert_seam_outcome_signals.outcome_signal_id` | `expert_seam_outcome_signals` | `outcome_signals` | Links per-seam rows back to the review aggregate that generated them |
| `outcome_signals.review_id` (UNIQUE) | `outcome_signals` | `reviews` | Enforces 1:1: one TECH_TEAM review → one aggregate signal record |
| `bid_versions.revision_request_id` | `bid_versions` | `bid_revision_requests` | Links each version snapshot to the specific request that caused it (NULL for v1) |
| `bid_conflict_overrides.bid_id` (UNIQUE) | `bid_conflict_overrides` | `capability_bids` | One override per bid; enforces the Surface D business rule at DB level |
| `engagements` type CHECK | — | — | `(type='PROJECT_BASED' AND project_id NOT NULL AND service_id NULL) OR (type IN ('SERVICE_PURCHASE','TECH_DISCOVERY') AND project_id NULL AND service_id NOT NULL)` |
| `escrow_accounts` one-parent CHECK | — | — | Exactly one of `milestone_id` or `engagement_id` must be set; prevents ambiguous escrow records |
| `disputes.escrow_account_id` | `disputes` | `escrow_accounts` | Direct FK makes escrow freeze O(1) and type-agnostic for both PROJECT_BASED and SERVICE_PURCHASE |
| `disputes.criterion_id` | `disputes` | `acceptance_criteria` | Disputes are always against a specific criterion (the objective contract element) |
| `addon_phase_requests.trigger_sprint_status_id` (UNIQUE partial) | `addon_phase_requests` | `sprint_status_updates` | One add-on per SCOPE_EVOLUTION event; prevents double-trigger |
| `addon_phase_requests.created_milestone_id` | `addon_phase_requests` | `milestones` | Reverse link: which add-on request created this milestone |
| `sprint_status_updates.milestone_sprint_id` | `sprint_status_updates` | `milestone_sprints` | Direct FK replaces implicit (milestone_id, sprint_number) lookup |
| `virtual_accounts.entity_id` (polymorphic) | `virtual_accounts` | users/milestones/engagements/subscriptions | No DB FK by design (polymorphic); `entity_type` determines the target table |
| `wallet_transactions` idempotency index | — | — | `UNIQUE (wallet_id, reference_id) WHERE reference_id IS NOT NULL` — prevents SePay IPN double-credit |
| `platform_settings.platform_wallet_id` | `platform_settings` | `wallets` | Singleton links to the seeded system wallet; fee rate read from here at transaction time |
| `platform_decisions.entity_type` | — | — | Required companion to polymorphic `entity_id`; enables Platform Integrity Monitor to JOIN correct source table |

---

# State Machine Quick Reference

| Entity | Initial State | Final States | Trigger |
|---|---|---|---|
| `elicitation_sessions.state` | `IN_PROGRESS` | `COMPLETED`, `ABANDONED`, `RETURNED` | Synthesis success/fail |
| `projects.state` | `DRAFT` | `PUBLISHED`, `RETURNED_TO_CLIENT`, `SUSPENDED` | Quality gate |
| `artifact_a.state` | `DRAFT` | `PUBLISHED`, `RETURNED_TO_CLIENT`, `SUSPENDED` | Mirrors projects |
| `capability_bids.state` | `DRAFT` | `SELECTED`, `DECLINED` | CEO selection |
| `engagements.state` | `PENDING` | `CLOSED`, `DISPUTED` | All milestones released |
| `milestones.state` | `DEFINED` | `RELEASED` | chi hộ credit IPN |
| `milestone_dod_items.status` | `PENDING` | `COMPLETED`, `NOT_APPLICABLE` (non-required only) | Expert self-mark |
| `sprint_status_updates.status` | *(per update)* | `ON_TRACK`, `DEVIATION`, `BLOCKER` | Weekly expert submission |
| `escrow_accounts.status` | `HELD` | `RELEASED`, `REFUNDED`, `SPLIT`, `FROZEN` | Approval / dispute |
| `disputes.state` | `LAYER_1_EVAL` | `AUTO_RESOLVED`, `MUTUAL_RESOLVED`, `DISPUTED_AUTO_RESOLVED` | LLM / 48h timer |
| `addon_phase_requests.status` | `PENDING` | `MILESTONE_CREATED`, `REJECTED` | Both approvals or either party rejects |
| `virtual_accounts.status` | `ACTIVE` | `USED`, `EXPIRED` | IPN fires or 24h elapses |
| `user_subscriptions.status` | `PENDING_PAYMENT` | `ACTIVE`, `EXPIRED` | IPN fires / time |
| `withdrawal_requests.status` | `PENDING` | `COMPLETED`, `FAILED` | chi hộ IPN / API error |
| `services.state` | `DRAFT` | `PUBLISHED`, `SUSPENDED` | Expert publishes / Admin pulls |
| `expert_seam_claims.verification_tier` | `CLAIMED` | `PLATFORM_DEMONSTRATED` | Signal accumulation |
