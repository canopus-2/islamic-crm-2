# Market Research Features Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Close the gap between the current MVP and a sellable UK mosque CRM by adding GASDS, HMRC Gift Aid export, trustee reporting, cash batch logging, donation exports, QR codes, fund expenditure tracking, email receipts, dashboard enhancements, financial year support, and donor giving statements.

**Architecture:** 3 new Prisma models (`GiftAidClaim`, `GasdsClaim`, `Expenditure`) plus a `ClaimStatus` enum and `title` field on Contact. New library utilities for financial year, GASDS rules, HMRC CSV generation, trustee exports, donation exports, donor statements, and Resend email. Each feature follows the existing pattern: Zod validation → API route → server component page → client component.

**Tech Stack:** Next.js 16, Prisma 7, Zod 4, Vitest, Resend (email), qrcode (QR generation), Tailwind CSS 4 with emerald theme.

---

## Phase 0: Foundation

### Task 1: Schema Migration — New Models

**Files:**
- Modify: `prisma/schema.prisma`

**Step 1:** Add the following to `prisma/schema.prisma`:

```prisma
// After existing enums
enum ClaimStatus {
  DRAFT
  SUBMITTED
  ACCEPTED
  REJECTED
}

// After GiftAidDeclaration model
model GiftAidClaim {
  id            String      @id @default(cuid())
  tenantId      String
  tenant        Tenant      @relation(fields: [tenantId], references: [id])
  status        ClaimStatus @default(DRAFT)
  donationIds   String[]    // IDs of donations included in this claim
  totalAmount   Decimal     @db.Decimal(10, 2) // Total donation amount
  giftAidAmount Decimal     @db.Decimal(10, 2) // 25% reclaim
  periodStart   DateTime    // Earliest donation date in claim
  periodEnd     DateTime    // Latest donation date in claim
  hmrcRef       String?     // Reference from HMRC after submission
  notes         String?
  createdAt     DateTime    @default(now())
  updatedAt     DateTime    @updatedAt

  @@index([tenantId])
  @@index([tenantId, status])
}

model GasdsClaim {
  id            String      @id @default(cuid())
  tenantId      String
  tenant        Tenant      @relation(fields: [tenantId], references: [id])
  taxYear       Int         // UK tax year start year (e.g. 2025 = 6 Apr 2025 – 5 Apr 2026)
  totalSmallDonations Decimal @db.Decimal(10, 2)
  gasdsAmount   Decimal     @db.Decimal(10, 2) // 25% top-up
  status        ClaimStatus @default(DRAFT)
  createdAt     DateTime    @default(now())
  updatedAt     DateTime    @updatedAt

  @@unique([tenantId, taxYear])
  @@index([tenantId])
}

model Expenditure {
  id          String   @id @default(cuid())
  tenantId    String
  tenant      Tenant   @relation(fields: [tenantId], references: [id])
  fundId      String
  fund        Fund     @relation(fields: [fundId], references: [id])
  amount      Decimal  @db.Decimal(10, 2)
  description String
  date        DateTime
  approvedBy  String?  // Name or user ID
  receiptUrl  String?  // Link to uploaded receipt
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  @@index([tenantId])
  @@index([tenantId, fundId])
}
```

Also add to the `Contact` model (after `lastName`):
```prisma
  title       String?     // Mr, Mrs, Dr, etc. — needed for HMRC Gift Aid export
```

Add reverse relations to `Tenant`:
```prisma
  giftAidClaims        GiftAidClaim[]
  gasdsClaims          GasdsClaim[]
  expenditures         Expenditure[]
```

Add reverse relation to `Fund`:
```prisma
  expenditures Expenditure[]
```

**Step 2:** Run migration

```bash
npx prisma migrate dev --name add_gasds_giftaidclaim_expenditure_contact_title
```

**Step 3:** Commit

```bash
git add prisma/
git commit -m "feat(schema): add GiftAidClaim, GasdsClaim, Expenditure models and Contact.title field"
```

---

### Task 2: UK Financial Year Utility

**Files:**
- Create: `src/lib/financial-year.ts`
- Create: `src/__tests__/lib/financial-year.test.ts`

**Step 1: Write failing tests**

