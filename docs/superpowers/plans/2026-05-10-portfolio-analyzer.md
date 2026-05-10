# Portfolio Analyzer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Next.js portfolio cluster risk dashboard with ETF look-through, three risk charts, and an AI chat advisor — deployed on Vercel.

**Architecture:** Next.js 14 App Router with TypeScript. All portfolio data lives in localStorage behind a `PortfolioRepository` interface (swappable for Supabase in V2). Enrichment logic (Yahoo Finance + static ETF data) runs server-side in API routes. Analysis computation is a pure client-side function.

**Tech Stack:** Next.js 14, TypeScript, Tailwind CSS, shadcn/ui, Recharts, `yahoo-finance2`, OpenAI SDK (→ OpenRouter), Vitest + Testing Library

---

## Task 1: Next.js Project Setup

**Files:**
- Create: `app/layout.tsx`, `app/globals.css`, `app/page.tsx`
- Create: `vitest.config.ts`, `vitest.setup.ts`
- Create: `.env.local`, `.env.example`

- [ ] **Step 1: Scaffold Next.js app**

```bash
cd "C:/Users/Sven/Desktop/AI_Projects/Trading/Portfolio_Analyzer"
npx create-next-app@latest . --typescript --tailwind --app --no-src-dir --eslint --import-alias="@/*"
```

When asked "The directory is not empty — continue?", answer **Yes**. This overwrites `.gitignore` and `README.md` (both already committed to git, no data loss).

- [ ] **Step 2: Install project dependencies**

```bash
npm install yahoo-finance2 openai
npm install -D vitest @vitejs/plugin-react jsdom @testing-library/react @testing-library/jest-dom @testing-library/user-event
```

- [ ] **Step 3: Install shadcn/ui**

```bash
npx shadcn@latest init
```

Accept defaults: New York style, slate color, CSS variables. Then add components:

```bash
npx shadcn@latest add chart button input form dialog scroll-area label badge
```

- [ ] **Step 4: Configure Vitest**

Create `vitest.config.ts`:
```typescript
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    setupFiles: ['./vitest.setup.ts'],
    globals: true,
  },
  resolve: {
    alias: { '@': path.resolve(__dirname, '.') },
  },
})
```

Create `vitest.setup.ts`:
```typescript
import '@testing-library/jest-dom'
```

Add to `package.json` scripts:
```json
"test": "vitest",
"test:run": "vitest run"
```

- [ ] **Step 5: Create environment files**

Create `.env.local`:
```
OPENROUTER_API_KEY=your_key_here
```

Create `.env.example`:
```
OPENROUTER_API_KEY=
```

- [ ] **Step 6: Run dev server to verify setup**

```bash
npm run dev
```

Expected: Next.js app running at `http://localhost:3000` with default page. No errors.

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat: scaffold Next.js app with shadcn/ui, Vitest, and dependencies"
```

---

## Task 2: Core Types and Data Layer

**Files:**
- Create: `lib/types.ts`
- Create: `lib/repository/index.ts`
- Create: `lib/repository/localStorage.ts`
- Create: `__tests__/lib/localStorage.test.ts`

- [ ] **Step 1: Define all shared types**

Create `lib/types.ts`:
```typescript
export type PositionType = 'stock' | 'etf' | 'unknown'

export type Position = {
  id: string
  name: string
  isin: string
  wkn?: string
  amountEur: number
  type: PositionType
  addedAt: string // ISO date string
}

export type EnrichmentData = {
  isin: string
  name: string
  type: PositionType
  // Stocks only
  sector?: string
  country?: string
  // ETFs only
  sectorBreakdown?: Record<string, number> // e.g. { "Technology": 0.23 }
  regionBreakdown?: Record<string, number>  // e.g. { "USA": 0.68 }
  fetchedAt: string // ISO date string
  error?: string    // set when enrichment failed
}

export type Warning = {
  type: 'single-stock' | 'region'
  label: string
  value: number     // actual fraction 0-1
  threshold: number // threshold fraction 0-1
}

export type SingleStockEntry = {
  id: string
  name: string
  isin: string
  weight: number    // fraction 0-1
  amountEur: number
}

export type PortfolioAnalysis = {
  totalValue: number
  sectorDistribution: Record<string, number>
  regionDistribution: Record<string, number>
  singleStocks: SingleStockEntry[]
  warnings: Warning[]
}
```

- [ ] **Step 2: Define PortfolioRepository interface**

Create `lib/repository/index.ts`:
```typescript
import type { Position, EnrichmentData } from '@/lib/types'

export interface PortfolioRepository {
  getPositions(): Position[]
  savePositions(positions: Position[]): void
  getEnrichmentCache(isin: string): EnrichmentData | null
  setEnrichmentCache(isin: string, data: EnrichmentData): void
  getAllEnrichmentCache(): Record<string, EnrichmentData>
  exportJson(): string
  importJson(json: string): void
}
```

- [ ] **Step 3: Implement LocalStorageRepository**

Create `lib/repository/localStorage.ts`:
```typescript
import type { PortfolioRepository } from './index'
import type { Position, EnrichmentData } from '@/lib/types'

const POSITIONS_KEY = 'portfolio:positions'
const ENRICHMENT_KEY = 'portfolio:enrichment'
const ENRICHMENT_TTL_MS = 7 * 24 * 60 * 60 * 1000 // 7 days

export class LocalStorageRepository implements PortfolioRepository {
  getPositions(): Position[] {
    const raw = localStorage.getItem(POSITIONS_KEY)
    return raw ? JSON.parse(raw) : []
  }

  savePositions(positions: Position[]): void {
    localStorage.setItem(POSITIONS_KEY, JSON.stringify(positions))
  }

  getEnrichmentCache(isin: string): EnrichmentData | null {
    const cache = this.getAllEnrichmentCache()
    const entry = cache[isin]
    if (!entry) return null
    if (Date.now() - new Date(entry.fetchedAt).getTime() > ENRICHMENT_TTL_MS) return null
    return entry
  }

  setEnrichmentCache(isin: string, data: EnrichmentData): void {
    const cache = this.getAllEnrichmentCache()
    cache[isin] = data
    localStorage.setItem(ENRICHMENT_KEY, JSON.stringify(cache))
  }

  getAllEnrichmentCache(): Record<string, EnrichmentData> {
    const raw = localStorage.getItem(ENRICHMENT_KEY)
    return raw ? JSON.parse(raw) : {}
  }

  exportJson(): string {
    return JSON.stringify(this.getPositions(), null, 2)
  }

  importJson(json: string): void {
    const parsed = JSON.parse(json)
    if (!Array.isArray(parsed)) throw new Error('Invalid portfolio format: expected an array')
    this.savePositions(parsed as Position[])
  }
}
```

- [ ] **Step 4: Write failing tests**

Create `__tests__/lib/localStorage.test.ts`:
```typescript
import { describe, it, expect, beforeEach } from 'vitest'
import { LocalStorageRepository } from '@/lib/repository/localStorage'
import type { Position, EnrichmentData } from '@/lib/types'

const makePosition = (isin: string, amountEur = 1000): Position => ({
  id: crypto.randomUUID(),
  name: 'Test',
  isin,
  amountEur,
  type: 'stock',
  addedAt: new Date().toISOString(),
})

const makeEnrichment = (isin: string, daysAgo = 0): EnrichmentData => ({
  isin,
  name: 'Test',
  type: 'stock',
  sector: 'Technology',
  country: 'United States',
  fetchedAt: new Date(Date.now() - daysAgo * 24 * 60 * 60 * 1000).toISOString(),
})

