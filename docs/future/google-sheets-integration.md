# Parked: Google Sheets Integration

**Status:** Parked — Phase 2
**Origin:** Brainstorming session 2026-05-10 — Sven's existing workflow uses a Google Sheet with ISIN, WKN, Euro amount

## The Concept

Allow users to connect their Google Sheet as a data source. The app reads the ISIN, WKN, and amount columns and imports them as positions automatically.

## Why It Is Parked

V1 uses direct entry in the dashboard, which is simpler and has no external dependencies. The Google Sheet only contains three columns (ISIN, WKN, amount) — re-entering this data in the dashboard is a one-time low-effort task. For V1 (single user, personal tool), the complexity of OAuth + Google Sheets API is not justified.

Becomes relevant in V2 when multiple users arrive — many may already track portfolios in spreadsheets and want a one-click import path rather than re-entering all positions.

## What Is Needed to Implement Later

| Requirement | Notes |
|-------------|-------|
| Google OAuth scope | `https://www.googleapis.com/auth/spreadsheets.readonly` |
| Google Sheets API v4 | Read the user's sheet by URL/ID |
| Column mapping UI | User specifies which column is ISIN, which is WKN, which is amount |
| Sync vs. one-time import | Decide if positions stay in sync or are a one-time copy |
| Conflict resolution | What happens if the same ISIN is entered manually AND imported? |

## Open Questions

- Should the Sheet be read live on each page load, or imported once and then managed in the app?
- How do we handle Google Sheet permissions (the sheet must be shared with the app's OAuth client)?
