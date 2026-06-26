# AITasker — Frontend ↔ Backend Integration Guide
## Main Flow 1 (MF-1): CEO Registration, Wallet Top-Up & Elicitation  
## Main Flow 2 (MF-2): Expert Registration, Profile & Tier 1→2 Verification

> **Audience:** Frontend developer  
> **Base URL (dev):** `http://localhost:3001`  
> **Auth pattern:** `Authorization: Bearer <access_token>` on every protected route  
> **Stack:** React + Vite + TanStack Query + Zustand (`auth.store.ts`) + axios `api-client.ts`  
> **Last synced against:** backend `src/` + BPMN diagrams (2026-06-26)

---

## Global Setup Notes

### Token Storage
After every auth call that returns a token, store it in `auth.store.ts` and attach it via `api-client.ts` interceptor. Never touch `localStorage`—use the Zustand store only.

```ts
// auth.store.ts (existing)
interface AuthState {
  accessToken:  string | null;
  refreshToken: string | null;
  user:         AuthUserResponse | null;
  setAuth: (tokens: { access_token: string; refresh_token: string }, user: AuthUserResponse) => void;
  clearAuth: () => void;
}
```

### Token Refresh
`POST /auth/refresh` with `{ refresh_token }` in body. No `Authorization` header needed. On 401 from any other endpoint, call refresh once, retry, then redirect to `/login` if it fails again.

### Subscription Guard
Most elicitation and portfolio routes return `403` with `{ statusCode: 403, message: "..." }` when the user's subscription tier is `"free"`. Always check this guard response and redirect the user to the subscription activation page.

---

---

# MAIN FLOW 1 — CEO Registration, Wallet, Subscription & Elicitation Wizard

This covers the full left-hand swimlane of the first BPMN diagram, from "Open register page" through "Receive project published notification".

---

## Phase A — Registration

### Step 1 · Open Register Page & Select Role

**Component:** `features/auth/RegisterPage.tsx`  
**Route:** `/register`

Render two role-selection options: **"I need AI help" (CEO/CLIENT)** and **"I provide AI services" (EXPERT)**. This selection controls the `roles` field sent to the API.

---

### Step 2 · Fill Registration Form → POST /auth/register

**Component:** `features/auth/RegisterPage.tsx`  

```
POST /auth/register
Body (JSON):
{
  email:        string,          // required
  password:     string,          // required, must be strong
  fullName:     string,          // required
  phone:        string,          // optional
  roles:        "CLIENT_CEO",    // for CEO path — always this literal string
  selfTechnical: boolean,        // optional, default false
  taxCode:      string           // optional — if provided, BE auto-fetches company name
}
```

**Success `201`:**
```json
{
  "access_token":  "<jwt>",
  "refresh_token": "<jwt>",
  "user": {
    "id":                     "uuid",
    "email":                  "...",
    "fullName":               "...",
    "activeRole":             "CLIENT",
    "clientSubtype":          "CEO",
    "subscriptionClientTier": "free",
    "subscriptionExpertTier": "free"
  }
}
```

**FE action:** Call `auth.store.setAuth(...)`. Issue JWT contains `activeRole`, `clientSubType`, `subscriptionClientTier`—store the whole user object. Redirect to `/dashboard` (CEO dashboard).

**Error cases:**
| HTTP | `message` | UI action |
|------|-----------|-----------|
| 409 | "Email already exist!" | Show inline field error |
| 400 | Validation array | Show per-field errors |

> ⚠️ **IMPORTANT:** After registration the user's `subscriptionClientTier` is `"free"`. They CANNOT start elicitation until they top-up their wallet and activate Client Pro. Guard the "Start AI Project" button accordingly.

---

## Phase B — Wallet Top-Up (Free Tier)

### Step 3 · View Dashboard on Free Tier

**Component:** `features/ceo/CeoDashboard.tsx`  
**Data needed:** `GET /wallets/me` and `GET /subscriptions/status`

```
GET /wallets/me
Headers: Authorization: Bearer <token>
```
**Response:**
```json
{
  "id":               "uuid",
  "userId":           "uuid",
  "availableBalance": 0,
  "lockedBalance":    0
}
```

```
GET /subscriptions/status
Headers: Authorization: Bearer <token>
```
**Response:**
```json
{
  "subscriptionTier":    "free",
  "subscriptionExpires": null
}
```

Show a banner when `subscriptionTier === "free"`: "Activate Client Pro to start AI projects — 500,000 VND / 6 months."

---

### Step 4 · Request Wallet Top-Up → POST /wallets/virtual-accounts/topup

**Component:** `features/ceo/onboarding/WalletTopUp.tsx`

```
POST /wallets/virtual-accounts/topup
Headers: Authorization: Bearer <token>
Body:
{
  "amount": 500000    // integer, minimum 2000 (in VND)
}
```

**Response `201`:**
```json
{
  "qrCodeUrl":        "https://qr.sepay.vn/img?bank=MBBank&acc=...&amount=500000&des=AITASKERXXXXX",
  "paymentReference": "AITASKERXXXXX"
}
```

**FE action:**
1. Render `qrCodeUrl` as an `<img>` tag (the URL itself is the QR code image — no further processing needed).
2. Display `paymentReference` as text so the user can copy it for manual bank transfer.
3. Show instructions: "Scan with any banking app. Transfer exactly 500,000 VND. Include the reference code in the memo."

```tsx
// VietQRPanel.tsx usage
<img src={qrCodeUrl} alt="VietQR payment code" className="w-48 h-48" />
<p>Reference: <code>{paymentReference}</code></p>
```

