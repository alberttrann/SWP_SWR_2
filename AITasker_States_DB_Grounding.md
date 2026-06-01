# AITasker — Schema Grounding Evidence
## How Every Concept in Sections 0.1–0.6 Is Fully Supported by the Physical Schema

> **Purpose:** This document proves, table by table and column by column, that every business rule, state machine, event trigger, and data operation described in the Master Reference Sheet (Sections 0.1–0.6 of the Internal Scope Document) and across all 18 Main Flows has a concrete, named home in the physical PostgreSQL schema. It is structured as a cross-examination reference: for any concept the teacher raises, the reader can immediately point to the exact table, column, constraint, or index that enforces it.

> **How to read:** Each section maps one conceptual layer to its schema backing. "Schema anchor" entries cite the exact table.column, constraint name, or SQL construct. "Why it can't be faked" entries explain what would go wrong at the database level if someone tried to bypass the rule.

---

## Part 1 — Section 0.1: The Six Capability Domains → Schema Backing

The six domains (A–F) are not free-text fields that users can invent. They are controlled vocabulary that flows through three expert capability tables.

### Domain claims live in `expert_domain_depths`

```
expert_domain_depths.domain_code  TEXT NOT NULL
  → stores 'DOMAIN_A' through 'DOMAIN_F'
expert_domain_depths.depth_level  TEXT NOT NULL
  CHECK (depth_level IN ('SURFACE','OPERATIONAL','DEEP'))
expert_domain_depths.verification_tier TEXT NOT NULL
  CHECK (verification_tier IN ('CLAIMED','EVIDENCE_BACKED','SCENARIO_VERIFIED','PLATFORM_DEMONSTRATED'))
UNIQUE (expert_id, domain_code)
```

An expert can declare at most one depth level per domain. The uniqueness constraint at `(expert_id, domain_code)` makes it physically impossible to hold duplicate or conflicting depth claims for the same domain.

### Project domain requirements live in `capability_footprints`

```
capability_footprints.required_domains_json  JSONB NOT NULL DEFAULT '[]'
  Format: [{"code":"DOMAIN_A","depth":"DEEP"}, ...]
```

The elicitation synthesis engine (FastAPI, MF-4 Stage 5) writes this field. The matching engine (MF-5) reads it and compares each required domain code and depth against the candidate expert's `expert_domain_depths` rows.

### How the 25% domain depth component is computed

For each entry in `required_domains_json`:
- Query: `SELECT depth_level FROM expert_domain_depths WHERE expert_id = ? AND domain_code = 'DOMAIN_X'`
- If expert's `depth_level` meets or exceeds the required depth → score 1.0 for that domain
- If expert has no row for that domain code → score 0.0

The UNIQUE constraint on `(expert_id, domain_code)` means this SELECT always returns at most one row — no ambiguity in the scoring query.

---

## Part 2 — Section 0.2: The Ten Seams → Schema Backing

Seams are the primary differentiator (40% of composite score). They are tracked across four distinct tables, each serving a different lifecycle purpose.

### Seam claims: `expert_seam_claims`

```
expert_seam_claims.seam_code  TEXT NOT NULL  (e.g. 'SEAM_AC', 'SEAM_DE')
expert_seam_claims.verification_tier TEXT NOT NULL DEFAULT 'CLAIMED'
  CHECK (verification_tier IN ('CLAIMED','EVIDENCE_BACKED','SCENARIO_VERIFIED','PLATFORM_DEMONSTRATED'))
expert_seam_claims.submission_count INT NOT NULL DEFAULT 0   -- 5-attempt throttle
expert_seam_claims.locked_until TIMESTAMPTZ NULL             -- 30-day lockout after 5th failure
UNIQUE (expert_id, seam_code)
```

The `seam_code` is the cross-reference point for every downstream table. The UNIQUE constraint prevents duplicate claims. The `submission_count` and `locked_until` columns enforce the anti-gaming throttle from Section 0.4.

### Project seam requirements: `capability_footprints.required_seams_json`

```json
[{"code":"SEAM_AC","criticality":"LOAD_BEARING"},
 {"code":"SEAM_CE","criticality":"SIGNIFICANT"},
 {"code":"SEAM_DE","criticality":"SIGNIFICANT"}]
```

Three criticality values — LOAD_BEARING, SIGNIFICANT, CONTRIBUTING — are stored as strings in the JSONB. The matching engine in FastAPI reads these and applies the weight multipliers (1.0 / 0.7 / 0.4) when computing Component 1 of the composite score.

### Seam gap map generation

For each required seam:
1. Query `expert_seam_claims WHERE expert_id = ? AND seam_code = ?`
2. Map `verification_tier` to confidence factor: CLAIMED→0.20, EVIDENCE_BACKED→0.55, SCENARIO_VERIFIED→0.80, PLATFORM_DEMONSTRATED→0.95
3. If no row found for a LOAD_BEARING seam → apply −0.1 penalty (not 0.0)
4. Return `{seam_code, coverage, color}` per seam for the gap map displayed to CEO

### Why the gap map is DB-derivable, not faked

Because `expert_seam_claims` has `UNIQUE (expert_id, seam_code)`, every lookup for `(expert_id, seam_code)` returns exactly 0 or 1 rows. There is no path for an expert to hold two different verification tiers for the same seam simultaneously.

### Portfolio submissions: `portfolio_submissions`

```
portfolio_submissions.seam_claim_id  UUID NOT NULL REFERENCES expert_seam_claims(id)
portfolio_submissions.llm_confidence FLOAT NULL
portfolio_submissions.status TEXT NOT NULL CHECK (status IN ('PENDING','APPROVED','REJECTED'))
```

The `seam_claim_id` FK is the critical design decision here. When FastAPI returns an LLM confidence ≥ 0.85, NestJS executes:
```sql
UPDATE expert_seam_claims SET verification_tier = 'EVIDENCE_BACKED'
  WHERE id = portfolio_submissions.seam_claim_id
```
Without this direct FK, the UPDATE would need to resolve the target row by `(expert_id, seam_code)` — which works, but introduces a race condition if two submissions for the same seam are in flight simultaneously.

### Scenario responses: `scenario_responses`

```
scenario_responses.seam_claim_id  UUID NOT NULL REFERENCES expert_seam_claims(id)
scenario_responses.pass_fail TEXT NULL CHECK (pass_fail IN ('PASS','FAIL','PENDING'))
scenario_responses.llm_rubric_scores_json JSONB NULL  -- per-dimension scores
```

Same FK pattern as `portfolio_submissions`. The `llm_rubric_scores_json` stores individual dimension scores (each must pass the per-question threshold independently — Section 0.4 rubric rule). When all dimensions pass, NestJS writes `verification_tier = 'SCENARIO_VERIFIED'` to the exact `expert_seam_claims` row via `seam_claim_id`.

### Outcome signals: `expert_seam_outcome_signals`

```
expert_seam_outcome_signals.seam_claim_id  UUID NOT NULL REFERENCES expert_seam_claims(id)
expert_seam_outcome_signals.signal_type TEXT NOT NULL
  CHECK (signal_type IN ('LOAD_BEARING','SIGNIFICANT','CONTRIBUTING'))
expert_seam_outcome_signals.valence TEXT NOT NULL
  CHECK (valence IN ('POSITIVE','NEGATIVE'))
```

This table is the Tier 4 accumulation store. The `signal_type` and `valence` columns directly implement the Section 0.4 signal accumulation rule:
- 1× LOAD_BEARING + POSITIVE → Tier 4
- 2× SIGNIFICANT + POSITIVE → Tier 4
- 3× CONTRIBUTING + POSITIVE → Tier 4

The matching hard gate also reads this table: `COUNT(*) WHERE valence='NEGATIVE' AND signal_type='LOAD_BEARING' AND seam_code = ANY(load_bearing_seams) >= 2` → exclude from shortlist.

---

## Part 3 — Section 0.3: Archetypes + Tiers → Schema Backing

### Archetype classification is stored in two places

**Session level** (locked at Stage 2):
```
elicitation_sessions.archetype  TEXT NULL  -- e.g. 'ARCHETYPE_1'; set at Stage 2
```

**Project level** (mirrored from session):
```
projects.archetype  TEXT NULL   -- set by elicitation synthesis at Stage 5
projects.tier       TEXT NULL CHECK (tier IN ('TIER_1','TIER_2','TIER_3'))
```

The archetype is determined by FastAPI during Stage 1 symptom extraction and confirmed by the CEO at Stage 2. It is then locked — the `elicitation_sessions.archetype` column is written once and not updated by any subsequent flow.

### Load-bearing seam enforcement

Each archetype has exactly one load-bearing seam (Section 0.3 table). This is a code-level constant in FastAPI (not a DB table), but its enforcement is DB-backed:

1. `capability_footprints.required_seams_json` always contains exactly one entry with `"criticality":"LOAD_BEARING"` for the archetype
2. The matching score computation applies the −0.1 penalty for a missing load-bearing seam
3. The hard gate filter excludes experts with 2+ NEGATIVE signals on a load-bearing seam code

### Archetype-tier congruence: `expert_profiles.archetype_history_json`

