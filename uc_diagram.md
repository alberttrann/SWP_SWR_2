# DIAGRAM 1: Project Setup & AI Verification

**Purpose:** Shows how users onboard, purchase required subscriptions, define highly technical projects using AI, and verify their expert capabilities using LLMs.

### 1. ACTORS 
**Primary Actors (Left Side):**
*   **User** (Base Actor)
    *   **Client** (Inherits from User)
        *   **CLIENT / CEO** (Inherits from Client)
        *   **CLIENT / TECH_TEAM** (Inherits from Client)
    *   **EXPERT** (Inherits from User)

**Secondary Actors (Right Side):**
*   **Anthropic Claude API** (The AI Engine)
*   **SePay Payment Gateway** (For Wallet Top-up)

---

### 2. BASE ACCOUNT & SUBSCRIPTION USE CASES

*   **Top Up Wallet via VA (UC02)**
    *   **Actors:** User, SePay Payment Gateway
*   **Purchase Subscription (UC03 / UC14)**
    *   **Actors:** User
    *   **Relationship:** `---<<include>>---> Top Up Wallet via VA` *(Because phải balance to buy the sub)*
*   **Manage Dual-Role Account** (Combines UC-DR1 & UC-DR2)
    *   **Actors:** User

---

### 3. CLIENT FLOW: AI Elicitation & Project Intake

*   **Submit Project via AI Elicitation Engine (UC01)**
    *   **Actors:** CLIENT / CEO, Anthropic Claude API
    *   **Relationships:** 
        *   `---<<include>>---> Run Automated Quality Gate` *(System always validates the final spec)*
        *   `---<<include>>---> Purchase Subscription` *(Guard: Must have Pro tier to use AI)*

*   **Complete Tech Team Architecture Handoff (UC01t)**
    *   **Actors:** CLIENT / TECH_TEAM
    *   **Relationship:** `---<<extend>>---> Submit Project via AI Elicitation Engine` *(Condition: Infrastructure threshold crossed)*

*   **Proceed via Tech Discovery Pathway - Scenario A (UC01a)**
    *   **Actors:** CLIENT / CEO
    *   **Relationship:** `---<<extend>>---> Submit Project via AI Elicitation Engine` *(Condition: No Tech Team exists)*

*   **Complete Stage 4 as Self-Technical - Scenario B (UC01b)**
    *   **Actors:** CLIENT / CEO
    *   **Relationship:** `---<<extend>>---> Submit Project via AI Elicitation Engine` *(Condition: CEO is technical)*

---

### 4. EXPERT FLOW: Profile Creation & AI Verification

*   **Create Taxonomy-Based Profile (UC13)**
    *   **Actors:** EXPERT

*   **Submit Portfolio Evidence for LLM Auto-Verification (UC15)**
    *   **Actors:** EXPERT, Anthropic Claude API
    *   **Relationships:**
        *   `---<<extend>>---> Create Taxonomy-Based Profile` *(Condition: Expert wants Tier 2 upgrade)*
        *   `---<<include>>---> Purchase Subscription` *(Guard: Must have Expert Pro to use AI verification)*

*   **Request & Complete Scenario Assessment (UC16)**
    *   **Actors:** EXPERT, Anthropic Claude API
    *   **Relationships:**
        *   `---<<extend>>---> Create Taxonomy-Based Profile` *(Condition: Expert wants Tier 3 upgrade)*

*   **Publish AI Service Listing via AI Generator (UC18)**
    *   **Actors:** EXPERT, Anthropic Claude API

---


# DIAGRAM 2: Bidding & Negotiation

**Purpose:** Shows how experts evaluate matched projects, negotiate technical scope and pricing via the 4 non-linear Bid Surfaces, and how the CEO ultimately signs the NDA to unlock the hidden architectural blueprints.

### 1. ACTORS 
**Primary Actors (Left Side):**
*   **Client** (Base Actor)
    *   **CLIENT / CEO** (Inherits from Client)
    *   **CLIENT / TECH_TEAM** (Inherits from Client)
*   **EXPERT** (Standalone Actor)


---

### 2. PRE-BIDDING & SHORTLIST VIEW

*   **View Artifact A and Expert Shortlist (UC04)**
    *   **Actors:** Client (CEO and Tech Team)

*   **Browse Project Shortlist & Gap Map (UC19)**
    *   **Actors:** EXPERT

*   **Surface A: File Spec Clarification Request** *(Expert asks a question before bidding)*
    *   **Actors:** EXPERT
    *   **Relationship:** `---<<extend>>---> Browse Project Shortlist & Gap Map` *(Condition: Expert needs more info)*

*   **Surface A: Respond to Spec Clarification (UC05 / UC05t)**
    *   **Actors:** Client (CEO and Tech Team)

---

### 3. BID SUBMISSION & TECH REVIEW (Surfaces B & D)

*   **Submit Structured Capability Bid (UC20)**
    *   **Actors:** EXPERT

*   **Review Capability Bids & Flag Recommendations (UC06a)**
    *   **Actors:** CLIENT / TECH_TEAM

*   **Surface B: Request Bid Component Revision (UC06r)** *(Tech Team forces Expert to rewrite a specific part of the bid)*
    *   **Actors:** CLIENT / TECH_TEAM
    *   **Relationship:** `---<<extend>>---> Review Capability Bids & Flag Recommendations` *(Condition: Tech approach is flawed)*

*   **Surface B: Revise Specific Bid Component (UC20r)**
    *   **Actors:** EXPERT
    *   **Relationship:** `---<<extend>>---> Submit Structured Capability Bid` *(Condition: Revision Requested by Tech Team)*