> **No webhook feedback to FE.** The IPN (SePay → BE webhook) fires server-side. The FE must poll `GET /wallets/me` to detect when `availableBalance` increases. Poll every 5 seconds, stop after balance ≥ `amount` OR after 30 minutes.

**Polling pattern:**
```ts
const pollWallet = async () => {
  const { data } = await apiClient.get('/wallets/me');
  if (data.availableBalance >= amount) { stopPolling(); onFunded(); }
};
const handle = setInterval(pollWallet, 5000);
// cleanup after 30min timeout
```

---

## Phase C — Subscription Activation

### Step 5 · Purchase Client Pro → POST /subscriptions/activate

**Component:** `features/ceo/onboarding/SubscriptionActivate.tsx`

```
POST /subscriptions/activate
Headers: Authorization: Bearer <token>
Body:
{
  "activeRole": "CLIENT"    // must match user's current activeRole
}
```

**Success `201`:**
```json
{
  "access_token": "<new-jwt-with-pro-tier-baked-in>"
}
```

**FE action (CRITICAL):** Replace the stored `access_token` with the new one. The new token contains updated `subscriptionClientTier: "pro"` in its payload. After storing the new token, the SubscriptionGuard on elicitation routes will pass.

```ts
const { data } = await apiClient.post('/subscriptions/activate', { activeRole: 'CLIENT' });
authStore.setToken(data.access_token); // update token in store
```

**Error cases:**
| HTTP | `message` | UI action |
|------|-----------|-----------|
| 422 | "INSUFFICIENT_BALANCE" | Show "Your wallet balance is too low. Please top up first." |
| 409 | "Your subscription is still available" | Show "You already have an active subscription." |
| 409 | "You must switch to the target role before activating subscription!" | Switch role first, then retry |

---

## Phase D — Elicitation Wizard (Stages 1–5)

All elicitation routes require `Authorization: Bearer <pro-token>` and return `403` if `subscriptionClientTier !== "pro"`.

**Component:** `features/ceo/elicitation/ElicitationWizard.tsx` (shell — renders sub-components per stage)

### Step 6 · Click "Start New AI Project" → POST /elicitation/sessions

**Component:** `features/ceo/elicitation/ElicitationWizard.tsx`

```
POST /elicitation/sessions
Headers: Authorization: Bearer <token>
Body: (empty — no body required)
```

**Response `201`:**
```json
{
  "id":           "uuid",
  "userId":       "uuid",
  "currentStage": 1,
  "state":        "IN_PROGRESS",
  "archetype":    null,
  "voidListJson": [],
  "createdAt":    "...",
  "updatedAt":    "..."
}
```

**FE action:** Store `session.id` in component state (or URL param). The wizard will use this `sessionId` for all subsequent stage calls. Navigate to Stage 1 view.

---

### Step 7 · Stage 1 — Answer Symptom Questions → PUT /elicitation/sessions/:id/stage1

**Component:** `features/ceo/elicitation/Stage1Symptoms.tsx`

```
PUT /elicitation/sessions/:sessionId/stage1
Headers: Authorization: Bearer <token>
Body:
{
  "symptomText": "We have a recommendation engine that..."   // free-text, at least a few sentences
}
```

**Response `200` (from BE after calling AI service):**
```json
{
  "id":           "uuid",
  "currentStage": 2,
  "state":        "IN_PROGRESS",
  "voidListJson": [
    { "void_code": "NO_GROUND_TRUTH", "severity": "HIGH", "injected": false },
    { "void_code": "UNCLEAR_SUCCESS_METRIC", "severity": "MEDIUM", "injected": false }
  ],
  "stage1SymptomsJson": ["symptom 1...", "symptom 2...", "..."],
  "updatedAt": "..."
}
```

**FE action:** 
- Display detected `voidListJson` to the user as warnings/info ("We noticed these gaps in your description: ...").
- `currentStage` is now `2` — navigate the wizard to Stage 2.
- Store `voidListJson` in component state; it will be sent forward through the wizard.

**UX note:** This call takes 10–30 seconds (LLM processing). Show a spinner with message "Analysing your project description…".

---

### Step 8 · Stage 2 — Select Project Archetype → PUT /elicitation/sessions/:id/stage2

**Component:** `features/ceo/elicitation/Stage2Archetype.tsx`

Display 6 archetype options to the user (rendered from this mapping — hardcode on FE):

| Code | Label |
|------|-------|
| `"1"` | AI Search & Q&A (RAG / document assistant) |
| `"2"` | Personalisation & Recommendations |
| `"3"` | Classification & Document Processing |
| `"4"` | Conversational Agent / Chatbot |
| `"5"` | Predictive Analytics / Forecasting |
| `"6"` | AI Process Automation |

```
PUT /elicitation/sessions/:sessionId/stage2
Headers: Authorization: Bearer <token>
Body:
{
  "archetype":            "1",              // string "1"–"6"
  "acknowledgedVoidCodes": ["NO_GROUND_TRUTH"]  // optional — user clicked "I understand" on voids
}
```

**Response `200`:**
```json
{
  "id":           "uuid",
  "currentStage": 3,
  "archetype":    "1",
  "state":        "IN_PROGRESS",
  "updatedAt":    "..."
}
```

**FE action:** Navigate wizard to Stage 3 with archetype-specific probe questions.

---

### Step 9 · Stage 3 — Answer Infrastructure Probe Questions → PUT /elicitation/sessions/:id/stage3

