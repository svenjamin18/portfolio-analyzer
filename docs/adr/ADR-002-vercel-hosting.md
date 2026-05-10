# ADR-002: Vercel for Hosting

**Date:** 2026-05-10
**Status:** Accepted

## Decision

Deploy to Vercel on the free Hobby tier.

## Context

The app needs to be publicly accessible (not just localhost) from day one so it can later be shared with others. Zero DevOps overhead is required for V1.

## Options Considered

| Option | Pros | Cons |
|--------|------|------|
| Vercel | Zero-config for Next.js, free, preview deploys per PR, instant | Vendor lock-in, free tier limits |
| Netlify | Similar to Vercel | Slightly less Next.js-native |
| Railway / Render | More control | Requires more config, costs more |
| Self-hosted VPS | Full control | DevOps overhead not justified for V1 |

## Reasoning

Vercel is the canonical host for Next.js applications. Free tier is sufficient for personal use and early testing. Every PR gets an automatic preview URL. Production deploys on push to `main`.

## Consequences

- `OPENROUTER_API_KEY` and any other secrets set in Vercel dashboard, not in code
- Free tier has 100k serverless invocations/month — sufficient until real users arrive
- If usage grows: upgrade to Vercel Pro ($20/month) before adding Supabase in V2