```
expert_profiles.archetype_history_json  JSONB NULL
  Format: [{"archetype_code":"ARCHETYPE_1","tier":"TIER_3","self_declared":true}]
```

This JSONB field seeds the 20% archetype-tier congruence component for cold-start experts (no closed engagements). For experts with prior engagements, FastAPI derives this at query time from `engagements → projects` (reading `projects.archetype` and `projects.tier`). The JSONB field is the fallback — it is not the authoritative source once real history exists.

### Tier classification from volume/complexity signals

The `elicitation_sessions.void_list_json` stores SDLC gaps detected in Stage 1. Infrastructure signals extracted in Stage 3 (Kafka/async flag, record counts, integration count) are what drive the TIER_1/TIER_2/TIER_3 classification written to `projects.tier`. This column then drives the subscription guard:
- `subscription_expert_tier = 'pro'` required to bid on Tier 2+ projects (checked at bid submission)

---

## Part 4 — Section 0.4: Expert Verification Tiers → Schema Backing

The four-tier verification system is grounded in a single column on `expert_seam_claims`, backed by three write-path tables.

### The canonical tier column

```
expert_seam_claims.verification_tier TEXT NOT NULL DEFAULT 'CLAIMED'
  CHECK (verification_tier IN ('CLAIMED','EVIDENCE_BACKED','SCENARIO_VERIFIED','PLATFORM_DEMONSTRATED'))
```

Every legitimate tier upgrade is a single `UPDATE` to this column. No other column stores the tier. The confidence factor (0.20 / 0.55 / 0.80 / 0.95) used in the composite score is a code-level mapping from this string — there is no `confidence_factor` column in the DB because it would be a redundant derivative.

### Tier 1 → Tier 2 (EVIDENCE_BACKED) write path

```
portfolio_submissions INSERT (status='PENDING')
  → FastAPI LLM extraction → llm_confidence ≥ 0.85 AND all required signals found
  → portfolio_submissions.status UPDATE 'APPROVED'
  → expert_seam_claims.verification_tier UPDATE 'EVIDENCE_BACKED'
  → platform_decisions INSERT (decision_type='PORTFOLIO_EVAL', decision='APPROVED')
```

Throttle enforcement:
```
expert_seam_claims.submission_count increments on every REJECTED submission
  → submission_count = 5 → locked_until = now() + 30 days
```
These two columns are the DB-level implementation of the "5 attempts then 30-day lockout" anti-gaming rule.

### Tier 2 → Tier 3 (SCENARIO_VERIFIED) write path

```
scenario_assessments SELECT (ORDER BY last_used_at ASC NULLS FIRST, LIMIT 1)
  → scenario_assessments.last_used_at UPDATE now()   -- prevents question reuse gaming
  → scenario_responses INSERT (pass_fail='PENDING')
  → FastAPI LLM rubric evaluation → all dimensions individually pass per-question threshold
  → scenario_responses.pass_fail UPDATE 'PASS'
  → scenario_responses.llm_rubric_scores_json UPDATE (per-dimension scores stored)
  → expert_seam_claims.verification_tier UPDATE 'SCENARIO_VERIFIED'
  → platform_decisions INSERT (decision_type='SCENARIO_EVAL', decision='PASS')
```

The `scenario_assessments.last_used_at` ordering is the anti-gaming mechanism: it ensures the same question bank entry is not repeatedly served to the same expert.

### Tier 3 → Tier 4 (PLATFORM_DEMONSTRATED) write path

```
engagements → CLOSED trigger:
  1. reviews INSERT (reviewer_role='TECH_TEAM', structured_signals_json set)
  2. outcome_signals INSERT (review_id FK set — UNIQUE enforces 1:1 per review)
  3. NestJS extracts per-seam rows:
     expert_seam_outcome_signals INSERT (seam_claim_id FK, signal_type, valence)
  4. Accumulation rule evaluated:
     SELECT COUNT(*) FROM expert_seam_outcome_signals
       WHERE expert_id=? AND seam_code=? AND signal_type='LOAD_BEARING' AND valence='POSITIVE'
     → COUNT ≥ 1 → expert_seam_claims.verification_tier UPDATE 'PLATFORM_DEMONSTRATED'
  5. platform_decisions INSERT (decision_type='SEAM_TIER_UPGRADE', decision='PLATFORM_DEMONSTRATED')
```

The `outcome_signals.review_id UNIQUE` constraint is what enforces the rule "one structured review produces one aggregate signal record." A TECH_TEAM member cannot submit two reviews for the same engagement to double-count Tier 4 signals.

---

## Part 5 — Section 0.5: Composite Match Score Weights → Schema Backing

The composite score is computed entirely in FastAPI, but every input to every component is sourced from exact DB columns. No component requires human input or admin judgment.

### Component 1 — Seam Alignment (40%)

| Input | DB Source |
|---|---|
| Required seams + criticality | `capability_footprints.required_seams_json` |
| Expert's seam coverage per seam | `expert_seam_claims.verification_tier` WHERE `(expert_id, seam_code)` |
| Confidence factor mapping | Code constant: CLAIMED→0.20, EVIDENCE_BACKED→0.55, SCENARIO_VERIFIED→0.80, PLATFORM_DEMONSTRATED→0.95 |
| Missing load-bearing penalty | −0.1 if no row found in `expert_seam_claims` for load-bearing seam |

### Component 2 — Domain Depth Coverage (25%)

| Input | DB Source |
|---|---|
| Required domains + depth | `capability_footprints.required_domains_json` |
| Expert's depth per domain | `expert_domain_depths.depth_level` WHERE `(expert_id, domain_code)` |

Depth ordering: SURFACE < OPERATIONAL < DEEP. Expert scores 1.0 per domain if `expert_depth >= required_depth`, else 0.0.

### Component 3 — Archetype-Tier Congruence (20%)

| Input | DB Source |
|---|---|
| Target archetype + tier | `projects.archetype`, `projects.tier` |
| Expert's prior engagement history | `engagements JOIN projects ON engagements.project_id = projects.id` WHERE `engagements.expert_id = ?` AND `projects.archetype = target AND projects.tier = target` |
| Cold-start fallback | `expert_profiles.archetype_history_json` (self-declared; confidence discounted) |

### Component 4 — Engagement Model Fit (10%)

| Input | DB Source |
|---|---|
| Required engagement model | Embedded in `capability_footprints` or derived from project type |
| Expert's declared model | `expert_profiles.engagement_model` CHECK (IN 'MILESTONE','HOURLY','HYBRID') |

### Component 5 — Stack Tag Overlap (5%)

| Input | DB Source |
|---|---|
| Required tags | Embedded in `capability_footprints` or project description |
| Expert's tags | `expert_stack_tags.tag` WHERE `expert_id = ?` |
| Recency weighting | `engagements.connected_at` for engagements where that tag was used |

### Hard Gate Filters (applied before scoring, not weighted)

**Gate 1 — Claimed-to-verified ratio > 4:1:**
```sql
SELECT expert_id FROM expert_seam_claims
GROUP BY expert_id
HAVING (COUNT(*) FILTER (WHERE verification_tier='CLAIMED')::FLOAT /
       NULLIF(COUNT(*) FILTER (WHERE verification_tier!='CLAIMED'), 0)) > 4
```
Source: `expert_seam_claims.verification_tier` aggregated per `expert_id`.

**Gate 2 — 2+ NEGATIVE signals on load-bearing seam:**
```sql
SELECT expert_id FROM expert_seam_outcome_signals
WHERE seam_code = ANY(load_bearing_seams) AND valence='NEGATIVE'
GROUP BY expert_id HAVING COUNT(*) >= 2
```
Source: `expert_seam_outcome_signals.valence`, `signal_type`, `seam_code`.

**Self-exclusion guard:**
```sql
SELECT 1 FROM projects WHERE client_id = candidate_id AND id = project_id
UNION
SELECT 1 FROM tech_team_profiles WHERE user_id = candidate_id AND linked_project_id = project_id
```
Source: `projects.client_id`, `tech_team_profiles.user_id + linked_project_id`.

---

## Part 6 — Section 0.6: All State Machines → Schema Backing

This is the most critical section for teacher examination. For each state machine, we trace every state value, every guard condition, and every trigger to its exact column and constraint.

---

### 6.1 Elicitation Session States

**State column:**
```
elicitation_sessions.state TEXT NOT NULL DEFAULT 'IN_PROGRESS'
  CHECK (state IN ('IN_PROGRESS','COMPLETED','ABANDONED','RETURNED'))
elicitation_sessions.current_stage INT NOT NULL DEFAULT 1 CHECK (current_stage BETWEEN 1 AND 5)
```

**State transition evidence:**

| State | Trigger event | Tables written |
|---|---|---|
| `IN_PROGRESS` (initial) | CEO clicks "Start New Project" | `elicitation_sessions` INSERT; `projects` INSERT (state='DRAFT'); `artifact_a` INSERT; `artifact_b` INSERT; `capability_footprints` INSERT |
| Stage advances 1→2→3→4→5 | CEO/TECH_TEAM complete each stage | `elicitation_sessions.current_stage` UPDATE |
| `COMPLETED` | Stage 5 synthesis succeeds | `elicitation_sessions.state` UPDATE; `projects.state` stays 'DRAFT' pending quality gate |
| `RETURNED` | Auto-publish quality gate fails | `elicitation_sessions.state` UPDATE; `elicitation_sessions.current_stage` SET to failing stage (not 1); `platform_decisions` INSERT (decision_type='SPEC_AUTO_RETURN') |
| `ABANDONED` | CEO navigates away | Session row persists at current `current_stage`; CEO can resume |

