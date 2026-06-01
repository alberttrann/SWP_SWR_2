## AITasker Conceptual ER — Unified Draw Plan

*(Merges: original 10-phase plan + Amendment Series A + Amendment Series B. B4 and B5 are absorbed — cardinality was already correct and the signal→claim relationship is covered in Phase 2.)*

---

## Entities to Draw — 51 Total

**Role & auth:** `users` · `client_profiles` · `expert_profiles` · `tech_team_profiles`

**Expert capability:** `expert_domain_depths` · `expert_seam_claims` · `expert_stack_tags` · `portfolio_submissions` · `scenario_assessments` · `scenario_responses` · `expert_seam_outcome_signals`

**Finance:** `wallets` · `wallet_transactions` · `virtual_accounts` · `user_subscriptions` · `withdrawal_requests` · `platform_settings`

**Org:** `organizations` · `organization_members`

**Elicitation & projects:** `elicitation_sessions` · `projects` · `capability_footprints` · `artifact_a` · `artifact_b`

**Services:** `services`

**Engagements & bids:** `engagements` · `spec_clarifications` · `capability_bids` · `bid_versions` · `bid_revision_requests` · `price_negotiations` · `bid_conflict_overrides`

**Milestones:** `milestones` · `acceptance_criteria` · `revision_requests` · `milestone_dod_items` · `milestone_dod_items` · `milestone_sprints` · `sprint_status_updates` · `milestone_submissions` · `paygated_documents` · `escrow_accounts` · `addon_phase_requests`

**Sprint & DoD comments:** `sprint_comments` · `dod_item_comments`

**Disputes:** `disputes` · `dispute_resolution_reports`

**Messaging & reads:** `messages` · `message_reads`

**Reviews & signals:** `reviews` · `outcome_signals`

**Platform:** `platform_decisions`

---

## Phase 0 — Elicitation Engine

Draw `elicitation_sessions` positioned between `users` and `projects` on the canvas. It will chain into `projects` in Phase 5.

- `users` 💠 `initiates` ➔ `elicitation_sessions` **(1:N)** — CEO initiates sessions; one session per Path A attempt

---

## Phase 1 — Users & Role Profile Subtypes

Draw `users` as the central anchor entity for the entire diagram.

- `users` 💠 `has` ➔ `client_profiles` **(1:1)**
- `users` 💠 `has` ➔ `expert_profiles` **(1:1)**
- `users` 💠 `has` ➔ `tech_team_profiles` **(1:1)**
- `users` 💠 `invited` ➔ `tech_team_profiles` **(1:N)** — via `linked_client_id`; CEO who issued the handoff link

*(The fifth relationship `projects → scopes → tech_team_profiles` is drawn in Phase 5 when `projects` exists.)*

---

## Phase 2 — Expert Capability Map

All branch from `expert_profiles`. Draw `expert_seam_outcome_signals` here too — it lives in the expert cluster but connects back to `outcome_signals` in Phase 10.

- `expert_profiles` 💠 `has` ➔ `expert_domain_depths` **(1:N)**
- `expert_profiles` 💠 `holds` ➔ `expert_seam_claims` **(1:N)**
- `expert_profiles` 💠 `has` ➔ `expert_stack_tags` **(1:N)**
- `expert_profiles` 💠 `submits` ➔ `portfolio_submissions` **(1:N)**
- `expert_profiles` 💠 `submits` ➔ `scenario_responses` **(1:N)**
- `scenario_assessments` 💠 `receives` ➔ `scenario_responses` **(1:N)**
- `expert_seam_claims` 💠 `updated by` ➔ `expert_seam_outcome_signals` **(1:N)** — via `seam_claim_id`; Tier 4 accumulation write-back
- `expert_profiles` 💠 `has` ➔ `expert_seam_outcome_signals` **(1:N)**

*(Two more lines into `expert_seam_outcome_signals` — from `engagements` and `outcome_signals` — are drawn in Phases 7 and 10 respectively when those entities exist.)*

---

## Phase 3 — Wallet, Finance & Subscriptions

Draw `wallets` beside `users`. Draw `platform_settings` as a singleton near the wallet cluster with a note: *"one row seeded at deploy"*.