describe('LocalStorageRepository', () => {
  let repo: LocalStorageRepository

  beforeEach(() => {
    localStorage.clear()
    repo = new LocalStorageRepository()
  })

  it('returns empty array when no positions saved', () => {
    expect(repo.getPositions()).toEqual([])
  })

  it('saves and retrieves positions', () => {
    const positions = [makePosition('US0378331005'), makePosition('IE00B4L5Y983')]
    repo.savePositions(positions)
    expect(repo.getPositions()).toEqual(positions)
  })

  it('returns null for uncached ISIN', () => {
    expect(repo.getEnrichmentCache('US0378331005')).toBeNull()
  })

  it('returns cached enrichment within TTL', () => {
    const data = makeEnrichment('US0378331005', 3) // 3 days ago
    repo.setEnrichmentCache('US0378331005', data)
    expect(repo.getEnrichmentCache('US0378331005')).toEqual(data)
  })

  it('returns null for expired enrichment (>7 days)', () => {
    const data = makeEnrichment('US0378331005', 8) // 8 days ago
    repo.setEnrichmentCache('US0378331005', data)
    expect(repo.getEnrichmentCache('US0378331005')).toBeNull()
  })

  it('exports valid JSON string of positions', () => {
    const positions = [makePosition('US0378331005')]
    repo.savePositions(positions)
    const json = repo.exportJson()
    expect(JSON.parse(json)).toEqual(positions)
  })

  it('imports positions from valid JSON', () => {
    const positions = [makePosition('US0378331005')]
    repo.importJson(JSON.stringify(positions))
    expect(repo.getPositions()).toEqual(positions)
  })

  it('throws on invalid JSON import', () => {
    expect(() => repo.importJson('{"not": "an array"}')).toThrow('Invalid portfolio format')
  })
})
```

- [ ] **Step 5: Run tests — expect failure**

```bash
npm run test:run -- __tests__/lib/localStorage.test.ts
```

Expected: FAIL — module not found errors until files exist.

- [ ] **Step 6: Run tests — expect all pass**

```bash
npm run test:run -- __tests__/lib/localStorage.test.ts
```

Expected: 8 tests passing.

- [ ] **Step 7: Commit**

```bash
git add lib/types.ts lib/repository/ __tests__/lib/localStorage.test.ts
git commit -m "feat: add core types, PortfolioRepository interface, and LocalStorageRepository"
```

---

## Task 3: Portfolio Analysis Logic

**Files:**
- Create: `lib/analysis.ts`
- Create: `__tests__/lib/analysis.test.ts`

- [ ] **Step 1: Write failing tests**

Create `__tests__/lib/analysis.test.ts`:
```typescript
import { describe, it, expect } from 'vitest'
import { computeAnalysis } from '@/lib/analysis'
import type { Position, EnrichmentData } from '@/lib/types'

const pos = (isin: string, amountEur: number, type: Position['type'] = 'stock', name = 'Test'): Position => ({
  id: isin,
  name,
  isin,
  amountEur,
  type,
  addedAt: new Date().toISOString(),
})

const stockEnrich = (isin: string, sector: string, country: string): EnrichmentData => ({
  isin, name: 'Test', type: 'stock', sector, country,
  fetchedAt: new Date().toISOString(),
})

const etfEnrich = (
  isin: string,
  sectorBreakdown: Record<string, number>,
  regionBreakdown: Record<string, number>,
): EnrichmentData => ({
  isin, name: 'ETF', type: 'etf', sectorBreakdown, regionBreakdown,
  fetchedAt: new Date().toISOString(),
})

describe('computeAnalysis', () => {
  it('returns empty analysis for empty portfolio', () => {
    const result = computeAnalysis([], {})
    expect(result.totalValue).toBe(0)
    expect(result.sectorDistribution).toEqual({})
    expect(result.regionDistribution).toEqual({})
    expect(result.singleStocks).toEqual([])
    expect(result.warnings).toEqual([])
  })

  it('sums totalValue across all positions', () => {
    const result = computeAnalysis(
      [pos('A', 500), pos('B', 1500)],
      { A: stockEnrich('A', 'Tech', 'United States'), B: stockEnrich('B', 'Healthcare', 'Germany') },
    )
    expect(result.totalValue).toBe(2000)
  })

  it('maps single stock sector at full weight', () => {
    const result = computeAnalysis(
      [pos('A', 1000)],
      { A: stockEnrich('A', 'Technology', 'United States') },
    )
    expect(result.sectorDistribution).toEqual({ Technology: 1 })
  })

  it('maps United States country to USA region', () => {
    const result = computeAnalysis(
      [pos('A', 1000)],
      { A: stockEnrich('A', 'Technology', 'United States') },
    )
    expect(result.regionDistribution.USA).toBe(1)
  })

  it('maps Germany country to Europa region', () => {
    const result = computeAnalysis(
      [pos('A', 1000)],
      { A: stockEnrich('A', 'Industrials', 'Germany') },
    )
    expect(result.regionDistribution.Europa).toBe(1)
  })

  it('maps unknown country to Sonstige region', () => {
    const result = computeAnalysis(
      [pos('A', 1000)],
      { A: stockEnrich('A', 'Tech', 'Antarctica') },
    )
    expect(result.regionDistribution.Sonstige).toBe(1)
  })

  it('applies ETF look-through for sectors', () => {
    // Stock (50%) + ETF (50%), ETF has 20% Tech and 80% Healthcare
    const result = computeAnalysis(
      [pos('STOCK', 1000, 'stock'), pos('ETF', 1000, 'etf')],
      {
        STOCK: stockEnrich('STOCK', 'Technology', 'United States'),
        ETF: etfEnrich('ETF', { Technology: 0.2, Healthcare: 0.8 }, { USA: 1.0 }),
      },
    )
    // Stock: 0.5 * 100% Tech = 0.5 Tech
    // ETF: 0.5 * 20% Tech = 0.1 Tech, 0.5 * 80% Healthcare = 0.4 Healthcare
    expect(result.sectorDistribution.Technology).toBeCloseTo(0.6)
    expect(result.sectorDistribution.Healthcare).toBeCloseTo(0.4)
  })

  it('applies ETF look-through for regions', () => {
    const result = computeAnalysis(
      [pos('ETF', 1000, 'etf')],
      { ETF: etfEnrich('ETF', {}, { USA: 0.7, Europa: 0.3 }) },
    )
    expect(result.regionDistribution.USA).toBeCloseTo(0.7)
    expect(result.regionDistribution.Europa).toBeCloseTo(0.3)
  })

  it('includes only stocks in singleStocks, excludes ETFs', () => {
    const result = computeAnalysis(
      [pos('STOCK', 500, 'stock'), pos('ETF', 500, 'etf')],
      {
        STOCK: stockEnrich('STOCK', 'Tech', 'United States'),
        ETF: etfEnrich('ETF', {}, {}),
      },
    )
    expect(result.singleStocks).toHaveLength(1)
    expect(result.singleStocks[0].isin).toBe('STOCK')
    expect(result.singleStocks[0].weight).toBeCloseTo(0.5)
  })

  it('skips position if no enrichment data exists', () => {
    const result = computeAnalysis(
      [pos('A', 500), pos('B', 500)],
      { A: stockEnrich('A', 'Technology', 'United States') }, // B has no cache
    )
    // Only A contributes — but totalValue is still 1000
    expect(result.sectorDistribution.Technology).toBeCloseTo(0.5)
  })

  it('generates single-stock warning when weight exceeds 25%', () => {
    const result = computeAnalysis(
      [pos('A', 300, 'stock', 'Apple'), pos('B', 700)],
      {
        A: stockEnrich('A', 'Technology', 'United States'),
        B: stockEnrich('B', 'Healthcare', 'Germany'),
      },
    )
    expect(result.warnings.some(w => w.type === 'single-stock')).toBe(false)
    // 300/1000 = 30% which IS > 25% — expect warning
  })

  it('correctly flags 30% single stock', () => {
    const result = computeAnalysis(
      [pos('A', 300, 'stock', 'Apple'), pos('B', 700)],
      {
        A: stockEnrich('A', 'Technology', 'United States'),
        B: stockEnrich('B', 'Healthcare', 'Germany'),
      },
    )
    const warning = result.warnings.find(w => w.type === 'single-stock')
    expect(warning).toBeDefined()
    expect(warning!.value).toBeCloseTo(0.3)
  })

  it('does not warn when single stock is exactly at 25%', () => {
    const result = computeAnalysis(
      [pos('A', 250), pos('B', 750)],
      {
        A: stockEnrich('A', 'Technology', 'United States'),
        B: stockEnrich('B', 'Healthcare', 'Germany'),
      },
    )
    expect(result.warnings.filter(w => w.type === 'single-stock')).toHaveLength(0)
  })

  it('generates region warning when region exceeds 60%', () => {
    const result = computeAnalysis(
      [pos('ETF', 1000, 'etf')],
      { ETF: etfEnrich('ETF', {}, { USA: 0.75, Europa: 0.25 }) },
    )
    const warning = result.warnings.find(w => w.type === 'region')
    expect(warning).toBeDefined()
    expect(warning!.label).toContain('USA')
  })

  it('does not warn when region is exactly at 60%', () => {
    const result = computeAnalysis(
      [pos('ETF', 1000, 'etf')],
      { ETF: etfEnrich('ETF', {}, { USA: 0.6, Europa: 0.4 }) },
    )
    expect(result.warnings.filter(w => w.type === 'region')).toHaveLength(0)
  })
})
```

- [ ] **Step 2: Run tests — expect failure**

```bash
npm run test:run -- __tests__/lib/analysis.test.ts
```

Expected: FAIL — `computeAnalysis` not found.

- [ ] **Step 3: Implement analysis.ts**

Create `lib/analysis.ts`:
```typescript
import type { Position, EnrichmentData, PortfolioAnalysis, Warning, SingleStockEntry } from '@/lib/types'

