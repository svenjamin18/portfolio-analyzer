# ADR-003: localStorage for V1 Data Persistence

**Date:** 2026-05-10
**Status:** Accepted

## Decision

Store portfolio positions and enrichment cache in browser localStorage for V1, behind a `PortfolioRepository` interface.

## Context

V1 is built for a single user (Sven) on a single device. No server-side storage is needed. However, the architecture must make V2 (multi-user, Supabase) a clean upgrade — not a rewrite.

## Options Considered

| Option | Pros | Cons |
|--------|------|------|
| localStorage (abstracted) | Zero infra, no GDPR issues, instant | Single-device, lost on cache clear |
| Supabase from day one | Cross-device, V2-ready | Overkill for V1, needs auth |
| URL state (portfolio in URL hash) | No storage, shareable | URLs get long, fragile |
| IndexedDB | More storage, structured | Complex API, overkill |

## Reasoning

V1 scope is explicitly single-user, single-device. localStorage is sufficient. The critical constraint is that all storage calls go through a `PortfolioRepository` interface — swapping the implementation for Supabase in V2 touches one file, not the entire codebase.

Export/Import (JSON file) provides a manual escape hatch for cross-device portability.

## Consequences

- Data is lost if browser cache is cleared — acceptable for V1 personal use
- `PortfolioRepository` interface must be defined before any component reads/writes positions
- V2 migration: implement `SupabaseRepository`, swap in dependency injection — no UI changes needed