```typescript
// src/__tests__/lib/financial-year.test.ts
import { describe, it, expect } from 'vitest'
import { getFinancialYear, getFinancialYearRange, formatFinancialYear } from '@/lib/financial-year'

describe('getFinancialYear', () => {
  it('returns previous year for dates before 6 April', () => {
    expect(getFinancialYear(new Date('2026-03-15'))).toBe(2025)
    expect(getFinancialYear(new Date('2026-04-05'))).toBe(2025)
  })

  it('returns current year for dates on or after 6 April', () => {
    expect(getFinancialYear(new Date('2026-04-06'))).toBe(2026)
    expect(getFinancialYear(new Date('2026-12-25'))).toBe(2026)
  })
})

describe('getFinancialYearRange', () => {
  it('returns correct date range for tax year 2025', () => {
    const { start, end } = getFinancialYearRange(2025)
    expect(start).toEqual(new Date(2025, 3, 6))  // 6 April 2025
    expect(end).toEqual(new Date(2026, 3, 5, 23, 59, 59, 999)) // 5 April 2026 end of day
  })
})

describe('formatFinancialYear', () => {
  it('formats as "2025/26"', () => {
    expect(formatFinancialYear(2025)).toBe('2025/26')
  })
})
```

**Step 2:** Run tests to verify they fail

```bash
npx vitest run src/__tests__/lib/financial-year.test.ts
```

**Step 3: Implement**

```typescript
// src/lib/financial-year.ts

/**
 * UK financial/tax year runs 6 April – 5 April.
 * getFinancialYear returns the start year (e.g. 2025 for 6 Apr 2025 – 5 Apr 2026).
 */
export function getFinancialYear(date: Date): number {
  const month = date.getMonth() // 0-indexed (3 = April)
  const day = date.getDate()
  // Before 6 April → previous tax year
  if (month < 3 || (month === 3 && day < 6)) {
    return date.getFullYear() - 1
  }
  return date.getFullYear()
}

export function getFinancialYearRange(taxYear: number): { start: Date; end: Date } {
  return {
    start: new Date(taxYear, 3, 6),         // 6 April
    end: new Date(taxYear + 1, 3, 5, 23, 59, 59, 999), // 5 April end of day
  }
}

export function formatFinancialYear(taxYear: number): string {
  return `${taxYear}/${(taxYear + 1) % 100}`
}
```

**Step 4:** Run tests to verify they pass

```bash
npx vitest run src/__tests__/lib/financial-year.test.ts
```

**Step 5:** Commit

```bash
git add src/lib/financial-year.ts src/__tests__/lib/financial-year.test.ts
git commit -m "feat(lib): add UK financial year utility functions with tests"
```

---

### Task 3: Install Dependencies

**Step 1:** Install packages

```bash
npm install resend qrcode
npm install -D @types/qrcode
```

**Step 2:** Commit

```bash
git add package.json package-lock.json
git commit -m "chore(deps): add resend, qrcode packages"
```

---

## Phase 1: Gift Aid Compliance (Tier 1)

### Task 4: GASDS Eligibility Logic

**Files:**
- Create: `src/lib/gasds.ts`
- Create: `src/__tests__/lib/gasds.test.ts`

**Step 1: Write failing tests**

```typescript
// src/__tests__/lib/gasds.test.ts
import { describe, it, expect } from 'vitest'
import {
  isGasdsEligible,
  calculateGasdsAmount,
  GASDS_MAX_DONATION,
  GASDS_ANNUAL_LIMIT,
} from '@/lib/gasds'

describe('isGasdsEligible', () => {
  it('returns true for cash donation <= £30', () => {
    expect(isGasdsEligible(30, 'CASH')).toBe(true)
    expect(isGasdsEligible(10, 'CONTACTLESS_KIOSK')).toBe(true)
  })

  it('returns false for donations > £30', () => {
    expect(isGasdsEligible(30.01, 'CASH')).toBe(false)
  })

  it('returns false for non-cash methods', () => {
    expect(isGasdsEligible(20, 'BANK_TRANSFER')).toBe(false)
    expect(isGasdsEligible(20, 'CARD_ONLINE')).toBe(false)
  })
})

describe('calculateGasdsAmount', () => {
  it('calculates 25% top-up', () => {
    expect(calculateGasdsAmount(1000)).toBe(250)
  })
})

describe('constants', () => {
  it('has correct limits', () => {
    expect(GASDS_MAX_DONATION).toBe(30)
    expect(GASDS_ANNUAL_LIMIT).toBe(8000)
  })
})
```

**Step 2:** Verify tests fail: `npx vitest run src/__tests__/lib/gasds.test.ts`

**Step 3: Implement**

