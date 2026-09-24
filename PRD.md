# Product Requirements Document (PRD)

**Project Name:** Pocketwise

**Platform:** Responsive Web Application (Optimized for Mobile/Desktop Viewports)

**Tech Stack:** Next.js (App Router), Node.js, PostgreSQL (Prisma ORM), Tailwind CSS, NextAuth.js

**Target Audience:** Students and early-career earners with variable or early incomes

**UI/UX Inspiration:** Design Buddies UI/UX Case Study (`#19664D` visual palette, clean cards, Insight Owl motif)

---

## 1. Executive Summary & Goals

### 1.1 Problem Statement

Young earners and students struggle with financial cognitive overload. Traditional finance tracking apps bombard users with dense spreadsheets, complex charts, and opaque financial terminology. This causes financial avoidance: users only check their money after overspending, resulting in persistent financial anxiety and decision paralysis.

### 1.2 Product Vision

Pocketwise delivers effortless, non-judgmental financial visibility. By pairing zero-friction manual transaction logging with a hybrid advisory engine (deterministic risk math + an LLM-powered "Insight Owl" for natural-language dashboard cards), users instantly see their financial position and know what concrete action to take.

### 1.3 Key Objectives (OKRs)

* **Time-to-Log:** Allow a user to record any transaction in under 5 seconds.


* **Cognitive Clarity:** Eliminate all opaque financial jargon; replace it with clean summaries (Income, Expenses, Net Worth).


* **Actionable Advisory:** Provide 100% deterministic portfolio recommendations paired with LLM-generated behavioral nudges on the dashboard.


* **Strict Multi-Tenancy:** Ensure strict database-level data isolation across all user sessions.

---

## 2. User Personas & Scenarios

### 2.1 Persona: "The Early Earner" (Soham / Hari)

* **Demographics:** 19–24 years old; college student or junior software engineer; variable income from stipends, part-time jobs, or freelancing.


* **Pain Points:**
* Avoids checking bank accounts when stressed.


* Struggles to understand whether weekend spending is sustainable.


* Overwhelmed by investment jargon (e.g., standard deviation, alpha, beta).




* **Core Need:** A clean dashboard that highlights outliers without shame and calculates exact asset allocation based on their risk tolerance.



---

## 3. System Architecture & Tech Stack

```
+-----------------------------------------------------------------------+
|                    Client Layer (Next.js App Router)                  |
|  - Tailwind CSS + Lucide React                                        |
|  - Pocketwise Design System: Primary #19664D, Neutral Slate Grays     |
|  - Recharts: Custom weekly spending bar charts, asset allocation pies |
+-----------------------------------+-----------------------------------+
                                    |
                           HTTPS / Server Actions
                                    |
+-----------------------------------v-----------------------------------+
|                    Application & API Layer (Node.js)                  |
|  - NextAuth.js v5 (Credentials + Google OAuth, JWT Strategy)          |
|  - Deterministic Portfolio Allocation Engine                          |
|  - Metric Aggregator (Daily spend, weekend spikes, savings ratios)    |
|  - LLM Advisory Service (Stateless prompt synthesis -> Insight Owl)   |
+-------------------+-------------------------------+-------------------+
                    |                               |
       Prisma ORM (PostgreSQL)            OpenAI / Gemini SDK
                    |                               |
+-------------------v---------------+ +-------------v-------------------+
|          PostgreSQL Database      | |      Stateless LLM Inference    |
|  - Multi-tenant data partition    | |  - Reads deterministic stats    |
|  - Row-level user_id constraints  | |  - Returns 2-line clean nudges  |
+-----------------------------------+ +---------------------------------+

```

---

## 4. Data Models & Database Schema (Prisma)

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

enum IncomePattern {
  FIXED_MONTHLY
  VARIABLE
  MULTIPLE_SOURCES
}

enum RiskTolerance {
  LOW
  MEDIUM
  HIGH
}

enum InvestmentGoal {
  SHORT_TERM    // 0-3 years
  MEDIUM_TERM   // 3-5 years
  LONG_TERM     // 5+ years
}

enum TransactionType {
  INCOME
  EXPENSE
}

enum TransactionCategory {
  HOME
  EDUCATION
  SHOPPING
  TRANSPORT
  NUTRITION
  FREELANCE
  SALARY
  OTHER
}

enum BalanceItemType {
  ASSET
  LIABILITY
}