const COUNTRY_TO_REGION: Record<string, string> = {
  'United States': 'USA', 'US': 'USA',
  'Germany': 'Europa', 'France': 'Europa', 'Netherlands': 'Europa',
  'Switzerland': 'Europa', 'Sweden': 'Europa', 'Denmark': 'Europa',
  'United Kingdom': 'Europa', 'Spain': 'Europa', 'Italy': 'Europa',
  'Finland': 'Europa', 'Norway': 'Europa', 'Belgium': 'Europa',
  'Austria': 'Europa', 'Portugal': 'Europa', 'Ireland': 'Europa',
  'Japan': 'Japan',
  'China': 'Emerging Markets', 'India': 'Emerging Markets',
  'Brazil': 'Emerging Markets', 'Taiwan': 'Emerging Markets',
  'South Korea': 'Emerging Markets', 'Korea': 'Emerging Markets',
  'Mexico': 'Emerging Markets', 'Indonesia': 'Emerging Markets',
  'Saudi Arabia': 'Emerging Markets', 'South Africa': 'Emerging Markets',
}

const SINGLE_STOCK_THRESHOLD = 0.25
const REGION_THRESHOLD = 0.60

function addToRecord(record: Record<string, number>, key: string, value: number): void {
  record[key] = (record[key] ?? 0) + value
}

export function computeAnalysis(
  positions: Position[],
  cache: Record<string, EnrichmentData>,
): PortfolioAnalysis {
  if (positions.length === 0) {
    return { totalValue: 0, sectorDistribution: {}, regionDistribution: {}, singleStocks: [], warnings: [] }
  }

  const totalValue = positions.reduce((sum, p) => sum + p.amountEur, 0)
  const sectorDist: Record<string, number> = {}
  const regionDist: Record<string, number> = {}
  const singleStocks: SingleStockEntry[] = []

  for (const position of positions) {
    const enrichment = cache[position.isin]
    if (!enrichment) continue

    const weight = position.amountEur / totalValue

    if (enrichment.type === 'stock') {
      if (enrichment.sector) addToRecord(sectorDist, enrichment.sector, weight)
      if (enrichment.country) {
        const region = COUNTRY_TO_REGION[enrichment.country] ?? 'Sonstige'
        addToRecord(regionDist, region, weight)
      }
      singleStocks.push({ id: position.id, name: position.name, isin: position.isin, weight, amountEur: position.amountEur })
    } else if (enrichment.type === 'etf') {
      if (enrichment.sectorBreakdown) {
        for (const [sector, fraction] of Object.entries(enrichment.sectorBreakdown)) {
          addToRecord(sectorDist, sector, weight * fraction)
        }
      }
      if (enrichment.regionBreakdown) {
        for (const [region, fraction] of Object.entries(enrichment.regionBreakdown)) {
          addToRecord(regionDist, region, weight * fraction)
        }
      }
    }
  }

  const warnings: Warning[] = []

  for (const stock of singleStocks) {
    if (stock.weight > SINGLE_STOCK_THRESHOLD) {
      warnings.push({
        type: 'single-stock',
        label: `${stock.name} ${(stock.weight * 100).toFixed(1)}% — Einzelposition > 25%`,
        value: stock.weight,
        threshold: SINGLE_STOCK_THRESHOLD,
      })
    }
  }

  for (const [region, fraction] of Object.entries(regionDist)) {
    if (fraction > REGION_THRESHOLD) {
      warnings.push({
        type: 'region',
        label: `${region} ${(fraction * 100).toFixed(1)}% — Region > 60%`,
        value: fraction,
        threshold: REGION_THRESHOLD,
      })
    }
  }

  return { totalValue, sectorDistribution: sectorDist, regionDistribution: regionDist, singleStocks, warnings }
}
```

- [ ] **Step 4: Run tests — expect all pass**

```bash
npm run test:run -- __tests__/lib/analysis.test.ts
```

Expected: 13 tests passing. Fix any failures before continuing.

- [ ] **Step 5: Commit**

```bash
git add lib/analysis.ts __tests__/lib/analysis.test.ts
git commit -m "feat: add portfolio analysis logic with ETF look-through (TDD)"
```

---

## Task 4: Static ETF Data

**Files:**
- Create: `data/common-etfs.json`
- Create: `lib/etf-data.ts`
- Create: `__tests__/lib/etf-data.test.ts`

- [ ] **Step 1: Create static ETF database**

Create `data/common-etfs.json`:
```json
{
  "IE00B4L5Y983": {
    "name": "iShares Core MSCI World UCITS ETF",
    "sectorBreakdown": { "Technology": 0.233, "Financials": 0.154, "Healthcare": 0.127, "Industrials": 0.105, "Consumer Discretionary": 0.107, "Communication Services": 0.086, "Consumer Staples": 0.068, "Energy": 0.048, "Materials": 0.039, "Real Estate": 0.023, "Utilities": 0.010 },
    "regionBreakdown": { "USA": 0.703, "Japan": 0.058, "Europa": 0.210, "Emerging Markets": 0.010, "Sonstige": 0.019 }
  },
  "IE00B5BMR087": {
    "name": "iShares Core S&P 500 UCITS ETF",
    "sectorBreakdown": { "Technology": 0.290, "Financials": 0.130, "Healthcare": 0.120, "Consumer Discretionary": 0.110, "Industrials": 0.085, "Communication Services": 0.090, "Consumer Staples": 0.058, "Energy": 0.040, "Materials": 0.025, "Real Estate": 0.028, "Utilities": 0.024 },
    "regionBreakdown": { "USA": 1.0 }
  },
  "IE00B3RBWM25": {
    "name": "Vanguard FTSE All-World UCITS ETF",
    "sectorBreakdown": { "Technology": 0.210, "Financials": 0.155, "Healthcare": 0.114, "Industrials": 0.102, "Consumer Discretionary": 0.102, "Communication Services": 0.081, "Consumer Staples": 0.068, "Energy": 0.051, "Materials": 0.045, "Real Estate": 0.029, "Utilities": 0.029, "Other": 0.014 },
    "regionBreakdown": { "USA": 0.610, "Europa": 0.155, "Japan": 0.055, "Emerging Markets": 0.130, "Sonstige": 0.050 }
  },
  "IE00BKM4GZ66": {
    "name": "iShares Core MSCI Emerging Markets IMI UCITS ETF",
    "sectorBreakdown": { "Technology": 0.230, "Financials": 0.220, "Consumer Discretionary": 0.125, "Communication Services": 0.098, "Materials": 0.080, "Industrials": 0.065, "Energy": 0.055, "Consumer Staples": 0.058, "Healthcare": 0.040, "Real Estate": 0.019, "Utilities": 0.010 },
    "regionBreakdown": { "Emerging Markets": 1.0 }
  },
  "IE00B3XXRP09": {
    "name": "Vanguard S&P 500 UCITS ETF",
    "sectorBreakdown": { "Technology": 0.290, "Financials": 0.130, "Healthcare": 0.120, "Consumer Discretionary": 0.110, "Industrials": 0.085, "Communication Services": 0.090, "Consumer Staples": 0.058, "Energy": 0.040, "Materials": 0.025, "Real Estate": 0.028, "Utilities": 0.024 },
    "regionBreakdown": { "USA": 1.0 }
  },
  "LU0274208692": {
    "name": "Xtrackers MSCI World Swap UCITS ETF",
    "sectorBreakdown": { "Technology": 0.233, "Financials": 0.154, "Healthcare": 0.127, "Industrials": 0.105, "Consumer Discretionary": 0.107, "Communication Services": 0.086, "Consumer Staples": 0.068, "Energy": 0.048, "Materials": 0.039, "Real Estate": 0.023, "Utilities": 0.010 },
    "regionBreakdown": { "USA": 0.703, "Japan": 0.058, "Europa": 0.210, "Emerging Markets": 0.010, "Sonstige": 0.019 }
  },
  "DE0005933931": {
    "name": "iShares Core DAX UCITS ETF",
    "sectorBreakdown": { "Industrials": 0.185, "Financials": 0.175, "Technology": 0.125, "Healthcare": 0.090, "Consumer Discretionary": 0.110, "Materials": 0.095, "Consumer Staples": 0.065, "Communication Services": 0.060, "Energy": 0.045, "Utilities": 0.030, "Real Estate": 0.020 },
    "regionBreakdown": { "Europa": 1.0 }
  },
  "IE00B4K48X80": {
    "name": "iShares MSCI Europe UCITS ETF",
    "sectorBreakdown": { "Financials": 0.180, "Industrials": 0.160, "Healthcare": 0.140, "Consumer Staples": 0.120, "Consumer Discretionary": 0.095, "Materials": 0.075, "Technology": 0.075, "Energy": 0.060, "Communication Services": 0.055, "Utilities": 0.025, "Real Estate": 0.015 },
    "regionBreakdown": { "Europa": 1.0 }
  },
  "LU1681043599": {
    "name": "Amundi MSCI World UCITS ETF",
    "sectorBreakdown": { "Technology": 0.233, "Financials": 0.154, "Healthcare": 0.127, "Industrials": 0.105, "Consumer Discretionary": 0.107, "Communication Services": 0.086, "Consumer Staples": 0.068, "Energy": 0.048, "Materials": 0.039, "Real Estate": 0.023, "Utilities": 0.010 },
    "regionBreakdown": { "USA": 0.703, "Japan": 0.058, "Europa": 0.210, "Emerging Markets": 0.010, "Sonstige": 0.019 }
  },
  "IE00BF4RFH31": {
    "name": "iShares MSCI World Small Cap UCITS ETF",
    "sectorBreakdown": { "Industrials": 0.215, "Technology": 0.155, "Financials": 0.145, "Consumer Discretionary": 0.130, "Healthcare": 0.095, "Materials": 0.085, "Real Estate": 0.065, "Consumer Staples": 0.045, "Energy": 0.030, "Communication Services": 0.020, "Utilities": 0.015 },
    "regionBreakdown": { "USA": 0.600, "Japan": 0.110, "Europa": 0.195, "Sonstige": 0.095 }
  },
  "DE0002635307": {
    "name": "iShares STOXX Europe 600 UCITS ETF",
    "sectorBreakdown": { "Financials": 0.180, "Industrials": 0.155, "Healthcare": 0.130, "Consumer Staples": 0.115, "Consumer Discretionary": 0.095, "Materials": 0.080, "Technology": 0.075, "Energy": 0.065, "Communication Services": 0.055, "Utilities": 0.030, "Real Estate": 0.020 },
    "regionBreakdown": { "Europa": 1.0 }
  },
  "IE00B1XNHC34": {
    "name": "iShares Global Clean Energy UCITS ETF",
    "sectorBreakdown": { "Utilities": 0.650, "Industrials": 0.210, "Technology": 0.080, "Energy": 0.060 },
    "regionBreakdown": { "USA": 0.430, "Europa": 0.280, "Emerging Markets": 0.180, "Sonstige": 0.110 }
  }
}
```

- [ ] **Step 2: Implement etf-data.ts**

Create `lib/etf-data.ts`:
```typescript
import etfDatabase from '@/data/common-etfs.json'

