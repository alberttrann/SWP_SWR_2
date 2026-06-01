1/ 
```mermaid
stateDiagram-v2
    direction LR
   
    state "ECOSYSTEM SETUP" as Setup {
        MF1_Client_Onboarding : MF-1 Client Reg (incl. MF-11 TopUp, MF-13 Sub)
        MF2_Expert_Onboarding : MF-2 Expert Reg (incl. MF-11 TopUp, MF-13 Sub)
        MF3_TechTeam_Handoff : MF-3 Tech Team Handoff
        MF15_Dual_Role : MF-15 Dual-Role Switch
    }
   
    state "PATH A: AI PROJECT FLOW" as PathA {
        MF4_Elicitation : MF-4 Elicitation (RQ2 Core)
        MF5_Matching : MF-5 Matching (RQ1 Core)
        MF6_Bidding : MF-6 Bid & Connection
        MF7_Escrow : MF-7 Milestones & Escrow (RQ3 Core)
    }
   
    state "PATH B: MARKETPLACE" as PathB {
        MF9_Service_Publish : MF-9 Service Publish
        MF10_Service_Buy : MF-10 Service / Tech Discovery Buy
    }

    state "TRUST & INTEGRITY" as Trust {
        MF8_Disputes : MF-8 Dispute Resolution (RQ3 Core)
        MF12_Withdrawal : MF-12 Auto-Withdrawal
        MF14_Scope_Evo : MF-14 Add-On Protocol
        MF16_Verification : MF-16 Tier 4 Auto-Upgrade
        MF17_Admin_Monitor : MF-17 Admin Monitor
        MF18_Fraud_Detection : MF-18 Fraud Detection
    }

    [*] --> Setup
    Setup --> PathA : CEO initiates project
    Setup --> PathB : CEO browses services
   
    PathA --> Trust : Engagement Active
    PathB --> Trust : Payment Held
   
    Trust --> [*] : Funds Released / Disputed

```

2/
```mermaid
stateDiagram-v2
    direction TB

    state "Layer 1: Identity & Qualification" as L1 {
        MF1_Client_Onboarding : MF-1 Client Reg & Sub
        MF2_Expert_Onboarding : MF-2 Expert Reg & Verify
        MF3_Tech_Handoff : MF-3 Tech Team Creation
        MF13_Subscription : MF-13 Sub Purchase
        MF15_Dual_Role : MF-15 Dual-Role Switch
    }

    state "Layer 2: Supply & Demand Generation" as L2 {
        MF4_Elicitation : MF-4 AI Elicitation (Path A Demand)
        MF9_Service_Pub : MF-9 Service Publishing (Path B Supply)
    }

    state "Layer 3: Discovery & Contracting" as L3 {
        MF5_Matching : MF-5 AI Matching
        MF6_Bidding : MF-6 Bid & Negotiation
    }

    state "Layer 4: Value Exchange & Delivery" as L4 {
        MF7_Milestones : MF-7 Milestones & Escrow
        MF14_AddOn : MF-14 Add-On Protocol
        MF10_Service_Buy : MF-10 Service Purchase (Path B)
    }

    state "Layer 5: Settlement, Trust & Infrastructure" as L5 {
        MF8_Disputes : MF-8 Dispute Resolution
        MF11_Wallet : MF-11 Wallet Top-Up
        MF12_Withdraw : MF-12 Auto-Withdrawal
        MF16_Tier4 : MF-16 Auto-Upgrade
        MF17_Admin : MF-17 Integrity Monitor
        MF18_Fraud : MF-18 Fraud Detection
    }

    [*] --> L1
    L1 --> L2 : Define needs/supply
    L2 --> L3 : Find counterparties
    L3 --> L4 : Execute contract
    L4 --> L5 : Settle & Evaluate
    
    L5 --> L1 : Upgrade Tier / Switch Role
    L5 --> L2 : Re-verify capabilities
```