model User {
  id            String          @id @default(uuid())
  name          String?
  email         String          @unique
  passwordHash  String?
  image         String?
  createdAt     DateTime        @default(now())
  updatedAt     DateTime        @updatedAt

  profile       UserProfile?
  transactions  Transaction[]
  balanceItems  BalanceItem[]
  portfolio     UserPortfolio?

  @@index([email])
}

model UserProfile {
  id                  String         @id @default(uuid())
  userId              String         @unique
  user                User           @relation(fields: [userId], references: [id], onDelete: Cascade)
  incomePattern       IncomePattern  @default(FIXED_MONTHLY)
  primaryFinancialGoal String?       // e.g., "Emergency Fund", "Vacation"
  targetSavingsAmount Decimal?       @db.Decimal(12, 2)
  createdAt           DateTime       @default(now())
  updatedAt           DateTime       @updatedAt
}

model Transaction {
  id          String              @id @default(uuid())
  userId      String
  user        User                @relation(fields: [userId], references: [id], onDelete: Cascade)
  amount      Decimal             @db.Decimal(12, 2)
  type        TransactionType
  category    TransactionCategory
  note        String?             @db.VarChar(255)
  date        DateTime            @default(now())
  isRecurring Boolean             @default(false)
  createdAt   DateTime            @default(now())
  updatedAt   DateTime            @updatedAt

  @@index([userId, date])
  @@index([userId, type])
}

model BalanceItem {
  id          String          @id @default(uuid())
  userId      String
  user        User            @relation(fields: [userId], references: [id], onDelete: Cascade)
  name        String          @db.VarChar(100)
  type        BalanceItemType
  valuation   Decimal         @db.Decimal(12, 2)
  updatedAt   DateTime        @updatedAt

  @@index([userId, type])
}

model UserPortfolio {
  id              String         @id @default(uuid())
  userId          String         @unique
  user            User           @relation(fields: [userId], references: [id], onDelete: Cascade)
  riskLevel       RiskTolerance  @default(MEDIUM)
  goalHorizon     InvestmentGoal @default(MEDIUM_TERM)
  fdAllocation    Int            // Percentage integer (e.g. 40)
  mfAllocation    Int            // Percentage integer (e.g. 40)
  bondAllocation  Int            // Percentage integer (e.g. 20)
  updatedAt       DateTime       @updatedAt
}

```

---

## 5. Functional Requirements & Feature Specifications

### Module 1: Authentication & Onboarding

* **Auth Options:** Email/Password (bcrypt-hashed) and Google OAuth provider via NextAuth.js.
* **Session Strategy:** JSON Web Token (JWT) containing `user.id`. Every API query enforces `WHERE userId = session.user.id`.
* **Onboarding Flow:**
1. *Welcome & Value Prop:* Non-intrusive splash screen emphasizing manual entry without bank credentials.


2. *Help Priority Selection:* Option chips ("Track where my money goes", "Follow a monthly budget", "Save more money").


3. *Income Pattern:* Radio options (`FIXED_MONTHLY`, `VARIABLE`, `MULTIPLE_SOURCES`).


4. *Initial Financial Goal:* Input goal title (e.g., "Emergency Fund") and target amount.





### Module 2: Dashboard Overview (Pocketwise Layout)

* **Weekly Spending Bar Chart:**
* Displays spending over the current week (Monday through Sunday).


* Programmatically computes the peak day and applies the active brand color (`#19664D`), with other days rendered in muted slate/gray.


* Displays baseline dynamic banner: `"You're ₹X over/under daily average"`.




* **Balance Sheet Snapshot:**
* **Saved This Month:** `SUM(Income) - SUM(Expense)` for current month.


* **Net Worth:** `SUM(Assets) - SUM(Liabilities)`.


* Cards include sub-labels for easy reading.




* **Stateless Insight Owl Card:**
* Embedded container with an owl icon and dynamic copy (e.g., *"Most of your spend happens in 2 days. Nearly a third of your weekly expenses come from the weekend alone."*).


* Strictly stateless: generated dynamically on dashboard load via an aggregated prompt sent to the LLM service.



### Module 3: Smart Manual Entry Modal

* **Segmented Toggle:** Buttons for `Income`, `Expense`, `Asset`, `Liability`.


* **Inputs:**
* Amount input field (numeric only).
* Category selector chips (`Home`, `Education`, `Shopping`, `Transport`, `Nutrition`, etc.).