**Component:** `features/ceo/elicitation/Stage3Probes.tsx`

The probe questions are **archetype-specific** and hardcoded on the FE (4 questions per archetype). You do NOT fetch these from the BE; they live in `elicitation.service.ts` on the backend but are mirrored here for FE rendering:

| Archetype | Q1 | Q2 | Q3 | Q4 |
|-----------|----|----|----|----|
| `"1"` | Roughly how many people will search/ask per day? | What happens when someone gets a wrong answer? | Does it pull from existing docs/systems? | How fast must an answer appear? |
| `"2"` | How many users see recommendations, how often? | What if someone ignores a recommendation? | Where do you track user preferences? | How fresh do recommendations need to be? |
| `"3"` | How many items need classifying per day? | What happens when classification confidence is low? | Who reviews borderline cases? | What format are the items (PDF, image, text)? |
| `"4"` | How many concurrent users will chat simultaneously? | What's the expected conversation length? | What systems must the agent access? | What escalation path exists when the agent can't help? |
| `"5"` | What time horizon do forecasts cover? | How much historical data is available? | What business decisions are made from forecasts? | What's the acceptable error margin? |
| `"6"` | Which manual steps need automating first? | What triggers the automation? | What downstream systems receive output? | How is failure/error handled today? |

```
PUT /elicitation/sessions/:sessionId/stage3
Headers: Authorization: Bearer <token>
Body:
{
  "probeResponses": {
    "q1": "About 500 searches per day...",
    "q2": "They usually rephrase and try again...",
    "q3": "Yes, we have a Confluence wiki and Notion docs...",
    "q4": "Under 3 seconds ideally..."
  }
}
```

**Response `200`:**
```json
{
  "id":           "uuid",
  "currentStage": 4,
  "state":        "IN_PROGRESS",
  "updatedAt":    "..."
}
```

**FE action:** Navigate to Stage 4. At this point the wizard must branch based on `session.scenarioType` or the user's `selfTechnical` flag.

---

### Step 10 · Stage 4 — Technical Context (Scenario A or B)

**Context:** At Stage 4, the CEO must provide infrastructure/technical details. There are two scenarios:

- **Scenario A** — CEO is technical (`selfTechnical: true` was set during registration OR user clicks "I'll fill this in myself"). CEO fills Stage 4 directly.
- **Scenario B** — CEO delegates to their Tech Team. CEO generates an invite link and waits for the Tech Team to submit.

#### Scenario A: CEO fills Stage 4 directly → PUT /elicitation/sessions/:id/stage4

**Component:** `features/ceo/elicitation/Stage4ScenarioA.tsx`

```
PUT /elicitation/sessions/:sessionId/stage4
Headers: Authorization: Bearer <token>
Body:
{
  "scaleAndInfrastructure": "We run on AWS EKS, ~500k req/day...",
  "integrationMethod":      "REST APIs with our internal microservices",
  "legacyVolume":           "~2TB of historical data in S3",
  "schemas":                ["https://company.com/schema/v2.json"],
  "contracts":              ["https://company.com/contracts/api-spec.yaml"]
}
```

**Response `200`:**
```json
{
  "id":           "uuid",
  "currentStage": 5,
  "state":        "IN_PROGRESS",
  "updatedAt":    "..."
}
```

**FE action:** Navigate to Stage 5 loading screen (synthesis in progress).

---

#### Scenario B: CEO delegates to Tech Team → POST /elicitation/sessions/:id/generate-handoff-link

**Component:** `features/ceo/elicitation/Stage4ScenarioB.tsx` (fully implemented — see existing file)

```
POST /elicitation/sessions/:sessionId/generate-handoff-link
Headers: Authorization: Bearer <token>
Body: (empty)
```

**Response `201`:**
```json
{
  "invite_link":  "http://localhost:5173/register/handoff/eyJhbGci...",
  "invite_token": "eyJhbGci...",
  "expires_in":   "72h"
}
```

**FE action:**
1. Display the `invite_link` with a "Copy" button.
2. Tell CEO: "Share this link with your tech team. It expires in 72 hours."
3. Begin polling `GET /elicitation/sessions/:id` every 5 seconds.
4. When `session.currentStage >= 5`, stop polling and move to Stage 5 loading.
5. Timeout after 30 minutes — offer "Generate a new link" or "Fill in myself instead" options.

> ⚠️ There is no email sending. The link is provided directly to the CEO to share via Slack, Zalo, etc.

---

#### Tech Team: Register via Handoff Link → POST /auth/register/handoff

**Component:** `features/tech-team/auth/HandoffRegister.tsx`  
**Route:** `/register/handoff/:token`

Extract `token` from URL params.

```
POST /auth/register/handoff
Body:
{
  "invite_token": "<token-from-url-param>",
  "email":        "techteam@company.com",
  "password":     "Str0ng!Pass123",
  "fullName":     "Tech Lead Name"
}
```

**Success `201`:**
```json
{
  "access_token":  "<jwt>",
  "refresh_token": "<jwt>",
  "user": {
    "id":           "uuid",
    "activeRole":   "CLIENT",
    "clientSubtype": "TECH_TEAM"
  }
}
```

**Error cases:**
| HTTP | `message` | UI action |
|------|-----------|-----------|
| 401 | "This invite link has expired or is invalid." | Redirect to `/register/handoff/expired` |
| 401 | "This invite link has been superseded by a newer one." | Show message, ask CEO to resend |
| 401 | "This invite link has already been used." | Show "Link already used" error |
| 409 | "An account with this email already exists." | Show inline email error |

