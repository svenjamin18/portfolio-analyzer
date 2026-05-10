# Infrastructure Specification — Portfolio Analyzer

**Last updated:** 2026-05-10

## Hosting

| Property | Value |
|----------|-------|
| Platform | Vercel |
| Tier | Free (Hobby) |
| Region | Auto (Vercel Edge Network) |
| URL (V1) | `portfolio-analyzer.vercel.app` (or custom subdomain) |

## Deployment

- **Trigger:** Push to `main` branch → automatic production deployment
- **Preview:** Every pull request gets an isolated preview URL
- **Build command:** `next build`
- **Output:** Static + serverless (Next.js default)

## Environment Variables

Set in Vercel dashboard under Project → Settings → Environment Variables.

| Variable | Required | Description |
|----------|----------|-------------|
| `OPENROUTER_API_KEY` | Yes | OpenRouter API key for AI chat |
| `YAHOO_FINANCE_CRUMB` | No | Yahoo Finance session crumb (if rate-limited) |

Never committed to git. Local development uses `.env.local` (in `.gitignore`).

## Traffic Flow

```
Browser
  └─> Vercel CDN (static assets: JS, CSS, fonts)
  └─> Vercel Serverless Functions (API routes)
        ├─> Yahoo Finance API   (stock enrichment)
        ├─> ETF Provider APIs   (ETF look-through)
        └─> OpenRouter API      (AI chat)
```

## Data Storage (V1)

No server-side database. All portfolio data lives in the user's browser localStorage.

## Scaling Considerations

Vercel free tier limits:
- 100 GB bandwidth/month
- 100,000 serverless function invocations/month
- Sufficient for personal use and early testing

If usage grows, upgrade to Vercel Pro ($20/month) before adding a database in V2.

## V2 Changes

When multi-user support is added (V2):
- Add Supabase (PostgreSQL) for portfolio storage
- Add NextAuth for authentication
- Migrate from `LocalStorageRepository` to `SupabaseRepository`
- No infrastructure changes needed — Supabase connects from existing serverless functions