```typescript
// src/lib/gasds.ts

/** Maximum single donation amount eligible for GASDS */
export const GASDS_MAX_DONATION = 30

/** Annual GASDS claim limit per charity */
export const GASDS_ANNUAL_LIMIT = 8000

/** GASDS rate (same as Gift Aid) */
export const GASDS_RATE = 0.25

/** Payment methods eligible for GASDS (cash and contactless) */
const GASDS_ELIGIBLE_METHODS = new Set(['CASH', 'CONTACTLESS_KIOSK'])

/**
 * Check if a single donation is GASDS-eligible.
 * Must be cash/contactless and ≤ £30.
 */
export function isGasdsEligible(amount: number, method: string): boolean {
  return GASDS_ELIGIBLE_METHODS.has(method) && amount <= GASDS_MAX_DONATION
}

/**
 * Calculate GASDS top-up amount (25% of eligible total).
 */
export function calculateGasdsAmount(eligibleTotal: number): number {
  return Math.round(eligibleTotal * GASDS_RATE * 100) / 100
}
```

**Step 4:** Verify tests pass: `npx vitest run src/__tests__/lib/gasds.test.ts`

**Step 5:** Commit

```bash
git add src/lib/gasds.ts src/__tests__/lib/gasds.test.ts
git commit -m "feat(gasds): add GASDS eligibility logic and calculation utilities"
```

---

### Task 5: GASDS Claim API

**Files:**
- Create: `src/lib/validations/gasds.ts`
- Create: `src/app/api/gasds/claim/route.ts`

The API should:
- GET: Return GASDS summary for a tax year (eligible donations total, claimable amount, 10x rule check, annual limit remaining)
- POST: Create a GasdsClaim record for the tax year

The **10x rule**: GASDS claim cannot exceed 10× the Gift Aid claim amount in the same tax year. Query total `giftAidClaimed` donations in the tax year and multiply by 10 to get the GASDS ceiling.

Zod schema for POST: `{ taxYear: z.number().int() }`

The GET endpoint accepts `?taxYear=2025` query param. It queries all GASDS-eligible donations (cash/contactless, ≤ £30, not already in a GASDS claim) in the financial year range, sums them, then applies:
1. 10x rule ceiling: `10 × totalGiftAidClaimedInYear`
2. Annual limit: `£8,000`
3. Already claimed GASDS in year

**Step 1: Implement validation**

```typescript
// src/lib/validations/gasds.ts
import { z } from 'zod'

export const gasdsClaimSchema = z.object({
  taxYear: z.number().int().min(2020).max(2100),
})
```

**Step 2: Implement API route**

The route should use `getFinancialYearRange()` to determine date bounds, query eligible donations using `isGasdsEligible` criteria via Prisma `where`, check limits, and create the `GasdsClaim` record.

**Step 3:** Commit

```bash
git add src/lib/validations/gasds.ts src/app/api/gasds/claim/route.ts
git commit -m "feat(gasds): add GASDS claim API with 10x rule and annual limit enforcement"
```

---

### Task 6: HMRC Gift Aid CSV Export

**Files:**
- Create: `src/lib/hmrc-export.ts`
- Create: `src/__tests__/lib/hmrc-export.test.ts`
- Create: `src/app/api/gift-aid/hmrc-export/route.ts`

**Step 1: Write failing tests**

Test the CSV generator function: given an array of donor+donation data, it produces HMRC-compliant CSV with columns: `Title, First Name, Last Name, House Name/Number, Postcode, Aggregated Donations, Sponsored Event`. The HMRC schedule limit is 1,000 rows; the generator should split into multiple schedules if needed.

```typescript
// src/__tests__/lib/hmrc-export.test.ts
import { describe, it, expect } from 'vitest'
import { generateHmrcCsv, HmrcDonorRow } from '@/lib/hmrc-export'

describe('generateHmrcCsv', () => {
  it('generates valid CSV header and rows', () => {
    const rows: HmrcDonorRow[] = [{
      title: 'Mr',
      firstName: 'Ahmad',
      lastName: 'Khan',
      houseNameOrNumber: '42',
      postcode: 'E1 6AN',
      aggregatedAmount: 150.00,
    }]
    const csv = generateHmrcCsv(rows)
    expect(csv).toContain('Title,First Name,Last Name,House Name or Number,Postcode,Aggregated Donations,Sponsored Event')
    expect(csv).toContain('Mr,Ahmad,Khan,42,E1 6AN,150.00,No')
  })

  it('escapes commas in fields', () => {
    const rows: HmrcDonorRow[] = [{
      title: 'Mrs',
      firstName: 'Fatima',
      lastName: 'Al-Hassan',
      houseNameOrNumber: 'Flat 3, Rose Court',
      postcode: 'SW1A 1AA',
      aggregatedAmount: 200.00,
    }]
    const csv = generateHmrcCsv(rows)
    expect(csv).toContain('"Flat 3, Rose Court"')
  })
})
```