3/
```mermaid
flowchart TD
    %% Styling
    classDef highwayA fill:#e1f5fe,stroke:#039be5,stroke-width:2px;
    classDef highwayB fill:#fff3e0,stroke:#fb8c00,stroke-width:2px;
    classDef infra fill:#f3e5f5,stroke:#8e24aa,stroke-width:2px;
    classDef layer fill:#fafafa,stroke:#bdbdbd,stroke-width:1px,stroke-dasharray: 5 5;

    %% Layers
    subgraph L1 [Layer 1: Identity & Qualification]
        direction LR
        MF1[MF-1: Client Reg & Sub]:::highwayA
        MF2[MF-2: Expert Reg & Verify]:::highwayB
        MF3[MF-3: Tech Team Handoff]:::highwayA
        MF13[MF-13: Sub Purchase]:::infra
        MF15[MF-15: Dual-Role Switch]:::infra
    end

    subgraph L2 [Layer 2: Supply & Demand Generation]
        direction LR
        MF4[MF-4: AI Elicitation Engine]:::highwayA
        MF9[MF-9: Service Publishing]:::highwayB
    end

    subgraph L3 [Layer 3: Discovery & Contracting]
        direction LR
        MF5[MF-5: AI Matching]:::highwayA
        MF6[MF-6: Bid & Negotiation]:::highwayA
    end

    subgraph L4 [Layer 4: Value Exchange & Delivery]
        direction LR
        MF7[MF-7: Milestones & Escrow]:::highwayA
        MF14[MF-14: Add-On Protocol]:::highwayA
        MF10[MF-10: Service Purchase]:::highwayB
    end

    subgraph L5 [Layer 5: Settlement, Trust & Infrastructure]
        direction LR
        MF8[MF-8: Dispute Resolution]:::infra
        MF11[MF-11: Wallet Top-Up]:::infra
        MF12[MF-12: Auto-Withdrawal]:::infra
        MF16[MF-16: Tier 4 Auto-Upgrade]:::infra
        MF17[MF-17: Admin Monitor]:::infra
        MF18[MF-18: Fraud Detection]:::infra
    end

    %% Highway A (Path A: Project-Based)
    MF1 -->|Needs AI| MF13
    MF13 -->|Unlocks| MF4
    MF4 -->|Hard-blocks CEO| MF3
    MF3 -->|Returns truth| MF4
    MF4 -->|Publishes Footprint| MF5
    MF5 -->|Shortlist| MF6
    MF6 -->|Connected| MF7
    MF7 -->|Scope Evolves| MF14
    MF14 -->|Appended| MF7

    %% Highway B (Path B: Marketplace)
    MF2 -->|Needs Pro| MF13
    MF2 -->|Creates listing| MF9
    MF9 -->|Visible to| MF1
    MF1 -->|Buys| MF10

    %% Layer 5 Interactions (Cross-cutting)
    MF7 -->|Funds needed| MF11
    MF7 -->|Approved| MF12
    MF7 -->|Disputed| MF8
    MF10 -->|Paid| MF12
    MF2 -->|Links Bank| MF18
    MF12 -->|Closed| MF16
    MF16 -->|Upgrades| MF5
    MF8 -->|Audited by| MF17
    MF1 <-->|Switches| MF15

    %% Cross-Layer routing
    L1 -.-> L2
    L2 -.-> L3
    L3 -.-> L4
    L4 -.-> L5
    L5 -.-> L1
```