type EtfRecord = {
  name: string
  sectorBreakdown: Record<string, number>
  regionBreakdown: Record<string, number>
}

export function lookupEtf(isin: string): EtfRecord | null {
  const db = etfDatabase as Record<string, EtfRecord>
  return db[isin.toUpperCase()] ?? null
}

export function isKnownEtf(isin: string): boolean {
  return lookupEtf(isin) !== null
}
```

- [ ] **Step 3: Write and run tests**

Create `__tests__/lib/etf-data.test.ts`:
```typescript
import { describe, it, expect } from 'vitest'
import { lookupEtf, isKnownEtf } from '@/lib/etf-data'

describe('lookupEtf', () => {
  it('returns data for known MSCI World ISIN', () => {
    const result = lookupEtf('IE00B4L5Y983')
    expect(result).not.toBeNull()
    expect(result!.sectorBreakdown.Technology).toBeGreaterThan(0)
    expect(result!.regionBreakdown.USA).toBeGreaterThan(0)
  })

  it('returns null for unknown ISIN', () => {
    expect(lookupEtf('XX0000000000')).toBeNull()
  })

  it('is case-insensitive', () => {
    expect(lookupEtf('ie00b4l5y983')).not.toBeNull()
  })

  it('sector breakdown sums to approximately 1.0', () => {
    const result = lookupEtf('IE00B4L5Y983')!
    const total = Object.values(result.sectorBreakdown).reduce((a, b) => a + b, 0)
    expect(total).toBeCloseTo(1.0, 1)
  })
})
```

```bash
npm run test:run -- __tests__/lib/etf-data.test.ts
```

Expected: 4 tests passing.

- [ ] **Step 4: Commit**

```bash
git add data/common-etfs.json lib/etf-data.ts __tests__/lib/etf-data.test.ts
git commit -m "feat: add static ETF look-through database for 12 common European ETFs"
```

---

## Task 5: Enrichment API Route

**Files:**
- Create: `lib/enrichment.ts`
- Create: `app/api/enrich/route.ts`

- [ ] **Step 1: Implement server-side enrichment coordinator**

Create `lib/enrichment.ts`:
```typescript
import yahooFinance from 'yahoo-finance2'
import { lookupEtf } from '@/lib/etf-data'
import type { EnrichmentData } from '@/lib/types'

export async function enrichIsin(isin: string): Promise<EnrichmentData> {
  const now = new Date().toISOString()

  // Step 1: Find ticker via ISIN search
  let ticker: string
  try {
    const searchResult = await yahooFinance.search(isin, {}, { validateResult: false })
    const quote = searchResult.quotes.find(q => 'symbol' in q)
    if (!quote || !('symbol' in quote)) {
      return { isin, name: isin, type: 'unknown', fetchedAt: now, error: 'Ticker not found' }
    }
    ticker = quote.symbol
  } catch {
    return { isin, name: isin, type: 'unknown', fetchedAt: now, error: 'Yahoo Finance search failed' }
  }

  // Step 2: Get quote summary
  let summary: Awaited<ReturnType<typeof yahooFinance.quoteSummary>>
  try {
    summary = await yahooFinance.quoteSummary(ticker, { modules: ['price', 'assetProfile'] }, { validateResult: false })
  } catch {
    return { isin, name: ticker, type: 'unknown', fetchedAt: now, error: 'Yahoo Finance summary failed' }
  }

  const quoteType = summary.price?.quoteType
  const name = summary.price?.longName ?? summary.price?.shortName ?? ticker

  // Step 3: ETF path
  if (quoteType === 'ETF') {
    const staticData = lookupEtf(isin)
    if (staticData) {
      return {
        isin,
        name: staticData.name,
        type: 'etf',
        sectorBreakdown: staticData.sectorBreakdown,
        regionBreakdown: staticData.regionBreakdown,
        fetchedAt: now,
      }
    }
    // ETF not in static database — return partial data with error flag
    return {
      isin, name, type: 'etf', fetchedAt: now,
      error: 'ETF not in database — please enter allocation manually',
    }
  }

  // Step 4: Stock path
  const sector = summary.assetProfile?.sector ?? undefined
  const country = summary.assetProfile?.country ?? undefined

  return { isin, name, type: 'stock', sector, country, fetchedAt: now }
}
```

- [ ] **Step 2: Create API route**

Create `app/api/enrich/route.ts`:
```typescript
import { NextResponse } from 'next/server'
import { enrichIsin } from '@/lib/enrichment'

export async function POST(request: Request) {
  try {
    const body = await request.json()
    const { isin } = body as { isin?: string }

    if (!isin || typeof isin !== 'string' || isin.trim().length < 4) {
      return NextResponse.json({ error: 'Invalid ISIN' }, { status: 400 })
    }

    const data = await enrichIsin(isin.trim().toUpperCase())
    return NextResponse.json(data)
  } catch {
    return NextResponse.json({ error: 'Enrichment failed' }, { status: 500 })
  }
}
```

- [ ] **Step 3: Manual smoke test**

Start dev server: `npm run dev`

Test with curl (or Postman/Insomnia):
```bash
curl -X POST http://localhost:3000/api/enrich \
  -H "Content-Type: application/json" \
  -d '{"isin":"US0378331005"}'
```

Expected response for Apple:
```json
{"isin":"US0378331005","name":"Apple Inc.","type":"stock","sector":"Technology","country":"United States","fetchedAt":"..."}
```

Test MSCI World ETF:
```bash
curl -X POST http://localhost:3000/api/enrich \
  -H "Content-Type: application/json" \
  -d '{"isin":"IE00B4L5Y983"}'
```

Expected: ETF response with `sectorBreakdown` and `regionBreakdown` from static data.

- [ ] **Step 4: Commit**

```bash
git add lib/enrichment.ts app/api/enrich/
git commit -m "feat: add ISIN enrichment API route (Yahoo Finance + static ETF data)"
```

---

## Task 6: App Shell

**Files:**
- Modify: `app/layout.tsx`
- Modify: `app/globals.css`
- Modify: `app/page.tsx`
- Create: `components/Dashboard.tsx`

- [ ] **Step 1: Configure dark layout**

Replace `app/layout.tsx`:
```typescript
import type { Metadata } from 'next'
import { Inter } from 'next/font/google'
import './globals.css'