**Why the session persists at the failing stage (not stage 1):**
The `elicitation_sessions.current_stage` UPDATE on RETURNED writes the specific stage where the void was detected (e.g., stage 2 for archetype confirmation failure), not 1. This is enforced by the application logic reading `void_list_json` to determine which stage to return to — the CEO is sent directly to the point of failure, not restarted from scratch.

**Void tracking column:**
```
elicitation_sessions.void_list_json JSONB NOT NULL DEFAULT '[]'
  Format: [{"void_code":"GROUND_TRUTH_VOID","injected":false}, ...]
```
The quality gate check reads this JSON: any entry with `injected=false` is a blocking void. The gate passes only when all void entries have `injected=true` OR the void list is empty.

---

### 6.2 Spec (projects + artifact_a) States

**State columns (tracked independently but must stay in sync):**
```
projects.state TEXT NOT NULL DEFAULT 'DRAFT'
  CHECK (state IN ('DRAFT','PUBLISHED','RETURNED_TO_CLIENT','SUSPENDED'))
artifact_a.state TEXT NOT NULL DEFAULT 'DRAFT'
  CHECK (state IN ('DRAFT','PUBLISHED','RETURNED_TO_CLIENT','SUSPENDED'))
```

**Why two separate state columns:** `projects.state` and `artifact_a.state` are managed independently to allow edge cases where the project metadata is in one state but the public-facing spec artifact is in another (e.g., a project could be suspended while artifact_a is under revision). In practice they mirror each other for all normal flows; both are written atomically in the same DB transaction.

**State transition evidence:**

| State | Trigger | Tables written |
|---|---|---|
| `DRAFT` (initial) | Session created | `projects` INSERT + `artifact_a` INSERT both in same BEGIN/COMMIT |
| `PUBLISHED` | Auto-publish quality gate passes | `projects.state` UPDATE 'PUBLISHED'; `artifact_a.state` UPDATE 'PUBLISHED' — same transaction |
| `RETURNED_TO_CLIENT` | Quality gate fails | `projects.state` UPDATE; `artifact_a.state` UPDATE; `elicitation_sessions.state` UPDATE 'RETURNED'; `platform_decisions` INSERT with advisory_note |
| `SUSPENDED` | Admin emergency pull-back | `projects.state` UPDATE 'SUSPENDED'; `artifact_a.state` UPDATE 'SUSPENDED' — admin-only route |

**Auto-publish quality gate — three checks backed by DB queries:**

Check 1 — Footprint completeness:
```sql
-- required_seams_json must contain >= archetype minimum count
-- No entries in void_list_json with injected=false
SELECT void_list_json FROM elicitation_sessions WHERE id = session_id
```

Check 2 — Matching pre-check:
```sql
SELECT COUNT(DISTINCT esc.expert_id) FROM expert_seam_claims esc
  WHERE esc.seam_code = ANY(required_seams_codes)
  AND esc.verification_tier != 'CLAIMED'
  AND EXISTS (SELECT 1 FROM users u WHERE u.id = esc.expert_id AND u.is_active=TRUE
              AND u.subscription_expert_tier='pro')
-- Must return >= 1
```

Check 3 — No unresolved hard voids: read from `void_list_json` JSON array.

---

### 6.3 Capability Bid States

**State column:**
```
capability_bids.state TEXT NOT NULL DEFAULT 'DRAFT'
  CHECK (state IN ('DRAFT','SUBMITTED','TECH_REVIEW','REVISION_REQUESTED','REVISED',
                   'TECH_APPROVED','TECH_DISAPPROVED','CONFLICT_PENDING',
                   'CONFLICT_RESOLVED_CEO_OVERRIDE','PRICE_NEGOTIATION',
                   'NEGOTIATION_COMPLETE','CEO_REVIEW','SELECTED','DECLINED'))
capability_bids.version_number INT NOT NULL DEFAULT 1
```

**State transition evidence:**

| Transition | Trigger | Tables written |
|---|---|---|
| DRAFT → SUBMITTED | Expert submits 3-component bid | `capability_bids.state` UPDATE; `bid_versions` INSERT (version_number=1, revision_request_id=NULL) |
| SUBMITTED → TECH_REVIEW | TECH_TEAM opens review | `capability_bids.state` UPDATE |
| TECH_REVIEW → REVISION_REQUESTED | TECH_TEAM flags component | `capability_bids.state` UPDATE; `bid_revision_requests` INSERT (flagged_component, reason, requested_by) |
| REVISION_REQUESTED → REVISED | Expert revises flagged component only | `capability_bids.state` UPDATE; `capability_bids.version_number` += 1; `bid_versions` INSERT (revision_request_id=revision_req_id, version_number=N) |
| TECH_DISAPPROVED → CONFLICT_PENDING | CEO decides to proceed | `capability_bids.state` UPDATE |
| CONFLICT_PENDING → CONFLICT_RESOLVED_CEO_OVERRIDE | CEO writes justification | `bid_conflict_overrides` INSERT; `capability_bids.state` UPDATE; `platform_decisions` INSERT (decision_type='CRITERION_QUALITY_GATE', decision='CEO_OVERRIDE') |
| CEO_REVIEW → PRICE_NEGOTIATION | CEO proposes price adjustment | `price_negotiations` INSERT (round_number=1) |
| PRICE_NEGOTIATION → NEGOTIATION_COMPLETE | Expert accepts | `price_negotiations.response` UPDATE 'ACCEPTED'; `capability_bids.state` UPDATE |
| NEGOTIATION_COMPLETE → SELECTED | CEO finalizes selection | `capability_bids.state` UPDATE; `engagements.state` advances |

**Revision versioning evidence — the audit chain:**
```
bid_versions.revision_request_id  UUID NULL REFERENCES bid_revision_requests(id)
  → NULL on version 1 (original submission — no request caused it)
  → revision_request_id SET on versions 2, 3, N (each version links back to the request that triggered it)
UNIQUE (bid_id, version_number)  -- prevents duplicate version numbers per bid
```

This FK chain means: given any `bid_versions` row, you can answer "why was this version created?" by following `revision_request_id → bid_revision_requests.reason + flagged_component`. Without this FK, the answer would require timestamp proximity reasoning.

**Price negotiation cap — DB-enforced:**
```
price_negotiations.round_number INT NOT NULL CHECK (round_number IN (1,2))
UNIQUE (bid_id, round_number)
```
It is physically impossible to INSERT a `price_negotiations` row with `round_number=3`. The CHECK constraint rejects it at the DB level, not just the application level.

**CEO override audit trail — DB-enforced minimum justification:**
```
bid_conflict_overrides.override_reason TEXT NOT NULL CHECK (char_length(override_reason) >= 50)
bid_conflict_overrides.bid_id UUID NOT NULL UNIQUE  -- one override per bid
bid_conflict_overrides.ceo_user_id UUID NOT NULL REFERENCES users(id)
```
The `char_length >= 50` CHECK constraint prevents rubber-stamp overrides. The UNIQUE on `bid_id` prevents the CEO from repeatedly overriding the same bid.

---

### 6.4 Spec Clarification States (Surface A)

**State column:**
```
spec_clarifications.status TEXT NOT NULL DEFAULT 'OPEN'
  CHECK (status IN ('OPEN','ANSWERED','RESOLVED'))
  -- 'CLOSED' is NOT in the CHECK list — any route writing status='CLOSED' gets a DB error
spec_clarifications.responded_by UUID NULL REFERENCES users(id)
  -- NULL until ANSWERED; set on ANSWERED transition
```

**State transition evidence:**

| Transition | Trigger | Tables written |
|---|---|---|
| OPEN (initial) | Expert files clarification | `spec_clarifications` INSERT (responded_by=NULL, status='OPEN') |
| OPEN → ANSWERED | TECH_TEAM or CEO responds | `spec_clarifications.response_text` UPDATE; `spec_clarifications.responded_by` UPDATE (FK set); `spec_clarifications.status` UPDATE 'ANSWERED' |
| ANSWERED → RESOLVED | Expert marks resolved | `spec_clarifications.status` UPDATE 'RESOLVED'; `spec_clarifications.resolved_at` UPDATE now() |

**Why `responded_by` matters:** It records which role answered the clarification. For Tier 2+ projects, route guard requires `client_subtype='TECH_TEAM'` to write this field. For Tier 1 projects, the CEO responds. Without this column, there is no audit trail of who provided what information — which matters if a dispute later references a clarification as a commitment.

---

### 6.5 Engagement States

**State column:**
```
engagements.state TEXT NOT NULL DEFAULT 'PENDING'
  CHECK (state IN ('PENDING','CONNECTED','ACTIVE','CLOSED','DISPUTED'))
engagements.client_nda_accepted_at TIMESTAMPTZ NULL
engagements.expert_nda_accepted_at TIMESTAMPTZ NULL
```

