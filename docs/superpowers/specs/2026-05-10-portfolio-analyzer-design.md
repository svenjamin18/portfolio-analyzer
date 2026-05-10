# Design Spec — Portfolio Analyzer

**Date:** 2026-05-10
**Status:** Approved
**Author:** Sven Geisler (brainstorming session with Claude)

---

## 1. What We Are Building

A web-based portfolio cluster risk dashboard. The user enters their investment positions (name, ISIN, optional WKN, Euro amount). The app enriches each position with sector and region data, looks through ETFs to their internal allocations, and visualises where concentration risks lie. An AI chat (Claude via OpenRouter) has full portfolio context and gives personalised recommendations.

**V1 scope:** Single user (Sven), browser-only storage, no login, publicly deployed on Vercel.

---

## 2. Architecture

```
Browser (Next.js frontend)
  ├── Portfolio sidebar (entry + position list)
  ├── Analysis dashboard (3 charts + risk warnings)
  └── AI chat panel (OpenRouter)

Next.js API Routes (serverside, Vercel serverless)
  ├── /api/enrich   → Yahoo Finance + ETF provider APIs
  └── /api/chat     → OpenRouter (claude-sonnet-4-5)
```

**Data layer:** All portfolio data in `localStorage` via `PortfolioRepository` interface. Swappable for Supabase in V2 without UI changes.

---

## 3. Layout — Sidebar + Dashboard

Chosen layout: **A — Sidebar left, analysis right**.

```
┌─────────────────────────────────────────────────────┐
│  ⚠ Risk warnings banner (shown when thresholds hit) │
├───────────────┬─────────────────────────────────────┤
│               │  Sektor chart  │  Region chart      │
│  Positions    ├────────────────┼────────────────────┤
│  sidebar      │  Einzeltitel   │  AI Chat           │
│               │  chart         │  panel             │
├───────────────┴────────────────┴────────────────────┤
│  + Add position   Export   Import                   │
└─────────────────────────────────────────────────────┘
```

Dark mode only (matches Bloomberg/financial tool aesthetic).

---

## 4. Portfolio Entry

- Fields: Name, ISIN (required), WKN (optional), Euro amount
- On save: position added to localStorage, enrichment triggered automatically
- Positions shown in sidebar as cards with: name, ISIN, Euro amount, % of total, colour-coded by risk level
- Edit and delete per position

---

## 5. Data Enrichment

**Trigger:** On position add/edit, or manual "Refresh" button.
**Caching:** Results stored in localStorage with 7-day TTL per ISIN.
**Route:** `POST /api/enrich` — accepts ISIN, returns `EnrichmentData`.

### Stocks
- `yahoo-finance2` `search(isin)` → ticker
- `yahoo-finance2` `quoteSummary(ticker, { modules: ['assetProfile'] })` → sector + country
- Result: `{ type: 'stock', sector: 'Technology', country: 'US' }`

### ETFs
1. Identify as ETF via Yahoo Finance `quoteType: 'ETF'`
2. Lookup by ISIN prefix / provider mapping:
   - iShares ISINs → BlackRock fund API
   - Vanguard ISINs → Vanguard fund API
   - Xtrackers ISINs → DWS fund API
   - Amundi ISINs → Amundi fund API
3. Fallback: static JSON with top 30 European ETFs (MSCI World, S&P 500, FTSE All-World, EM, etc.)
4. If all fail: position flagged with warning, manual input offered
- Result: `{ type: 'etf', sectorBreakdown: {...}, regionBreakdown: {...} }`

---

## 6. Risk Analysis (Computed Client-Side)

All three views apply ETF look-through by weighting each ETF's breakdown by its share of total portfolio.

### Sektor Chart (Donut)
- Groups: Technology, Healthcare, Financials, Industrials, Consumer, Energy, Materials, Real Estate, Utilities, Other
- Each stock contributes its sector at full position weight
- Each ETF contributes its internal sector breakdown weighted by its portfolio share

### Region Chart (Donut)
- Regions: USA, Europa, Japan, Emerging Markets, Sonstige
- Same look-through logic as sector

### Einzeltitel Chart (Horizontal bar)
- Only individual stocks (type = 'stock'), not ETFs
- Shows % of total portfolio per stock
- Red bar + warning if any single stock > 25%

### Risk Warning Banner
- Shown at top when: any single stock > 25%, or any single region > 60%
- Lists each violation specifically ("Apple 29.9% — Einzelposition > 25%")

---

## 7. AI Chat

**Route:** `POST /api/chat` — streaming response.
**Model:** `anthropic/claude-sonnet-4-5` via OpenRouter.
**SDK:** OpenAI SDK with `baseURL: "https://openrouter.ai/api/v1"`.

### System Prompt (injected automatically)
```
You are a professional portfolio advisor. The user's current portfolio:

Total value: €{total}
Positions:
{positions list with name, type, weight%}

Risk analysis:
- Sector distribution: {sector breakdown}
- Region distribution: {region breakdown}
- Single-stock positions: {stocks with weight%}
- Active warnings: {list of threshold violations}

Today's date: {date}

Before making recommendations, ask the user about their investment horizon, 
risk tolerance, and any specific preferences or constraints. 
Recommendations should cover: missing regions, macro/political trends, 
technology trends, and specific stock or ETF ideas for a satellite portfolio.
```

### UX
- Chat panel bottom-right of dashboard
- Streaming responses
- Conversation history maintained in component state (not persisted)
- AI opens with a brief portfolio summary + first clarifying question

---

## 8. Export / Import

- **Export:** Serialise `Position[]` from localStorage to JSON, trigger browser download as `portfolio-{date}.json`
- **Import:** File picker → parse JSON → validate schema → write to localStorage → trigger enrichment for any new ISINs

---

## 9. Tech Stack

| Layer | Choice |
|-------|--------|
| Framework | Next.js 14, App Router, TypeScript |
| Styling | Tailwind CSS + shadcn/ui |
| Charts | Recharts (via shadcn/ui chart) |
| Stock data | `yahoo-finance2` npm |
| ETF data | Provider APIs + static fallback |
| AI | OpenAI SDK → OpenRouter → `anthropic/claude-sonnet-4-5` |
| Persistence | `localStorage` via `PortfolioRepository` interface |
| Hosting | Vercel (free tier) |

---

## 10. V2 Upgrade Path

The architecture is designed so V2 (multi-user, auth, DB) is an upgrade, not a rewrite:

| V1 | V2 change |
|----|-----------|
| `LocalStorageRepository` | Swap for `SupabaseRepository` |
| No auth | Add NextAuth (one middleware file) |
| No DB | Add Supabase (one new service) |
| No GDPR flow | Add consent banner + data processing agreement |

See `docs/future/multi-user-auth.md` for full V2 requirements.
