# Pocketwise

Pocketwise is a full-stack personal finance and robo-advisory platform for tracking day-to-day cash flow, maintaining a personal balance sheet, and building diversified mutual fund SIP recommendations.

Built with Next.js App Router, Auth.js, Prisma, PostgreSQL, and Google Gemini, the application keeps transactional spending separate from long-term wealth so each metric remains clear and useful.

## Highlights

- Secure email/password and Google OAuth authentication
- Guided onboarding for income, expenses, savings, and liabilities
- Monthly "Left to Spend" budget with real-time progress
- Weekly expense trends and transaction management
- Dedicated financial profile and balance sheet
- Live Indian mutual fund NAV data from MFapi.in
- Gemini-powered Core & Satellite SIP recommendations
- Deterministic recommendations when an external service is unavailable
- Strict tenant isolation across all user-owned records

## Product Tour

### Authentication and onboarding

New users can create an account with email and password or continue with Google. The onboarding flow captures the financial baseline used throughout the dashboard.

<table>
  <tr>
    <td width="42%" align="center">
      <img src="image.png" alt="Pocketwise login screen" width="360" />
      <br />
      <sub><strong>Secure sign-in</strong></sub>
    </td>
    <td width="58%" align="center">
      <img src="image-1.png" alt="Pocketwise financial baseline onboarding form" width="520" />
      <br />
      <sub><strong>Financial baseline setup</strong></sub>
    </td>
  </tr>
</table>

### Money overview

The dashboard brings the monthly budget, net worth, spending insight, weekly activity, and recent transactions into one responsive workspace.

<p align="center">
  <img src="image-2.png" alt="Pocketwise money overview dashboard" width="900" />
  <br />
  <sub><strong>Dashboard with budget, net worth, insights, and transaction activity</strong></sub>
</p>

### Robo-advisory journey

The advisory wizard combines the user's risk comfort and investment horizon with a curated mutual fund universe, live NAV data, and strict diversification rules.

<table>
  <tr>
    <td width="50%" align="center">
      <img src="image-3.png" alt="Pocketwise investment risk selection" width="460" />
      <br />
      <sub><strong>1. Select risk comfort</strong></sub>
    </td>
    <td width="50%" align="center">
      <img src="image-4.png" alt="Pocketwise investment horizon selection" width="460" />
      <br />
      <sub><strong>2. Choose an investment horizon</strong></sub>
    </td>
  </tr>
</table>

<p align="center">
  <img src="image-5.png" alt="Pocketwise personalized SIP recommendation" width="620" />
  <br />
  <sub><strong>3. Review the personalized Core &amp; Satellite SIP allocation</strong></sub>
</p>

> **Disclaimer:** Pocketwise provides a conceptual portfolio recommendation for demonstration purposes. It does not provide financial advice.

## Tech Stack

| Area | Technology |
| --- | --- |
| Application | Next.js 16, React 19, TypeScript |
| Styling | Tailwind CSS, Lucide React |
| Authentication | Auth.js v5, Prisma Adapter, bcryptjs |
| Data | PostgreSQL, Prisma ORM, Decimal.js |
| Validation | Zod |
| Charts | Recharts |
| AI | Google Gemini |
| Market data | MFapi.in |
| Deployment | Vercel, Neon Postgres |

## Architecture

```text
src/
  actions/                  Authenticated server actions and domain operations
  app/                      App Router pages, layouts, and route handlers
  components/
    advisory/               Robo-advisory wizard
    auth/                   Authentication forms and controls
    dashboard/              Dashboard cards, charts, and transaction UI
    profile/                Financial profile and balance sheet forms
    ui/                     Shared UI primitives
  lib/                      Validation, market data, AI, and domain utilities
  auth.ts                   Auth.js configuration
  proxy.ts                  Protected-route authorization

prisma/
  schema.prisma             PostgreSQL schema
  seed.ts                   Demo data and curated mutual fund universe

lib/
  prisma.ts                 Development-safe Prisma singleton
  session.ts                Authenticated-user helper
  utils.ts                  Tailwind class composition helper
```

Server Components load dashboard data directly. Client Components are used only where interaction or browser state is required. Every user-owned database query is scoped by the authenticated `userId`.

## Getting Started

### Prerequisites

- Node.js 22 or newer
- npm
- PostgreSQL, locally or through a managed provider such as Neon

### 1. Install dependencies

```bash
npm install
```

### 2. Configure the environment

Copy the example environment file:

```powershell
Copy-Item .env.example .env
```

On macOS or Linux:

```bash
cp .env.example .env
```

Required variables:

```env
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/pocketwise?schema=public"
AUTH_SECRET="replace-with-a-strong-random-secret"
```

Optional integrations:

```env
GOOGLE_CLIENT_ID=""
GOOGLE_CLIENT_SECRET=""
GEMINI_API_KEY=""
GEMINI_RECOMMENDATION_TIMEOUT_MS="15000"
```