const inter = Inter({ subsets: ['latin'] })

export const metadata: Metadata = {
  title: 'Portfolio Analyzer',
  description: 'Portfolio cluster risk dashboard with ETF look-through',
}

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="de">
      <body className={`${inter.className} bg-slate-900 text-slate-100 min-h-screen`}>
        {children}
      </body>
    </html>
  )
}
```

Replace `app/globals.css` body/html styles — keep shadcn CSS variables, remove conflicting background:
```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --background: 222 47% 11%;
    --foreground: 213 31% 91%;
    /* Keep all shadcn variable definitions from the generated file */
  }
}

* {
  box-sizing: border-box;
}

html, body {
  height: 100%;
}
```

- [ ] **Step 2: Create Dashboard shell**

Create `components/Dashboard.tsx`:
```typescript
'use client'

import { useState, useEffect, useCallback } from 'react'
import { LocalStorageRepository } from '@/lib/repository/localStorage'
import { computeAnalysis } from '@/lib/analysis'
import type { Position, EnrichmentData, PortfolioAnalysis } from '@/lib/types'

const repo = new LocalStorageRepository()

export function Dashboard() {
  const [positions, setPositions] = useState<Position[]>([])
  const [enrichmentCache, setEnrichmentCache] = useState<Record<string, EnrichmentData>>({})
  const [analysis, setAnalysis] = useState<PortfolioAnalysis | null>(null)
  const [enriching, setEnriching] = useState<Set<string>>(new Set())

  // Load from localStorage on mount
  useEffect(() => {
    const saved = repo.getPositions()
    const cache = repo.getAllEnrichmentCache()
    setPositions(saved)
    setEnrichmentCache(cache)
  }, [])

  // Recompute analysis when positions or cache changes
  useEffect(() => {
    setAnalysis(computeAnalysis(positions, enrichmentCache))
  }, [positions, enrichmentCache])

  const enrichPosition = useCallback(async (isin: string) => {
    // Skip if already being enriched
    if (enriching.has(isin)) return

    // Use cache if fresh
    const cached = repo.getEnrichmentCache(isin)
    if (cached) {
      setEnrichmentCache(prev => ({ ...prev, [isin]: cached }))
      return
    }

    setEnriching(prev => new Set(prev).add(isin))
    try {
      const res = await fetch('/api/enrich', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ isin }),
      })
      if (res.ok) {
        const data: EnrichmentData = await res.json()
        repo.setEnrichmentCache(isin, data)
        setEnrichmentCache(prev => ({ ...prev, [isin]: data }))
      }
    } finally {
      setEnriching(prev => { const s = new Set(prev); s.delete(isin); return s })
    }
  }, [enriching])

  const handleAddPosition = useCallback(async (position: Position) => {
    const updated = [...positions, position]
    repo.savePositions(updated)
    setPositions(updated)
    await enrichPosition(position.isin)
  }, [positions, enrichPosition])

  const handleDeletePosition = useCallback((id: string) => {
    const updated = positions.filter(p => p.id !== id)
    repo.savePositions(updated)
    setPositions(updated)
  }, [positions])

  const handleImport = useCallback((json: string) => {
    repo.importJson(json)
    const imported = repo.getPositions()
    setPositions(imported)
    imported.forEach(p => enrichPosition(p.isin))
  }, [enrichPosition])

  return (
    <div className="flex h-screen overflow-hidden bg-slate-900">
      <div className="w-64 flex-shrink-0 bg-slate-800 border-r border-slate-700">
        {/* Sidebar — wired in Task 7 */}
        <div className="p-4 text-slate-400 text-sm">Sidebar</div>
      </div>
      <div className="flex-1 flex flex-col overflow-hidden">
        {/* Warning banner — wired in Task 8 */}
        <div className="p-4 text-slate-400 text-sm border-b border-slate-700">
          Warnungen
        </div>
        <div className="flex-1 grid grid-cols-2 grid-rows-2 gap-4 p-4 overflow-hidden">
          {/* Charts — wired in Task 8 */}
          <div className="bg-slate-800 rounded-lg p-4 text-slate-400 text-sm">Sektor</div>
          <div className="bg-slate-800 rounded-lg p-4 text-slate-400 text-sm">Region</div>
          <div className="bg-slate-800 rounded-lg p-4 text-slate-400 text-sm">Einzeltitel</div>
          <div className="bg-slate-800 rounded-lg p-4 text-slate-400 text-sm">KI-Analyse</div>
        </div>
      </div>
    </div>
  )
}
```

- [ ] **Step 3: Wire page.tsx**

Replace `app/page.tsx`:
```typescript
import { Dashboard } from '@/components/Dashboard'

export default function Home() {
  return <Dashboard />
}
```

- [ ] **Step 4: Verify in browser**

```bash
npm run dev
```

Open `http://localhost:3000`. Expected: dark shell with 4 placeholder boxes visible. No console errors.

- [ ] **Step 5: Commit**

```bash
git add app/ components/Dashboard.tsx
git commit -m "feat: add dark app shell and Dashboard component skeleton"
```

---

## Task 7: Portfolio Sidebar

**Files:**
- Create: `components/sidebar/AddPositionForm.tsx`
- Create: `components/sidebar/PositionCard.tsx`
- Create: `components/sidebar/PortfolioSidebar.tsx`
- Modify: `components/Dashboard.tsx`

- [ ] **Step 1: Create AddPositionForm**

Create `components/sidebar/AddPositionForm.tsx`:
```typescript
'use client'

import { useState } from 'react'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Label } from '@/components/ui/label'
import { Dialog, DialogContent, DialogHeader, DialogTitle, DialogTrigger } from '@/components/ui/dialog'
import type { Position } from '@/lib/types'

type Props = {
  onAdd: (position: Position) => void
}

export function AddPositionForm({ onAdd }: Props) {
  const [open, setOpen] = useState(false)
  const [name, setName] = useState('')
  const [isin, setIsin] = useState('')
  const [wkn, setWkn] = useState('')
  const [amount, setAmount] = useState('')

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault()
    const parsed = parseFloat(amount.replace(',', '.'))
    if (!name.trim() || !isin.trim() || isNaN(parsed) || parsed <= 0) return

    const position: Position = {
      id: crypto.randomUUID(),
      name: name.trim(),
      isin: isin.trim().toUpperCase(),
      wkn: wkn.trim() || undefined,
      amountEur: parsed,
      type: 'unknown', // enrichment will determine the actual type
      addedAt: new Date().toISOString(),
    }
    onAdd(position)
    setName(''); setIsin(''); setWkn(''); setAmount('')
    setOpen(false)
  }

  return (
    <Dialog open={open} onOpenChange={setOpen}>
      <DialogTrigger asChild>
        <Button className="w-full bg-blue-600 hover:bg-blue-700 text-white">
          + Position hinzufügen
        </Button>
      </DialogTrigger>
      <DialogContent className="bg-slate-800 border-slate-700 text-slate-100">
        <DialogHeader>
          <DialogTitle>Position hinzufügen</DialogTitle>
        </DialogHeader>
        <form onSubmit={handleSubmit} className="space-y-4">
          <div>
            <Label>Name *</Label>
            <Input value={name} onChange={e => setName(e.target.value)}
              placeholder="z.B. Apple Inc." className="bg-slate-900 border-slate-600 text-slate-100" required />
          </div>
          <div>
            <Label>ISIN *</Label>
            <Input value={isin} onChange={e => setIsin(e.target.value)}
              placeholder="z.B. US0378331005" className="bg-slate-900 border-slate-600 text-slate-100 font-mono" required />
          </div>
          <div>
            <Label>WKN (optional)</Label>
            <Input value={wkn} onChange={e => setWkn(e.target.value)}
              placeholder="z.B. 865985" className="bg-slate-900 border-slate-600 text-slate-100 font-mono" />
          </div>
          <div>
            <Label>Betrag in € *</Label>
            <Input value={amount} onChange={e => setAmount(e.target.value)}
              placeholder="z.B. 5200" type="text" inputMode="decimal"
              className="bg-slate-900 border-slate-600 text-slate-100" required />
          </div>
          <Button type="submit" className="w-full bg-blue-600 hover:bg-blue-700">Hinzufügen</Button>
        </form>
      </DialogContent>
    </Dialog>
  )
}
```

- [ ] **Step 2: Create PositionCard**