**State transition evidence:**

| Transition | Trigger | Tables written | Guards |
|---|---|---|---|
| PENDING (initial) | CEO selects expert via bid SELECTED | `engagements` INSERT or state UPDATE 'PENDING' | Bid must be SELECTED |
| PENDING → CONNECTED | Both NDA timestamps set | `engagements.client_nda_accepted_at` UPDATE; `engagements.expert_nda_accepted_at` UPDATE; `engagements.state` UPDATE 'CONNECTED'; `engagements.connected_at` UPDATE now() | Both timestamps must be NOT NULL before state UPDATE is permitted |
| CONNECTED → ACTIVE | First milestone FUNDED (IPN fires) | `engagements.state` UPDATE 'ACTIVE' | `milestones.state='FUNDED'` must exist for this engagement |
| ACTIVE → DISPUTED | Dispute filed on any milestone | `disputes` INSERT; `escrow_accounts.status` UPDATE 'FROZEN'; `engagements.state` UPDATE 'DISPUTED' | Escrow must be HELD |
| DISPUTED → ACTIVE | Dispute resolved | `disputes.state` UPDATE (AUTO_RESOLVED/MUTUAL_RESOLVED/DISPUTED_AUTO_RESOLVED); `engagements.state` UPDATE 'ACTIVE' | Ledger settled per resolution |
| ACTIVE → CLOSED | Final milestone reaches RELEASED | `engagements.state` UPDATE 'CLOSED' → triggers Tier 4 evaluation pipeline | All milestones RELEASED |

**SERVICE_PURCHASE skip path:**
```
engagements.type CHECK constraint:
  (type='PROJECT_BASED' AND project_id IS NOT NULL AND service_id IS NULL) OR
  (type IN ('SERVICE_PURCHASE','TECH_DISCOVERY') AND project_id IS NULL AND service_id IS NOT NULL)
```
SERVICE_PURCHASE engagements are created directly in state='ACTIVE', bypassing PENDING/CONNECTED. This is permitted by the state machine spec and enforced by the type CHECK constraint ensuring the correct FK pattern exists.

**Artifact B access gate — DB-enforced read guard:**
```sql
SELECT ab.* FROM artifact_b ab
WHERE ab.project_id = ?
  AND EXISTS (
    SELECT 1 FROM engagements e
    WHERE e.project_id = ab.project_id
      AND e.client_nda_accepted_at IS NOT NULL
      AND e.expert_nda_accepted_at IS NOT NULL
      AND (e.expert_id = :current_user_id
           OR EXISTS (SELECT 1 FROM tech_team_profiles ttp
                      WHERE ttp.user_id = :current_user_id
                      AND ttp.linked_project_id = ab.project_id))
  )
-- CEO permanently excluded: no CEO branch in the EXISTS subquery
```
This is not an application-level check that can be bypassed by a misconfigured route — it is embedded in the FastAPI data access layer and runs against the actual `client_nda_accepted_at` + `expert_nda_accepted_at` columns. A CEO user_id will never satisfy either branch of the inner EXISTS.

---

### 6.6 Milestone States

**State column:**
```
milestones.state TEXT NOT NULL DEFAULT 'DEFINED'
  CHECK (state IN ('DEFINED','AWAITING_PAYMENT','FUNDED','IN_PROGRESS',
                   'SUBMITTED','IN_REVISION','APPROVED','RELEASED','DISPUTED'))
milestones.review_clock_paused_at TIMESTAMPTZ NULL     -- NULL = clock running; SET = paused
milestones.total_paused_seconds INT NOT NULL DEFAULT 0  -- accumulates SLA pause time
```

**State transition evidence:**

| Transition | Trigger | Tables written | Guards enforced |
|---|---|---|---|
| DEFINED (initial) | CEO creates milestone | `milestones` INSERT; `acceptance_criteria` INSERTs; `milestone_dod_items` INSERTs | `engagements.state='CONNECTED'` |
| DEFINED → AWAITING_PAYMENT | CEO clicks "Fund" | `milestones.state` UPDATE; `milestones.va_number` SET; `milestones.va_expires_at` SET; `virtual_accounts` INSERT (entity_type='MILESTONE', fixed_amount set) | CEO wallet must exist |
| AWAITING_PAYMENT → FUNDED | SePay IPN fires on milestone VA | `wallets.available_balance -= amount`; `wallets.locked_balance += amount`; `wallet_transactions` INSERT (ESCROW_LOCK); `escrow_accounts` INSERT (status='HELD'); `milestones.state` UPDATE 'FUNDED'; `milestones.funded_at` SET; `virtual_accounts.status` UPDATE 'USED'; `paygated_documents.release_state` UPDATE 'RELEASED' for staged docs | IPN amount must exactly match `virtual_accounts.fixed_amount` |
| FUNDED → IN_PROGRESS | Auto-advance | `milestones.state` UPDATE 'IN_PROGRESS' | Immediate, no guard |
| IN_PROGRESS → SUBMITTED | Expert submits deliverable | `milestone_submissions` INSERT; `milestones.state` UPDATE 'SUBMITTED'; `milestones.submitted_at` SET | **DoD gate: SELECT COUNT(*) FROM milestone_dod_items WHERE milestone_id=? AND is_required=TRUE AND status!='COMPLETED' → must return 0** |
| SUBMITTED → IN_REVISION | Sign-off authority requests revision | `revision_requests` INSERT; `milestones.state` UPDATE 'IN_REVISION' | Verified role matches `acceptance_criteria.verified_by_role` |
| IN_REVISION → SUBMITTED | Expert resubmits | `milestone_submissions` INSERT; `milestones.state` UPDATE 'SUBMITTED' | DoD gate runs again |
| SUBMITTED/IN_REVISION → DISPUTED | Party files dispute | `disputes` INSERT; `escrow_accounts.status` UPDATE 'FROZEN'; `milestones.state` UPDATE 'DISPUTED' | `escrow_accounts.status='HELD'` |
| SUBMITTED/IN_REVISION → APPROVED | All required criteria verified | `acceptance_criteria.verified_at` UPDATE (per criterion); `milestones.state` UPDATE 'APPROVED'; `milestones.approved_at` SET; escrow release ledger entries; `withdrawal_requests` INSERT (type='MILESTONE_RELEASE') | **Criteria gate: SELECT COUNT(*) FROM acceptance_criteria WHERE milestone_id=? AND is_required=TRUE AND verified_at IS NULL → must return 0** |
| APPROVED → RELEASED | Chi hộ credit IPN fires | `milestones.state` UPDATE 'RELEASED'; `milestones.released_at` SET; `withdrawal_requests.status` UPDATE 'COMPLETED' | IPN reference matches `withdrawal_requests.disbursement_id` |

**BLOCKER sprint status → review clock pause:**
```
sprint_status_updates.blocker_type = 'TECH_CONSTRAINT' (or similar)
  → milestones.review_clock_paused_at = now()

TECH_TEAM resolves blocker:
  → milestones.total_paused_seconds += EXTRACT(EPOCH FROM (now() - review_clock_paused_at))
  → milestones.review_clock_paused_at = NULL
```
Both columns on `milestones` implement the SLA pause tracking. The `total_paused_seconds` provides cumulative pause audit for the Platform Integrity Monitor.

**SCOPE_EVOLUTION → add-on phase trigger:**
```
sprint_status_updates.scope_evolution_flag BOOLEAN NOT NULL DEFAULT FALSE
  → set to TRUE when blocker_type='SCOPE_EVOLUTION'
  → NestJS auto-creates addon_phase_requests row
  → milestones.review_clock_paused_at = now()

UNIQUE INDEX addon_per_sprint_event ON addon_phase_requests(trigger_sprint_status_id)
  WHERE trigger_sprint_status_id IS NOT NULL
  -- Prevents double-trigger if NestJS handler retries
```

---

### 6.7 DoD Checklist Item States

**State column:**
```
milestone_dod_items.status TEXT NOT NULL DEFAULT 'PENDING'
  CHECK (status IN ('PENDING','COMPLETED','NOT_APPLICABLE'))
milestone_dod_items.is_required BOOLEAN NOT NULL DEFAULT TRUE
```

**The critical constraint:**
```sql
CONSTRAINT dod_required_cannot_be_na
  CHECK (NOT (is_required = TRUE AND status = 'NOT_APPLICABLE'))
```
This DB-level CHECK constraint directly implements the spec rule: "NOT_APPLICABLE: only for non-required items." No application code change can bypass this — the DB itself rejects the UPDATE.

**The submission gate — SQL query run by NestJS before allowing SUBMITTED transition:**
```sql
SELECT COUNT(*) FROM milestone_dod_items
  WHERE milestone_id = ? AND is_required = TRUE AND status != 'COMPLETED'
-- Returns > 0 → 422 DOD_INCOMPLETE, list of blocking items returned to expert
-- Returns 0 → proceed to milestone_submissions INSERT
```

**State transition evidence:**