**Step 2:** Verify fail: `npx vitest run src/__tests__/lib/hmrc-export.test.ts`

**Step 3: Implement generator**

```typescript
// src/lib/hmrc-export.ts

export interface HmrcDonorRow {
  title: string
  firstName: string
  lastName: string
  houseNameOrNumber: string
  postcode: string
  aggregatedAmount: number
}

const HMRC_HEADER = 'Title,First Name,Last Name,House Name or Number,Postcode,Aggregated Donations,Sponsored Event'

function escapeCsv(value: string): string {
  if (value.includes(',') || value.includes('"') || value.includes('\n')) {
    return `"${value.replace(/"/g, '""')}"`
  }
  return value
}

export function generateHmrcCsv(rows: HmrcDonorRow[]): string {
  const lines = [HMRC_HEADER]
  for (const row of rows) {
    lines.push([
      escapeCsv(row.title),
      escapeCsv(row.firstName),
      escapeCsv(row.lastName),
      escapeCsv(row.houseNameOrNumber),
      escapeCsv(row.postcode),
      row.aggregatedAmount.toFixed(2),
      'No',
    ].join(','))
  }
  return lines.join('\n')
}

/** HMRC schedule limit */
export const HMRC_SCHEDULE_LIMIT = 1000
```

**Step 4:** Verify pass: `npx vitest run src/__tests__/lib/hmrc-export.test.ts`

**Step 5:** Implement API route at `src/app/api/gift-aid/hmrc-export/route.ts`

The GET endpoint accepts `?claimId=xxx` and:
1. Loads the GiftAidClaim record
2. Loads all donations by their IDs with contact details
3. Aggregates donations per donor (by contact ID)
4. Generates HMRC CSV using the utility
5. Returns as downloadable CSV

**Step 6:** Commit

```bash
git add src/lib/hmrc-export.ts src/__tests__/lib/hmrc-export.test.ts src/app/api/gift-aid/hmrc-export/route.ts
git commit -m "feat(hmrc): add HMRC Gift Aid CSV export with schedule splitting"
```

---

### Task 7: Gift Aid Claim History & Audit Trail

**Files:**
- Create: `src/lib/validations/gift-aid-claim.ts`
- Create: `src/app/api/gift-aid/claims/route.ts`
- Create: `src/app/api/gift-aid/claims/[id]/route.ts`
- Create: `src/app/(dashboard)/gift-aid/claims/page.tsx`

**Step 1:** Create Zod validation for claim creation:

```typescript
// src/lib/validations/gift-aid-claim.ts
import { z } from 'zod'

export const giftAidClaimCreateSchema = z.object({
  donationIds: z.array(z.string().min(1)).min(1, 'At least one donation required'),
  notes: z.string().max(500).optional(),
})

export const giftAidClaimUpdateSchema = z.object({
  status: z.enum(['SUBMITTED', 'ACCEPTED', 'REJECTED']),
  hmrcRef: z.string().max(100).optional(),
  notes: z.string().max(500).optional(),
})
```

**Step 2:** Implement claims API:

- `GET /api/gift-aid/claims` — List all claims for tenant (paginated, ordered by createdAt desc)
- `POST /api/gift-aid/claims` — Create a new DRAFT claim from selected donation IDs. Validates all donations are unclaimed, calculates totals, creates GiftAidClaim record, marks donations as `giftAidClaimed = true`.

**Step 3:** Implement claim detail API:

- `GET /api/gift-aid/claims/[id]` — Get single claim with summary
- `PATCH /api/gift-aid/claims/[id]` — Update status (SUBMITTED/ACCEPTED/REJECTED), add HMRC ref

**Step 4:** Implement claims history page at `src/app/(dashboard)/gift-aid/claims/page.tsx`

Server component that lists all GiftAidClaim records in a table: date, status badge, total amount, Gift Aid amount, donation count, HMRC ref. Link to HMRC export download. Add a link to this page from the existing Gift Aid page.

**Step 5:** Commit

```bash
git add src/lib/validations/gift-aid-claim.ts src/app/api/gift-aid/claims/ src/app/\(dashboard\)/gift-aid/claims/
git commit -m "feat(gift-aid): add claim history, audit trail, and status tracking"
```

---

### Task 8: Trustee Reporting Pack

**Files:**
- Create: `src/lib/trustee-export.ts`
- Create: `src/__tests__/lib/trustee-export.test.ts`
- Create: `src/app/api/reports/trustee-pack/donations-by-fund/route.ts`
- Create: `src/app/api/reports/trustee-pack/gift-aid-summary/route.ts`
- Create: `src/app/api/reports/trustee-pack/campaign-progress/route.ts`
- Create: `src/app/(dashboard)/reports/trustee-pack/page.tsx`
- Create: `src/components/reports/FinancialYearPicker.tsx`

**Step 1: Write failing tests for CSV generators**

```typescript
// src/__tests__/lib/trustee-export.test.ts
import { describe, it, expect } from 'vitest'
import { generateDonationsByFundCsv, generateGiftAidSummaryCsv, generateCampaignProgressCsv } from '@/lib/trustee-export'