* Date picker (defaults to today; supports past entries).


* Note field (`Optional`, string, max 255 chars).


* Recurring toggle (`Daily`, `Weekly`, `Monthly`).




* **Submission Action:** Instant optimistic UI update followed by database persistence via Server Action or API route.

### Module 4: 5-Step Risk Assessment & Robo-Advisory Engine

Implements the 5-step portfolio recommendation pipeline:

```
[Step 1: Select Risk] -> [Step 2: Risk Profile Confirmation] -> [Step 3: Horizon Selection]
                                                                            |
                                                                            v
[Step 5: Visual Breakdown & Why Card]  <--  [Step 4: Recommended Portfolio Overview]

```

1. **Step 1 (Risk Comfort Selection):** Single-select choice between Low, Medium, High.


2. **Step 2 (Risk Confirmation):** Shows descriptive card explaining the selection (e.g., Medium = *"Balanced risk for balanced growth; suitable for medium-term investments"*).


3. **Step 3 (Investment Horizon):** User picks target timeline:
* Short-Term (0–3 years)


* Medium-Term (3–5 years)


* Long-Term (5+ years)




4. **Step 4 (Target Summary):** Calculates recommended monthly investment target based on the user's monthly net surplus (`Monthly Income - Monthly Expenses`).


5. **Step 5 (Asset Allocation & Portfolio Card):**
* Computes asset allocation percentages deterministically:
* *Low Risk:* 50% Fixed Deposits, 30% Govt Bonds, 20% Large Cap Mutual Funds.
* *Medium Risk:* 40% Fixed Deposits, 40% Mutual Funds, 20% Bonds.


* *High Risk:* 70% Diversified/Equity Mutual Funds, 20% Stocks, 10% Fixed Deposits/Liquid.


* Visualized via a Recharts Donut/Pie Chart.


* Includes *"Why This Portfolio Works for You"* card (highlighting capital protection vs. growth).


* Mandated disclaimer badge: *"Disclaimer: This is a conceptual UI recommendation. No real financial advice is given."*




---

## 6. Hybrid Advisory & Stateless LLM Pipeline

To guarantee mathematical precision and eliminate hallucinations, the LLM is never allowed to calculate financial aggregates. All metrics are calculated by deterministic database queries and injected as context.

### 6.1 Metric Synthesis Pipeline

```
Raw Transactions (Postgres)
           |
           v
Deterministic Math Engine (Compute weekly spikes, top category %, savings rate)
           |
           v
JSON Snapshot Context
           |
           v
LLM Inference Service (Stateless Prompt)
           |
           v
Insight Card Response (Headline + 1-Sentence Actionable Nudge)

```

### 6.2 Pre-Computed Aggregates Schema

```typescript
interface FinancialSnapshot {
  currentMonthIncome: number;
  currentMonthExpense: number;
  netSavings: number;
  savingsRatePct: number;
  topCategory: string;
  topCategoryPct: number;
  weekendSpendRatio: number; // e.g., 0.38 indicates 38% spent on Sat/Sun
  dailyAverageSpend: number;
}

```

### 6.3 System Prompt Template (Insight Owl)

```text
You are the "Insight Owl" for the Pocketwise personal finance application.
Your role is to act as a calm, wise, and non-judgmental financial peer.

Input Data:
- Total Income: ₹{currentMonthIncome}
- Total Expense: ₹{currentMonthExpense}
- Net Savings: ₹{netSavings} (Savings Rate: {savingsRatePct}%)
- Top Spending Category: {topCategory} ({topCategoryPct}% of total spending)
- Weekend Spending Share: {weekendSpendRatio * 100}%
- Average Daily Spend: ₹{dailyAverageSpend}

Output Guidelines:
1. Provide a 1-sentence headline summarizing the primary financial observation.
2. Provide a 1-sentence actionable, jargon-free recommendation.
3. Keep the tone friendly, reassuring, and completely free of moralizing guilt.
4. Output strict JSON with keys "headline" and "actionableTip".

```

---

## 7. UI/UX & Design Tokens

* **Color Palette:**
* **Primary Brand:** `#19664D` (Pocketwise Deep Forest Green)


* **Primary Hover/Active:** `#134e3a`
* **Surface Light:** `#F7F9F8` (App Canvas background)
* **Card Surface:** `#FFFFFF` with `border border-slate-100` and `shadow-sm`
* **Text Primary:** `#1E293B` (Slate-800)
* **Text Secondary:** `#64748B` (Slate-500)
* **Accent/Highlight:** `#22C55E` (Growth green), `#EF4444` (Expense alert red)