| Transition | Trigger | Tables written |
|---|---|---|
| PENDING (initial) | CEO or expert creates DoD item | `milestone_dod_items` INSERT |
| PENDING → COMPLETED | Expert self-marks | `milestone_dod_items.status` UPDATE 'COMPLETED'; `milestone_dod_items.completed_at` SET; `milestone_dod_items.completion_note` SET (required) |
| PENDING → NOT_APPLICABLE | Expert marks N/A (non-required only) | `milestone_dod_items.status` UPDATE (DB CHECK prevents this on is_required=TRUE); `milestone_dod_items.not_applicable_note` SET |

**Per-item TECH_TEAM comments:**
```
dod_item_comments.dod_item_id UUID NOT NULL REFERENCES milestone_dod_items(id)
  -- Route guard: write = TECH_TEAM only (client_subtype='TECH_TEAM')
  -- CEO permanently excluded from reading this thread
```
The `dod_item_comments` table provides the TECH_TEAM-only annotation layer on each DoD item, separate from the engagement-level `messages` channel. This separation is intentional — DoD comments are technical quality notes, not contractual communications.

---

### 6.8 Sprint Status States

**State column:**
```
sprint_status_updates.status TEXT NOT NULL
  CHECK (status IN ('ON_TRACK','DEVIATION','BLOCKER'))
sprint_status_updates.blocker_type TEXT NULL
  CHECK (blocker_type IN ('SCOPE_GAP','TECH_CONSTRAINT','EXTERNAL_DEPENDENCY','SCOPE_EVOLUTION'))
sprint_status_updates.milestone_sprint_id UUID NOT NULL REFERENCES milestone_sprints(id)
  -- Direct FK replaces implicit (milestone_id, sprint_number) lookup
  -- Prevents orphan status updates referencing non-existent sprint plans
```

**State transition evidence:**

| Status | Trigger | Tables written | Side effects |
|---|---|---|---|
| ON_TRACK | Expert submits weekly update | `sprint_status_updates` INSERT | No milestone state change |
| DEVIATION | Expert reports deviation | `sprint_status_updates` INSERT (deviation_note required) | Notification to client; no clock pause |
| BLOCKER (non-SCOPE_EVOLUTION) | Expert reports technical blocker | `sprint_status_updates` INSERT; `milestones.review_clock_paused_at` UPDATE now() | TECH_TEAM notified; TECH_TEAM resolves → `milestones.total_paused_seconds` updated; `review_clock_paused_at` cleared |
| BLOCKER (SCOPE_EVOLUTION) | Expert reports scope evolution | `sprint_status_updates` INSERT (scope_evolution_flag=TRUE); `addon_phase_requests` INSERT; `milestones.review_clock_paused_at` UPDATE now() | F8 add-on flow triggered; unique index prevents double-trigger |

**Per-sprint TECH_TEAM comments:**
```
sprint_comments.milestone_sprint_id UUID NOT NULL REFERENCES milestone_sprints(id)
  -- Route guard: write = TECH_TEAM only
  -- Read = TECH_TEAM + EXPERT; CEO sees summary only
```
Separate from `messages` for the same reason as `dod_item_comments` — technical sprint feedback is distinct from contractual project communications.

---

### 6.9 Add-On Phase Request States

**State column:**
```
addon_phase_requests.status TEXT NOT NULL DEFAULT 'PENDING'
  CHECK (status IN ('PENDING','TECH_APPROVED','AWAITING_CEO','FULLY_APPROVED','REJECTED','MILESTONE_CREATED'))
addon_phase_requests.trigger_sprint_status_id UUID NULL REFERENCES sprint_status_updates(id)
addon_phase_requests.created_milestone_id UUID NULL REFERENCES milestones(id)
addon_phase_requests.tech_approved_at TIMESTAMPTZ NULL
addon_phase_requests.ceo_approved_at TIMESTAMPTZ NULL
addon_phase_requests.rejected_at TIMESTAMPTZ NULL
addon_phase_requests.rejected_by UUID NULL REFERENCES users(id)
```

**State transition evidence:**

| Transition | Trigger | Tables written |
|---|---|---|
| PENDING (initial) | NestJS auto-creates from SCOPE_EVOLUTION event | `addon_phase_requests` INSERT (trigger_sprint_status_id FK set) |
| PENDING → TECH_APPROVED | TECH_TEAM approves technical justification | `addon_phase_requests.status` UPDATE; `addon_phase_requests.tech_approved_at` SET; `addon_phase_requests.tech_approved_by` SET |
| TECH_APPROVED → FULLY_APPROVED | CEO approves budget | `addon_phase_requests.status` UPDATE; `addon_phase_requests.ceo_approved_at` SET; `addon_phase_requests.ceo_approved_by` SET |
| FULLY_APPROVED → MILESTONE_CREATED | System auto-creates new milestone | `milestones` INSERT (new milestone row); `addon_phase_requests.created_milestone_id` SET; `addon_phase_requests.status` UPDATE 'MILESTONE_CREATED'; `milestones.review_clock_paused_at` cleared on original milestone |
| Any → REJECTED | Either party rejects | `addon_phase_requests.rejected_at` SET; `addon_phase_requests.rejected_by` SET; `addon_phase_requests.rejection_reason` SET; original milestone clock resumed |

**The AWAITING_CEO status is a UI-facing label only** — there is no additional DB write between TECH_APPROVED and FULLY_APPROVED. The spec explicitly states "UI label only — no DB state change," which is why the CHECK constraint includes it but there is no trigger condition that sets it from a DB operation.

**Why `created_milestone_id` FK matters:** It creates the reverse audit link: given the newly created add-on milestone, you can answer "which scope evolution event caused this milestone to be created?" by joining `milestones.id → addon_phase_requests.created_milestone_id`. Without this, you would need timestamp proximity reasoning — fragile under concurrent inserts.

---

### 6.10 Wallet Transaction Types (Internal Ledger)

**Transaction type column:**
```
wallet_transactions.transaction_type TEXT NOT NULL
  CHECK (transaction_type IN ('TOP_UP','SUBSCRIPTION','ESCROW_LOCK','ESCROW_RELEASE',
                               'PLATFORM_FEE','ESCROW_REFUND','ESCROW_SPLIT','WITHDRAWAL'))
wallet_transactions.amount BIGINT NOT NULL CHECK (amount > 0)   -- always positive
wallet_transactions.reference_id TEXT NULL                        -- polymorphic idempotency key
```

**Balance invariant — every type modifies exactly two columns:**

| Type | Effect | Tables written |
|---|---|---|
| TOP_UP | `wallets.available_balance += amount` | `wallets` UPDATE + `wallet_transactions` INSERT |
| SUBSCRIPTION | `wallets.available_balance -= price` | `wallets` UPDATE + `wallet_transactions` INSERT + `user_subscriptions.status` UPDATE 'ACTIVE' + `users.subscription_*_tier` UPDATE |
| ESCROW_LOCK | `wallets.available_balance -= amount`; `wallets.locked_balance += amount` | Both `wallets` UPDATE columns + `wallet_transactions` INSERT + `escrow_accounts` INSERT (status='HELD') |
| ESCROW_RELEASE | `wallets.locked_balance -= amount` (client); `wallets.available_balance += net` (expert) | Both wallets UPDATE + `wallet_transactions` INSERT (ESCROW_RELEASE) + `wallet_transactions` INSERT (PLATFORM_FEE) |
| PLATFORM_FEE | `wallets.available_balance += fee` (platform wallet) | Platform wallet UPDATE + `wallet_transactions` INSERT |
| ESCROW_REFUND | `wallets.locked_balance -= amount` (client); `wallets.available_balance += amount` (client) | Client wallet UPDATE + `wallet_transactions` INSERT |
| ESCROW_SPLIT | `wallets.locked_balance -= amount` (client); both `wallets.available_balance += amount/2` | Both wallets UPDATE + two `wallet_transactions` INSERTs |
| WITHDRAWAL | `wallets.available_balance -= amount` | Expert wallet UPDATE + `wallet_transactions` INSERT + `withdrawal_requests` INSERT |

**Idempotency constraint — prevents SePay IPN double-credit:**
```sql
CREATE UNIQUE INDEX wallet_tx_idempotency
  ON wallet_transactions(wallet_id, reference_id)
  WHERE reference_id IS NOT NULL;
-- reference_id format: 'TOP_UP:{va_number}:{transfer_ref}'
-- Second IPN retry → INSERT fails unique constraint → caught and ignored safely
```
This is a DB-level uniqueness guarantee, not an application-level check. Even if two NestJS workers process the same IPN simultaneously, only one INSERT can succeed.

**Platform fee is read from DB, never hardcoded:**
```
platform_settings.platform_fee_pct FLOAT NOT NULL DEFAULT 0.05
  CHECK (platform_fee_pct BETWEEN 0 AND 1)
platform_settings.platform_wallet_id UUID NOT NULL UNIQUE REFERENCES wallets(id)
```
Every ESCROW_RELEASE transaction reads `SELECT platform_fee_pct, platform_wallet_id FROM platform_settings LIMIT 1` before computing the split. This means the fee rate can be changed without a code deploy.

---

### 6.11 Withdrawal States

