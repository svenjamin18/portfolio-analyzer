# System Design Document — Portfolio Analyzer

**Project:** Portfolio Analyzer
**Last updated:** 2026-05-10
**Status:** Agreed

## Frontend

| Decision | Choice | Reason |
|----------|--------|--------|
| Framework | Next.js 14 (App Router) | Full-stack in one repo, Vercel-native, easy V2 upgrade path |
| Language | TypeScript | Type safety for financial data structures |
| Styling | Tailwind CSS | Utility-first, fast iteration |
| Component library | shadcn/ui | Accessible, unstyled primitives, consistent design system |
| Charts | Recharts (via shadcn chart) | React-native, composable, good donut + bar support |
| State | React state + localStorage via data layer abstraction | No external state library needed for V1 |

## Backend

| Decision | Choice | Reason |
|----------|--------|--------|
| API layer | Next.js API Routes (App Router route handlers) | No separate backend needed, API keys stay server-side |
| Stock enrichment | `yahoo-finance2` npm package | Reliable ISIN search, returns sector + country |
| ETF look-through | ETF provider REST endpoints + static fallback | iShares/Vanguard/Xtrackers have queryable data |
| ISIN → ticker | Yahoo Finance `search()` method | Handles ISIN input natively |
| WKN handling | Stored for display only — ISIN used for all API calls | Keeps enrichment logic simple |
| Enrichment caching | localStorage (7-day TTL per ISIN) | Avoids repeated API calls for unchanged positions |

## Data Layer (Abstracted)

The data layer is behind a `PortfolioRepository` interface so V1 (localStorage) can be swapped for V2 (Supabase) without touching UI code.

```typescript
interface PortfolioRepository {
  getPositions(): Position[]
  savePositions(positions: Position[]): void
  getEnrichmentCache(isin: string): EnrichmentData | null
  setEnrichmentCache(isin: string, data: EnrichmentData): void
}
```

V1 implementation: `LocalStorageRepository`
V2 implementation: `SupabaseRepository` (not built yet — see `docs/future/multi-user-auth.md`)

## Data Model

```typescript
type Position = {
  id: string           // uuid
  name: string
  isin: string
  wkn?: string
  amountEur: number
  type: 'stock' | 'etf' | 'unknown'
  addedAt: string      // ISO date
}

type EnrichmentData = {
  isin: string
  name: string
  type: 'stock' | 'etf'
  // For stocks:
  sector?: string
  country?: string
  // For ETFs:
  sectorBreakdown?: Record<string, number>   // e.g. { "Technology": 0.23, "Healthcare": 0.12 }
  regionBreakdown?: Record<string, number>   // e.g. { "USA": 0.68, "Europe": 0.17 }
  fetchedAt: string    // ISO date — for cache TTL check
}
```

## ETF Look-Through Logic

1. Determine if position is ETF via Yahoo Finance `quoteSummary` (quoteType: `ETF`)
2. If ETF, attempt to fetch sector + region breakdown:
   - **iShares**: BlackRock fund data API
   - **Vanguard**: Vanguard fund data endpoint
   - **Xtrackers (DWS)**: Xtrackers fund data endpoint
   - **Amundi**: Amundi fund data endpoint
   - **Fallback**: Static JSON file of pre-filled data for the 30 most common European ETFs
3. If look-through fails: flag position in UI with a warning, allow manual input
4. Apply ETF weight to its breakdown when computing portfolio totals

## AI Integration

| Decision | Choice | Reason |
|----------|--------|--------|
| Provider | OpenRouter | Sven has existing OpenRouter API key |
| Model | `anthropic/claude-sonnet-4-5` | Fast, high quality, cost-effective |
| SDK | OpenAI SDK with custom `baseURL` | OpenRouter is OpenAI API-compatible |
| Context injection | Portfolio positions + computed risk metrics in system prompt | AI has full picture before user asks |
| Interaction style | Chat with clarifying questions before recommendations | Preferences (horizon, risk tolerance) must be known first |

System prompt injects: total portfolio value, each position with weight and type, computed sector/region breakdown, current risk warnings, today's date.

## Infrastructure & Hosting

Full details → `docs/infrastructure-spec.md`

| Layer | Choice | Reason |
|-------|--------|--------|
| Hosting | Vercel | Free tier, zero-config Next.js, instant deploys |
| Domain | Vercel subdomain (V1) | No cost, immediate |
| Environment variables | Vercel dashboard | `OPENROUTER_API_KEY`, `YAHOO_FINANCE_API_KEY` (if needed) |

## Authentication

**V1:** None. App is publicly accessible.
**V2:** NextAuth + Supabase — see `docs/future/multi-user-auth.md`

## Development Workflow

1. Local: `npm run dev` → `http://localhost:3000`
2. Feature branch → push → Vercel preview deployment auto-created
3. PR merged to `main` → Vercel production deployment auto-triggered

## Out of Scope (V1)

- Database (→ `docs/future/multi-user-auth.md`)
- Authentication (→ `docs/future/multi-user-auth.md`)
- Google Sheets integration (→ `docs/future/google-sheets-integration.md`)
- Real-time price data
- N8N integration (→ `docs/future/n8n-integration.md`)