* **Typography:**
* Sans-Serif font family: `Inter` or `Roboto`.


* Numbers/Currency: Tabular figures (`font-mono` or `tabular-nums`) to prevent jitter during updates.


* **Component Styling:**
* Container radius: `rounded-2xl` for cards, `rounded-xl` for interactive pills/buttons.
* Inputs: Large tap targets (minimum `h-11`), clear active focus rings (`ring-2 ring-[#19664D]`).



---

## 8. API Specifications

### 8.1 Transaction Endpoints

* `POST /api/transactions`
* **Payload:** `{ amount: number, type: "INCOME"|"EXPENSE", category: string, note?: string, date: string, isRecurring: boolean }`
* **Auth:** Enforced via session token.
* **Response:** Created transaction object.


* `GET /api/transactions`
* **Query Params:** `?startDate=YYYY-MM-DD&endDate=YYYY-MM-DD&limit=50`
* **Response:** Array of user transactions.



### 8.2 Dashboard & Insights

* `GET /api/dashboard/summary`
* **Response:**
```json
{
  "savedThisMonth": 20000,
  "netWorth": 640000,
  "weeklyOverview": [
    { "day": "Mon", "amount": 380, "date": "2026-09-07" },
    { "day": "Tue", "amount": 30, "date": "2026-09-08" },
    { "day": "Wed", "amount": 300, "date": "2026-09-09" },
    { "day": "Thu", "amount": 240, "date": "2026-09-10" },
    { "day": "Fri", "amount": 200, "date": "2026-09-11" },
    { "day": "Sat", "amount": 500, "date": "2026-09-12", "isPeak": true },
    { "day": "Sun", "amount": 350, "date": "2026-09-13" }
  ],
  "overDailyAverage": 65,
  "insight": {
    "headline": "Most of your spend happens in 2 days",
    "actionableTip": "Nearly a third of your weekly expenses come from the weekend alone."
  }
}

```





### 8.3 Portfolio Recommendation Engine

* `POST /api/portfolio/recommend`
* **Payload:** `{ riskLevel: "LOW"|"MEDIUM"|"HIGH", goalHorizon: "SHORT_TERM"|"MEDIUM_TERM"|"LONG_TERM" }`
* **Response:**
```json
{
  "allocations": {
    "fixedDeposits": 40,
    "mutualFunds": 40,
    "bonds": 20
  },
  "projectedReturn": "7.2%",
  "riskScore": "Medium",
  "whyItWorks": "Combines capital preservation with moderate inflation-beating equity exposure."
}

```





---

## 9. Non-Functional Requirements & Security

1. **Tenant Isolation:** Every database operation must explicitly include `userId` within the Prisma `where` clause. Never rely solely on client-passed identifiers.
2. **Precision Arithmetic:** Currency amounts must be stored as `Decimal(12, 2)` to avoid floating-point rounding errors.
3. **Optimistic UI:** Manual entries should reflect instantly on the UI while syncing asynchronously in the background.
4. **Stateless LLM Reliability:** If the LLM provider times out (>2500ms) or fails, fallback to hardcoded rule-based string templates (e.g., *"You've logged X transactions this week. Keep going!"*) without failing dashboard page loads.

---

## 10. Phased Implementation Roadmap

* **Sprint 1: Core Foundation & Auth**
* Initialize Next.js project with Tailwind CSS and brand colors (`#19664D`).


* Set up PostgreSQL database and deploy Prisma migrations.
* Implement NextAuth (Credentials + Google OAuth) with protected route middleware.


* **Sprint 2: Transaction Logging & Dashboard Math**
* Build the Smart Manual Entry Modal (Income, Expense, Asset, Liability).


* Implement dashboard aggregation endpoints (Weekly Spending, Net Worth, Savings).


* Render the weekly spending bar chart using Recharts.




* **Sprint 3: 5-Step Advisory & Insight Owl**
* Build the 5-step portfolio recommendation interactive wizard.


* Construct deterministic allocation tables and Recharts donut visualization.


* Integrate stateless LLM service with failover fallbacks for dashboard Insight Owl cards.




* **Sprint 4: Polish & Edge Cases**
* Zero-state handling (empty screens for new accounts).
* Mobile viewport responsiveness testing and touch optimizations.