**State column:**
```
withdrawal_requests.status TEXT NOT NULL DEFAULT 'PENDING'
  CHECK (status IN ('PENDING','PROCESSING','COMPLETED','FAILED'))
withdrawal_requests.type TEXT NOT NULL DEFAULT 'EXPERT_MANUAL'
  CHECK (type IN ('MILESTONE_RELEASE','EXPERT_MANUAL'))
withdrawal_requests.bank_account_xid TEXT NOT NULL  -- sourced from users.sepay_bank_account_xid
withdrawal_requests.disbursement_id TEXT NULL        -- set after chi hộ API responds
```

**State transition evidence:**

| Transition | Trigger | Tables written |
|---|---|---|
| PENDING (initial) | Expert requests withdrawal OR milestone APPROVED auto-trigger | `wallets.available_balance -= amount` (atomic with INSERT); `wallet_transactions` INSERT (WITHDRAWAL); `withdrawal_requests` INSERT (status='PENDING') |
| PENDING → PROCESSING | Chi hộ API responds with disbursement_id | `withdrawal_requests.disbursement_id` SET; `withdrawal_requests.status` UPDATE 'PROCESSING' |
| PROCESSING → COMPLETED | SePay credit IPN fires on expert's bank | `withdrawal_requests.status` UPDATE 'COMPLETED'; `withdrawal_requests.confirmed_at` SET |
| PENDING/PROCESSING → FAILED | Chi hộ API error or bounce IPN | `wallets.available_balance += amount` (reversal); `wallet_transactions` INSERT (WITHDRAWAL reversal); `withdrawal_requests.status` UPDATE 'FAILED' |

**Debit-first pattern prevents double-spend:** The wallet debit and the withdrawal_requests INSERT happen in the same `BEGIN/COMMIT` block, atomically, before the external SePay API call. If two withdrawal requests are submitted simultaneously, only one can decrement `available_balance` without violating the `CHECK (available_balance >= 0)` constraint on `wallets`.

**MILESTONE_RELEASE vs EXPERT_MANUAL distinction:** The `type` column allows the earnings dashboard to show "Milestone Release (auto)" separately from "Manual Cash-Out." MILESTONE_RELEASE rows are created automatically by NestJS when `milestones.state` transitions to APPROVED. EXPERT_MANUAL rows are created by the expert clicking "Withdraw."

---

### 6.12 Subscription States

**State column:**
```
user_subscriptions.status TEXT NOT NULL DEFAULT 'PENDING_PAYMENT'
  CHECK (status IN ('PENDING_PAYMENT','ACTIVE','EXPIRING_SOON','EXPIRED'))
user_subscriptions.role_type TEXT NOT NULL CHECK (role_type IN ('client','expert'))
```

**Mirror column on users (for JWT guard performance):**
```
users.subscription_client_tier TEXT NOT NULL DEFAULT 'free' CHECK (IN ('free','pro'))
users.subscription_expert_tier TEXT NOT NULL DEFAULT 'free' CHECK (IN ('free','pro'))
users.sub_client_expires_at TIMESTAMPTZ NULL
users.sub_expert_expires_at TIMESTAMPTZ NULL
```

**Why two tables?** `user_subscriptions` is the authoritative record with full payment history. `users.subscription_*_tier` and `users.sub_*_expires_at` are denormalized for JWT guard performance — every protected route checks `users.subscription_client_tier='pro' AND sub_client_expires_at > now()` rather than doing a JOIN to `user_subscriptions` on every request.

**State transition evidence:**

| Transition | Trigger | Tables written |
|---|---|---|
| PENDING_PAYMENT (initial) | User initiates subscription purchase | `user_subscriptions` INSERT (status='PENDING_PAYMENT'); `virtual_accounts` INSERT (entity_type='SUBSCRIPTION', entity_id=user_subscriptions.id, fixed_amount set) |
| PENDING_PAYMENT → ACTIVE | SePay IPN fires on SUBSCRIPTION VA | `wallets.available_balance -= price`; `wallet_transactions` INSERT (SUBSCRIPTION); `user_subscriptions.status` UPDATE 'ACTIVE'; `user_subscriptions.activated_at` SET; `user_subscriptions.expires_at` SET; `users.subscription_*_tier` UPDATE 'pro'; `users.sub_*_expires_at` UPDATE; `virtual_accounts.status` UPDATE 'USED' |
| ACTIVE → EXPIRING_SOON | Scheduled job: expires_at - 7 days | `user_subscriptions.status` UPDATE 'EXPIRING_SOON'; notification sent |
| EXPIRING_SOON/ACTIVE → EXPIRED | expires_at passes | `user_subscriptions.status` UPDATE 'EXPIRED'; `users.subscription_*_tier` UPDATE 'free' |
| EXPIRED → ACTIVE | User renews | New `user_subscriptions` INSERT + new `virtual_accounts` INSERT → same IPN cycle |