4/
```mermaid
stateDiagram-v2
    direction TB

    state "Layer 1: Identity & Qualification" as L1_Setup {
        direction LR
        state "MF-1 Client Reg" as MF1_Client
        state "MF-2 Expert Reg & Verify" as MF2_Expert
        state "MF-3 Tech Team Handoff" as MF3_TechTeam
        state "MF-11 Wallet Top-Up" as MF11_Wallet
        state "MF-13 Subscription Purchase" as MF13_Sub
        state "MF-15 Dual-Role Switch" as MF15_Dual
    }

    state "Layer 2: Supply & Demand Generation" as L2_Gen {
        direction LR
        state "MF-4 AI Elicitation (Path A)" as MF4_Elicitation
        state "MF-9 Service Publishing (Path B)" as MF9_Service
    }

    state "Layer 3: Discovery & Contracting" as L3_Contract {
        direction LR
        state "MF-5 AI Matching" as MF5_Matching
        state "MF-6 Bid & Negotiation" as MF6_Bidding
    }

    state "Layer 4: Value Exchange & Delivery" as L4_Delivery {
        direction LR
        state "MF-7 Milestones & Escrow" as MF7_Escrow
        state "MF-14 Add-On Protocol" as MF14_AddOn
        state "MF-10 Service Purchase (Path B)" as MF10_BuyService
    }

    state "Layer 5: Settlement, Trust & Infrastructure" as L5_Trust {
        direction LR
        state "MF-8 Dispute Resolution" as MF8_Dispute
        state "MF-12 Auto-Withdrawal" as MF12_Withdraw
        state "MF-16 Tier 4 Auto-Upgrade" as MF16_Upgrade
        state "MF-17 Admin Monitor" as MF17_Admin
        state "MF-18 Fraud Detection" as MF18_Fraud
    }

    [*] --> L1_Setup : User enters ecosystem

    L1_Setup --> L2_Gen : Identity established and Pro unlocked

    %% HIGHWAY A - Project-Based Flow (Traces through Layers 2 -> 3 -> 4)
    MF4_Elicitation --> MF5_Matching : HIGHWAY A - Spec PUBLISHED
    MF5_Matching --> MF6_Bidding : HIGHWAY A - Shortlist generated
    MF6_Bidding --> MF7_Escrow : HIGHWAY A - Expert SELECTED and Connected
    MF7_Escrow --> MF14_AddOn : HIGHWAY A - Sprint hits SCOPE EVOLUTION
    MF14_AddOn --> MF7_Escrow : Add-on appended

    %% HIGHWAY B - Marketplace Flow (Bypasses Layer 3, goes 2 -> 4)
    MF9_Service --> MF10_BuyService : HIGHWAY B - Client clicks Buy Now

    %% Layer Transitions
    L4_Delivery --> L5_Trust : Milestone APPROVED or DISPUTED

    L5_Trust --> L1_Setup : MF-16 upgrades profile or MF-15 switches context
    L5_Trust --> [*] : Ledger settled and funds withdrawn

    note right of L5_Trust
        MF-17 and MF-18 run continuously
        as cross-cutting integrity guards
        across all layers.
    end note
```
5/


## Part 1: High-Level Navigation Map

The AITasker UI is **role-driven**. The navigation bar completely re-renders based on the JWT's `active_role`.

```mermaid
stateDiagram-v2
    direction TB

    state "Authentication Stack" as Auth {
        Login_Register : Login / Register
        Role_Switcher : Role Switcher (Top Nav)
    }

    state "CLIENT_CEO View" as CEO {
        CEO_Dash : CEO Dashboard
        Elicitation : Elicitation Wizard (MF-4)
        Shortlist : Shortlist View (MF-5)
        Bid_Review_CEO : Bid Review (Business/Surface C&D)
        Project_Mgmt_CEO : Project Management (Fund / Approve Business)
        Marketplace : Marketplace Browser (MF-10)
    }

    state "CLIENT_TECH_TEAM View" as Tech {
        Tech_Dash : Tech Dashboard
        Handoff : Architecture Handoff (MF-4 Stage 4)
        Bid_Review_Tech : Bid Review (Technical/Surface B)
        Sprint_View : Sprint View (Comment/Resolve Blockers)
        Signoff_Tech : Sign-off (Technical Milestones)
    }

    state "EXPERT View" as Expert {
        Expert_Dash : Expert Dashboard
        Profile_Verify : Profile & Verification (MF-2)
        Market_Browse : Project Market (View Artifact A)
        Bid_Workspace : Bid Workspace (MF-6)
        Delivery : Delivery Workspace (Sprint/DoD/Submit)
        Wallet_Withdraw : Wallet & Withdrawal (MF-11/12)
    }

    Auth --> CEO : active_role = CLIENT, subtype = CEO
    Auth --> Tech : active_role = CLIENT, subtype = TECH_TEAM
    Auth --> Expert : active_role = EXPERT
    
    CEO --> Role_Switcher : If roles.length > 1
    Tech --> Role_Switcher : (Disabled - Tech cannot switch)
    Expert --> Role_Switcher : If roles.length > 1
    
    Role_Switcher --> CEO
    Role_Switcher --> Expert
```

