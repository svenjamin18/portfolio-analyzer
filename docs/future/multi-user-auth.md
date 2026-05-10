# Parked: Multi-User Support + Authentication

**Status:** Parked — Phase 2
**Origin:** Brainstorming session 2026-05-10 — user wants to eventually share the tool with others

## The Concept

Allow multiple users to each have their own portfolio, persisted server-side and accessible from any device. Each user logs in with email/password or OAuth (Google).

## Why It Is Parked

V1 is built for a single user (Sven) on a single device. Adding auth and a database before the core analysis logic is validated would add significant complexity with no immediate benefit. The data layer abstraction (`PortfolioRepository` interface) already makes this upgrade clean.

Privacy concern: financial portfolio data (ISIN + amounts) is sensitive personal data under GDPR. Storing it server-side requires a proper data processing agreement, consent flow, encryption at rest, and a clear data retention policy. This is a non-trivial compliance effort.

## What Is Needed to Implement Later

| Requirement | Notes |
|-------------|-------|
| Supabase project | Free tier sufficient to start, PostgreSQL |
| `SupabaseRepository` implementation | Swaps in for `LocalStorageRepository` via the existing interface |
| NextAuth setup | Email + Google OAuth at minimum |
| GDPR consent flow | Cookie banner + data processing consent on first login |
| Data encryption | Supabase Row Level Security (RLS) + encrypted sensitive fields |
| Migration UX | Allow existing localStorage users to "claim" their data on sign-up |

## Open Questions

- Should portfolio amounts be encrypted at the field level, or is Supabase RLS + HTTPS sufficient?
- Which OAuth providers to support in V2? (Google likely, GitHub optional)
- What is the data retention policy if a user deletes their account?