describe('generateDonationsByFundCsv', () => {
  it('generates CSV with fund name, category, donated, spent, balance', () => {
    const csv = generateDonationsByFundCsv([
      { fundName: 'Zakat Fund', category: 'ZAKAT', restricted: true, totalDonated: 5000, totalSpent: 1200, balance: 3800 },
    ])
    expect(csv).toContain('Fund,Category,Restricted,Total Donated,Total Spent,Balance')
    expect(csv).toContain('Zakat Fund,Zakat,Yes,5000.00,1200.00,3800.00')
  })
})
```

**Step 2:** Verify fail, then implement `src/lib/trustee-export.ts` with three CSV generator functions.

**Step 3:** Implement three API routes — each accepts `?taxYear=2025` (defaults to current FY), queries Prisma, generates CSV, returns as download.

**Step 4:** Implement `FinancialYearPicker.tsx` — client component with a select dropdown for tax years (current ± 3 years).

**Step 5:** Implement trustee pack page — shows three download buttons (Donations by Fund, Gift Aid Summary, Campaign Progress) with the FinancialYearPicker.

**Step 6:** Commit

```bash
git add src/lib/trustee-export.ts src/__tests__/lib/trustee-export.test.ts src/app/api/reports/trustee-pack/ src/app/\(dashboard\)/reports/trustee-pack/ src/components/reports/FinancialYearPicker.tsx
git commit -m "feat(trustees): add trustee reporting pack with CSV exports"
```

---

## Phase 2: Operational Features (Tier 2)

### Task 9: Cash Batch Logging

**Files:**
- Create: `src/lib/validations/cash-batch.ts`
- Create: `src/app/api/donations/cash-batch/route.ts`
- Create: `src/app/(dashboard)/donations/cash-batch/page.tsx`
- Create: `src/components/donations/CashBatchForm.tsx`

**Step 1:** Zod validation — array of `{ amount, fundId, giftAidEligible?, notes? }` with a batch date:

```typescript
// src/lib/validations/cash-batch.ts
import { z } from 'zod'

export const cashBatchItemSchema = z.object({
  amount: z.number().min(0.01).max(50000),
  fundId: z.string().min(1),
  giftAidEligible: z.boolean().default(false),
  notes: z.string().max(500).optional(),
})