- `users` 💠 `owns` ➔ `wallets` **(1:1)**
- `wallets` 💠 `records` ➔ `wallet_transactions` **(1:N)**
- `users` 💠 `has` ➔ `user_subscriptions` **(1:N)** — 1:N because a dual-role user holds two rows, one per role_type
- `expert_profiles` 💠 `creates` ➔ `withdrawal_requests` **(1:N)**
- `platform_settings` 💠 `references` ➔ `wallets` **(1:1)** — the seeded platform fee wallet row

*(The `virtual_accounts` relationship is drawn in Phase 9 when `milestones` exists.)*

---

## Phase 4 — Organizations

- `users` 💠 `owns` ➔ `organizations` **(1:N)** — via `owner_user_id`
- `organizations` 💠 `has` ➔ `organization_members` **(1:N)**
- `users` 💠 `joins` ➔ `organization_members` **(1:N)**

---

## Phase 5 — Projects & Dual Artifacts

Draw `projects` centrally — it is the second major hub of the diagram.

- `users` 💠 `creates` ➔ `projects` **(1:N)** — via `client_id`; CEO only
- `elicitation_sessions` 💠 `produces` ➔ `projects` **(1:1)** — Stage 5 synthesis output; primary Path A route
- `projects` 💠 `scopes` ➔ `tech_team_profiles` **(1:N)** — via `linked_project_id`; completes the handoff link chain from Phase 1
- `projects` 💠 `has` ➔ `capability_footprints` **(1:1)**
- `capability_footprints` 💠 `drives` ➔ `artifact_a` **(1:1)** — synthesis engine constructs Artifact A from the footprint
- `projects` 💠 `has` ➔ `artifact_b` **(1:1)** — data parent is always Project; access is gated by engagement state

*(The `engagements → gates access to → artifact_b` line is drawn in Phase 7 when `engagements` exists.)*

---

## Phase 6 — Services (Path B)

Draw `services` beside `expert_profiles`.

- `expert_profiles` 💠 `creates` ➔ `services` **(1:N)**

*(The `services → generates → engagements` line is drawn in Phase 7.)*

---

## Phase 7 — Engagements

Draw `engagements` as the second major hub. Five entities feed into it; two access gate lines close back to Phase 5.

**Inbound feeds:**
- `projects` 💠 `has` ➔ `engagements` **(1:N)** — Path A; `project_id` nullable for Path B
- `expert_profiles` 💠 `joins` ➔ `engagements` **(1:N)**
- `organizations` 💠 `participates in` ➔ `engagements` **(1:N)** — via `organization_id`; nullable
- `services` 💠 `generates` ➔ `engagements` **(1:N)** — Path B only; `service_id` nullable for Path A

**Artifact B access gate** (closes Phase 5):
- `engagements` 💠 `gates access to` ➔ `artifact_b` **(N:1)** — annotate: *"state ≥ CONNECTED + NDA accepted; CEO permanently excluded"*

**Surface A — Spec clarifications:**
- `artifact_a` 💠 `receives` ➔ `spec_clarifications` **(1:N)** — pre-bid questions filed against Artifact A
- `expert_profiles` 💠 `files` ➔ `spec_clarifications` **(1:N)** — who asked
- `users` 💠 `responds to` ➔ `spec_clarifications` **(1:N)** — via `responded_by`; TECH_TEAM on Tier 2+ projects, CEO on Tier 1

**Expert seam outcome signals** (closes Phase 2):
- `engagements` 💠 `has` ➔ `expert_seam_outcome_signals` **(1:N)**

**Escrow Path B** (SERVICE_PURCHASE only — no milestone exists at IPN time):
- `engagements` 💠 `holds escrow for` ➔ `escrow_accounts` **(1:1)** — annotate: *"SERVICE_PURCHASE only"*

---

## Phase 8 — Non-Linear Bid System (Surfaces A–D)

Draw `capability_bids` branching from `engagements`.

**Core bid:**
- `engagements` 💠 `has` ➔ `capability_bids` **(1:1)** — one bid per engagement; absent for SERVICE_PURCHASE

