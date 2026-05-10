# Parked: N8N Workflow Integration

**Status:** Parked — potential future option
**Origin:** Brainstorming session 2026-05-10 — Sven had a partial N8N workflow for portfolio analysis

## The Concept

Use N8N as a data enrichment pipeline: the dashboard sends ISINs to an N8N webhook, N8N fetches sector/region/ETF data via its existing node integrations, and returns enriched data to the dashboard.

Alternatively: use the N8N MCP server to create and manage N8N workflows directly from Claude Code.

## Why It Is Parked

For V1, the enrichment logic is built directly into the Next.js API routes (Yahoo Finance + ETF provider APIs). This keeps the architecture simple with no external orchestration dependency. N8N adds operational overhead (running N8N instance, webhook management) that is not justified when the enrichment can be done inline.

N8N would only add value if:
- Enrichment logic becomes complex enough to warrant a visual workflow editor
- Multiple data sources need to be orchestrated with retry/error handling
- The same enrichment pipeline needs to serve multiple apps

## What Is Needed to Implement Later

| Requirement | Notes |
|-------------|-------|
| Running N8N instance | Self-hosted or N8N Cloud |
| N8N MCP server | Allows Claude Code to create/manage workflows via MCP |
| Webhook endpoint | N8N exposes a webhook the dashboard calls |
| Auth between app and N8N | Shared secret or token on the webhook |

## Open Questions

- Is there a scenario where N8N's orchestration capabilities justify the added complexity?
- If N8N is used, should it replace the Next.js API routes or complement them?