export const cashBatchSchema = z.object({
  date: z.string().datetime({ offset: true }).or(z.string().date()),
  items: z.array(cashBatchItemSchema).min(1, 'At least one item required').max(200),
})
```

**Step 2:** API route `POST /api/donations/cash-batch` — validates, creates Donation records in a transaction with `method: 'CASH'`, auto-computes hijriDate. Returns count + total.

**Step 3:** Client component `CashBatchForm.tsx` — date picker, dynamic rows (add/remove), fund select, amount input, Gift Aid checkbox per row. Subtotal display. Submit creates all donations.

**Step 4:** Page at `/donations/cash-batch` with link from main donations page.

**Step 5:** Commit

```bash
git add src/lib/validations/cash-batch.ts src/app/api/donations/cash-batch/route.ts src/app/\(dashboard\)/donations/cash-batch/ src/components/donations/CashBatchForm.tsx
git commit -m "feat(cash-batch): add bulk cash donation entry with batch logging"
```

---

### Task 10: Donation CSV Export

**Files:**
- Create: `src/lib/donation-export.ts`
- Create: `src/__tests__/lib/donation-export.test.ts`
- Create: `src/app/api/donations/export/route.ts`

**Step 1: Write failing tests**

Test that the CSV generator produces correct columns: Date, Donor, Amount, Fund, Category, Method, Gift Aid Eligible, Gift Aid Claimed, Reference.

**Step 2:** Implement generator in `src/lib/donation-export.ts`.

**Step 3:** API route `GET /api/donations/export` — accepts query params `dateFrom`, `dateTo`, `fundId`, `givingCategory`, `method`. Queries donations with filters, generates CSV, returns as download.

**Step 4:** Add an "Export CSV" button to the existing donations page (`src/app/(dashboard)/donations/page.tsx`) that links to the export endpoint with current filters.

**Step 5:** Commit

```bash
git add src/lib/donation-export.ts src/__tests__/lib/donation-export.test.ts src/app/api/donations/export/route.ts
git commit -m "feat(donations): add CSV export with filter support"
```

---

### Task 11: QR Code for Donation Page

**Files:**
- Create: `src/app/api/donate/qr/[slug]/route.ts`
- Create: `src/app/(dashboard)/donate-qr/page.tsx`

**Step 1:** API route `GET /api/donate/qr/[slug]` — generates QR code SVG for `{NEXT_PUBLIC_APP_URL}/donate/{slug}` using the `qrcode` package. Returns SVG with `Content-Type: image/svg+xml`.

```typescript
import QRCode from 'qrcode'

// In handler:
const url = `${process.env.NEXT_PUBLIC_APP_URL}/donate/${slug}`
const svg = await QRCode.toString(url, { type: 'svg', width: 400, margin: 2 })
return new NextResponse(svg, {
  headers: { 'Content-Type': 'image/svg+xml' },
})
```

**Step 2:** Page at `/donate-qr` — server component that fetches tenant slug and renders the QR code image, the donation URL, and a print button. Clean layout for printing (hide nav in print media query).

**Step 3:** Add to sidebar navigation.

**Step 4:** Commit

```bash
git add src/app/api/donate/qr/ src/app/\(dashboard\)/donate-qr/
git commit -m "feat(qr): add QR code generation and printable page for donation link"
```

---

### Task 12: Fund Expenditure Tracking

**Files:**
- Create: `src/lib/validations/expenditure.ts`
- Create: `src/app/api/expenditures/route.ts`
- Create: `src/app/api/expenditures/[id]/route.ts`
- Modify: `src/app/(dashboard)/funds/page.tsx`

**Step 1:** Zod validation:

```typescript
// src/lib/validations/expenditure.ts
import { z } from 'zod'

export const expenditureCreateSchema = z.object({
  fundId: z.string().min(1),
  amount: z.number().min(0.01).max(1000000),
  description: z.string().min(1).max(500),
  date: z.string().date(),
  approvedBy: z.string().max(200).optional(),
  receiptUrl: z.string().url().max(500).optional(),
})

export const expenditureUpdateSchema = expenditureCreateSchema.partial().refine(
  (data) => Object.values(data).some((v) => v !== undefined),
  { message: 'At least one field must be provided' }
)
```

**Step 2:** CRUD API routes:
- `GET /api/expenditures` — list with `?fundId=` filter
- `POST /api/expenditures` — create (admin/treasurer only)
- `GET /api/expenditures/[id]` — detail
- `PATCH /api/expenditures/[id]` — update
- `DELETE /api/expenditures/[id]` — delete (admin only)

**Step 3:** Modify funds page — add expenditure total and balance columns:
- Query `Expenditure.groupBy({ by: ['fundId'], _sum: { amount: true } })` alongside existing donation aggregation
- Display: Total Donated | Total Spent | Balance

**Step 4:** Commit

```bash
git add src/lib/validations/expenditure.ts src/app/api/expenditures/ src/app/\(dashboard\)/funds/page.tsx
git commit -m "feat(expenditure): add expenditure tracking and fund balance display"
```

---

### Task 13: Donation Receipt Emails (Resend)

**Files:**
- Create: `src/lib/email.ts`
- Create: `src/__tests__/lib/email.test.ts`
- Modify: `src/app/api/donations/route.ts`
- Modify: `src/app/api/stripe/webhook/route.ts`

**Step 1: Write failing tests**

Test the `buildReceiptHtml` function that generates receipt email HTML (not the send itself — mock Resend).

**Step 2: Implement email utility**

```typescript
// src/lib/email.ts
import { Resend } from 'resend'

