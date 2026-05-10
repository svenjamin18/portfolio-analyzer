# ADR-001: Next.js as Application Framework

**Date:** 2026-05-10
**Status:** Accepted

## Decision

Use Next.js 14 (App Router) with TypeScript as the full-stack framework.

## Context

The project needs a frontend for the dashboard UI and a backend for securely calling external APIs (Yahoo Finance, ETF providers, OpenRouter) without exposing API keys to the browser. Both concerns need to be solved in a single deployable unit for simplicity.

## Options Considered

| Option | Pros | Cons |
|--------|------|------|
| Next.js | One repo, API routes built-in, Vercel-native, strong V2 upgrade path | TypeScript over Python |
| Python FastAPI + React | Best Python finance libs, familiar if Python-first | Two services to deploy, more infra |
| Next.js + Python serverless | Python logic + good UI | Two languages, most complex setup |
| Streamlit | Fastest prototype | Poor UX, not extensible for multi-user |

## Reasoning

Next.js solves both frontend and backend in one codebase. The data enrichment logic (Yahoo Finance via `yahoo-finance2` npm package) is available and well-maintained in Node.js. The V2 upgrade path (add Supabase + NextAuth) is a documented, well-trodden path in the Next.js ecosystem. Vercel deployment is zero-config.

## Consequences

- All code is TypeScript — no Python financial libraries available
- `yahoo-finance2` npm package handles stock enrichment adequately
- ETF look-through uses provider REST APIs, not Python pandas — acceptable for V1
- V2 auth and DB upgrade requires no framework change