**Surface B — Revision chain:**
- `capability_bids` 💠 `has` ➔ `bid_versions` **(1:N)** — immutable snapshot on every REVISED transition
- `capability_bids` 💠 `receives` ➔ `bid_revision_requests` **(1:N)** — TECH_TEAM flags one component
- `bid_revision_requests` 💠 `produces` ➔ `bid_versions` **(1:1)** — each request produces one version snapshot
- `users` 💠 `requests` ➔ `bid_revision_requests` **(1:N)** — via `requested_by`
- `users` 💠 `authors` ➔ `bid_versions` **(1:N)** — via `reviser_id`

**Surface C — Price negotiation:**
- `capability_bids` 💠 `has` ➔ `price_negotiations` **(1:N)** — max 2 rounds; bounded by CHECK constraint

**Surface D — CEO conflict override:**
- `capability_bids` 💠 `has` ➔ `bid_conflict_overrides` **(1:N)**
- `users` 💠 `files override` ➔ `bid_conflict_overrides` **(1:N)** — via `ceo_user_id`; CEO who bypassed TECH disapproval
- `bid_conflict_overrides` 💠 `logged in` ➔ `platform_decisions` **(1:1)** — Platform Integrity Monitor feed

---

## Phase 9 — Milestone System (3 Layers) + Add-On + Disputes

Draw `milestones` branching from `engagements`. This is the most complex phase — work layer by layer.

**Engagement → Milestones:**
- `engagements` 💠 `has` ➔ `milestones` **(1:N)**

---

**Layer 1 — Acceptance Criteria (payment contract):**
- `milestones` 💠 `has` ➔ `acceptance_criteria` **(1:N)** — all required criteria must have `verified_at` set before APPROVED
- `acceptance_criteria` 💠 `has` ➔ `revision_requests` **(1:N)**
- `users` 💠 `files` ➔ `revision_requests` **(1:N)** — via `requested_by`
- `acceptance_criteria` 💠 `evaluation logged in` ➔ `platform_decisions` **(1:1)** — LLM quality gate on criterion save; subjective language flagged and returned

---

**Layer 2 — DoD Checklist (gates SUBMITTED):**
- `milestones` 💠 `has` ➔ `milestone_dod_items` **(1:N)** — all required items must reach COMPLETED before submission is permitted
- `milestone_dod_items` 💠 `maps to` ➔ `acceptance_criteria` **(N:1)** — via `maps_to_criterion_id`; nullable; optional upward link
- `milestone_dod_items` 💠 `has` ➔ `dod_item_comments` **(1:N)** — TECH_TEAM per-DoD-item comment thread; separate from main channel
- `users` 💠 `writes` ➔ `dod_item_comments` **(1:N)** — TECH_TEAM only at route level

---

**Layer 3 — Sprint Plan & Status:**
- `milestones` 💠 `has` ➔ `milestone_sprints` **(1:N)** — temporal decomposition; no payment consequence
- `milestone_sprints` 💠 `has` ➔ `sprint_comments` **(1:N)** — TECH_TEAM per-sprint comment thread
- `users` 💠 `writes` ➔ `sprint_comments` **(1:N)** — TECH_TEAM only at route level
- `milestone_sprints` 💠 `receives` ➔ `sprint_status_updates` **(1:N)** — via `milestone_sprint_id`; direct FK replacing implicit sprint_number link
- `users` 💠 `submits` ➔ `sprint_status_updates` **(1:N)** — EXPERT only

---

**Add-On Phase (F8 Scope Evolution):**
- `sprint_status_updates` 💠 `triggers` ➔ `addon_phase_requests` **(1:1)** — annotate: *"SCOPE_EVOLUTION blocker only"*; UNIQUE constraint on trigger side
- `engagements` 💠 `has` ➔ `addon_phase_requests` **(1:N)**
- `users` 💠 `submits` ➔ `addon_phase_requests` **(1:N)** — via `submitted_by`; EXPERT files the brief
- `users` 💠 `tech approves` ➔ `addon_phase_requests` **(1:N)** — via `tech_approved_by`; independent approval
- `users` 💠 `ceo approves` ➔ `addon_phase_requests` **(1:N)** — via `ceo_approved_by`; independent approval

---