*   **Surface D: Override TECH_TEAM Disapproval (UC06c)** *(CEO ignores the Tech Team's warning and forces the bid through)*
    *   **Actors:** CLIENT / CEO
    *   **Relationship:** `---<<extend>>---> Approve Expert Selection` *(Condition: Bid state is TECH_DISAPPROVED)*

---

### 4. PRICE NEGOTIATION & FINAL SELECTION (Surface C)

*   **Approve Expert Selection (UC06b)**
    *   **Actors:** CLIENT / CEO

*   **Surface C: Propose Price Adjustment (UC06d)** *(CEO haggles on price)*
    *   **Actors:** CLIENT / CEO
    *   **Relationships:** 
        *   `---<<extend>>---> Approve Expert Selection` *(Condition: CEO wants cheaper milestone pricing)*
        *   `---<<include>>---> Accept / Counter / Decline Price Proposal` *(Expert MUST respond to the haggle)*

*   **Surface C: Accept / Counter / Decline Price Proposal (UC20c)**
    *   **Actors:** EXPERT

---

### 5. Connection & Dual Artifact Unlocking

*   **Send Connection Request & Complete NDA (UC07)**
    *   **Actors:** CLIENT / CEO

*   **Accept Connection and Access Artifact B (UC21)**
    *   **Actors:** EXPERT

*   **Access Artifact B Post-Connection (UC07b)** *(Tech Team can now see the shared vault too)*
    *   **Actors:** CLIENT / TECH_TEAM

---

# DIAGRAM 3: Milestone Execution & Escrow Payments

**Purpose:** Shows how money is escrowed via VietQR, how Experts manage their work using Sprints/DoD, how Scope Evolution is handled automatically, and how automated payouts and dispute resolutions occur without Admin intervention.

### 1. ACTORS 
**Primary Actors (Left Side):**
*   **User** (Base Actor)
    *   **Client** (Inherits from User)
        *   **CLIENT / CEO** (Inherits from Client)
        *   **CLIENT / TECH_TEAM** (Inherits from Client)
    *   **EXPERT** (Inherits from User)

**Secondary Actors (Right Side):**
*   **SePay Payment Gateway** (Handles Inbound QR payments & Outbound Chi Hộ)
*   **Anthropic Claude API** (Handles Layer 1 Dispute Resolution)
*   **Admin** (Read-Only Monitor)

---

### 2. ESCROW FUNDING & SPRINT PLANNING

*   **Fund Milestone via Per-Milestone VA QR (UC08)**
    *   **Actors:** CLIENT / CEO, SePay Payment Gateway
    *   **Relationship:** `---<<include>>---> Top Up Wallet via VA` *(Guard: If CEO chooses to fund from wallet instead of direct scan, balance is checked)*

*   **Create Sprint Plan & DoD Checklist (UC23)**
    *   **Actors:** EXPERT

*   **Read Sprint Plan & Comment on Thread (UC13t)**
    *   **Actors:** CLIENT / TECH_TEAM

*   **Submit Weekly Sprint Status Update (UC24)**
    *   **Actors:** EXPERT

---

### 3. SCOPE EVOLUTION & ADD-ON PHASE (F8)

*   **Submit Add-On Phase Brief with Causal Chain (UC26)**
    *   **Actors:** EXPERT
    *   **Relationship:** `---<<extend>>---> Submit Weekly Sprint Status Update` *(Condition: Status is marked as SCOPE_EVOLUTION blocker)*

*   **Approve Add-On Phase - Tech Dimension (UC10t)**
    *   **Actors:** CLIENT / TECH_TEAM

*   **Approve Add-On Phase - Budget Dimension (UC10)**
    *   **Actors:** CLIENT / CEO
    *   **Relationship:** `---<<include>>---> Approve Add-On Phase - Tech Dimension` *(Guard: Tech Team MUST approve tech justification before CEO can approve budget)*

---

### 4. DELIVERABLES, APPROVALS & AUTOMATED PAYOUT

*   **Submit Milestone Deliverable (UC25)**
    *   **Actors:** EXPERT

*   **Stage Pay-Gated Reasoning Documents (UC22)**
    *   **Actors:** EXPERT

*   **Sign Off Technical Milestones (UC08t)**
    *   **Actors:** CLIENT / TECH_TEAM

*   **Approve Business/Budget Milestones (UC09)**
    *   **Actors:** CLIENT / CEO

*   **Request Withdrawal to Bank Account (UC27)**
    *   **Actors:** EXPERT, SePay Payment Gateway
    *   **Relationship:** `---<<include>>---> Link Bank Account via Bank Hub` *(Guard: Cannot withdraw if bank account isn't linked)*

*   **Link Bank Account via Bank Hub (UC17)**
    *   **Actors:** EXPERT, SePay Payment Gateway

---

### 5. DISPUTES, REVIEWS & ADMIN MONITORING

*   **File and Resolve Dispute (3-Layer Engine) (UC29)**
    *   **Actors:** User, Anthropic Claude API, SePay Payment Gateway
    *   **Relationships:** 
        *   `---<<extend>>---> Submit Milestone Deliverable` *(Condition: Client rejects the delivery)*
        *   `---<<extend>>---> Sign Off Technical Milestones` *(Condition: Tech Team rejects delivery)*

*   **Complete Post-Engagement Review (UC12a / UC12b / UC28)**
    *   **Actors:** User

*   **Monitor Platform Integrity & Escrow Ledger (UC-A2 / UC-A3)**
    *   **Actors:** Admin