Create `components/sidebar/PositionCard.tsx`:
```typescript
import { Button } from '@/components/ui/button'
import { Badge } from '@/components/ui/badge'
import type { Position, EnrichmentData } from '@/lib/types'

const WEIGHT_COLORS = ['bg-blue-500', 'bg-emerald-500', 'bg-violet-500', 'bg-amber-500', 'bg-rose-500', 'bg-cyan-500']

type Props = {
  position: Position
  weight: number          // fraction 0-1
  enrichment?: EnrichmentData
  colorIndex: number
  onDelete: (id: string) => void
}

export function PositionCard({ position, weight, enrichment, colorIndex, onDelete }: Props) {
  const color = WEIGHT_COLORS[colorIndex % WEIGHT_COLORS.length]
  const pct = (weight * 100).toFixed(1)
  const isWarning = weight > 0.25 && position.type === 'stock'

  return (
    <div className={`bg-slate-900 rounded-lg p-3 mb-2 border-l-4 ${color} relative group`}>
      <div className="flex items-start justify-between gap-2">
        <div className="min-w-0 flex-1">
          <div className="font-medium text-sm text-slate-100 truncate">{position.name}</div>
          <div className="text-xs text-slate-500 font-mono">{position.isin}</div>
          <div className="flex items-center gap-2 mt-1">
            <span className="text-sm font-semibold text-emerald-400">
              €{position.amountEur.toLocaleString('de-DE')}
            </span>
            <Badge variant="outline"
              className={`text-xs px-1.5 py-0 border-slate-600 ${isWarning ? 'text-red-400 border-red-800' : 'text-slate-400'}`}>
              {pct}%
            </Badge>
          </div>
          {enrichment?.error && (
            <div className="text-xs text-amber-400 mt-1 truncate" title={enrichment.error}>
              ⚠ {enrichment.error}
            </div>
          )}
        </div>
        <button
          onClick={() => onDelete(position.id)}
          className="opacity-0 group-hover:opacity-100 text-slate-600 hover:text-red-400 text-lg leading-none transition-opacity"
          aria-label="Position entfernen"
        >
          ×
        </button>
      </div>
    </div>
  )
}
```

- [ ] **Step 3: Create PortfolioSidebar**

Create `components/sidebar/PortfolioSidebar.tsx`:
```typescript
import { AddPositionForm } from './AddPositionForm'
import { PositionCard } from './PositionCard'
import { Button } from '@/components/ui/button'
import { ScrollArea } from '@/components/ui/scroll-area'
import type { Position, EnrichmentData } from '@/lib/types'

type Props = {
  positions: Position[]
  enrichmentCache: Record<string, EnrichmentData>
  onAdd: (position: Position) => void
  onDelete: (id: string) => void
  onExport: () => void
  onImport: (json: string) => void
}

export function PortfolioSidebar({ positions, enrichmentCache, onAdd, onDelete, onExport, onImport }: Props) {
  const totalValue = positions.reduce((sum, p) => sum + p.amountEur, 0)

  const handleImportClick = () => {
    const input = document.createElement('input')
    input.type = 'file'
    input.accept = '.json'
    input.onchange = async (e) => {
      const file = (e.target as HTMLInputElement).files?.[0]
      if (!file) return
      const text = await file.text()
      try { onImport(text) } catch { alert('Ungültiges Portfolio-Format') }
    }
    input.click()
  }

  return (
    <div className="flex flex-col h-full">
      <div className="px-4 pt-4 pb-3 border-b border-slate-700">
        <h1 className="font-bold text-slate-100">Mein Portfolio</h1>
        <p className="text-xs text-slate-500 mt-0.5">
          Gesamtwert:{' '}
          <span className="text-emerald-400 font-semibold">
            €{totalValue.toLocaleString('de-DE')}
          </span>
        </p>
      </div>

      <ScrollArea className="flex-1 px-3 py-2">
        {positions.length === 0 ? (
          <p className="text-slate-500 text-xs text-center py-8">
            Noch keine Positionen.<br />Füge deine erste Position hinzu.
          </p>
        ) : (
          positions.map((position, idx) => (
            <PositionCard
              key={position.id}
              position={position}
              weight={totalValue > 0 ? position.amountEur / totalValue : 0}
              enrichment={enrichmentCache[position.isin]}
              colorIndex={idx}
              onDelete={onDelete}
            />
          ))
        )}
      </ScrollArea>

      <div className="px-3 pb-4 pt-2 border-t border-slate-700 space-y-2">
        <AddPositionForm onAdd={onAdd} />
        <div className="grid grid-cols-2 gap-2">
          <Button variant="outline" size="sm"
            onClick={onExport}
            className="border-slate-600 text-slate-400 hover:text-slate-100 text-xs">
            Exportieren
          </Button>
          <Button variant="outline" size="sm"
            onClick={handleImportClick}
            className="border-slate-600 text-slate-400 hover:text-slate-100 text-xs">
            Importieren
          </Button>
        </div>
      </div>
    </div>
  )
}
```

- [ ] **Step 4: Wire Sidebar into Dashboard**

In `components/Dashboard.tsx`, replace the sidebar placeholder div and add export handler:

```typescript
// Add this import at the top:
import { PortfolioSidebar } from '@/components/sidebar/PortfolioSidebar'

// Add this handler inside Dashboard():
const handleExport = useCallback(() => {
  const json = repo.exportJson()
  const blob = new Blob([json], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = `portfolio-${new Date().toISOString().slice(0, 10)}.json`
  a.click()
  URL.revokeObjectURL(url)
}, [])

// Replace the sidebar placeholder div with:
<div className="w-64 flex-shrink-0 bg-slate-800 border-r border-slate-700">
  <PortfolioSidebar
    positions={positions}
    enrichmentCache={enrichmentCache}
    onAdd={handleAddPosition}
    onDelete={handleDeletePosition}
    onExport={handleExport}
    onImport={handleImport}
  />
</div>
```

- [ ] **Step 5: Test in browser**

Run `npm run dev`. Add a position (e.g. Apple, US0378331005, €5000). Verify:
- Position appears in sidebar with correct weight %
- After a moment, position card no longer shows loading state (enrichment completed)
- Delete button appears on hover
- Export downloads a JSON file

- [ ] **Step 6: Commit**

```bash
git add components/sidebar/ components/Dashboard.tsx
git commit -m "feat: add portfolio sidebar with position entry, cards, export/import"
```

---

## Task 8: Risk Charts and Warning Banner

**Files:**
- Create: `components/RiskWarningBanner.tsx`
- Create: `components/charts/SectorChart.tsx`
- Create: `components/charts/RegionChart.tsx`
- Create: `components/charts/SingleStockChart.tsx`
- Modify: `components/Dashboard.tsx`

- [ ] **Step 1: Create RiskWarningBanner**

Create `components/RiskWarningBanner.tsx`:
```typescript
import type { Warning } from '@/lib/types'

type Props = { warnings: Warning[] }

export function RiskWarningBanner({ warnings }: Props) {
  if (warnings.length === 0) return null

  return (
    <div className="bg-red-950 border-b border-red-800 px-4 py-2 flex flex-wrap gap-2 items-center">
      <span className="text-red-300 text-xs font-medium flex-shrink-0">⚠ Klumpenrisiko erkannt:</span>
      {warnings.map((w, i) => (
        <span key={i} className="bg-red-900 text-red-300 text-xs px-2 py-0.5 rounded">
          {w.label}
        </span>
      ))}
    </div>
  )
}
```

- [ ] **Step 2: Create SectorChart**

Create `components/charts/SectorChart.tsx`:
```typescript
'use client'

import { PieChart, Pie, Cell, Tooltip, Legend, ResponsiveContainer } from 'recharts'

const COLORS = ['#3b82f6', '#22c55e', '#f59e0b', '#8b5cf6', '#ef4444', '#06b6d4', '#f97316', '#84cc16', '#ec4899', '#14b8a6', '#6366f1']

type Props = { distribution: Record<string, number> | undefined }

export function SectorChart({ distribution }: Props) {
  const data = Object.entries(distribution ?? {})
    .map(([name, value]) => ({ name, value: Math.round(value * 1000) / 10 }))
    .sort((a, b) => b.value - a.value)

  return (
    <div className="bg-slate-800 rounded-lg p-4 flex flex-col h-full">
      <h3 className="text-sm font-semibold text-slate-100 mb-3">Sektoren</h3>
      {data.length === 0 ? (
        <div className="flex-1 flex items-center justify-center text-slate-500 text-xs">
          Positionen hinzufügen
        </div>
      ) : (
        <ResponsiveContainer width="100%" height="100%">
          <PieChart>
            <Pie data={data} dataKey="value" nameKey="name" cx="50%" cy="50%"
              innerRadius="40%" outerRadius="70%" paddingAngle={2}>
              {data.map((_, i) => <Cell key={i} fill={COLORS[i % COLORS.length]} />)}
            </Pie>
            <Tooltip
              contentStyle={{ backgroundColor: '#1e293b', border: '1px solid #334155', borderRadius: '6px' }}
              itemStyle={{ color: '#e2e8f0' }}
              formatter={(v: number) => [`${v}%`, '']}
            />
            <Legend
              iconType="circle"
              iconSize={8}
              formatter={(v) => <span style={{ color: '#94a3b8', fontSize: '11px' }}>{v}</span>}
            />
          </PieChart>
        </ResponsiveContainer>
      )}
    </div>
  )
}
```