const resend = process.env.RESEND_API_KEY
  ? new Resend(process.env.RESEND_API_KEY)
  : null

export interface ReceiptData {
  donorName: string
  donorEmail: string
  amount: number
  currency: string
  fundName: string
  category: string
  date: string
  giftAidEligible: boolean
  charityName: string
  charityNumber: string | null
}

export function buildReceiptHtml(data: ReceiptData): string {
  // Generate clean HTML receipt email with charity branding
  // Include: amount, fund, date, Gift Aid status, charity details
}

export async function sendDonationReceipt(data: ReceiptData): Promise<void> {
  if (!resend) {
    console.warn('RESEND_API_KEY not set — skipping receipt email')
    return
  }

  const html = buildReceiptHtml(data)

  await resend.emails.send({
    from: `${data.charityName} <donations@${process.env.RESEND_DOMAIN ?? 'resend.dev'}>`,
    to: data.donorEmail,
    subject: `Donation Receipt — ${data.charityName}`,
    html,
  })
}
```

**Step 3:** Hook into existing donation creation route (`POST /api/donations`) — after successful creation, if contact has email, fire-and-forget `sendDonationReceipt()`.

**Step 4:** Hook into Stripe webhook (`handlePaymentIntentSucceeded`) — after donation created, if contactId is provided and contact has email, send receipt.

**Step 5:** Add `RESEND_API_KEY` and `RESEND_DOMAIN` to `.env.example`.

**Step 6:** Commit

```bash
git add src/lib/email.ts src/__tests__/lib/email.test.ts src/app/api/donations/route.ts src/app/api/stripe/webhook/route.ts .env.example
git commit -m "feat(email): add donation receipt emails via Resend"
```

---

## Phase 3: Dashboard & UX (Tier 3)

### Task 14: Dashboard — Gift Aid Metric + Campaign Progress

**Files:**
- Modify: `src/app/(dashboard)/dashboard/page.tsx`
- Modify: `src/app/api/dashboard/stats/route.ts`
- Create: `src/components/dashboard/CampaignProgress.tsx`

**Step 1:** Add to dashboard stats API:
- `unclaimedGiftAid`: sum of `amount * 0.25` for donations where `giftAidEligible = true` AND `giftAidClaimed = false`
- `activeCampaigns`: list of active campaigns with `name`, `goalAmount`, `totalRaised` (sum of campaign donations)

**Step 2:** Add "Unclaimed Gift Aid" card to `GivingSummaryCards.tsx` (or directly in dashboard page).

**Step 3:** Create `CampaignProgress.tsx` — client component showing campaign name, progress bar (raised/goal), percentage. Emerald colour scheme.

**Step 4:** Add both to the dashboard page.

**Step 5:** Commit

```bash
git add src/app/\(dashboard\)/dashboard/page.tsx src/app/api/dashboard/stats/route.ts src/components/dashboard/CampaignProgress.tsx
git commit -m "feat(dashboard): add unclaimed Gift Aid metric and campaign progress bars"
```

---

### Task 15: Financial Year Support in Reports

**Files:**
- Modify: `src/app/(dashboard)/reports/page.tsx`
- Reuse: `src/components/reports/FinancialYearPicker.tsx` (created in Task 8)

**Step 1:** Modify the reports page to:
- Default date range to current UK financial year (6 Apr – 5 Apr) instead of no filter
- Add FinancialYearPicker component to allow switching between tax years
- Pass the selected FY date range to the existing giving-by-category API

**Step 2:** Commit

```bash
git add src/app/\(dashboard\)/reports/page.tsx
git commit -m "feat(reports): default to current financial year and add FY picker"
```

---

### Task 16: Donor Giving Statements

**Files:**
- Create: `src/lib/donor-statement.ts`
- Create: `src/__tests__/lib/donor-statement.test.ts`
- Create: `src/app/api/contacts/[id]/statement/route.ts`
- Modify: `src/app/(dashboard)/contacts/[id]/page.tsx`

**Step 1: Write failing tests**

```typescript
// src/__tests__/lib/donor-statement.test.ts
import { describe, it, expect } from 'vitest'
import { generateDonorStatementCsv, DonorStatementRow } from '@/lib/donor-statement'