**SUBSCRIPTION VA entity_id design:** The VA is created with `entity_id = user_subscriptions.id` (the subscription row's UUID), not the user's ID. This allows the IPN handler to resolve directly to the exact subscription row without an ambiguous lookup — a user could have multiple subscription rows across role_types.

---

### 6.13 Dispute Resolution Layer States

**State column:**
```
disputes.state TEXT NOT NULL DEFAULT 'PENDING'
  CHECK (state IN ('PENDING','LAYER_1_EVAL','AUTO_RESOLVED','LAYER_2_MUTUAL',
                   'MUTUAL_RESOLVED','LAYER_3_FORCED','DISPUTED_AUTO_RESOLVED'))
disputes.llm_confidence FLOAT NULL CHECK (llm_confidence BETWEEN 0 AND 1)
disputes.resolution_layer INT NULL CHECK (resolution_layer IN (1,2,3))
disputes.layer2_deadline TIMESTAMPTZ NULL    -- 48h window set when Layer 1 fails
disputes.escrow_account_id UUID NOT NULL REFERENCES escrow_accounts(id)
disputes.criterion_id UUID NOT NULL REFERENCES acceptance_criteria(id)
```

**State transition evidence:**

| Transition | Trigger | Tables written |
|---|---|---|
| LAYER_1_EVAL (initial) | Party files dispute | `disputes` INSERT (state='LAYER_1_EVAL'); `escrow_accounts.status` UPDATE 'FROZEN' |
| LAYER_1_EVAL → AUTO_RESOLVED | FastAPI LLM confidence ≥ 0.80 | `disputes.state` UPDATE; `disputes.resolution_layer` SET 1; `disputes.llm_confidence` SET; escrow release ledger entries; `milestones.state` UPDATE 'RELEASED' |
| LAYER_1_EVAL → LAYER_2_MUTUAL | FastAPI LLM confidence < 0.80 | `disputes.state` UPDATE; `disputes.layer2_deadline` SET (now()+48h); `disputes.llm_confidence` SET |
| LAYER_2_MUTUAL → MUTUAL_RESOLVED | Both parties agree same option within 48h | `disputes.state` UPDATE; `disputes.resolution_layer` SET 2; custom split ledger entries |
| LAYER_2_MUTUAL → LAYER_3_FORCED | 48h expires without agreement | `disputes.state` UPDATE (triggered by scheduled job checking `layer2_deadline`) |
| LAYER_3_FORCED → DISPUTED_AUTO_RESOLVED | Automatic | `disputes.state` UPDATE; `disputes.resolution_layer` SET 3; 50/50 split ledger entries; `milestones.state` UPDATE 'RELEASED' |

**Why `disputes.escrow_account_id` is a direct FK (not derived via milestone):**
```
disputes.escrow_account_id UUID NOT NULL REFERENCES escrow_accounts(id)
-- O(1) lookup for freeze/release regardless of engagement type
-- Works identically for PROJECT_BASED (escrow.milestone_id set) and SERVICE_PURCHASE (escrow.engagement_id set)
-- Without this FK: dispute handler would need to JOIN through milestone OR engagement to find escrow
```

**Why `disputes.criterion_id` is required:**
```
disputes.criterion_id UUID NOT NULL REFERENCES acceptance_criteria(id)
-- Disputes must always target a specific, objective contract element
-- Prevents "I am generally unhappy" disputes with no measurable criterion
-- Layer 1 LLM evaluation is scoped to the specific criterion text vs deliverable
```

**Escrow one-parent constraint — prevents invalid escrow records:**
```sql
CONSTRAINT escrow_has_one_parent CHECK (
  (milestone_id IS NOT NULL AND engagement_id IS NULL) OR
  (milestone_id IS NULL AND engagement_id IS NOT NULL)
)
```
This constraint ensures that every `escrow_accounts` row is unambiguously associated with either a milestone (PROJECT_BASED path) or an engagement (SERVICE_PURCHASE path), never both or neither. The `disputes.escrow_account_id` FK therefore always resolves to a well-formed escrow record.

---

## Part 7 — RBAC Constraints → Schema Backing (Section 0.7)

Every RBAC rule from Section 0.7 has a DB-backed enforcement point, not just an application-level guard.

### Role storage

```
users.roles JSONB NOT NULL DEFAULT '[]'         -- e.g. '["CLIENT_CEO","EXPERT"]'
users.active_role TEXT NOT NULL
  CHECK (active_role IN ('CLIENT','EXPERT','ADMIN'))
users.client_subtype TEXT NULL
  CHECK (client_subtype IN ('CEO','TECH_TEAM'))  -- NULL for EXPERT and ADMIN
```

### TECH_TEAM cannot self-register

Enforced at the application level (no self-registration route exists for TECH_TEAM), backed by:
```
tech_team_profiles.linked_client_id UUID NOT NULL REFERENCES users(id)
tech_team_profiles.linked_project_id UUID NOT NULL REFERENCES projects(id)
```
A TECH_TEAM account without a corresponding `tech_team_profiles` row pointing to a valid project is structurally incomplete — every route that requires TECH_TEAM functionality checks `tech_team_profiles.linked_project_id` against the requested project.

### TECH_TEAM scope guard

```
tech_team_profiles.linked_project_id UUID NOT NULL REFERENCES projects(id)
-- Route guard: tech_team_profiles.linked_project_id = requested_project_id
-- TECH_TEAM cannot access bid reviews, Artifact B, or milestone sign-offs for any other project
```

### CEO never sees Artifact B

Enforced in the FastAPI access gate query — no CEO branch exists in the `EXISTS` subquery. No application code path can grant CEO access without rewriting the query.

### Expert Pro required for Tier 2+ bidding

```
users.subscription_expert_tier TEXT NOT NULL DEFAULT 'free'
users.sub_expert_expires_at TIMESTAMPTZ NULL
-- Guard at bid submission: subscription_expert_tier='pro' AND sub_expert_expires_at > now()
-- Also: projects.tier = 'TIER_2' or 'TIER_3' → requires pro subscription to bid
```

### Bank Hub link required before chi hộ

```
users.sepay_bank_account_xid TEXT NULL
-- Guard at CONNECTED transition: expert must have sepay_bank_account_xid IS NOT NULL
-- Guard at withdrawal: bank_account_xid required in withdrawal_requests
```

### is_active = FALSE is the only suspension mechanism

```
users.is_active BOOLEAN NOT NULL DEFAULT TRUE
-- No is_suspended column exists in schema
-- Every JWT guard reads users.is_active = TRUE on every protected request
-- Setting is_active=FALSE immediately blocks all route access on the next request
```

---

## Part 8 — Payment Architecture → Schema Backing (Section 0.8)

### Virtual account polymorphism

```
virtual_accounts.entity_type TEXT NOT NULL
  CHECK (entity_type IN ('WALLET_TOPUP','MILESTONE','SERVICE','SUBSCRIPTION'))
virtual_accounts.entity_id TEXT NOT NULL            -- polymorphic; no DB-level FK
virtual_accounts.fixed_amount BIGINT NULL           -- NULL for WALLET_TOPUP; SET for all others
virtual_accounts.expires_at TIMESTAMPTZ NULL        -- NULL for WALLET_TOPUP (permanent)
virtual_accounts.status TEXT NOT NULL DEFAULT 'ACTIVE'
  CHECK (status IN ('ACTIVE','EXPIRED','USED'))
```

The `entity_type` field is the router: the IPN handler reads it first to determine which table and column to look up with `entity_id`. This is why there is no DB-level FK on `entity_id` — a polymorphic FK would require four separate REFERENCES clauses, which PostgreSQL does not support. The application contract (entity_type → target table mapping) substitutes for referential integrity here.

**Per-entity_type entity_id mapping:**
| entity_type | entity_id resolves to | Purpose |
|---|---|---|
| WALLET_TOPUP | `users.id` | IPN → credit user's wallet |
| MILESTONE | `milestones.id` | IPN → FUNDED transition + escrow lock |
| SERVICE | `engagements.id` | IPN → SERVICE_PURCHASE ACTIVE transition |
| SUBSCRIPTION | `user_subscriptions.id` | IPN → ACTIVE subscription transition |

### Escrow dual-path support

```
escrow_accounts.milestone_id UUID NULL REFERENCES milestones(id)
escrow_accounts.engagement_id UUID NULL REFERENCES engagements(id)
CONSTRAINT escrow_has_one_parent CHECK (
  (milestone_id IS NOT NULL AND engagement_id IS NULL) OR
  (milestone_id IS NULL AND engagement_id IS NOT NULL)
)
CREATE UNIQUE INDEX escrow_milestone_unique ON escrow_accounts(milestone_id) WHERE milestone_id IS NOT NULL;
CREATE UNIQUE INDEX escrow_engagement_unique ON escrow_accounts(engagement_id) WHERE engagement_id IS NOT NULL;
```

Two partial unique indexes instead of one full unique index enables: one escrow record per milestone on Path A, one escrow record per engagement on Path B, while the CHECK constraint prevents any escrow from being parentless or double-parented.

---

## Part 9 — Path A vs Path B Structural Differences → Schema Backing

### Path A (Project-Based) structural requirements in schema

| Requirement | Schema enforcement |
|---|---|
| Elicitation session must exist | `projects.elicitation_session_id REFERENCES elicitation_sessions(id)` |
| Capability footprint must be populated | `capability_footprints` — 1:1 with projects, populated by synthesis engine |
| Artifact A must be published | `artifact_a.state = 'PUBLISHED'` before matching engine runs |
| Artifact B is gated | `artifact_b` access guard — NDA timestamps + engagement state |
| Bid system required | `engagements` INSERT + `capability_bids` INSERT + bid state machine |
| Multi-milestone escrow | `escrow_accounts.milestone_id IS NOT NULL` (one per milestone) |
| Tech team handoff (Scenario A) | `tech_team_profiles.linked_project_id` + `artifact_b` population |

### Path B (Service Marketplace) structural requirements in schema

| Requirement | Schema enforcement |
|---|---|
| No elicitation required | `engagements.project_id IS NULL` enforced by type CHECK constraint |
| Service must exist and be published | `services.state='PUBLISHED'` + `services.price_vnd > 0` |
| Direct engagement creation in ACTIVE | `engagements` created with state='ACTIVE' (SERVICE_PURCHASE bypasses PENDING/CONNECTED) |
| Per-service-order escrow | `escrow_accounts.engagement_id IS NOT NULL` (one per engagement) |
| No bid system | `capability_bids` has no row for SERVICE_PURCHASE engagements |
| Free-tier clients can buy | No subscription guard on SERVICE_PURCHASE route |

### Engagement type immutability

```
engagements.type TEXT NOT NULL CHECK (type IN ('PROJECT_BASED','SERVICE_PURCHASE','TECH_DISCOVERY'))
-- No UPDATE path exists for this column after creation
-- The type-to-FK CHECK constraint enforces structural consistency:
CONSTRAINT engagement_type_fk_consistency CHECK (
  (type = 'PROJECT_BASED' AND project_id IS NOT NULL AND service_id IS NULL) OR
  (type IN ('SERVICE_PURCHASE','TECH_DISCOVERY') AND project_id IS NULL AND service_id IS NOT NULL)
)
```

---

## Part 10 — FastAPI vs NestJS Write Boundaries → Schema Backing

The system has a clear and auditable boundary: FastAPI writes to `platform_decisions`; NestJS never writes to that table.

### Tables written exclusively by FastAPI

| Table | Written by FastAPI when |
|---|---|
| `platform_decisions` | Every LLM evaluation result (elicitation synthesis, spec auto-return, portfolio eval, scenario eval, dispute L1 eval, criterion quality gate, seam tier upgrade, CEO override log) |

### Why `platform_decisions.entity_type` is required

```
platform_decisions.entity_type TEXT NOT NULL
  CHECK (entity_type IN ('project','capability_bid','acceptance_criterion',
                         'portfolio_submission','scenario_response','spec_clarification','service'))
platform_decisions.entity_id TEXT NOT NULL  -- polymorphic UUID stored as TEXT; no DB FK
```

Without `entity_type`, the Platform Integrity Monitor (MF-17) cannot determine which table to JOIN when rendering the detail view for a decision. For example, `decision_type='PORTFOLIO_EVAL'` with `entity_type='portfolio_submission'` → JOIN `portfolio_submissions` → JOIN `expert_seam_claims` to get the seam code and outcome. If `entity_type` were absent, this JOIN chain would require inspecting `decision_type` string matching — fragile and non-atomic.

### Tables written exclusively by NestJS (not FastAPI)

All operational state tables: `engagements`, `milestones`, `wallets`, `wallet_transactions`, `capability_bids`, `bid_versions`, `bid_revision_requests`, `expert_seam_claims` (tier UPDATE on FastAPI signal return), `withdrawal_requests`, `escrow_accounts`, `sprint_status_updates`, `addon_phase_requests`, `reviews`, `outcome_signals`, `expert_seam_outcome_signals`.

---

## Part 11 — Fraud Detection & Account Integrity → Schema Backing (MF-18)

### Bank account deduplication

```
users.sepay_bank_account_xid TEXT NULL UNIQUE
-- UNIQUE constraint: no two users can hold the same bank_account_xid
-- Deduplication check at Bank Hub webhook:
SELECT id FROM users WHERE sepay_bank_account_xid = :incoming_xid AND id != :current_user_id
-- If row found → auto-block both accounts
```

### Suspension mechanism

```
users.is_active BOOLEAN NOT NULL DEFAULT TRUE
-- UPDATE users SET is_active=FALSE WHERE id = target_user_id
-- Immediate effect: every JWT guard returns 401 on next protected request
-- Wallets and escrow balances are NOT touched by suspension
-- Active engagements continue after reactivation
```

### Fraud audit trail

```
platform_decisions INSERT (decision_type='CONTENT_MODERATION',
                           entity_id=user_id,
                           decision='FRAUD_AUTO_BLOCK' or 'ADMIN_SUSPENSION' or 'ADMIN_REACTIVATION',
                           advisory_note='reason')
```

---

## Summary Table — Every Master Reference Concept → Its Schema Home

| Concept (Section) | Primary table | Enforcing column/constraint |
|---|---|---|
| Six domains (0.1) | `expert_domain_depths` | `domain_code`, UNIQUE `(expert_id, domain_code)` |
| Domain depth levels (0.1) | `expert_domain_depths` | `depth_level CHECK ('SURFACE','OPERATIONAL','DEEP')` |
| Project domain requirements (0.1) | `capability_footprints` | `required_domains_json JSONB` |
| Ten seams (0.2) | `expert_seam_claims` | `seam_code`, UNIQUE `(expert_id, seam_code)` |
| Seam criticality (0.2) | `capability_footprints` | `required_seams_json` (criticality in JSON value) |
| Archetype (0.3) | `elicitation_sessions`, `projects` | `archetype TEXT` columns |
| Tier (0.3) | `projects` | `tier CHECK ('TIER_1','TIER_2','TIER_3')` |
| Load-bearing seam (0.3) | `capability_footprints` | `required_seams_json` criticality:'LOAD_BEARING' |
| Verification tiers (0.4) | `expert_seam_claims` | `verification_tier CHECK (4 values)` |
| 5-attempt throttle (0.4) | `expert_seam_claims` | `submission_count`, `locked_until` |
| 30-day lockout (0.4) | `expert_seam_claims` | `locked_until TIMESTAMPTZ NULL` |
| Tier 2 LLM threshold ≥ 0.85 (0.4) | `portfolio_submissions` | `llm_confidence FLOAT NULL` |
| Tier 3 per-dimension rubric (0.4) | `scenario_responses` | `llm_rubric_scores_json JSONB` |
| Tier 4 signal accumulation (0.4) | `expert_seam_outcome_signals` | `signal_type`, `valence`, seam_claim_id FK |
| Composite score Component 1 (0.5) | `expert_seam_claims`, `capability_footprints` | seam_code + verification_tier + required_seams_json |
| Composite score Component 2 (0.5) | `expert_domain_depths`, `capability_footprints` | depth_level + required_domains_json |
| Composite score Component 3 (0.5) | `expert_profiles`, `engagements`, `projects` | `archetype_history_json`, `projects.archetype` |
| Composite score Component 4 (0.5) | `expert_profiles` | `engagement_model CHECK (3 values)` |
| Composite score Component 5 (0.5) | `expert_stack_tags` | `tag`, UNIQUE `(expert_id, tag)` |
| Hard gate: 4:1 ratio (0.5) | `expert_seam_claims` | `verification_tier` aggregated |
| Hard gate: 2× negative signals (0.5) | `expert_seam_outcome_signals` | `valence='NEGATIVE'` count |
| Self-exclusion (0.5) | `projects`, `tech_team_profiles` | `client_id`, `linked_project_id` |
| Elicitation session states (0.6) | `elicitation_sessions` | `state CHECK (4 values)`, `current_stage CHECK (1-5)` |
| Void tracking (0.6) | `elicitation_sessions` | `void_list_json JSONB` |
| Spec states (0.6) | `projects`, `artifact_a` | `state CHECK (4 values)` — both columns |
| Auto-publish quality gate (0.6) | `capability_footprints`, `expert_seam_claims` | JSON completeness + matching pre-check SQL |
| Bid states (0.6) | `capability_bids` | `state CHECK (14 values)` |
| Bid versioning (0.6) | `bid_versions` | `revision_request_id FK NULL`, UNIQUE `(bid_id, version_number)` |
| Price negotiation cap (0.6) | `price_negotiations` | `round_number CHECK (IN (1,2))`, UNIQUE `(bid_id, round_number)` |
| CEO override justification (0.6) | `bid_conflict_overrides` | `char_length(override_reason) >= 50`, `bid_id UNIQUE` |
| Spec clarification states (0.6) | `spec_clarifications` | `status CHECK ('OPEN','ANSWERED','RESOLVED')` — 'CLOSED' excluded |
| Engagement states (0.6) | `engagements` | `state CHECK (5 values)`, `client_nda_accepted_at`, `expert_nda_accepted_at` |
| Engagement type immutability (0.6) | `engagements` | `type CHECK` + table-level `engagement_type_fk_consistency` |
| Artifact B NDA gate (0.6) | `engagements` | `client_nda_accepted_at IS NOT NULL AND expert_nda_accepted_at IS NOT NULL` |
| SERVICE_PURCHASE skip PENDING (0.6) | `engagements` | State created as ACTIVE; type CHECK enforces FK pattern |
| Milestone states (0.6) | `milestones` | `state CHECK (9 values)` |
| Review clock pause (0.6) | `milestones` | `review_clock_paused_at TIMESTAMPTZ NULL`, `total_paused_seconds INT` |
| DoD gate on SUBMITTED (0.6) | `milestone_dod_items` | `is_required`, `status != 'COMPLETED'` COUNT guard |
| DoD NOT_APPLICABLE blocked on required (0.6) | `milestone_dod_items` | `CHECK (NOT (is_required=TRUE AND status='NOT_APPLICABLE'))` |
| Criteria gate on APPROVED (0.6) | `acceptance_criteria` | `is_required`, `verified_at IS NULL` COUNT guard |
| Platform fee from DB (0.6) | `platform_settings` | `platform_fee_pct FLOAT`, `platform_wallet_id FK` |
| Escrow one-parent rule (0.6) | `escrow_accounts` | `CONSTRAINT escrow_has_one_parent CHECK` |
| Sprint status types (0.6) | `sprint_status_updates` | `status CHECK ('ON_TRACK','DEVIATION','BLOCKER')` |
| BLOCKER clock pause (0.6) | `sprint_status_updates`, `milestones` | `blocker_type`, `review_clock_paused_at` |
| SCOPE_EVOLUTION auto-trigger (0.6) | `sprint_status_updates`, `addon_phase_requests` | `scope_evolution_flag BOOLEAN`, UNIQUE index on `trigger_sprint_status_id` |
| Add-on phase states (0.6) | `addon_phase_requests` | `status CHECK (6 values)`, dual approval timestamps |
| Add-on milestone reverse link (0.6) | `addon_phase_requests` | `created_milestone_id UUID NULL REFERENCES milestones(id)` |
| Wallet transaction types (0.6) | `wallet_transactions` | `transaction_type CHECK (8 values)` |
| IPN idempotency (0.6) | `wallet_transactions` | UNIQUE INDEX `(wallet_id, reference_id) WHERE reference_id IS NOT NULL` |
| Withdrawal states (0.6) | `withdrawal_requests` | `status CHECK (4 values)` |
| Debit-first pattern (0.6) | `wallets`, `withdrawal_requests` | Atomic BEGIN/COMMIT: wallet UPDATE before chi hộ API call |
| Subscription states (0.6) | `user_subscriptions` | `status CHECK (4 values)` incl. PENDING_PAYMENT |
| Subscription mirror for guard (0.6) | `users` | `subscription_*_tier`, `sub_*_expires_at` denormalized columns |
| SUBSCRIPTION VA entity_id (0.6) | `virtual_accounts` | `entity_id = user_subscriptions.id` (not user_id) |
| Dispute resolution layers (0.6) | `disputes` | `state CHECK (7 values)`, `resolution_layer CHECK (1,2,3)`, `layer2_deadline` |
| Dispute criterion targeting (0.6) | `disputes` | `criterion_id UUID NOT NULL REFERENCES acceptance_criteria(id)` |
| Escrow freeze on dispute (0.6) | `escrow_accounts` | `status='FROZEN'` written on disputes INSERT |
| Layer 2 48h deadline (0.6) | `disputes` | `layer2_deadline TIMESTAMPTZ NULL` set on Layer 2 entry |
| RBAC: roles + active_role (0.7) | `users` | `roles JSONB`, `active_role CHECK`, `client_subtype CHECK` |
| RBAC: TECH_TEAM scope lock (0.7) | `tech_team_profiles` | `linked_project_id REFERENCES projects(id)` |
| RBAC: account suspension (0.7) | `users` | `is_active BOOLEAN` — only suspension mechanism |
| Payment: VA polymorphism (0.8) | `virtual_accounts` | `entity_type CHECK (4 values)`, `entity_id TEXT` |
| Payment: chi hộ automation (0.8) | `withdrawal_requests` | `bank_account_xid`, `disbursement_id`, `type` field |
| Fraud: bank dedup (0.7/0.8) | `users` | `sepay_bank_account_xid TEXT NULL UNIQUE` |
| FastAPI-only audit log (design) | `platform_decisions` | `decision_type`, `entity_type`, `entity_id` — never written by NestJS |