---

## Part 2: Screen Inventory & Components

###  1. CLIENT / CEO Screens
*Goal: Help non-technical users define problems, pick the right expert, and manage budgets.*

| Screen Name | Core Purpose | Key UI Components | Mapped Flows |
|---|---|---|---|
| **CEO Dashboard** | Command center | Wallet Balance Card, Pro Subscription Badge, "Start New AI Project" CTA, Active Project Cards | MF-1, MF-11, MF-13 |
| **AI Elicitation Wizard** | Guided conversation | Step Indicator (1-5), Chat-style Input, Archetype Selector Cards, "Inject Phase 0" system message, **Hard-Block Modal** (Generate Handoff Link) | MF-4, MF-3 |
| **Shortlist View** | Compare matched experts | Match Cards (Strong/Qualified labels), **Seam Gap Map Visual**, Stack Tag Overlaps, "Select Expert" button | MF-5 |
| **Bid Review (CEO)** | Negotiate business terms | Bid Version History, Tech Team Recommendation Badge, **Surface C (Price Negotiation Panel)**, **Surface D (Override Modal with mandatory reason input)** | MF-6 |
| **Project Management** | Track budget & milestones | Milestone Timeline, "Fund Milestone" button (generates VietQR modal), Business Milestone Sign-off button | MF-7 |
| **Marketplace Browser** | Buy direct services | Service Cards, "Buy Now" button, Service Detail Page | MF-9, MF-10 |

###  2. CLIENT / TECH_TEAM Screens
*Goal: Provide technical reality, validate expert approaches, and track code-level delivery.*

| Screen Name | Core Purpose | Key UI Components | Mapped Flows |
|---|---|---|---|
| **Tech Dashboard** | Technical tasks | Pending Handoff Alerts, Active Sprint Alerts, Technical Sign-off Queue | MF-3, MF-7 |
| **Architecture Handoff** | Populate Artifact B | Stack Tag Multi-select, Integration Method Dropdown, File Upload Zone (schema/topology) | MF-4 (Stage 4) |
| **Bid Review (Tech)** | Validate tech approach | Bid Version History, **Surface B (Request Revision Modal - targeting specific component)**, Recommend/Not Recommend Toggle | MF-6 |
| **Sprint View** | Monitor delivery | Sprint Status Badge (On Track/Blocker), **Blocker Resolution Input**, Sprint Comment Thread | MF-7, MF-14 |
| **Sign-off (Tech)** | Approve technical deliverables | DoD Checklist Read-only View, Technical Acceptance Criteria, Sign-off Button | MF-7 |

###  3. EXPERT Screens
*Goal: Prove capability, win bids, deliver work, get paid automatically.*

| Screen Name | Core Purpose | Key UI Components | Mapped Flows |
|---|---|---|---|
| **Expert Dashboard** | Earnings & tasks | Earnings Chart, Pro Badge, Bank Link Status, Active Bids/Sprints, "Claim Seams" CTA | MF-2, MF-12 |
| **Profile & Verification** | Build taxonomy map | Domain Depth Selector, **Seam Cards (Tier 1/2/3 badges)**, "Verify with AI" button (Tier 2 file upload / Tier 3 scenario chat) | MF-2 |
| **Project Market** | Find work | Artifact A Viewer, **Surface A (Spec Clarification Panel - Ask question)**, "Submit Bid" button | MF-5, MF-6 |
| **Bid Workspace** | Construct & negotiate bid | 3-Component Form (Tech/Delivery/Finance), **Surface B (Revision Update Form - only flagged component editable)**, **Surface C (Accept/Counter Price)** | MF-6 |
| **Delivery Workspace** | Execute work | Milestone Timeline, Sprint Plan Editor, **DoD Checklist (Checkboxes with Completion Notes)**, "Submit Deliverable" button | MF-7 |
| **Wallet & Withdrawal** | Manage funds | Top-Up VietQR Display, Withdrawal Form (Amount input), Withdrawal History (Pending/Completed) | MF-11, MF-12 |