describe('generateDonorStatementCsv', () => {
  it('generates CSV with donor header and donation rows', () => {
    const rows: DonorStatementRow[] = [
      { date: '2025-06-15', amount: 100, fund: 'Zakat Fund', category: 'ZAKAT', method: 'CASH', giftAidEligible: true, giftAidClaimed: false },
    ]
    const csv = generateDonorStatementCsv('Ahmad Khan', '2025/26', rows)
    expect(csv).toContain('Ahmad Khan')
    expect(csv).toContain('2025/26')
    expect(csv).toContain('Zakat Fund')
    expect(csv).toContain('100.00')
  })
})
```

**Step 2:** Implement `src/lib/donor-statement.ts`.

**Step 3:** API route `GET /api/contacts/[id]/statement?taxYear=2025` — queries donations for the contact in the FY, generates CSV, returns as download.

**Step 4:** Add "Download Statement" link to the contact detail page.

**Step 5:** Commit

```bash
git add src/lib/donor-statement.ts src/__tests__/lib/donor-statement.test.ts src/app/api/contacts/\[id\]/statement/route.ts src/app/\(dashboard\)/contacts/\[id\]/page.tsx
git commit -m "feat(statements): add donor giving statement CSV export"
```

---

## Verification

After all tasks are complete:

```bash
# Type check
npx tsc --noEmit

# Run all tests
npx vitest run

# Start dev server and manually verify:
npm run dev
```

Manual verification checklist:
1. `/gift-aid` — create a claim, view claims history, download HMRC export CSV
2. `/gift-aid/claims` — verify audit trail shows all claims with status
3. `/reports/trustee-pack` — download all three CSVs for a financial year
4. `/donations/cash-batch` — log a batch of cash donations
5. `/donations` — export filtered donations as CSV
6. `/donate-qr` — view and print QR code
7. `/funds` — verify balance column shows donated - spent
8. `/dashboard` — verify Gift Aid metric and campaign progress bars
9. `/reports` — verify FY picker defaults to current financial year
10. `/contacts/[id]` — download giving statement
11. Create a donation for a contact with email — verify receipt email sent (check Resend dashboard)

---

## New File Inventory

| Directory | Files |
|-----------|-------|
| `src/lib/` | `financial-year.ts`, `gasds.ts`, `hmrc-export.ts`, `trustee-export.ts`, `donation-export.ts`, `donor-statement.ts`, `email.ts` |
| `src/lib/validations/` | `gasds.ts`, `gift-aid-claim.ts`, `cash-batch.ts`, `expenditure.ts` |
| `src/app/api/gasds/claim/` | `route.ts` |
| `src/app/api/gift-aid/hmrc-export/` | `route.ts` |
| `src/app/api/gift-aid/claims/` | `route.ts`, `[id]/route.ts` |
| `src/app/api/donations/cash-batch/` | `route.ts` |
| `src/app/api/donations/export/` | `route.ts` |
| `src/app/api/donate/qr/[slug]/` | `route.ts` |
| `src/app/api/expenditures/` | `route.ts`, `[id]/route.ts` |
| `src/app/api/reports/trustee-pack/` | `donations-by-fund/route.ts`, `gift-aid-summary/route.ts`, `campaign-progress/route.ts` |
| `src/app/api/contacts/[id]/statement/` | `route.ts` |
| `src/app/(dashboard)/gift-aid/claims/` | `page.tsx` |
| `src/app/(dashboard)/donations/cash-batch/` | `page.tsx` |
| `src/app/(dashboard)/donate-qr/` | `page.tsx` |
| `src/app/(dashboard)/reports/trustee-pack/` | `page.tsx` |
| `src/components/donations/` | `CashBatchForm.tsx` |
| `src/components/dashboard/` | `CampaignProgress.tsx` |
| `src/components/reports/` | `FinancialYearPicker.tsx` |
| `src/__tests__/lib/` | `financial-year.test.ts`, `gasds.test.ts`, `hmrc-export.test.ts`, `trustee-export.test.ts`, `donation-export.test.ts`, `donor-statement.test.ts`, `email.test.ts` |

**Modified existing files:**
- `prisma/schema.prisma`
- `src/app/(dashboard)/dashboard/page.tsx`
- `src/app/api/dashboard/stats/route.ts`
- `src/app/(dashboard)/funds/page.tsx`
- `src/app/(dashboard)/reports/page.tsx`
- `src/app/api/donations/route.ts`
- `src/app/api/stripe/webhook/route.ts`
- `src/app/(dashboard)/contacts/[id]/page.tsx`
- `src/components/layout/Sidebar.tsx`
- `.env.example`
- `package.json`