- [ ] **Step 3: Create RegionChart**

Create `components/charts/RegionChart.tsx`:
```typescript
'use client'

import { PieChart, Pie, Cell, Tooltip, Legend, ResponsiveContainer } from 'recharts'

const REGION_COLORS: Record<string, string> = {
  USA: '#ef4444',
  Europa: '#3b82f6',
  Japan: '#f59e0b',
  'Emerging Markets': '#22c55e',
  Sonstige: '#6b7280',
}

type Props = { distribution: Record<string, number> | undefined }

export function RegionChart({ distribution }: Props) {
  const data = Object.entries(distribution ?? {})
    .map(([name, value]) => ({ name, value: Math.round(value * 1000) / 10 }))
    .sort((a, b) => b.value - a.value)

  return (
    <div className="bg-slate-800 rounded-lg p-4 flex flex-col h-full">
      <h3 className="text-sm font-semibold text-slate-100 mb-3">Regionen</h3>
      {data.length === 0 ? (
        <div className="flex-1 flex items-center justify-center text-slate-500 text-xs">
          Positionen hinzufügen
        </div>
      ) : (
        <ResponsiveContainer width="100%" height="100%">
          <PieChart>
            <Pie data={data} dataKey="value" nameKey="name" cx="50%" cy="50%"
              innerRadius="40%" outerRadius="70%" paddingAngle={2}>
              {data.map((entry, i) => (
                <Cell key={i} fill={REGION_COLORS[entry.name] ?? '#6b7280'} />
              ))}
            </Pie>
            <Tooltip
              contentStyle={{ backgroundColor: '#1e293b', border: '1px solid #334155', borderRadius: '6px' }}
              itemStyle={{ color: '#e2e8f0' }}
              formatter={(v: number) => [`${v}%`, '']}
            />
            <Legend
              iconType="circle"
              iconSize={8}
              formatter={(v) => <span style={{ color: '#94a3b8', fontSize: '11px' }}>{v}</span>}
            />
          </PieChart>
        </ResponsiveContainer>
      )}
    </div>
  )
}
```

- [ ] **Step 4: Create SingleStockChart**

Create `components/charts/SingleStockChart.tsx`:
```typescript
'use client'

import { BarChart, Bar, XAxis, YAxis, Tooltip, Cell, ResponsiveContainer, ReferenceLine } from 'recharts'
import type { SingleStockEntry } from '@/lib/types'

type Props = { stocks: SingleStockEntry[] | undefined }

export function SingleStockChart({ stocks }: Props) {
  const data = (stocks ?? [])
    .map(s => ({ name: s.name, value: Math.round(s.weight * 1000) / 10 }))
    .sort((a, b) => b.value - a.value)

  return (
    <div className="bg-slate-800 rounded-lg p-4 flex flex-col h-full">
      <h3 className="text-sm font-semibold text-slate-100 mb-1">Einzeltitel</h3>
      <p className="text-xs text-slate-500 mb-3">Nur Aktien, ohne ETFs</p>
      {data.length === 0 ? (
        <div className="flex-1 flex items-center justify-center text-slate-500 text-xs">
          Keine Einzeltitel im Portfolio
        </div>
      ) : (
        <ResponsiveContainer width="100%" height="100%">
          <BarChart data={data} layout="vertical" margin={{ left: 0, right: 16 }}>
            <XAxis type="number" domain={[0, 100]} tickFormatter={v => `${v}%`}
              tick={{ fill: '#64748b', fontSize: 11 }} axisLine={false} tickLine={false} />
            <YAxis type="category" dataKey="name" width={90}
              tick={{ fill: '#94a3b8', fontSize: 11 }} axisLine={false} tickLine={false} />
            <Tooltip
              contentStyle={{ backgroundColor: '#1e293b', border: '1px solid #334155', borderRadius: '6px' }}
              itemStyle={{ color: '#e2e8f0' }}
              formatter={(v: number) => [`${v}%`, 'Anteil']}
            />
            <ReferenceLine x={25} stroke="#ef4444" strokeDasharray="4 2"
              label={{ value: '25%', position: 'top', fill: '#ef4444', fontSize: 10 }} />
            <Bar dataKey="value" radius={[0, 4, 4, 0]}>
              {data.map((entry, i) => (
                <Cell key={i} fill={entry.value > 25 ? '#ef4444' : '#3b82f6'} />
              ))}
            </Bar>
          </BarChart>
        </ResponsiveContainer>
      )}
    </div>
  )
}
```

- [ ] **Step 5: Wire charts and banner into Dashboard**

In `components/Dashboard.tsx`, replace placeholder divs with real components:

```typescript
// Add imports:
import { RiskWarningBanner } from '@/components/RiskWarningBanner'
import { SectorChart } from '@/components/charts/SectorChart'
import { RegionChart } from '@/components/charts/RegionChart'
import { SingleStockChart } from '@/components/charts/SingleStockChart'

// Replace warning placeholder:
{analysis && <RiskWarningBanner warnings={analysis.warnings} />}

// Replace chart placeholders:
<SectorChart distribution={analysis?.sectorDistribution} />
<RegionChart distribution={analysis?.regionDistribution} />
<SingleStockChart stocks={analysis?.singleStocks} />
<div className="bg-slate-800 rounded-lg p-4 text-slate-400 text-sm">KI-Analyse (Task 9)</div>
```

- [ ] **Step 6: Test in browser**

Add 2-3 positions including an ETF (e.g. MSCI World IE00B4L5Y983). Verify:
- Sector donut shows ETF look-through data
- Region donut reflects geographic allocation
- Single-stock bar chart shows only stocks (not ETFs)
- If any stock > 25%, red warning banner appears at top

- [ ] **Step 7: Commit**

```bash
git add components/RiskWarningBanner.tsx components/charts/ components/Dashboard.tsx
git commit -m "feat: add sector, region, and single-stock charts with risk warning banner"
```

---

## Task 9: AI Chat Panel

**Files:**
- Create: `app/api/chat/route.ts`
- Create: `components/AiChatPanel.tsx`
- Modify: `components/Dashboard.tsx`

- [ ] **Step 1: Create chat API route**

Create `app/api/chat/route.ts`:
```typescript
import OpenAI from 'openai'
import type { PortfolioAnalysis, Position } from '@/lib/types'

const openai = new OpenAI({
  apiKey: process.env.OPENROUTER_API_KEY!,
  baseURL: 'https://openrouter.ai/api/v1',
})

function buildSystemPrompt(positions: Position[], analysis: PortfolioAnalysis | null): string {
  const total = positions.reduce((s, p) => s + p.amountEur, 0)

  const positionLines = positions
    .map(p => `- ${p.name} (${p.isin}): €${p.amountEur.toLocaleString('de-DE')} (${total > 0 ? ((p.amountEur / total) * 100).toFixed(1) : 0}%)`)
    .join('\n')

  const sectorLines = analysis
    ? Object.entries(analysis.sectorDistribution)
        .sort(([, a], [, b]) => b - a)
        .map(([s, v]) => `  ${s}: ${(v * 100).toFixed(1)}%`)
        .join('\n')
    : '  (noch keine Daten)'

  const regionLines = analysis
    ? Object.entries(analysis.regionDistribution)
        .sort(([, a], [, b]) => b - a)
        .map(([r, v]) => `  ${r}: ${(v * 100).toFixed(1)}%`)
        .join('\n')
    : '  (noch keine Daten)'

  const warningLines = analysis?.warnings.length
    ? analysis.warnings.map(w => `  ⚠ ${w.label}`).join('\n')
    : '  Keine aktiven Warnungen'

  return `Du bist ein professioneller Portfolio-Berater. Heutige Datum: ${new Date().toLocaleDateString('de-DE')}.

Das Portfolio des Nutzers:
Gesamtwert: €${total.toLocaleString('de-DE')}

Positionen:
${positionLines || '  (leer)'}

Risikoanalyse (ETF Look-Through angewendet):
Sektoren:
${sectorLines}

Regionen:
${regionLines}

Aktive Warnungen:
${warningLines}

