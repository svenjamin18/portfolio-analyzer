# Portfolio Analyzer

A web-based portfolio cluster risk dashboard built with Next.js.

Enter your investment positions (ISIN, WKN, Euro amount). The app enriches each position with sector and region data — looking through ETFs to their internal allocations — and shows you where your concentration risks lie. An AI chat (Claude via OpenRouter) has full portfolio context and gives personalised recommendations.

## Features (V1)

- Direct portfolio entry with ISIN-based data enrichment
- ETF look-through — sector and region analysis accounts for ETF internal allocations
- Cluster risk views: Sector, Region, Individual Stock weight
- Risk warnings when thresholds are exceeded (single stock >25%, region >60%)
- AI portfolio advisor chat (OpenRouter + claude-sonnet-4-5)
- Export / Import portfolio as JSON

## Tech Stack

- **Framework:** Next.js 14 (App Router), TypeScript
- **UI:** Tailwind CSS + shadcn/ui
- **Charts:** Recharts
- **Data:** Yahoo Finance (stocks), ETF provider APIs (ETF look-through)
- **AI:** OpenRouter → `anthropic/claude-sonnet-4-5`
- **Hosting:** Vercel

## Docs

- [`docs/PRD.md`](docs/PRD.md) — Product requirements and roadmap
- [`docs/SDD.md`](docs/SDD.md) — System design and tech decisions
- [`docs/infrastructure-spec.md`](docs/infrastructure-spec.md) — Deployment and infrastructure
- [`docs/adr/`](docs/adr/) — Architecture decision records
- [`docs/future/`](docs/future/) — Parked features for later phases

## Local Development

```bash
npm install
cp .env.example .env.local
# Add your OPENROUTER_API_KEY to .env.local
npm run dev
```

Open [http://localhost:3000](http://localhost:3000).