Generate a secure Auth.js secret with:

```bash
npx auth secret
```

Google OAuth must use this authorized redirect URI in Google Cloud:

```text
http://localhost:3000/api/auth/callback/google
```

Use the corresponding HTTPS URL for production. For example:

```text
https://your-domain.com/api/auth/callback/google
```

### 3. Prepare the database

```bash
npm run prisma:generate
npx prisma db push
npm run prisma:seed
```

### 4. Start the application

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

After seeding, use the demo account:

```text
Email:    demo@pocketwise.local
Password: pocketwise-demo-2026
```

## Main Routes

| Route | Purpose |
| --- | --- |
| `/` | Entry point; routes users to authentication or the dashboard |
| `/login` | Email/password and Google sign-in |
| `/register` | Account creation |
| `/onboarding` | Initial financial baseline setup |
| `/dashboard` | Monthly budget and activity overview |
| `/dashboard/profile` | Personal profile and balance sheet management |
| `/dashboard/invest` | Mutual fund advisory wizard |

## Financial Model

Pocketwise deliberately separates cash flow from wealth state.

**Cash flow** is stored in `Transaction`. Quick Add records only income and expenses, which drive weekly spending and recent activity.

**Wealth state** is stored in `UserProfile`, `BalanceItem`, and `UserPortfolio`. Assets and liabilities are managed independently from daily transactions.

```text
monthlyBudget = baselineMonthlyIncome - baselineMonthlyExpenses

leftToSpend = monthlyBudget - currentMonthExpenseTransactions

netWorth = totalAssets - totalLiabilities
```

Financial values use PostgreSQL decimals and `decimal.js` where exact arithmetic is required.

## Recommendation Pipeline

The advisory workflow uses a multi-level retrieval and generation pipeline:

1. Load the authenticated user's profile, balance sheet, and available monthly budget.
2. Build a category-aware basket from the curated `MutualFund` table.
3. Hydrate each candidate with its latest NAV from MFapi.in.
4. Ask Gemini for exactly three funds under Core, Satellite, and Liquid Buffer guardrails.
5. Validate the structured JSON response before returning it to the UI.
6. Use a risk-aware deterministic allocation if Gemini or MFapi is unavailable.

## Scripts

| Command | Description |
| --- | --- |
| `npm run dev` | Start the development server |
| `npm run build` | Generate Prisma Client, sync the schema, and create a production build |
| `npm run start` | Start the production server |
| `npm run lint` | Run ESLint |
| `npx tsc --noEmit` | Run TypeScript checks |
| `npm run prisma:generate` | Generate Prisma Client |
| `npm run prisma:migrate` | Create and apply a development migration |
| `npm run prisma:seed` | Seed demo and mutual fund data |

> `npm run build` executes `prisma db push`. Confirm that `DATABASE_URL` points to the intended database before running it.

## Production Deployment

For Vercel, configure all required variables in the Production environment and redeploy after making changes. Never prefix database credentials, OAuth secrets, or AI keys with `NEXT_PUBLIC_`.

Before deployment, verify the application locally:

```bash
npm run lint
npx tsc --noEmit
npm run build
```

Production checklist:

- Use a managed PostgreSQL database with SSL enabled.
- Generate a unique, strong `AUTH_SECRET`.
- Add the exact production callback URL to the Google OAuth client.
- Keep `GEMINI_API_KEY` server-side.
- Rotate any credential that has been committed or shared.
- Confirm that `.env` files are excluded from version control.

## Troubleshooting

### Google sign-in returns `Configuration`

Verify `AUTH_SECRET`, `GOOGLE_CLIENT_ID`, and `GOOGLE_CLIENT_SECRET` in the deployment environment. Confirm that the OAuth client and secret belong together and that the callback URL exactly matches:

```text
https://your-domain.com/api/auth/callback/google
```

Auth.js logs configuration failures in the server or Vercel function logs under `[Auth.js Error]`.

### Recommendations use the fallback allocation

Check the server logs for `[Gemini Portfolio Error]` or `[Gemini JSON Parse Error]`. Typical causes are a missing API key, a network timeout, or an invalid model response. Increase `GEMINI_RECOMMENDATION_TIMEOUT_MS` when necessary and restart the server after changing `.env`.

### Prisma changes do not appear

```bash
npx prisma db push
npm run prisma:generate
```

Run `npm run prisma:seed` again when the curated mutual fund universe is missing or stale.

## Security

- Environment files and build artifacts are excluded through `.gitignore`.
- AI and database credentials are read only by server-side modules.
- Server actions authenticate every request before accessing user data.
- Mutations and queries enforce ownership using the current user's ID.
- Passwords are hashed with bcrypt and never stored in plain text.

## License

This project is private and does not currently include an open-source license.
