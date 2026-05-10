# Product Requirements Document — Portfolio Analyzer

**Project name:** Portfolio Analyzer
**Author:** Sven Geisler
**Date:** 2026-05-10
**Status:** Planning

## What We Are Building

A web-based portfolio analysis dashboard that lets an investor enter their holdings (ISIN, WKN, Euro amount) and immediately see where their concentration risks lie — broken down by sector, geographic region, and individual stock weight. ETFs are analysed by looking through to their internal allocations so the cluster risk picture is complete and not distorted by ETF wrappers.

An integrated AI chat (Claude via OpenRouter) knows the current portfolio state and provides personalised recommendations: which regions are missing, which macro or technology trends are underrepresented, and what individual stocks or ETFs could round out a satellite portfolio.

## Why This Exists

Manual spreadsheet analysis of concentration risk is tedious and incomplete — especially for ETFs whose internal sector/region weights are invisible. No lightweight, self-hosted tool exists that combines automated ETF look-through with an AI advisor that understands the full portfolio context.

## Phase 1 — MVP (V1, for Sven only)

| Feature | Description |
|---------|-------------|
| Portfolio entry | Add/edit/delete positions: name, ISIN, optional WKN, Euro amount |
| Data persistence | Browser localStorage — data survives page reloads on same device |
| Stock enrichment | Yahoo Finance API → sector + country per ISIN |
| ETF look-through | ETF provider APIs (iShares, Vanguard, Xtrackers, Amundi) + static fallback for common ETFs → sector + region breakdown |
| Sector chart | Donut chart — portfolio-weighted sector distribution (look-through applied) |
| Region chart | Donut chart — portfolio-weighted regional distribution (look-through applied) |
| Single-stock chart | Bar chart — only individual equities (not ETFs), with % of total portfolio |
| Risk warnings | Red banner when single stock > 25 % or single region > 60 % |
| AI chat | OpenRouter + claude-sonnet-4-5, portfolio data injected as context, AI asks clarifying questions before recommending |
| Export / Import | JSON export and import of portfolio for device portability |
| Deployment | Vercel (free tier), publicly accessible URL |

## Phase 2 — Next

| Feature | Description |
|---------|-------------|
| Multi-user support | Supabase database + NextAuth authentication |
| Cross-device sync | Portfolio stored server-side per user instead of localStorage |
| Google Sheets import | One-click sync from a user's own Google Sheet |
| GDPR consent flow | Explicit consent + data processing agreement for EU users |
| Onboarding wizard | Step-by-step guide for new users entering their first portfolio |

## Phase 3 — Future / Parked

- N8N workflow integration (see `docs/future/n8n-integration.md`)
- Real-time price updates and portfolio valuation
- Tax reporting and realised gains tracking
- Mobile native app
- Multiple portfolios per user

## Tech Stack (Summary)

| Layer | Technology |
|-------|-----------|
| Framework | Next.js 14 (App Router) |
| Language | TypeScript |
| Styling | Tailwind CSS + shadcn/ui |
| Charts | Recharts (via shadcn/ui chart primitives) |
| Data layer | localStorage (V1), Supabase (V2) |
| AI | OpenRouter API — `anthropic/claude-sonnet-4-5` |
| Hosting | Vercel |

Full details → `docs/SDD.md`

## Open Items

- [ ] Confirm ETF provider API access and rate limits for iShares, Vanguard, Xtrackers
- [ ] Define exact threshold values for risk warnings (25 % single stock, 60 % region — to be tuned)
- [ ] Decide on fallback UX when ETF look-through data is unavailable

## What Is Not Being Built (V1)

- User authentication or accounts
- Real-time price updates or portfolio valuation
- Trading functionality
- Tax or gains reporting
- Mobile native app
- N8N integration