**Component:** `features/tech-team/auth/LinkExpiredError.tsx` — show at `/register/handoff/expired`

---

#### Tech Team: Submit Stage 4 → PUT /elicitation/sessions/:id/stage4-handoff

**Component:** `features/tech-team/stage4/Stage4Form.tsx`

After Tech Team registers, they need the `sessionId`. It is embedded in the JWT invite token as a claim — decode it on the FE using `atob` on the payload portion, or have the BE return it in the handoff registration response.

> **Note:** The `sessionId` is in the invite token JWT payload as `sessionId`. Decode it: `JSON.parse(atob(invite_token.split('.')[1])).sessionId`

```
PUT /elicitation/sessions/:sessionId/stage4-handoff
Headers: Authorization: Bearer <tech-team-jwt>
Body:
{
  "scaleAndInfrastructure": "We are on GCP GKE, autoscaling 200-800 nodes...",
  "integrationMethod":      "Pub/Sub with BigQuery sink",
  "legacyVolume":           "4 years of transaction data, ~3TB in BigQuery",
  "schemas":                [],
  "contracts":              []
}
```

**Response `200`:**
```json
{
  "id":           "uuid",
  "currentStage": 5,
  "state":        "IN_PROGRESS",
  "updatedAt":    "..."
}
```

**FE action (CEO side):** The CEO's polling loop (Scenario B component) detects `currentStage === 5` and advances the wizard.

---

### Step 11 · Stage 5 — Synthesis (Loading) + Quality Gate

**Component:** `features/ceo/elicitation/Stage5Loading.tsx`

Stage 5 synthesis is triggered automatically by the BE when Stage 4 completes. The CEO sees a loading screen and polls `GET /elicitation/sessions/:id` every 5 seconds.

```
GET /elicitation/sessions/:sessionId
Headers: Authorization: Bearer <token>
```

**Response `200`:**
```json
{
  "id":           "uuid",
  "currentStage": 5,
  "state":        "IN_PROGRESS" | "COMPLETED" | "RETURNED",
  "archetype":    "1",
  "voidListJson": [...],
  "updatedAt":    "..."
}
```

Poll until `state !== "IN_PROGRESS"`, then check:

| `state` | `currentStage` | Action |
|---------|----------------|--------|
| `"COMPLETED"` | any | Gate **passed** → navigate to `QualityGatePassed` |
| `"RETURNED"` | 1–4 | Gate **failed** → navigate to `QualityGateFailed` with `returnToStage` |

> The actual synthesis result (project_id, advisory_note, return_to_stage) comes from the **last PUT stage response** or from a `GET /elicitation/sessions/:id` extension. Store the data from the last stage call response; it contains `gate_passed`, `completeness_score`, `flagged_void`, `return_to_stage`, `advisory_note`, `project_id`.

---

### Step 12 · Quality Gate Passed → Project Published

**Component:** `features/ceo/elicitation/QualityGatePassed.tsx`

Display to CEO:
- "Your project has been published! ✓"
- Show `completeness_score` (e.g., "Completeness: 87%")
- "AI experts are being matched to your project. You'll be notified when bids arrive."

No additional API call needed. `project_id` is returned by the Stage 4/4-handoff response when gate passes.

---

### Step 13 · Quality Gate Failed → Return to Stage with Advisory

**Component:** `features/ceo/elicitation/QualityGateFailed.tsx`

The final stage response (from PUT stage4 or stage4-handoff) contains the failure data when `gate_passed: false`:

```json
{
  "gate_passed":        false,
  "completeness_score": 0.58,
  "flagged_void":       "UNCLEAR_SUCCESS_METRIC",
  "return_to_stage":    3,
  "advisory_note":      "Your project specification scored 58% completeness (minimum 70% required). Please revisit Stage 3 and provide more detail about unclear success metric."
}
```

Display `advisory_note` prominently. Provide a "Go back to Stage N" button that resets `ElicitationWizard` to `currentStage = return_to_stage`.

**CEO re-enters at the correct stage:** Call the appropriate stage endpoint again with corrected data (same `sessionId`), then the wizard retriggers synthesis on Stage 4 submission.

---

### Step 14 · Re-enter Elicitation (RETURNED_TO_CLIENT State)

If the user navigates away and comes back to an in-progress session, fetch the session and resume at `currentStage`:

```
GET /elicitation/sessions/:sessionId
```

Check `session.state`:
- `"IN_PROGRESS"` → resume at `session.currentStage`
- `"RETURNED"` → show advisory note, offer to resume at `session.currentStage`
- `"COMPLETED"` → show "Project published" screen

---

---

# MAIN FLOW 2 — Expert Registration, Profile & Tier 1→2 Verification

This covers the Expert swimlane of the second BPMN diagram: from "Open register page" through "Mark expert eligible for matching and withdrawals".

---

## Phase A — Expert Registration

### Step 1 · Register as Expert → POST /auth/register

**Component:** `features/expert/auth/ExpertRegister.tsx` or `features/auth/RegisterPage.tsx`

```
POST /auth/register
Body:
{
  "email":    "expert@company.com",
  "password": "Str0ng!Pass123",
  "fullName": "Expert Full Name",
  "phone":    "0901234567",          // optional
  "roles":    "EXPERT"               // exactly this string for expert path
}
```

**Success `201`:**
```json
{
  "access_token":  "<jwt>",
  "refresh_token": "<jwt>",
  "user": {
    "id":                     "uuid",
    "email":                  "...",
    "activeRole":             "EXPERT",
    "clientSubtype":          null,
    "subscriptionExpertTier": "free",
    "subscriptionClientTier": "free"
  }
}
```

