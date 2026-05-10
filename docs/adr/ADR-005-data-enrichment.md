# ADR-005: Data Enrichment Strategy (Stocks + ETFs)

**Date:** 2026-05-10
**Status:** Accepted

## Decision

Use Yahoo Finance (`yahoo-finance2`) for individual stock enrichment and ETF provider APIs with a static fallback for ETF look-through.

## Context

Every position needs sector and region/country data to compute cluster risk. This data is not entered by the user — it must be fetched automatically. ETFs add complexity because their risk must be "looked through" to internal allocations.

## Options Considered

| Option | Pros | Cons |
|--------|------|------|
| Yahoo Finance (yahoo-finance2) | Free, TypeScript package, ISIN search, sector + country for stocks | ETF look-through not available |
| OpenFIGI | Free, ISIN → metadata mapping | Limited sector/region data |
| Financial Modeling Prep | Good ETF data | Paid for full access |
| ETF provider APIs directly | Authoritative source | Different API per provider, no standard |
| Static ETF database | Simple, reliable | Requires manual maintenance |

## Reasoning — Stocks

`yahoo-finance2` `search()` accepts ISINs and returns ticker, sector, country. Runs server-side in Next.js API routes (API key not exposed). 7-day enrichment cache in localStorage avoids repeated calls.

## Reasoning — ETFs

ETF look-through requires the internal allocation data. The most reliable free source is the ETF providers themselves:

- **iShares (BlackRock)**: `https://www.blackrock.com/...` fund data endpoints
- **Vanguard**: Vanguard fund API
- **Xtrackers (DWS)**: Xtrackers fund data endpoint
- **Amundi**: Amundi fund data endpoint

Fallback chain:
1. Try provider API based on ISIN prefix / known fund mapping
2. If provider API fails: use static JSON of pre-seeded data for top 30 European ETFs
3. If not in static data: show warning in UI, allow manual sector/region input

## Consequences

- Enrichment logic runs server-side (Next.js API route `/api/enrich`)
- Results cached in localStorage with `fetchedAt` timestamp, TTL = 7 days
- WKN stored for display only — ISIN is the canonical identifier for all API calls
- ETFs not in the fallback list require manual input — acceptable for V1
- If Yahoo Finance changes structure: update `yahoo-finance2` package version