**Deliverables & Pay-gated Documents:**
- `milestones` 💠 `has` ➔ `milestone_submissions` **(1:N)**
- `expert_profiles` 💠 `submits` ➔ `milestone_submissions` **(1:N)** — via `expert_id`
- `milestones` 💠 `triggers release of` ➔ `paygated_documents` **(1:N)** — `release_state` flips STAGED→RELEASED when milestone IPN fires; TECH_TEAM-only read

---

**Escrow (closes Phase 7 Path B line):**
- `milestones` 💠 `held in` ➔ `escrow_accounts` **(1:1)** — PROJECT_BASED path; one escrow per milestone
- `wallets` 💠 `funds` ➔ `escrow_accounts` **(1:N)** — via `client_wallet_id`
- `wallets` 💠 `receives from` ➔ `escrow_accounts` **(1:N)** — via `expert_wallet_id`
- `milestones` 💠 `allocates` ➔ `virtual_accounts` **(1:N)** — per-milestone VA for VietQR payment

---

**Disputes:**
- `engagements` 💠 `has` ➔ `disputes` **(1:N)**
- `milestones` 💠 `has` ➔ `disputes` **(1:N)**
- `acceptance_criteria` 💠 `subject of` ➔ `disputes` **(1:N)** — disputes always filed against a specific criterion
- `users` 💠 `files` ➔ `disputes` **(1:N)** — CEO, TECH_TEAM, or EXPERT can all file
- `disputes` 💠 `freezes` ➔ `escrow_accounts` **(1:1)** — escrow.status → FROZEN immediately on filing
- `disputes` 💠 `has` ➔ `dispute_resolution_reports` **(1:N)**
- `users` 💠 `files` ➔ `dispute_resolution_reports` **(1:N)** — the party who felt the Layer 3 resolution was unfair

---

## Phase 10 — Messaging, Reviews & Platform Decisions

**Messaging:**
- `engagements` 💠 `has` ➔ `messages` **(1:N)** — direct `engagement_id` FK; no thread table
- `users` 💠 `sends` ➔ `messages` **(1:N)** — via `sender_id`; any of the three transactional roles
- `messages` 💠 `has` ➔ `message_reads` **(1:N)** — per-user read record
- `users` 💠 `generates` ➔ `message_reads` **(1:N)** — one row per user × message pair; enables per-role unread badge

**Reviews & Outcome Signals:**
- `engagements` 💠 `has` ➔ `reviews` **(1:N)**
- `users` 💠 `writes` ➔ `reviews` **(1:N)** — via `reviewer_id`
- `users` 💠 `reviewed as` ➔ `reviews` **(1:N)** — via `target_id`
- `engagements` 💠 `has` ➔ `outcome_signals` **(1:N)**
- `reviews` 💠 `generates` ➔ `outcome_signals` **(1:1)** — TECH_TEAM structured review produces one aggregate signal record

**Closes Phase 2 — Outcome Signals → Expert Seam Outcome Signals:**
- `outcome_signals` 💠 `generates` ➔ `expert_seam_outcome_signals` **(1:N)** — NestJS extracts per-seam rows from the JSONB aggregate after each review save; these rows power Tier 4 accumulation and the matching hard gate

**Platform Decisions** — passive audit log, written by FastAPI only. Already has two incoming lines from Phase 8 (`bid_conflict_overrides`) and Phase 9 (`acceptance_criteria`). No further relationships — `entity_id` and `entity_type` are polymorphic by design.

---

## Final Count

| Phase | Entities | Relationships |
|---|---|---|
| 0 — Elicitation Engine | 1 | 1 |
| 1 — Users & Role Subtypes | 4 | 4 |
| 2 — Expert Capability | 7 | 8 |
| 3 — Wallet & Finance | 6 | 5 |
| 4 — Organizations | 2 | 3 |
| 5 — Projects & Artifacts | 5 | 6 |
| 6 — Services | 1 | 1 |
| 7 — Engagements | 1 | 8 |
| 8 — Bid System | 7 | 10 |
| 9 — Milestones, Add-On, Disputes | 13 | 35 |
| 10 — Messaging, Reviews, Decisions | 4 | 10 |
| **Total** | **51** | **91** |