**FE action:** Store auth. Redirect to Expert dashboard. The user's `subscriptionExpertTier` is `"free"` — they cannot submit portfolio evidence yet.

---

## Phase B — Profile Build (Tier 1 — CLAIMED)

### Step 2 · Fill Domain Depths → POST /expert-profile/domains

**Component:** `features/expert/profile/DomainDepthForm.tsx`  
**This is Tier 1 — the claim is self-declared (CLAIMED), no verification needed yet.**

Render 6 domain options with 3 depth levels each:

| Domain Code | Label |
|-------------|-------|
| `"A"` | LLM App Engineering |
| `"B"` | MLOps / LLMOps |
| `"C"` | AI Evaluation & Quality |
| `"D"` | Vector DB & Embeddings |
| `"E"` | Data & Pipeline Engineering |
| `"F"` | ML Modeling & Fine-Tuning |

| Depth Level | Meaning |
|-------------|---------|
| `"SURFACE"` | Familiar — used in projects but not primary |
| `"OPERATIONAL"` | Proficient — regular primary usage |
| `"DEEP"` | Expert — can architect and teach |

```
POST /expert-profile/domains
Headers: Authorization: Bearer <token>
Body:
{
  "domainCode": "A",         // one of: A B C D E F
  "depthLevel": "DEEP"       // one of: SURFACE OPERATIONAL DEEP
}
```

**Success `201`:**
```json
{
  "id":               "uuid",
  "expertId":         "uuid",
  "domainCode":       "A",
  "depthLevel":       "DEEP",
  "verificationTier": "CLAIMED"
}
```

**FE note:** Allow the expert to add multiple domains. Each `POST` creates one domain depth record. There is no batch endpoint — call once per domain.

---

### Step 3 · Add Seam Claims → POST /expert-profile/seams

**Component:** `features/expert/profile/SeamClaimsForm.tsx`

Seams are cross-domain boundaries the expert claims expertise in. Display as checkboxes:

| Seam Code | Boundary |
|-----------|----------|
| `"A↔C"` | LLM App ↔ AI Evaluation |
| `"A↔D"` | LLM App ↔ Vector DB |
| `"A↔F"` | LLM App ↔ ML Modeling |
| `"A↔B"` | LLM App ↔ MLOps |
| `"D↔E"` | Vector DB ↔ Data Pipeline |
| `"D↔F"` | Vector DB ↔ ML Modeling |
| `"C↔E"` | AI Eval ↔ Data Pipeline |
| `"C↔F"` | AI Eval ↔ ML Modeling |
| `"E↔F"` | Data Pipeline ↔ ML Modeling |
| `"B↔E"` | MLOps ↔ Data Pipeline |

```
POST /expert-profile/seams
Headers: Authorization: Bearer <token>
Body:
{
  "seamCode": "A↔D"    // must use exact Unicode arrow ↔ (U+2194)
}
```

**Success `201`:**
```json
{
  "id":               "uuid",
  "expertId":         "uuid",
  "seamCode":         "A↔D",
  "verificationTier": "CLAIMED",
  "submissionCount":  0,
  "lockedUntil":      null
}
```

**CRITICAL — Arrow encoding:** The seam code uses the Unicode bidirectional arrow `↔` (U+2194), NOT two separate arrows or a dash. Always send this exact character. Verify the encoding in your form values.

Store `seamClaimId` (the returned `id`) — it is required for portfolio submission in Phase C.

---

### Step 4 · Set Stack Tags & Engagement Model → PUT /expert-profile/me

**Component:** `features/expert/profile/ExpertProfileForm.tsx`

```
PUT /expert-profile/me
Headers: Authorization: Bearer <token>
Body:
{
  "engagementModel":  "MILESTONE",          // one of: MILESTONE HOURLY HYBRID
  "stackTagsJson":    ["Python", "Kafka", "Go", "Langchain"],
  "archetypeHistoryJson": []                // optional — can be filled later
}
```

**Success `200`:**
```json
{
  "userId":           "uuid",
  "engagementModel":  "MILESTONE",
  "stackTagsJson":    ["Python", "Kafka", "Go", "Langchain"],
  "archetypeHistoryJson": []
}
```

---

### Step 5 · Verify Profile → GET /expert-profile/me

```
GET /expert-profile/me
Headers: Authorization: Bearer <token>
```

**Response `200`:**
```json
{
  "user": {
    "email":      "...",
    "fullName":   "...",
    "activeRole": "EXPERT"
  },
  "profile": {
    "engagementModel":     "MILESTONE",
    "stackTagsJson":       ["Python", "Kafka"],
    "archetypeHistoryJson": []
  },
  "domainDepths": [
    { "domainCode": "A", "depthLevel": "DEEP", "verificationTier": "CLAIMED" }
  ],
  "seamClaims": [
    { "id": "uuid", "seamCode": "A↔D", "verificationTier": "CLAIMED", "submissionCount": 0, "lockedUntil": null }
  ]
}
```

---

## Phase C — Tier 2 Verification (Portfolio Submission → LLM Evaluation)

**Prerequisite:** Expert must have `subscriptionExpertTier: "pro"` to submit portfolio evidence. Gate the submission UI accordingly. The subscription guard returns `403` with message `"Expert Pro subscription required"`.

### Step 6 · Fund Expert Wallet (Same flow as CEO — Phase B MF-1)