---

## Part 3: UI/UX State-Driven Rules

This is the most important section for your frontend teammate. AITasker's UI is heavily dictated by backend state machines. Components must conditionally render based on these rules:

### Rule 1: The Subscription Guard (Feature Gating)
*   **IF** `user.subscription_tier === 'free'`:
    *   "Start New AI Project" button is disabled/hidden.
    *   Clicking AI features triggers a `403` interceptor → Show **"Upgrade to Pro" Upsell Modal**.
    *   Expert Seam Verification buttons (Tier 2/3) are locked.

### Rule 2: The Elicitation Hard Block (MF-4 Stage 4)
*   **IF** `project.self_technical === false` AND `infra_signals === 'HIGH'`:
    *   The Elicitation Wizard *must not* allow the CEO to proceed to Stage 4.
    *   Show a full-screen "Hard Block" overlay: *"Your project involves live infrastructure..."* with a "Generate Handoff Link" button.

### Rule 3: Milestone Submission Gate (DoD)
*   **IF** `milestone.state === 'IN_PROGRESS'`:
    *   The "Submit Deliverable" button **must** be disabled UNTIL the frontend verifies that *all* DoD items with `is_required = true` are checked `COMPLETED`.
    *   If a required item is marked `NOT_APPLICABLE` by the expert, the UI should throw a validation error (backend will throw 403).

### Rule 4: State-Gated Artifact Access
*   **Artifact A:** Visible to all matched experts and the client.
*   **Artifact B:** 
    *   **IF** `engagement.state < CONNECTED`: UI must *not* render the Artifact B tab/endpoint for the Expert.
    *   **IF** `engagement.state >= CONNECTED`: Artifact B tab appears for Expert and Tech Team.

### Rule 5: Bid Surface Routing (Non-Linear Bids)
*   The Bid Detail screen must route the user based on the `bid.state`:
    *   `TECH_REVIEW`: Tech Team sees the review form.
    *   `REVISION_REQUESTED`: Expert sees *only* the flagged component text area editable; other fields are read-only.
    *   `CONFLICT_PENDING`: CEO sees the Override Modal (must input reason before viewing bid).

### Rule 6: The Dual-Role Switcher
*   **IF** `user.roles.length > 1`:
    *   Render a persistent toggle in the Top Navigation (e.g., "Switch to Expert").
    *   Clicking it hits `/auth/switch-role`, returns a new JWT, and the entire app shell (nav, dashboard) re-mounts.
    *   **Self-Exclusion:** Even when in Expert mode, the Project Market shortlist API will automatically filter out any project where the user is the CEO. No special UI needed, but good to know.

### Rule 7: The Escrow & VietQR Flow
*   When a CEO clicks "Fund Milestone":
    *   Show a Modal with a loading spinner ("Generating secure QR...").
    *   Render the VietQR image and a 24-hour countdown timer.
    *   Use WebSockets or polling: Once the SePay IPN hits the backend, the modal should auto-close and the Milestone state should live-update to `FUNDED` without a manual page refresh.

### Rule 8: Sprint Status & Add-On Protocol
*   **IF** Expert selects `Sprint Status = BLOCKER` and `blocker_type = SCOPE_EVOLUTION`:
    *   The UI must immediately prompt: *"This requires a scope change. Fill out the Add-On Proposal form."*
    *   This links the Sprint UI directly to the MF-14 Add-On creation flow.