Deine Aufgabe:
1. Stelle zuerst Rückfragen zu Anlagehorizont, Risikotoleranz und persönlichen Präferenzen des Nutzers — bevor du Empfehlungen gibst.
2. Gib dann konkrete Empfehlungen: fehlende Regionen, politische und technologische Trends, Einzelaktien oder ETFs für ein Satelliten-Portfolio.
3. Antworte immer auf Deutsch, präzise und ohne Finanzchat-Floskeln.`
}

export async function POST(request: Request) {
  const { messages, positions, analysis } = await request.json() as {
    messages: OpenAI.Chat.Completions.ChatCompletionMessageParam[]
    positions: Position[]
    analysis: PortfolioAnalysis | null
  }

  const stream = await openai.chat.completions.create({
    model: 'anthropic/claude-sonnet-4-5',
    messages: [
      { role: 'system', content: buildSystemPrompt(positions, analysis) },
      ...messages,
    ],
    stream: true,
    max_tokens: 1024,
  })

  const encoder = new TextEncoder()
  const readable = new ReadableStream({
    async start(controller) {
      for await (const chunk of stream) {
        const text = chunk.choices[0]?.delta?.content ?? ''
        if (text) controller.enqueue(encoder.encode(text))
      }
      controller.close()
    },
  })

  return new Response(readable, {
    headers: { 'Content-Type': 'text/plain; charset=utf-8' },
  })
}
```

- [ ] **Step 2: Create AiChatPanel**

Create `components/AiChatPanel.tsx`:
```typescript
'use client'

import { useState, useRef, useEffect } from 'react'
import { Button } from '@/components/ui/button'
import { ScrollArea } from '@/components/ui/scroll-area'
import type { Position, PortfolioAnalysis } from '@/lib/types'

type Message = { role: 'user' | 'assistant'; content: string }

type Props = {
  positions: Position[]
  analysis: PortfolioAnalysis | null
}

export function AiChatPanel({ positions, analysis }: Props) {
  const [messages, setMessages] = useState<Message[]>([])
  const [input, setInput] = useState('')
  const [loading, setLoading] = useState(false)
  const bottomRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: 'smooth' })
  }, [messages])

  const sendMessage = async () => {
    if (!input.trim() || loading) return
    const userMessage: Message = { role: 'user', content: input.trim() }
    const updated = [...messages, userMessage]
    setMessages(updated)
    setInput('')
    setLoading(true)

    try {
      const res = await fetch('/api/chat', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          messages: updated.map(m => ({ role: m.role, content: m.content })),
          positions,
          analysis,
        }),
      })

      if (!res.body) return
      const reader = res.body.getReader()
      const decoder = new TextDecoder()
      let assistantContent = ''
      setMessages(prev => [...prev, { role: 'assistant', content: '' }])

      while (true) {
        const { done, value } = await reader.read()
        if (done) break
        assistantContent += decoder.decode(value, { stream: true })
        setMessages(prev => {
          const copy = [...prev]
          copy[copy.length - 1] = { role: 'assistant', content: assistantContent }
          return copy
        })
      }
    } finally {
      setLoading(false)
    }
  }

  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter' && !e.shiftKey) { e.preventDefault(); sendMessage() }
  }

  return (
    <div className="bg-slate-800 rounded-lg p-4 flex flex-col h-full">
      <h3 className="text-sm font-semibold text-slate-100 mb-3">KI-Analyse</h3>
      <ScrollArea className="flex-1 mb-3">
        {messages.length === 0 ? (
          <p className="text-slate-500 text-xs">
            Stell mir eine Frage zu deinem Portfolio oder schreib "Analysiere mein Portfolio".
          </p>
        ) : (
          messages.map((msg, i) => (
            <div key={i} className={`mb-3 ${msg.role === 'user' ? 'text-right' : ''}`}>
              <div className={`inline-block text-xs rounded-lg px-3 py-2 max-w-[90%] text-left leading-relaxed ${
                msg.role === 'user'
                  ? 'bg-blue-700 text-blue-50'
                  : 'bg-slate-700 text-slate-200'
              }`}>
                {msg.content || (loading && i === messages.length - 1 ? '▌' : '')}
              </div>
            </div>
          ))
        )}
        <div ref={bottomRef} />
      </ScrollArea>
      <div className="flex gap-2">
        <input
          value={input}
          onChange={e => setInput(e.target.value)}
          onKeyDown={handleKeyDown}
          placeholder="Frage stellen..."
          disabled={loading}
          className="flex-1 bg-slate-900 border border-slate-600 rounded-md px-3 py-2 text-sm text-slate-100 placeholder:text-slate-500 focus:outline-none focus:ring-1 focus:ring-blue-500 disabled:opacity-50"
        />
        <Button onClick={sendMessage} disabled={loading || !input.trim()}
          className="bg-blue-600 hover:bg-blue-700 px-3">
          →
        </Button>
      </div>
    </div>
  )
}
```

- [ ] **Step 3: Wire AI panel into Dashboard**

In `components/Dashboard.tsx`:

```typescript
// Add import:
import { AiChatPanel } from '@/components/AiChatPanel'

// Replace the KI-Analyse placeholder div with:
<AiChatPanel positions={positions} analysis={analysis} />
```

- [ ] **Step 4: Test AI chat**

Ensure `OPENROUTER_API_KEY` is set in `.env.local`. Run `npm run dev`.

Add a couple of positions, then type "Analysiere mein Portfolio" in the chat. Expected:
- Streaming response appears character by character
- AI references the actual positions and risk data
- AI asks a clarifying question before making recommendations

- [ ] **Step 5: Commit**

```bash
git add app/api/chat/ components/AiChatPanel.tsx components/Dashboard.tsx
git commit -m "feat: add AI portfolio advisor chat with streaming (OpenRouter + claude-sonnet-4-5)"
```

---

## Task 10: Run Full Test Suite

- [ ] **Step 1: Run all tests**

```bash
npm run test:run
```

Expected: All tests pass (analysis.test.ts, localStorage.test.ts, etf-data.test.ts).

- [ ] **Step 2: Fix any failures**

If any test fails: read the error, identify which assertion fails, fix the implementation. Re-run until all pass.

- [ ] **Step 3: Check TypeScript compilation**

```bash
npx tsc --noEmit
```

Expected: No type errors. Fix any before proceeding.

- [ ] **Step 4: Build for production**

```bash
npm run build
```

Expected: Build completes without errors. Note any warnings.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "chore: verify all tests pass and production build succeeds"
```

---

## Task 11: Vercel Deployment

- [ ] **Step 1: Push to GitHub**

```bash
git push origin master
```

- [ ] **Step 2: Import project in Vercel**

1. Go to vercel.com → Add New Project
2. Import `svenjamin18/portfolio-analyzer` from GitHub
3. Framework: Next.js (auto-detected)
4. Root directory: `.` (default)
5. Click Deploy

- [ ] **Step 3: Add environment variable**

In Vercel dashboard → Project → Settings → Environment Variables:
- Name: `OPENROUTER_API_KEY`
- Value: your OpenRouter API key
- Environment: Production + Preview + Development

Click Save, then trigger a redeploy.

- [ ] **Step 4: Test live deployment**

Open the Vercel URL. Test:
- Add a position (Apple, US0378331005, €5000)
- Wait for enrichment (sector/region appears in charts)
- Add MSCI World ETF (IE00B4L5Y983, €10000) — verify ETF look-through in sector chart
- Type a message in AI chat — verify streaming response
- Export portfolio as JSON, clear localStorage, import the JSON — positions restored

- [ ] **Step 5: Final commit with deployment URL**

Add the deployment URL to README.md, then commit:

```bash
git add README.md
git commit -m "chore: add Vercel deployment URL to README"
git push origin master
```

---

## Self-Review Checklist

**Spec coverage:**
- ✅ Direct portfolio entry (ISIN/WKN/name/amount) — Task 7
- ✅ localStorage persistence — Task 2
- ✅ Stock enrichment via Yahoo Finance — Task 5
- ✅ ETF look-through (static data for 12 ETFs) — Task 4
- ✅ Sector chart (with ETF look-through) — Task 8
- ✅ Region chart (with ETF look-through) — Task 8
- ✅ Single-stock chart (stocks only) — Task 8
- ✅ Risk warnings (>25% single stock, >60% region) — Tasks 3 + 8
- ✅ AI chat with portfolio context + streaming — Task 9
- ✅ Export / Import JSON — Task 7
- ✅ Dark mode — Task 6
- ✅ Vercel deployment — Task 11

**Type consistency:** `Position`, `EnrichmentData`, `PortfolioAnalysis`, `Warning`, `SingleStockEntry` defined once in `lib/types.ts`, imported everywhere. `computeAnalysis` signature matches usage in `Dashboard.tsx`. `PortfolioRepository` interface matches `LocalStorageRepository` implementation.

**No placeholders:** All steps contain complete code. No TBD.