```
POST /wallets/virtual-accounts/topup
Body: { "amount": 300000 }   // Expert Pro costs 300,000 VND
```

Poll `GET /wallets/me` until balance ≥ 300,000.

---

### Step 7 · Activate Expert Pro → POST /subscriptions/activate

```
POST /subscriptions/activate
Headers: Authorization: Bearer <token>
Body:
{
  "activeRole": "EXPERT"    // MUST match current activeRole
}
```

**Success `201`:**
```json
{
  "access_token": "<new-jwt-with-pro-tier>"
}
```

**FE action:** Replace stored `access_token`. The new token encodes `subscriptionExpertTier: "pro"`.

---

### Step 8 · Submit Portfolio Evidence → POST /portfolio-submissions

**Component:** `features/expert/profile/PortfolioSubmissionForm.tsx`

This submits real-world project evidence for a specific seam claim. The BE calls the AI service for LLM evaluation. Response takes 10–30 seconds.

```
POST /portfolio-submissions
Headers: Authorization: Bearer <pro-expert-token>
Body:
{
  "seamClaimId":        "uuid",                    // the ID from POST /expert-profile/seams
  "projectDescription": "Built a production RAG pipeline for a 500-lawyer law firm...",
                        // minimum 50 characters
  "decisionPoints":     "Evaluated BERTScore vs ROUGE for output quality; chose BERTScore..."
                        // minimum 20 characters
}
```

**Success `201`:**
```json
{
  "id":                     "uuid",
  "status":                 "APPROVED" | "REJECTED",
  "llmConfidence":          0.91,
  "evaluationTierUpgraded": true,
  "advisoryNote":           null,                  // non-null when REJECTED, explains gaps
  "evaluatedAt":            "2026-06-26T10:30:00Z"
}
```

**FE action based on `status`:**

**If `status === "APPROVED"` && `evaluationTierUpgraded === true`:**
- Show success: "Your seam claim has been upgraded to Tier 2 (Evidence-Backed)! ✓"
- Show `llmConfidence` as a percentage: "Confidence: 91%"
- The seam's `verificationTier` is now `"EVIDENCE_BACKED"` — refresh profile display.

**If `status === "REJECTED"`:**
- Show: "Submission not approved (Confidence: `llmConfidence * 100`%)"
- Display `advisoryNote` as improvement guidance.
- Show remaining attempts: `MAX_ATTEMPTS - submissionCount` (BE: max 5 attempts, then 30-day lockout).
- Check `lockedUntil` on the seam claim — if non-null and in future, show lockout countdown.

**Error cases:**
| HTTP | Code/Message | UI action |
|------|-------------|-----------|
| 403 | SubscriptionGuard | "Expert Pro required. Please activate your subscription first." |
| 404 | "Seam claim not found" | Edge case — show generic error |
| 422 | "ALREADY_VERIFIED_OR_HIGHER" | "This seam is already verified at a higher tier." |
| 429 | `{ code: "TOO_MANY_ATTEMPTS", lockedUntil: "..." }` | Show: "Too many attempts. Try again after `lockedUntil`." |
| 503 | LLM unavailable | "Evaluation service is temporarily unavailable. Please try again in a moment." |

**UX:** Show a loading spinner for the full duration (up to 30 seconds) with "Evaluating your portfolio evidence with AI...". Do not let the user submit again until response arrives.

---

### Step 9 · Check Submission Status → GET /portfolio-submissions/:id

**Component:** `features/expert/profile/PortfolioSubmissionResult.tsx`

```
GET /portfolio-submissions/:submissionId
Headers: Authorization: Bearer <token>
```

**Response `200`:**
```json
{
  "id":          "uuid",
  "seamClaimId": "uuid",
  "status":      "APPROVED" | "REJECTED" | "PENDING",
  "llmConfidence": 0.91,
  "advisoryNote":  "The submission lacks specific technical decision rationale...",
  "submittedAt":   "...",
  "evaluatedAt":   "..."
}
```

---

## Phase D — Bank Account Link

This is required before the expert can receive withdrawals. After Tier 2 verification, guide the expert to link their bank account.

### Step 10 · Link Bank Account → POST /bank-hub/initiate-link

**Component:** `features/expert/profile/BankLinkForm.tsx`

> **Note:** In the current implementation there is no real SePay Hosted Link flow on FE. The expert provides their bank account XID directly (workaround approach — see MF-2 validation script). The SePay OTP flow described in the BPMN is not yet wired in the backend beyond this direct POST.

```
POST /bank-hub/initiate-link
Headers: Authorization: Bearer <expert-token>
Body:
{
  "bank_account_xid": "BANKXID12345",    // SePay bank account XID
  "holder_name":      "Expert Full Name"
}
```

**Success `201`:**
```json
{
  "success": true
}
```

**Error cases:**
| HTTP | Message | UI action |
|------|---------|-----------|
| 409 | "Bank already linked!" | Show "Bank account is already linked." |

After successful link: "Your bank account has been linked. You can now receive milestone payments and request withdrawals."

---

## Phase E — Post-Verification State: Eligible for Matching

After bank link, expert is now:
- `verificationTier: "EVIDENCE_BACKED"` on at least one seam
- `subscriptionExpertTier: "pro"`
- Bank account linked

**FE action:** Refresh `GET /expert-profile/me` and update the dashboard to show "Profile Active — You are eligible for matching on projects requiring your seam expertise."

The expert appears in the shortlist when the AI matching engine scores eligible experts against published projects.

---

---

# Shared Utility Patterns

## Error Handling Template

Every API call should handle these standardized error shapes:

```ts
interface ApiErrorResponse {
  statusCode: number;
  message:    string | string[];   // can be array for validation errors
  path:       string;
  timestamp:  string;
}
```

For validation errors, `message` is an array of field-level strings. Map them to form field errors via Formik/React Hook Form.

```ts
catch (err: any) {
  if (err.response?.status === 422) {
    const msgs = err.response.data.message;
    if (Array.isArray(msgs)) {
      // map to field errors
    }
  }
  if (err.response?.status === 403) {
    // check if SubscriptionGuard: redirect to /subscribe
    if (err.response.data.message?.includes('subscription')) {
      navigate('/subscribe');
    }
  }
}
```

---

## Wallet Balance Display

Amounts from the BE are always integers in **VND** (Vietnamese Dong). Format them:

```ts
const formatVND = (amount: number) =>
  new Intl.NumberFormat('vi-VN', { style: 'currency', currency: 'VND' }).format(amount);
// → "500.000 ₫"
```

---

## Polling Helper

Both MF-1 (wallet top-up) and MF-2 (wallet + Stage 4 Scenario B) use polling. Use this pattern:

```ts
function usePoll(fn: () => Promise<boolean>, intervalMs = 5000, timeoutMs = 30 * 60_000) {
  const handle  = useRef<ReturnType<typeof setInterval>>();
  const started = useRef(Date.now());
  
  const start = useCallback(() => {
    fn(); // immediate first call
    handle.current = setInterval(async () => {
      const done = await fn();
      if (done || Date.now() - started.current > timeoutMs) stop();
    }, intervalMs);
  }, [fn]);
  
  const stop = useCallback(() => clearInterval(handle.current), []);
  
  useEffect(() => () => stop(), []);
  return { start, stop };
}
```

---

## JWT Payload Decode (for Tech Team session ID extraction)

```ts
function decodeJwtPayload<T>(token: string): T {
  const base64 = token.split('.')[1].replace(/-/g, '+').replace(/_/g, '/');
  return JSON.parse(atob(base64));
}

// Usage in HandoffRegister:
const { sessionId, ceoId } = decodeJwtPayload<{ sessionId: string; ceoId: string }>(inviteToken);
```

---

## Subscription Gate Check (FE-side Guard)

Before rendering protected UI elements, check the token payload:

```ts
import { useAuthStore } from '@/store/auth.store';

function useIsClientPro() {
  const user = useAuthStore(s => s.user);
  return user?.subscriptionClientTier === 'pro';
}

function useIsExpertPro() {
  const user = useAuthStore(s => s.user);
  return user?.subscriptionExpertTier === 'pro';
}
```

After `POST /subscriptions/activate` returns a new `access_token`, decode it and update the user object in the store. The new token payload includes the updated tier field.

---

---

# Complete API Reference (Both Main Flows)

| Method | Path | Auth | Role | Sub Gate | Description |
|--------|------|------|------|----------|-------------|
| POST | `/auth/register` | ❌ | — | — | Register CEO or Expert |
| POST | `/auth/login` | ❌ | — | — | Login |
| POST | `/auth/refresh` | ❌ | — | — | Refresh access token |
| POST | `/auth/register/handoff` | ❌ | — | — | Tech Team register via invite link |
| PUT | `/auth/switch-role` | ✅ | CLIENT/EXPERT | — | Switch active role |
| GET | `/wallets/me` | ✅ | CLIENT/EXPERT | — | Get wallet balance |
| GET | `/wallets/me/transactions` | ✅ | CLIENT/EXPERT | — | Get transaction history |
| POST | `/wallets/virtual-accounts/topup` | ✅ | CLIENT/EXPERT | — | Generate QR for top-up |
| POST | `/subscriptions/activate` | ✅ | CLIENT/EXPERT | — | Activate Pro subscription |
| GET | `/subscriptions/status` | ✅ | CLIENT/EXPERT | — | Check subscription status |
| POST | `/elicitation/sessions` | ✅ | CLIENT (CEO) | ❌ free OK | Create elicitation session |
| GET | `/elicitation/sessions/:id` | ✅ | CLIENT (CEO) | ✅ Pro-C | Get session state |
| PUT | `/elicitation/sessions/:id/stage1` | ✅ | CLIENT (CEO) | ✅ Pro-C | Submit symptoms text |
| PUT | `/elicitation/sessions/:id/stage2` | ✅ | CLIENT (CEO) | ✅ Pro-C | Select archetype |
| PUT | `/elicitation/sessions/:id/stage3` | ✅ | CLIENT (CEO) | ✅ Pro-C | Answer probe questions |
| PUT | `/elicitation/sessions/:id/stage4` | ✅ | CLIENT (CEO) | ✅ Pro-C | Submit technical context (Scenario A) |
| PUT | `/elicitation/sessions/:id/stage4-handoff` | ✅ | CLIENT (TECH_TEAM) | ❌ no gate | Submit tech context (Scenario B) |
| POST | `/elicitation/sessions/:id/generate-handoff-link` | ✅ | CLIENT (CEO) | ✅ Pro-C | Generate Tech Team invite link |
| PUT | `/elicitation/sessions/:id/self-technical` | ✅ | CLIENT (CEO) | ✅ Pro-C | Toggle self-technical flag |
| POST | `/elicitation/sessions/:id/retry-synthesis` | ✅ | CLIENT (CEO) | ✅ Pro-C | Retry failed synthesis |
| GET | `/expert-profile/me` | ✅ | EXPERT | — | Get expert profile |
| PUT | `/expert-profile/me` | ✅ | EXPERT | — | Update stack tags / engagement model |
| POST | `/expert-profile/domains` | ✅ | EXPERT | — | Add domain depth claim (Tier 1) |
| POST | `/expert-profile/seams` | ✅ | EXPERT | — | Add seam claim (Tier 1) |
| POST | `/portfolio-submissions` | ✅ | EXPERT | ✅ Pro-E | Submit portfolio for LLM evaluation |
| GET | `/portfolio-submissions/:id` | ✅ | EXPERT/ADMIN | — | Get submission result |
| POST | `/bank-hub/initiate-link` | ✅ | EXPERT | — | Link bank account |

---

# Component-to-Route Mapping

| File | Endpoint(s) |
|------|-------------|
| `features/auth/RegisterPage.tsx` | `POST /auth/register` |
| `features/auth/LoginPage.tsx` | `POST /auth/login`, `POST /auth/refresh` |
| `features/ceo/CeoDashboard.tsx` | `GET /wallets/me`, `GET /subscriptions/status` |
| `features/ceo/onboarding/WalletTopUp.tsx` | `POST /wallets/virtual-accounts/topup`, `GET /wallets/me` (poll) |
| `features/ceo/onboarding/SubscriptionActivate.tsx` | `POST /subscriptions/activate` |
| `features/ceo/elicitation/ElicitationWizard.tsx` | `POST /elicitation/sessions`, `GET /elicitation/sessions/:id` (poll) |
| `features/ceo/elicitation/Stage1Symptoms.tsx` | `PUT /elicitation/sessions/:id/stage1` |
| `features/ceo/elicitation/Stage2Archetype.tsx` | `PUT /elicitation/sessions/:id/stage2` |
| `features/ceo/elicitation/Stage3Probes.tsx` | `PUT /elicitation/sessions/:id/stage3` |
| `features/ceo/elicitation/Stage4ScenarioA.tsx` | `PUT /elicitation/sessions/:id/stage4` |
| `features/ceo/elicitation/Stage4ScenarioB.tsx` | `POST /elicitation/sessions/:id/generate-handoff-link`, `GET /elicitation/sessions/:id` (poll) |
| `features/ceo/elicitation/Stage5Loading.tsx` | `GET /elicitation/sessions/:id` (poll) |
| `features/ceo/elicitation/QualityGatePassed.tsx` | Display only (data from stage response) |
| `features/ceo/elicitation/QualityGateFailed.tsx` | Display only (data from stage response) |
| `features/tech-team/auth/HandoffRegister.tsx` | `POST /auth/register/handoff` |
| `features/tech-team/auth/LinkExpiredError.tsx` | Display only |
| `features/tech-team/stage4/Stage4Form.tsx` | `PUT /elicitation/sessions/:id/stage4-handoff` |
| `features/expert/auth/ExpertRegister.tsx` | `POST /auth/register` |
| `features/expert/ExpertDashboard.tsx` | `GET /wallets/me`, `GET /subscriptions/status` |
| `features/expert/profile/DomainDepthForm.tsx` | `POST /expert-profile/domains` |
| `features/expert/profile/SeamClaimsForm.tsx` | `POST /expert-profile/seams`, `GET /expert-profile/me` |
| `features/expert/profile/ExpertProfileForm.tsx` | `PUT /expert-profile/me` |
| `features/expert/profile/PortfolioSubmissionForm.tsx` | `POST /portfolio-submissions` |
| `features/expert/profile/PortfolioSubmissionResult.tsx` | `GET /portfolio-submissions/:id` |
| `features/expert/profile/BankLinkForm.tsx` | `POST /bank-hub/initiate-link` |
| `components/wallet/WalletCard.tsx` | `GET /wallets/me` |
| `components/wallet/VietQRPanel.tsx` | Renders `qrCodeUrl` from topup response |

---

# Critical Implementation Notes

1. **Wallet amounts are BigInt on the BE** — they're serialized as `Number` in the JSON response. Always treat as integers. Format with `Intl.NumberFormat` for display.

2. **Token replacement after subscription activate** — failing to replace the token after `POST /subscriptions/activate` is the single most common integration bug. The old free-tier token will get 403 on all gated routes even after payment.

3. **Seam arrow encoding** — `↔` is U+2194 (LEFT RIGHT ARROW). Copy-paste from this doc into form values. If it arrives as `???` or `?` on the BE, the request will fail validation.

4. **Stage 4 Handoff — no email infrastructure** — the `invite_link` must be displayed and copied by the CEO. Do not show a "Send by email" button unless you build the email service.

5. **Poll cleanup** — always `clearInterval` in `useEffect` cleanup. Missing this causes memory leaks and state updates on unmounted components.

6. **LLM calls are slow** — Stage 1, Stage 5 synthesis, and Portfolio Submission all call Gemini. Show a loading state for up to 90 seconds for Stage 5. Disable re-submit buttons while in-flight.

7. **Expert Tier 1 vs Tier 2** — Tier 1 (`CLAIMED`) is just a self-declaration. Tier 2 (`EVIDENCE_BACKED`) requires portfolio submission and LLM approval. The gap map shown to CEOs distinguishes these visually (amber = CLAIMED, green = EVIDENCE_BACKED, red = missing).

8. **No logout endpoint** — JWTs expire after 7 days. "Logout" on FE means `authStore.clearAuth()` and redirect to `/login`. No BE call needed.
