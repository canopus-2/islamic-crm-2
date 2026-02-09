# Islamic Donor CRM — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a full-stack Islamic Donor CRM (SaaS) that replaces Beacon for UK mosques — natively understanding Islamic giving categories, the Hijri calendar, Gift Aid, and mosque-specific donor intelligence.

**Architecture:** Next.js 16 App Router with TypeScript, PostgreSQL via Prisma ORM, multi-tenant via row-level `tenantId` on every table. Auth.js v5 (NextAuth) for authentication. Stripe Connect Express for payments, GoCardless for Direct Debit. HMRC XML API for Gift Aid. Xero OAuth 2.0 for accounting sync. Deployed via Docker.

**Tech Stack:**
- Runtime: Node.js 22, TypeScript 5.x (strict)
- Framework: Next.js 16 (App Router, Server Components)
- Database: PostgreSQL 16 + Prisma ORM (latest)
- Auth: Auth.js v5 (`next-auth@beta`, `@auth/prisma-adapter`)
- Styling: Tailwind CSS v4
- Validation: Zod
- Payments: Stripe Connect (Express), GoCardless
- Calendar: `Intl.DateTimeFormat` with `islamic-umalqura` calendar + `moment-hijri` for complex calculations
- Gift Aid: HMRC Charities Online XML API
- Accounting: Xero API (OAuth 2.0)
- Email: Resend (or SendGrid)
- Testing: Vitest + React Testing Library + Playwright (e2e)
- Containerisation: Docker + docker-compose

**Project root:** `/Users/sohailsaddique/Documents/GitHub/Islamic-CRM`

---

## Phase 1: Foundation (Tasks 1–6)

### Task 1: Project Scaffolding

**Files:**
- Create: `package.json` (via create-next-app)
- Create: `tsconfig.json` (via create-next-app)
- Create: `tailwind.config.ts` (via create-next-app)
- Create: `.env.example`
- Create: `.gitignore`
- Create: `docker-compose.yml`

**Step 1: Scaffold Next.js project**

```bash
cd /Users/sohailsaddique/Documents/GitHub/Islamic-CRM
npx create-next-app@latest . --typescript --tailwind --eslint --app --src-dir --import-alias "@/*" --use-npm --yes
```

Expected: Next.js project created in current directory with App Router, TypeScript, Tailwind, ESLint, `src/` directory.

**Step 2: Install core dependencies**

```bash
cd /Users/sohailsaddique/Documents/GitHub/Islamic-CRM
npm install prisma @prisma/client next-auth@beta @auth/prisma-adapter zod moment-hijri date-fns
npm install -D vitest @vitejs/plugin-react @testing-library/react @testing-library/jest-dom jsdom
```

**Step 3: Create docker-compose.yml for local PostgreSQL**

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: crm_dev
      POSTGRES_PASSWORD: crm_dev_password
      POSTGRES_DB: islamic_crm_dev
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

**Step 4: Create .env.example**

```env
# Database
DATABASE_URL="postgresql://crm_dev:crm_dev_password@localhost:5432/islamic_crm_dev"

# Auth
NEXTAUTH_URL="http://localhost:3000"
NEXTAUTH_SECRET="generate-with-openssl-rand-base64-32"

# Stripe
STRIPE_SECRET_KEY=""
STRIPE_PUBLISHABLE_KEY=""
STRIPE_WEBHOOK_SECRET=""

# Xero
XERO_CLIENT_ID=""
XERO_CLIENT_SECRET=""

# HMRC Gift Aid
HMRC_SENDER_ID=""
HMRC_SENDER_PASSWORD=""
```

**Step 5: Copy .env.example to .env and start database**

```bash
cp .env.example .env
# Edit .env with real NEXTAUTH_SECRET:
# openssl rand -base64 32
docker compose up -d db
```

Expected: PostgreSQL running on localhost:5432.

**Step 6: Verify dev server starts**

```bash
npm run dev
```

Expected: Next.js dev server at http://localhost:3000.

**Step 7: Commit**

```bash
git add -A
git commit -m "chore: scaffold Next.js 16 project with Tailwind, Prisma, Auth.js deps"
```

---

### Task 2: Prisma Schema — Core Models

**Files:**
- Create: `prisma/schema.prisma`

**Step 1: Initialise Prisma**

```bash
cd /Users/sohailsaddique/Documents/GitHub/Islamic-CRM
npx prisma init
```

Expected: Creates `prisma/schema.prisma` and updates `.env` with DATABASE_URL.

**Step 2: Write the Prisma schema**

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ─── Multi-tenancy ──────────────────────────────────────

model Tenant {
  id        String   @id @default(cuid())
  name      String   // e.g. "East London Mosque"
  slug      String   @unique // e.g. "east-london-mosque"
  charityNumber String? // Charity Commission number
  hmrcRef   String?  // HMRC Charities Online ref
  currency  String   @default("GBP")
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  users       User[]
  contacts    Contact[]
  donations   Donation[]
  funds       Fund[]
  campaigns   Campaign[]
  giftAidDeclarations GiftAidDeclaration[]
}

// ─── Auth ────────────────────────────────────────────────

model User {
  id            String    @id @default(cuid())
  tenantId      String
  tenant        Tenant    @relation(fields: [tenantId], references: [id])
  email         String
  name          String?
  passwordHash  String?
  role          UserRole  @default(MEMBER)
  emailVerified DateTime?
  image         String?
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt

  accounts Account[]
  sessions Session[]

  @@unique([tenantId, email])
}

enum UserRole {
  ADMIN
  TREASURER
  MEMBER
}

model Account {
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String? @db.Text
  access_token      String? @db.Text
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String? @db.Text
  session_state     String?
  user              User    @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerAccountId])
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model VerificationToken {
  identifier String
  token      String   @unique
  expires    DateTime

  @@unique([identifier, token])
}

// ─── Contacts (Donors) ──────────────────────────────────

model Contact {
  id          String   @id @default(cuid())
  tenantId    String
  tenant      Tenant   @relation(fields: [tenantId], references: [id])
  firstName   String
  lastName    String
  email       String?
  phone       String?
  address1    String?
  address2    String?
  city        String?
  postcode    String?
  country     String   @default("GB")
  householdId String?
  household   Household? @relation(fields: [householdId], references: [id])
  tags        String[]   @default([])
  notes       String?
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  donations   Donation[]
  giftAidDeclarations GiftAidDeclaration[]

  @@index([tenantId])
  @@index([tenantId, email])
  @@index([tenantId, lastName])
}

model Household {
  id        String   @id @default(cuid())
  tenantId  String
  name      String   // e.g. "The Ahmed Family"
  createdAt DateTime @default(now())

  contacts  Contact[]

  @@index([tenantId])
}

// ─── Islamic Giving Categories ──────────────────────────

enum GivingCategory {
  ZAKAT
  SADAQAH
  SADAQAH_JARIYAH
  LILLAH
  FIDYAH
  KAFFARAH
  WAQF
  GENERAL
}

// ─── Funds ──────────────────────────────────────────────

model Fund {
  id              String        @id @default(cuid())
  tenantId        String
  tenant          Tenant        @relation(fields: [tenantId], references: [id])
  name            String        // e.g. "Zakat Fund", "Building Fund"
  givingCategory  GivingCategory
  isRestricted    Boolean       @default(false) // Zakat = restricted, Lillah = unrestricted
  xeroAccountCode String?      // Maps to Xero nominal code
  isActive        Boolean       @default(true)
  createdAt       DateTime      @default(now())

  donations Donation[]

  @@unique([tenantId, name])
  @@index([tenantId])
}

// ─── Donations ──────────────────────────────────────────

model Donation {
  id              String        @id @default(cuid())
  tenantId        String
  tenant          Tenant        @relation(fields: [tenantId], references: [id])
  contactId       String?
  contact         Contact?      @relation(fields: [contactId], references: [id])
  fundId          String
  fund            Fund          @relation(fields: [fundId], references: [id])
  amount          Decimal       @db.Decimal(10, 2)
  currency        String        @default("GBP")
  givingCategory  GivingCategory
  method          PaymentMethod
  reference       String?       // External payment ref (Stripe charge ID, etc.)
  hijriDate       String?       // Stored as "1446-09-15" for Hijri date
  giftAidEligible Boolean       @default(false)
  giftAidClaimed  Boolean       @default(false)
  campaignId      String?
  campaign        Campaign?     @relation(fields: [campaignId], references: [id])
  notes           String?
  createdAt       DateTime      @default(now())
  updatedAt       DateTime      @updatedAt

  @@index([tenantId])
  @@index([tenantId, contactId])
  @@index([tenantId, givingCategory])
  @@index([tenantId, createdAt])
  @@index([tenantId, giftAidClaimed])
}

enum PaymentMethod {
  CARD_ONLINE
  DIRECT_DEBIT
  BANK_TRANSFER
  CASH
  CHEQUE
  CONTACTLESS_KIOSK
  OTHER
}

// ─── Campaigns ──────────────────────────────────────────

model Campaign {
  id          String   @id @default(cuid())
  tenantId    String
  tenant      Tenant   @relation(fields: [tenantId], references: [id])
  name        String   // e.g. "Ramadan 2026", "Masjid Extension"
  fundId      String?
  goalAmount  Decimal? @db.Decimal(10, 2)
  startDate   DateTime
  endDate     DateTime?
  isActive    Boolean  @default(true)
  createdAt   DateTime @default(now())

  donations Donation[]

  @@index([tenantId])
}

// ─── Gift Aid ───────────────────────────────────────────

model GiftAidDeclaration {
  id          String   @id @default(cuid())
  tenantId    String
  contactId   String
  contact     Contact  @relation(fields: [contactId], references: [id])
  startDate   DateTime
  endDate     DateTime?
  isActive    Boolean  @default(true)
  declarationText String
  signedAt    DateTime
  createdAt   DateTime @default(now())

  @@index([tenantId, contactId])
}
```

**Step 3: Run initial migration**

```bash
npx prisma migrate dev --name init
```

Expected: Migration created in `prisma/migrations/`, database tables created.

**Step 4: Generate Prisma client**

```bash
npx prisma generate
```

Expected: Prisma Client generated.

**Step 5: Commit**

```bash
git add prisma/ .env.example
git commit -m "feat: add Prisma schema with multi-tenant Islamic CRM models"
```

---

### Task 3: Prisma Client Singleton & Tenant Context

**Files:**
- Create: `src/lib/db.ts`
- Create: `src/lib/tenant.ts`
- Test: `src/__tests__/lib/tenant.test.ts`

**Step 1: Write the failing test for tenant context**

```typescript
// src/__tests__/lib/tenant.test.ts
import { describe, it, expect } from 'vitest'
import { createTenantContext, TenantContext } from '@/lib/tenant'

describe('createTenantContext', () => {
  it('returns a context with tenantId', () => {
    const ctx: TenantContext = createTenantContext('tenant_abc123')
    expect(ctx.tenantId).toBe('tenant_abc123')
  })

  it('throws if tenantId is empty', () => {
    expect(() => createTenantContext('')).toThrow('tenantId is required')
  })
})
```

**Step 2: Create vitest config**

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    include: ['src/**/*.test.{ts,tsx}'],
  },
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
})
```

Add to `package.json` scripts:
```json
"test": "vitest run",
"test:watch": "vitest"
```

**Step 3: Run test to verify it fails**

```bash
npx vitest run src/__tests__/lib/tenant.test.ts
```

Expected: FAIL — module `@/lib/tenant` not found.

**Step 4: Write Prisma singleton**

```typescript
// src/lib/db.ts
import { PrismaClient } from '@prisma/client'

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined
}

export const prisma = globalForPrisma.prisma ?? new PrismaClient()

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma
```

**Step 5: Write tenant context**

```typescript
// src/lib/tenant.ts
export interface TenantContext {
  tenantId: string
}

export function createTenantContext(tenantId: string): TenantContext {
  if (!tenantId) {
    throw new Error('tenantId is required')
  }
  return { tenantId }
}
```

**Step 6: Run test to verify it passes**

```bash
npx vitest run src/__tests__/lib/tenant.test.ts
```

Expected: PASS (2 tests).

**Step 7: Commit**

```bash
git add src/lib/db.ts src/lib/tenant.ts src/__tests__/lib/tenant.test.ts vitest.config.ts package.json
git commit -m "feat: add Prisma singleton and tenant context utility"
```

---

### Task 4: Auth.js v5 Setup with Prisma Adapter

**Files:**
- Create: `src/lib/auth.ts`
- Create: `src/app/api/auth/[...nextauth]/route.ts`
- Modify: `src/app/layout.tsx` (wrap with SessionProvider)
- Create: `src/components/auth/SessionProvider.tsx`

**Step 1: Create auth config**

```typescript
// src/lib/auth.ts
import NextAuth from 'next-auth'
import { PrismaAdapter } from '@auth/prisma-adapter'
import Credentials from 'next-auth/providers/credentials'
import { prisma } from '@/lib/db'
import { z } from 'zod'
import bcrypt from 'bcryptjs'

const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
})

export const { handlers, auth, signIn, signOut } = NextAuth({
  adapter: PrismaAdapter(prisma),
  session: { strategy: 'jwt' },
  providers: [
    Credentials({
      credentials: {
        email: { label: 'Email', type: 'email' },
        password: { label: 'Password', type: 'password' },
      },
      async authorize(credentials) {
        const parsed = loginSchema.safeParse(credentials)
        if (!parsed.success) return null

        const user = await prisma.user.findFirst({
          where: { email: parsed.data.email },
        })
        if (!user || !user.passwordHash) return null

        const valid = await bcrypt.compare(parsed.data.password, user.passwordHash)
        if (!valid) return null

        return { id: user.id, email: user.email, name: user.name }
      },
    }),
  ],
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        const dbUser = await prisma.user.findUnique({
          where: { id: user.id },
          select: { tenantId: true, role: true },
        })
        if (dbUser) {
          token.tenantId = dbUser.tenantId
          token.role = dbUser.role
        }
      }
      return token
    },
    async session({ session, token }) {
      if (token) {
        session.user.id = token.sub!
        session.user.tenantId = token.tenantId as string
        session.user.role = token.role as string
      }
      return session
    },
  },
  pages: {
    signIn: '/login',
  },
})
```

**Step 2: Install bcryptjs**

```bash
npm install bcryptjs
npm install -D @types/bcryptjs
```

**Step 3: Create route handler**

```typescript
// src/app/api/auth/[...nextauth]/route.ts
import { handlers } from '@/lib/auth'

export const { GET, POST } = handlers
```

**Step 4: Create SessionProvider wrapper**

```typescript
// src/components/auth/SessionProvider.tsx
'use client'

import { SessionProvider as NextAuthSessionProvider } from 'next-auth/react'

export function SessionProvider({ children }: { children: React.ReactNode }) {
  return <NextAuthSessionProvider>{children}</NextAuthSessionProvider>
}
```

**Step 5: Create auth type augmentation**

```typescript
// src/types/next-auth.d.ts
import 'next-auth'

declare module 'next-auth' {
  interface Session {
    user: {
      id: string
      email: string
      name?: string | null
      tenantId: string
      role: string
    }
  }
}

declare module 'next-auth/jwt' {
  interface JWT {
    tenantId?: string
    role?: string
  }
}
```

**Step 6: Wrap layout with SessionProvider**

Modify `src/app/layout.tsx` — wrap `{children}` with `<SessionProvider>`.

**Step 7: Verify dev server starts without errors**

```bash
npm run dev
```

Expected: No errors. Auth routes available at `/api/auth/*`.

**Step 8: Commit**

```bash
git add src/lib/auth.ts src/app/api/auth/ src/components/auth/ src/types/ src/app/layout.tsx package.json package-lock.json
git commit -m "feat: add Auth.js v5 with Prisma adapter and JWT sessions"
```

---

### Task 5: Zod Validation Schemas

**Files:**
- Create: `src/lib/validations/contact.ts`
- Create: `src/lib/validations/donation.ts`
- Create: `src/lib/validations/fund.ts`
- Test: `src/__tests__/lib/validations/donation.test.ts`

**Step 1: Write the failing test**

```typescript
// src/__tests__/lib/validations/donation.test.ts
import { describe, it, expect } from 'vitest'
import { donationCreateSchema } from '@/lib/validations/donation'

describe('donationCreateSchema', () => {
  it('validates a correct donation', () => {
    const result = donationCreateSchema.safeParse({
      contactId: 'clxyz123',
      fundId: 'fund_abc',
      amount: 100.50,
      givingCategory: 'ZAKAT',
      method: 'CARD_ONLINE',
      giftAidEligible: true,
    })
    expect(result.success).toBe(true)
  })

  it('rejects negative amount', () => {
    const result = donationCreateSchema.safeParse({
      fundId: 'fund_abc',
      amount: -10,
      givingCategory: 'ZAKAT',
      method: 'CARD_ONLINE',
    })
    expect(result.success).toBe(false)
  })

  it('rejects invalid giving category', () => {
    const result = donationCreateSchema.safeParse({
      fundId: 'fund_abc',
      amount: 50,
      givingCategory: 'INVALID',
      method: 'CASH',
    })
    expect(result.success).toBe(false)
  })
})
```

**Step 2: Run test to verify it fails**

```bash
npx vitest run src/__tests__/lib/validations/donation.test.ts
```

Expected: FAIL — module not found.

**Step 3: Write validation schemas**

```typescript
// src/lib/validations/donation.ts
import { z } from 'zod'

export const GIVING_CATEGORIES = [
  'ZAKAT', 'SADAQAH', 'SADAQAH_JARIYAH', 'LILLAH',
  'FIDYAH', 'KAFFARAH', 'WAQF', 'GENERAL',
] as const

export const PAYMENT_METHODS = [
  'CARD_ONLINE', 'DIRECT_DEBIT', 'BANK_TRANSFER',
  'CASH', 'CHEQUE', 'CONTACTLESS_KIOSK', 'OTHER',
] as const

export const donationCreateSchema = z.object({
  contactId: z.string().optional(),
  fundId: z.string().min(1),
  amount: z.number().positive('Amount must be positive'),
  givingCategory: z.enum(GIVING_CATEGORIES),
  method: z.enum(PAYMENT_METHODS),
  giftAidEligible: z.boolean().default(false),
  campaignId: z.string().optional(),
  notes: z.string().optional(),
  reference: z.string().optional(),
})

export type DonationCreate = z.infer<typeof donationCreateSchema>
```

```typescript
// src/lib/validations/contact.ts
import { z } from 'zod'

export const contactCreateSchema = z.object({
  firstName: z.string().min(1, 'First name is required'),
  lastName: z.string().min(1, 'Last name is required'),
  email: z.string().email().optional().or(z.literal('')),
  phone: z.string().optional(),
  address1: z.string().optional(),
  address2: z.string().optional(),
  city: z.string().optional(),
  postcode: z.string().optional(),
  country: z.string().default('GB'),
  tags: z.array(z.string()).default([]),
  notes: z.string().optional(),
})

export type ContactCreate = z.infer<typeof contactCreateSchema>
```

```typescript
// src/lib/validations/fund.ts
import { z } from 'zod'
import { GIVING_CATEGORIES } from './donation'

export const fundCreateSchema = z.object({
  name: z.string().min(1, 'Fund name is required'),
  givingCategory: z.enum(GIVING_CATEGORIES),
  isRestricted: z.boolean().default(false),
  xeroAccountCode: z.string().optional(),
})

export type FundCreate = z.infer<typeof fundCreateSchema>
```

**Step 4: Run test to verify it passes**

```bash
npx vitest run src/__tests__/lib/validations/donation.test.ts
```

Expected: PASS (3 tests).

**Step 5: Commit**

```bash
git add src/lib/validations/ src/__tests__/lib/validations/
git commit -m "feat: add Zod validation schemas for contacts, donations, funds"
```

---

### Task 6: Hijri Calendar Utility

**Files:**
- Create: `src/lib/hijri.ts`
- Test: `src/__tests__/lib/hijri.test.ts`

**Step 1: Write the failing test**

```typescript
// src/__tests__/lib/hijri.test.ts
import { describe, it, expect } from 'vitest'
import {
  toHijri,
  toGregorian,
  isRamadan,
  isDhulHijjah,
  formatHijriDate,
} from '@/lib/hijri'

describe('Hijri calendar utilities', () => {
  it('converts a known Gregorian date to Hijri', () => {
    // 1 Ramadan 1446 = approx 1 March 2025
    const hijri = toHijri(new Date('2025-03-01'))
    expect(hijri.year).toBe(1446)
    expect(hijri.month).toBe(9) // Ramadan = month 9
  })

  it('formats a Hijri date string', () => {
    const formatted = formatHijriDate(new Date('2025-03-01'))
    expect(formatted).toMatch(/1446/)
    expect(formatted).toMatch(/Ramadan|Ramaḍān/)
  })

  it('detects Ramadan correctly', () => {
    // During Ramadan 1446
    expect(isRamadan(new Date('2025-03-15'))).toBe(true)
    // Outside Ramadan
    expect(isRamadan(new Date('2025-01-15'))).toBe(false)
  })

  it('detects Dhul Hijjah correctly', () => {
    // Dhul Hijjah 1446 ~ June 2025
    expect(isDhulHijjah(new Date('2025-06-07'))).toBe(true)
  })
})
```

**Step 2: Run test to verify it fails**

```bash
npx vitest run src/__tests__/lib/hijri.test.ts
```

Expected: FAIL — module not found.

**Step 3: Write the Hijri utility**

```typescript
// src/lib/hijri.ts

export interface HijriDate {
  year: number
  month: number
  day: number
  monthName: string
}

const HIJRI_MONTHS = [
  'Muharram', 'Safar', 'Rabi al-Awwal', 'Rabi al-Thani',
  'Jumada al-Ula', 'Jumada al-Thani', 'Rajab', 'Sha\'ban',
  'Ramadan', 'Shawwal', 'Dhul Qi\'dah', 'Dhul Hijjah',
]

/**
 * Convert Gregorian Date to Hijri using Intl (Umm al-Qura calendar).
 */
export function toHijri(date: Date): HijriDate {
  const formatter = new Intl.DateTimeFormat('en-u-ca-islamic-umalqura', {
    year: 'numeric',
    month: 'numeric',
    day: 'numeric',
  })

  const parts = formatter.formatToParts(date)
  const year = parseInt(parts.find(p => p.type === 'year')?.value ?? '0')
  const month = parseInt(parts.find(p => p.type === 'month')?.value ?? '0')
  const day = parseInt(parts.find(p => p.type === 'day')?.value ?? '0')

  return {
    year,
    month,
    day,
    monthName: HIJRI_MONTHS[month - 1] ?? 'Unknown',
  }
}

/**
 * Format a Gregorian date as a Hijri string: "15 Ramadan 1446"
 */
export function formatHijriDate(date: Date): string {
  const h = toHijri(date)
  return `${h.day} ${h.monthName} ${h.year}`
}

/**
 * Check if a Gregorian date falls in Ramadan (Hijri month 9).
 */
export function isRamadan(date: Date): boolean {
  return toHijri(date).month === 9
}

/**
 * Check if a Gregorian date falls in Dhul Hijjah (Hijri month 12).
 */
export function isDhulHijjah(date: Date): boolean {
  return toHijri(date).month === 12
}

/**
 * Convert Hijri date back to approximate Gregorian.
 * Uses moment-hijri for reverse calculation.
 */
export function toGregorian(hijriYear: number, hijriMonth: number, hijriDay: number): Date {
  // Approximate algorithm — for display purposes
  // More accurate: use moment-hijri for critical calculations
  const momentHijri = require('moment-hijri')
  const m = momentHijri(`${hijriYear}/${hijriMonth}/${hijriDay}`, 'iYYYY/iM/iD')
  return m.toDate()
}
```

**Step 4: Run test to verify it passes**

```bash
npx vitest run src/__tests__/lib/hijri.test.ts
```

Expected: PASS (4 tests). Note: exact Hijri date assertions may need adjustment based on Umm al-Qura calendar precision — verify outputs and adjust expected values if off by 1 day.

**Step 5: Commit**

```bash
git add src/lib/hijri.ts src/__tests__/lib/hijri.test.ts
git commit -m "feat: add Hijri calendar utilities with Intl.DateTimeFormat"
```

---

## Phase 2: CRUD API Routes (Tasks 7–10)

### Task 7: Contacts CRUD API

**Files:**
- Create: `src/app/api/contacts/route.ts` (GET list, POST create)
- Create: `src/app/api/contacts/[id]/route.ts` (GET one, PATCH update, DELETE)
- Create: `src/lib/api-utils.ts` (shared helpers)
- Test: `src/__tests__/api/contacts.test.ts`

**Step 1: Write the failing test**

```typescript
// src/__tests__/api/contacts.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { contactCreateSchema } from '@/lib/validations/contact'

describe('Contact validation', () => {
  it('validates a correct contact', () => {
    const result = contactCreateSchema.safeParse({
      firstName: 'Ahmed',
      lastName: 'Khan',
      email: 'ahmed@example.com',
      postcode: 'E1 1AA',
    })
    expect(result.success).toBe(true)
  })

  it('rejects missing first name', () => {
    const result = contactCreateSchema.safeParse({
      lastName: 'Khan',
    })
    expect(result.success).toBe(false)
  })

  it('rejects invalid email', () => {
    const result = contactCreateSchema.safeParse({
      firstName: 'Ahmed',
      lastName: 'Khan',
      email: 'not-an-email',
    })
    expect(result.success).toBe(false)
  })
})
```

**Step 2: Run test to verify it fails (or passes if validation already exists)**

```bash
npx vitest run src/__tests__/api/contacts.test.ts
```

Expected: PASS (validation already written in Task 5).

**Step 3: Write API utility helpers**

```typescript
// src/lib/api-utils.ts
import { NextRequest, NextResponse } from 'next/server'
import { auth } from '@/lib/auth'
import { ZodSchema } from 'zod'

export async function getSession() {
  const session = await auth()
  if (!session?.user?.tenantId) {
    return null
  }
  return session
}

export function unauthorized() {
  return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
}

export function badRequest(message: string) {
  return NextResponse.json({ error: message }, { status: 400 })
}

export function notFound(resource = 'Resource') {
  return NextResponse.json({ error: `${resource} not found` }, { status: 404 })
}

export async function parseBody<T>(req: NextRequest, schema: ZodSchema<T>) {
  const body = await req.json()
  return schema.safeParse(body)
}
```

**Step 4: Write contacts list + create route**

```typescript
// src/app/api/contacts/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@/lib/db'
import { getSession, unauthorized, badRequest, parseBody } from '@/lib/api-utils'
import { contactCreateSchema } from '@/lib/validations/contact'

export async function GET(req: NextRequest) {
  const session = await getSession()
  if (!session) return unauthorized()

  const { searchParams } = new URL(req.url)
  const page = parseInt(searchParams.get('page') ?? '1')
  const limit = parseInt(searchParams.get('limit') ?? '50')
  const search = searchParams.get('search') ?? ''

  const where = {
    tenantId: session.user.tenantId,
    ...(search && {
      OR: [
        { firstName: { contains: search, mode: 'insensitive' as const } },
        { lastName: { contains: search, mode: 'insensitive' as const } },
        { email: { contains: search, mode: 'insensitive' as const } },
      ],
    }),
  }

  const [contacts, total] = await Promise.all([
    prisma.contact.findMany({
      where,
      skip: (page - 1) * limit,
      take: limit,
      orderBy: { lastName: 'asc' },
    }),
    prisma.contact.count({ where }),
  ])

  return NextResponse.json({ data: contacts, total, page, limit })
}

export async function POST(req: NextRequest) {
  const session = await getSession()
  if (!session) return unauthorized()

  const parsed = await parseBody(req, contactCreateSchema)
  if (!parsed.success) {
    return badRequest(parsed.error.errors[0].message)
  }

  const contact = await prisma.contact.create({
    data: {
      ...parsed.data,
      tenantId: session.user.tenantId,
    },
  })

  return NextResponse.json(contact, { status: 201 })
}
```

**Step 5: Write single contact route**

```typescript
// src/app/api/contacts/[id]/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@/lib/db'
import { getSession, unauthorized, notFound, badRequest, parseBody } from '@/lib/api-utils'
import { contactCreateSchema } from '@/lib/validations/contact'

export async function GET(
  req: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const session = await getSession()
  if (!session) return unauthorized()

  const { id } = await params
  const contact = await prisma.contact.findFirst({
    where: { id, tenantId: session.user.tenantId },
    include: {
      donations: { orderBy: { createdAt: 'desc' }, take: 10 },
      household: true,
    },
  })

  if (!contact) return notFound('Contact')
  return NextResponse.json(contact)
}

export async function PATCH(
  req: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const session = await getSession()
  if (!session) return unauthorized()

  const { id } = await params
  const parsed = await parseBody(req, contactCreateSchema.partial())
  if (!parsed.success) {
    return badRequest(parsed.error.errors[0].message)
  }

  const contact = await prisma.contact.updateMany({
    where: { id, tenantId: session.user.tenantId },
    data: parsed.data,
  })

  if (contact.count === 0) return notFound('Contact')

  const updated = await prisma.contact.findUnique({ where: { id } })
  return NextResponse.json(updated)
}

export async function DELETE(
  req: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const session = await getSession()
  if (!session) return unauthorized()

  const { id } = await params
  const result = await prisma.contact.deleteMany({
    where: { id, tenantId: session.user.tenantId },
  })

  if (result.count === 0) return notFound('Contact')
  return NextResponse.json({ deleted: true })
}
```

**Step 6: Commit**

```bash
git add src/app/api/contacts/ src/lib/api-utils.ts src/__tests__/api/
git commit -m "feat: add contacts CRUD API with tenant isolation"
```

---

### Task 8: Funds CRUD API

**Files:**
- Create: `src/app/api/funds/route.ts`
- Create: `src/app/api/funds/[id]/route.ts`
- Test: `src/__tests__/api/funds.test.ts`

**Step 1: Write the failing test**

```typescript
// src/__tests__/api/funds.test.ts
import { describe, it, expect } from 'vitest'
import { fundCreateSchema } from '@/lib/validations/fund'

describe('Fund validation', () => {
  it('validates a Zakat fund', () => {
    const result = fundCreateSchema.safeParse({
      name: 'Zakat Fund',
      givingCategory: 'ZAKAT',
      isRestricted: true,
    })
    expect(result.success).toBe(true)
    expect(result.data?.isRestricted).toBe(true)
  })

  it('rejects empty fund name', () => {
    const result = fundCreateSchema.safeParse({
      name: '',
      givingCategory: 'SADAQAH',
    })
    expect(result.success).toBe(false)
  })
})
```

**Step 2: Run test to verify**

```bash
npx vitest run src/__tests__/api/funds.test.ts
```

Expected: PASS (validation already written).

**Step 3: Write funds route**

```typescript
// src/app/api/funds/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@/lib/db'
import { getSession, unauthorized, badRequest, parseBody } from '@/lib/api-utils'
import { fundCreateSchema } from '@/lib/validations/fund'

export async function GET(req: NextRequest) {
  const session = await getSession()
  if (!session) return unauthorized()

  const funds = await prisma.fund.findMany({
    where: { tenantId: session.user.tenantId },
    orderBy: { name: 'asc' },
  })

  return NextResponse.json({ data: funds })
}

export async function POST(req: NextRequest) {
  const session = await getSession()
  if (!session) return unauthorized()

  const parsed = await parseBody(req, fundCreateSchema)
  if (!parsed.success) {
    return badRequest(parsed.error.errors[0].message)
  }

  const fund = await prisma.fund.create({
    data: {
      ...parsed.data,
      tenantId: session.user.tenantId,
    },
  })

  return NextResponse.json(fund, { status: 201 })
}
```

```typescript
// src/app/api/funds/[id]/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@/lib/db'
import { getSession, unauthorized, notFound, badRequest, parseBody } from '@/lib/api-utils'
import { fundCreateSchema } from '@/lib/validations/fund'

export async function PATCH(
  req: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const session = await getSession()
  if (!session) return unauthorized()

  const { id } = await params
  const parsed = await parseBody(req, fundCreateSchema.partial())
  if (!parsed.success) {
    return badRequest(parsed.error.errors[0].message)
  }

  const fund = await prisma.fund.updateMany({
    where: { id, tenantId: session.user.tenantId },
    data: parsed.data,
  })

  if (fund.count === 0) return notFound('Fund')

  const updated = await prisma.fund.findUnique({ where: { id } })
  return NextResponse.json(updated)
}

export async function DELETE(
  req: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const session = await getSession()
  if (!session) return unauthorized()

  const { id } = await params
  const result = await prisma.fund.deleteMany({
    where: { id, tenantId: session.user.tenantId },
  })

  if (result.count === 0) return notFound('Fund')
  return NextResponse.json({ deleted: true })
}
```

**Step 4: Commit**

```bash
git add src/app/api/funds/ src/__tests__/api/funds.test.ts
git commit -m "feat: add funds CRUD API with Islamic giving categories"
```

---

### Task 9: Donations CRUD API

**Files:**
- Create: `src/app/api/donations/route.ts`
- Create: `src/app/api/donations/[id]/route.ts`

**Step 1: Write donations route**

```typescript
// src/app/api/donations/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@/lib/db'
import { getSession, unauthorized, badRequest, parseBody } from '@/lib/api-utils'
import { donationCreateSchema } from '@/lib/validations/donation'
import { toHijri } from '@/lib/hijri'

export async function GET(req: NextRequest) {
  const session = await getSession()
  if (!session) return unauthorized()

  const { searchParams } = new URL(req.url)
  const page = parseInt(searchParams.get('page') ?? '1')
  const limit = parseInt(searchParams.get('limit') ?? '50')
  const category = searchParams.get('category')
  const fundId = searchParams.get('fundId')
  const contactId = searchParams.get('contactId')

  const where = {
    tenantId: session.user.tenantId,
    ...(category && { givingCategory: category as any }),
    ...(fundId && { fundId }),
    ...(contactId && { contactId }),
  }

  const [donations, total] = await Promise.all([
    prisma.donation.findMany({
      where,
      skip: (page - 1) * limit,
      take: limit,
      orderBy: { createdAt: 'desc' },
      include: {
        contact: { select: { firstName: true, lastName: true, email: true } },
        fund: { select: { name: true, givingCategory: true } },
        campaign: { select: { name: true } },
      },
    }),
    prisma.donation.count({ where }),
  ])

  return NextResponse.json({ data: donations, total, page, limit })
}

export async function POST(req: NextRequest) {
  const session = await getSession()
  if (!session) return unauthorized()

  const parsed = await parseBody(req, donationCreateSchema)
  if (!parsed.success) {
    return badRequest(parsed.error.errors[0].message)
  }

  const now = new Date()
  const hijri = toHijri(now)
  const hijriDate = `${hijri.year}-${String(hijri.month).padStart(2, '0')}-${String(hijri.day).padStart(2, '0')}`

  const donation = await prisma.donation.create({
    data: {
      ...parsed.data,
      amount: parsed.data.amount,
      tenantId: session.user.tenantId,
      hijriDate,
    },
    include: {
      contact: { select: { firstName: true, lastName: true } },
      fund: { select: { name: true } },
    },
  })

  return NextResponse.json(donation, { status: 201 })
}
```

```typescript
// src/app/api/donations/[id]/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@/lib/db'
import { getSession, unauthorized, notFound } from '@/lib/api-utils'

export async function GET(
  req: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const session = await getSession()
  if (!session) return unauthorized()

  const { id } = await params
  const donation = await prisma.donation.findFirst({
    where: { id, tenantId: session.user.tenantId },
    include: {
      contact: true,
      fund: true,
      campaign: true,
    },
  })

  if (!donation) return notFound('Donation')
  return NextResponse.json(donation)
}

export async function DELETE(
  req: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const session = await getSession()
  if (!session) return unauthorized()

  const { id } = await params
  const result = await prisma.donation.deleteMany({
    where: { id, tenantId: session.user.tenantId },
  })

  if (result.count === 0) return notFound('Donation')
  return NextResponse.json({ deleted: true })
}
```

**Step 2: Commit**

```bash
git add src/app/api/donations/
git commit -m "feat: add donations CRUD API with Hijri date auto-tagging"
```

---

### Task 10: Campaigns CRUD API

**Files:**
- Create: `src/app/api/campaigns/route.ts`
- Create: `src/app/api/campaigns/[id]/route.ts`
- Create: `src/lib/validations/campaign.ts`

**Step 1: Write campaign validation**

```typescript
// src/lib/validations/campaign.ts
import { z } from 'zod'

export const campaignCreateSchema = z.object({
  name: z.string().min(1, 'Campaign name is required'),
  fundId: z.string().optional(),
  goalAmount: z.number().positive().optional(),
  startDate: z.string().datetime(),
  endDate: z.string().datetime().optional(),
})

export type CampaignCreate = z.infer<typeof campaignCreateSchema>
```

**Step 2: Write campaigns routes (same pattern as funds)**

```typescript
// src/app/api/campaigns/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@/lib/db'
import { getSession, unauthorized, badRequest, parseBody } from '@/lib/api-utils'
import { campaignCreateSchema } from '@/lib/validations/campaign'

export async function GET(req: NextRequest) {
  const session = await getSession()
  if (!session) return unauthorized()

  const campaigns = await prisma.campaign.findMany({
    where: { tenantId: session.user.tenantId },
    orderBy: { startDate: 'desc' },
    include: {
      _count: { select: { donations: true } },
    },
  })

  return NextResponse.json({ data: campaigns })
}

export async function POST(req: NextRequest) {
  const session = await getSession()
  if (!session) return unauthorized()

  const parsed = await parseBody(req, campaignCreateSchema)
  if (!parsed.success) {
    return badRequest(parsed.error.errors[0].message)
  }

  const campaign = await prisma.campaign.create({
    data: {
      ...parsed.data,
      startDate: new Date(parsed.data.startDate),
      endDate: parsed.data.endDate ? new Date(parsed.data.endDate) : null,
      tenantId: session.user.tenantId,
    },
  })

  return NextResponse.json(campaign, { status: 201 })
}
```

```typescript
// src/app/api/campaigns/[id]/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@/lib/db'
import { getSession, unauthorized, notFound } from '@/lib/api-utils'

export async function GET(
  req: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const session = await getSession()
  if (!session) return unauthorized()

  const { id } = await params
  const campaign = await prisma.campaign.findFirst({
    where: { id, tenantId: session.user.tenantId },
    include: {
      donations: {
        orderBy: { createdAt: 'desc' },
        take: 20,
        include: {
          contact: { select: { firstName: true, lastName: true } },
        },
      },
      _count: { select: { donations: true } },
    },
  })

  if (!campaign) return notFound('Campaign')
  return NextResponse.json(campaign)
}

export async function DELETE(
  req: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const session = await getSession()
  if (!session) return unauthorized()

  const { id } = await params
  const result = await prisma.campaign.deleteMany({
    where: { id, tenantId: session.user.tenantId },
  })

  if (result.count === 0) return notFound('Campaign')
  return NextResponse.json({ deleted: true })
}
```

**Step 3: Commit**

```bash
git add src/app/api/campaigns/ src/lib/validations/campaign.ts
git commit -m "feat: add campaigns CRUD API"
```

---

## Phase 3: UI Shell & Pages (Tasks 11–16)

### Task 11: App Shell — Layout, Sidebar, Navigation

**Files:**
- Create: `src/components/layout/AppShell.tsx`
- Create: `src/components/layout/Sidebar.tsx`
- Create: `src/components/layout/Header.tsx`
- Modify: `src/app/(dashboard)/layout.tsx`

**Step 1: Create dashboard route group layout**

```bash
mkdir -p src/app/\(dashboard\)
```

**Step 2: Write Sidebar component**

```tsx
// src/components/layout/Sidebar.tsx
'use client'

import Link from 'next/link'
import { usePathname } from 'next/navigation'

const navItems = [
  { href: '/dashboard', label: 'Dashboard', icon: 'LayoutDashboard' },
  { href: '/contacts', label: 'Contacts', icon: 'Users' },
  { href: '/donations', label: 'Donations', icon: 'Heart' },
  { href: '/funds', label: 'Funds', icon: 'Wallet' },
  { href: '/campaigns', label: 'Campaigns', icon: 'Target' },
  { href: '/gift-aid', label: 'Gift Aid', icon: 'FileCheck' },
  { href: '/reports', label: 'Reports', icon: 'BarChart3' },
  { href: '/settings', label: 'Settings', icon: 'Settings' },
]

export function Sidebar() {
  const pathname = usePathname()

  return (
    <aside className="w-64 border-r border-gray-200 bg-white min-h-screen p-4">
      <div className="mb-8">
        <h1 className="text-xl font-bold text-gray-900">Islamic CRM</h1>
        <p className="text-sm text-gray-500">Donor Management</p>
      </div>
      <nav className="space-y-1">
        {navItems.map((item) => {
          const isActive = pathname === item.href || pathname.startsWith(item.href + '/')
          return (
            <Link
              key={item.href}
              href={item.href}
              className={`block px-3 py-2 rounded-md text-sm font-medium transition-colors ${
                isActive
                  ? 'bg-emerald-50 text-emerald-700'
                  : 'text-gray-700 hover:bg-gray-50'
              }`}
            >
              {item.label}
            </Link>
          )
        })}
      </nav>
    </aside>
  )
}
```

**Step 3: Write Header component**

```tsx
// src/components/layout/Header.tsx
'use client'

import { useSession, signOut } from 'next-auth/react'
import { formatHijriDate } from '@/lib/hijri'

export function Header() {
  const { data: session } = useSession()
  const today = new Date()
  const hijriToday = formatHijriDate(today)

  return (
    <header className="h-16 border-b border-gray-200 bg-white px-6 flex items-center justify-between">
      <div>
        <p className="text-sm text-gray-500">{hijriToday}</p>
      </div>
      <div className="flex items-center gap-4">
        {session?.user && (
          <>
            <span className="text-sm text-gray-700">{session.user.name ?? session.user.email}</span>
            <button
              onClick={() => signOut()}
              className="text-sm text-gray-500 hover:text-gray-700"
            >
              Sign out
            </button>
          </>
        )}
      </div>
    </header>
  )
}
```

**Step 4: Write AppShell**

```tsx
// src/components/layout/AppShell.tsx
import { Sidebar } from './Sidebar'
import { Header } from './Header'

export function AppShell({ children }: { children: React.ReactNode }) {
  return (
    <div className="flex min-h-screen bg-gray-50">
      <Sidebar />
      <div className="flex-1 flex flex-col">
        <Header />
        <main className="flex-1 p-6">{children}</main>
      </div>
    </div>
  )
}
```

**Step 5: Create dashboard layout**

```tsx
// src/app/(dashboard)/layout.tsx
import { AppShell } from '@/components/layout/AppShell'
import { auth } from '@/lib/auth'
import { redirect } from 'next/navigation'

export default async function DashboardLayout({
  children,
}: {
  children: React.ReactNode
}) {
  const session = await auth()
  if (!session) redirect('/login')

  return <AppShell>{children}</AppShell>
}
```

**Step 6: Commit**

```bash
git add src/components/layout/ src/app/\(dashboard\)/
git commit -m "feat: add app shell with sidebar, header, Hijri date display"
```

---

### Task 12: Login Page

**Files:**
- Create: `src/app/login/page.tsx`
- Create: `src/components/auth/LoginForm.tsx`

**Step 1: Write LoginForm**

```tsx
// src/components/auth/LoginForm.tsx
'use client'

import { signIn } from 'next-auth/react'
import { useState } from 'react'
import { useRouter } from 'next/navigation'

export function LoginForm() {
  const [email, setEmail] = useState('')
  const [password, setPassword] = useState('')
  const [error, setError] = useState('')
  const [loading, setLoading] = useState(false)
  const router = useRouter()

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault()
    setError('')
    setLoading(true)

    const result = await signIn('credentials', {
      email,
      password,
      redirect: false,
    })

    setLoading(false)

    if (result?.error) {
      setError('Invalid email or password')
    } else {
      router.push('/dashboard')
    }
  }

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      {error && (
        <div className="p-3 text-sm text-red-700 bg-red-50 rounded-md">{error}</div>
      )}
      <div>
        <label htmlFor="email" className="block text-sm font-medium text-gray-700">
          Email
        </label>
        <input
          id="email"
          type="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          required
          className="mt-1 block w-full rounded-md border border-gray-300 px-3 py-2 text-sm focus:border-emerald-500 focus:outline-none focus:ring-1 focus:ring-emerald-500"
        />
      </div>
      <div>
        <label htmlFor="password" className="block text-sm font-medium text-gray-700">
          Password
        </label>
        <input
          id="password"
          type="password"
          value={password}
          onChange={(e) => setPassword(e.target.value)}
          required
          minLength={8}
          className="mt-1 block w-full rounded-md border border-gray-300 px-3 py-2 text-sm focus:border-emerald-500 focus:outline-none focus:ring-1 focus:ring-emerald-500"
        />
      </div>
      <button
        type="submit"
        disabled={loading}
        className="w-full rounded-md bg-emerald-600 px-4 py-2 text-sm font-medium text-white hover:bg-emerald-700 disabled:opacity-50"
      >
        {loading ? 'Signing in...' : 'Sign in'}
      </button>
    </form>
  )
}
```

**Step 2: Write login page**

```tsx
// src/app/login/page.tsx
import { LoginForm } from '@/components/auth/LoginForm'

export default function LoginPage() {
  return (
    <div className="min-h-screen flex items-center justify-center bg-gray-50">
      <div className="w-full max-w-md">
        <div className="text-center mb-8">
          <h1 className="text-2xl font-bold text-gray-900">Islamic CRM</h1>
          <p className="text-gray-500 mt-1">Sign in to your mosque dashboard</p>
        </div>
        <div className="bg-white rounded-lg shadow-sm border border-gray-200 p-6">
          <LoginForm />
        </div>
      </div>
    </div>
  )
}
```

**Step 3: Commit**

```bash
git add src/app/login/ src/components/auth/LoginForm.tsx
git commit -m "feat: add login page with credentials auth"
```

---

### Task 13: Dashboard Page with Giving Summary

**Files:**
- Create: `src/app/(dashboard)/dashboard/page.tsx`
- Create: `src/components/dashboard/GivingSummaryCards.tsx`
- Create: `src/components/dashboard/RecentDonations.tsx`
- Create: `src/app/api/dashboard/stats/route.ts`

**Step 1: Write dashboard stats API**

```typescript
// src/app/api/dashboard/stats/route.ts
import { NextResponse } from 'next/server'
import { prisma } from '@/lib/db'
import { getSession, unauthorized } from '@/lib/api-utils'
import { isRamadan } from '@/lib/hijri'

export async function GET() {
  const session = await getSession()
  if (!session) return unauthorized()

  const tenantId = session.user.tenantId
  const now = new Date()
  const thirtyDaysAgo = new Date(now.getTime() - 30 * 24 * 60 * 60 * 1000)

  const [totalDonors, totalDonations30d, byCategory, recentDonations] = await Promise.all([
    prisma.contact.count({ where: { tenantId } }),

    prisma.donation.aggregate({
      where: { tenantId, createdAt: { gte: thirtyDaysAgo } },
      _sum: { amount: true },
      _count: true,
    }),

    prisma.donation.groupBy({
      by: ['givingCategory'],
      where: { tenantId, createdAt: { gte: thirtyDaysAgo } },
      _sum: { amount: true },
      _count: true,
    }),

    prisma.donation.findMany({
      where: { tenantId },
      orderBy: { createdAt: 'desc' },
      take: 5,
      include: {
        contact: { select: { firstName: true, lastName: true } },
        fund: { select: { name: true } },
      },
    }),
  ])

  return NextResponse.json({
    totalDonors,
    last30Days: {
      total: totalDonations30d._sum.amount ?? 0,
      count: totalDonations30d._count,
    },
    byCategory,
    recentDonations,
    isRamadan: isRamadan(now),
  })
}
```

**Step 2: Write GivingSummaryCards**

```tsx
// src/components/dashboard/GivingSummaryCards.tsx
'use client'

interface Stats {
  totalDonors: number
  last30Days: { total: number; count: number }
  byCategory: Array<{ givingCategory: string; _sum: { amount: number }; _count: number }>
  isRamadan: boolean
}

export function GivingSummaryCards({ stats }: { stats: Stats }) {
  return (
    <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
      <Card
        label="Total Donors"
        value={stats.totalDonors.toString()}
      />
      <Card
        label="Last 30 Days"
        value={`£${Number(stats.last30Days.total).toLocaleString('en-GB', { minimumFractionDigits: 2 })}`}
        sub={`${stats.last30Days.count} donations`}
      />
      {stats.byCategory.slice(0, 2).map((cat) => (
        <Card
          key={cat.givingCategory}
          label={cat.givingCategory.replace('_', ' ')}
          value={`£${Number(cat._sum.amount).toLocaleString('en-GB', { minimumFractionDigits: 2 })}`}
          sub={`${cat._count} donations`}
        />
      ))}
    </div>
  )
}

function Card({ label, value, sub }: { label: string; value: string; sub?: string }) {
  return (
    <div className="bg-white rounded-lg border border-gray-200 p-4">
      <p className="text-sm text-gray-500">{label}</p>
      <p className="text-2xl font-semibold text-gray-900 mt-1">{value}</p>
      {sub && <p className="text-sm text-gray-400 mt-1">{sub}</p>}
    </div>
  )
}
```

**Step 3: Write RecentDonations**

```tsx
// src/components/dashboard/RecentDonations.tsx
'use client'

interface RecentDonation {
  id: string
  amount: string
  givingCategory: string
  createdAt: string
  contact: { firstName: string; lastName: string } | null
  fund: { name: string }
}

export function RecentDonations({ donations }: { donations: RecentDonation[] }) {
  return (
    <div className="bg-white rounded-lg border border-gray-200">
      <div className="px-4 py-3 border-b border-gray-200">
        <h3 className="text-sm font-medium text-gray-900">Recent Donations</h3>
      </div>
      <div className="divide-y divide-gray-100">
        {donations.map((d) => (
          <div key={d.id} className="px-4 py-3 flex items-center justify-between">
            <div>
              <p className="text-sm font-medium text-gray-900">
                {d.contact ? `${d.contact.firstName} ${d.contact.lastName}` : 'Anonymous'}
              </p>
              <p className="text-xs text-gray-500">{d.fund.name} — {d.givingCategory.replace('_', ' ')}</p>
            </div>
            <div className="text-right">
              <p className="text-sm font-semibold text-gray-900">
                £{Number(d.amount).toLocaleString('en-GB', { minimumFractionDigits: 2 })}
              </p>
              <p className="text-xs text-gray-400">
                {new Date(d.createdAt).toLocaleDateString('en-GB')}
              </p>
            </div>
          </div>
        ))}
        {donations.length === 0 && (
          <p className="px-4 py-6 text-sm text-gray-400 text-center">No donations yet</p>
        )}
      </div>
    </div>
  )
}
```

**Step 4: Write dashboard page (Server Component fetching data)**

```tsx
// src/app/(dashboard)/dashboard/page.tsx
import { auth } from '@/lib/auth'
import { prisma } from '@/lib/db'
import { isRamadan } from '@/lib/hijri'
import { GivingSummaryCards } from '@/components/dashboard/GivingSummaryCards'
import { RecentDonations } from '@/components/dashboard/RecentDonations'

export default async function DashboardPage() {
  const session = await auth()
  if (!session) return null

  const tenantId = session.user.tenantId
  const now = new Date()
  const thirtyDaysAgo = new Date(now.getTime() - 30 * 24 * 60 * 60 * 1000)

  const [totalDonors, totalDonations30d, byCategory, recentDonations] = await Promise.all([
    prisma.contact.count({ where: { tenantId } }),
    prisma.donation.aggregate({
      where: { tenantId, createdAt: { gte: thirtyDaysAgo } },
      _sum: { amount: true },
      _count: true,
    }),
    prisma.donation.groupBy({
      by: ['givingCategory'],
      where: { tenantId, createdAt: { gte: thirtyDaysAgo } },
      _sum: { amount: true },
      _count: true,
    }),
    prisma.donation.findMany({
      where: { tenantId },
      orderBy: { createdAt: 'desc' },
      take: 5,
      include: {
        contact: { select: { firstName: true, lastName: true } },
        fund: { select: { name: true } },
      },
    }),
  ])

  const stats = {
    totalDonors,
    last30Days: {
      total: Number(totalDonations30d._sum.amount ?? 0),
      count: totalDonations30d._count,
    },
    byCategory: byCategory.map((c) => ({
      ...c,
      _sum: { amount: Number(c._sum.amount ?? 0) },
    })),
    isRamadan: isRamadan(now),
  }

  const serializedDonations = recentDonations.map((d) => ({
    ...d,
    amount: d.amount.toString(),
    createdAt: d.createdAt.toISOString(),
  }))

  return (
    <div className="space-y-6">
      <div>
        <h2 className="text-lg font-semibold text-gray-900">Dashboard</h2>
        {stats.isRamadan && (
          <p className="text-sm text-emerald-600 font-medium mt-1">Ramadan Mubarak — giving season is active</p>
        )}
      </div>
      <GivingSummaryCards stats={stats} />
      <RecentDonations donations={serializedDonations} />
    </div>
  )
}
```

**Step 5: Commit**

```bash
git add src/app/\(dashboard\)/dashboard/ src/components/dashboard/ src/app/api/dashboard/
git commit -m "feat: add dashboard page with giving summary and recent donations"
```

---

### Task 14: Contacts List & Detail Pages

**Files:**
- Create: `src/app/(dashboard)/contacts/page.tsx`
- Create: `src/app/(dashboard)/contacts/[id]/page.tsx`
- Create: `src/app/(dashboard)/contacts/new/page.tsx`
- Create: `src/components/contacts/ContactsTable.tsx`
- Create: `src/components/contacts/ContactForm.tsx`

This task follows the same Server Component data-fetching pattern as Task 13. The contacts page fetches from Prisma directly in a Server Component, renders a table with search/pagination, links to detail pages showing donation history per contact. The form uses client-side validation with the Zod schema from Task 5, submits to the API from Task 7.

**Step 1:** Write `ContactsTable` — a client component receiving serialised contacts as props, with search input and pagination controls.

**Step 2:** Write `ContactForm` — a client component with controlled inputs matching `contactCreateSchema`, POST to `/api/contacts`.

**Step 3:** Write the list page as a Server Component fetching contacts with Prisma.

**Step 4:** Write the detail page showing contact info + their donation history.

**Step 5:** Write the new contact page wrapping `ContactForm`.

**Step 6: Commit**

```bash
git add src/app/\(dashboard\)/contacts/ src/components/contacts/
git commit -m "feat: add contacts list, detail, and create pages"
```

---

### Task 15: Donations List & Record Page

**Files:**
- Create: `src/app/(dashboard)/donations/page.tsx`
- Create: `src/app/(dashboard)/donations/new/page.tsx`
- Create: `src/components/donations/DonationsTable.tsx`
- Create: `src/components/donations/DonationForm.tsx`

Same pattern as Task 14. Key differences:

- `DonationsTable` shows: date (Gregorian + Hijri), donor name, amount, giving category (colour-coded badge), fund, Gift Aid status.
- `DonationForm` has a **giving category dropdown** (Zakat/Sadaqah/Lillah/etc.), fund selector, contact autocomplete, amount, payment method, Gift Aid toggle.
- Filter bar: filter by giving category, fund, date range.

**Step 1–5:** Same pattern as Task 14 adapted for donations.

**Step 6: Commit**

```bash
git add src/app/\(dashboard\)/donations/ src/components/donations/
git commit -m "feat: add donations list and recording pages with Islamic category selection"
```

---

### Task 16: Funds & Campaigns Pages

**Files:**
- Create: `src/app/(dashboard)/funds/page.tsx`
- Create: `src/app/(dashboard)/campaigns/page.tsx`
- Create: `src/app/(dashboard)/campaigns/[id]/page.tsx`

Funds page: simple list + create form. Each fund shows name, giving category, restricted status, total donated.

Campaigns page: list of campaigns with progress bars (raised / goal). Campaign detail shows donations within that campaign.

**Commit after completion.**

---

## Phase 4: Gift Aid & Integrations (Tasks 17–19)

### Task 17: Gift Aid Declaration Management

**Files:**
- Create: `src/app/(dashboard)/gift-aid/page.tsx`
- Create: `src/components/gift-aid/GiftAidBatchTable.tsx`
- Create: `src/app/api/gift-aid/batch/route.ts`
- Create: `src/lib/gift-aid.ts`

**Step 1: Write Gift Aid utility**

```typescript
// src/lib/gift-aid.ts
import { prisma } from '@/lib/db'

export async function getUnclaimedDonations(tenantId: string) {
  return prisma.donation.findMany({
    where: {
      tenantId,
      giftAidEligible: true,
      giftAidClaimed: false,
    },
    include: {
      contact: {
        include: {
          giftAidDeclarations: {
            where: { isActive: true },
          },
        },
      },
      fund: true,
    },
    orderBy: { createdAt: 'asc' },
  })
}

export function calculateGiftAidAmount(donationAmount: number): number {
  // UK Gift Aid: charity can claim 25% on top of donation
  return Math.round(donationAmount * 0.25 * 100) / 100
}
```

**Step 2:** Write batch API route that marks donations as claimed and generates a summary.

**Step 3:** Write the Gift Aid page showing unclaimed donations, total reclaimable amount, and a "Generate Claim" button.

**Step 4: Commit**

```bash
git add src/app/\(dashboard\)/gift-aid/ src/components/gift-aid/ src/app/api/gift-aid/ src/lib/gift-aid.ts
git commit -m "feat: add Gift Aid management with batch claiming"
```

---

### Task 18: Seed Script (Dev Data)

**Files:**
- Create: `prisma/seed.ts`
- Modify: `package.json` (add prisma seed config)

**Step 1: Write seed script**

```typescript
// prisma/seed.ts
import { PrismaClient } from '@prisma/client'
import bcrypt from 'bcryptjs'

const prisma = new PrismaClient()

async function main() {
  // Create tenant
  const tenant = await prisma.tenant.upsert({
    where: { slug: 'demo-mosque' },
    update: {},
    create: {
      name: 'Demo Mosque',
      slug: 'demo-mosque',
      charityNumber: '1234567',
    },
  })

  // Create admin user
  const passwordHash = await bcrypt.hash('password123', 10)
  await prisma.user.upsert({
    where: { tenantId_email: { tenantId: tenant.id, email: 'admin@demo-mosque.org' } },
    update: {},
    create: {
      tenantId: tenant.id,
      email: 'admin@demo-mosque.org',
      name: 'Mosque Admin',
      passwordHash,
      role: 'ADMIN',
    },
  })

  // Create funds
  const funds = await Promise.all([
    prisma.fund.upsert({
      where: { tenantId_name: { tenantId: tenant.id, name: 'Zakat Fund' } },
      update: {},
      create: { tenantId: tenant.id, name: 'Zakat Fund', givingCategory: 'ZAKAT', isRestricted: true },
    }),
    prisma.fund.upsert({
      where: { tenantId_name: { tenantId: tenant.id, name: 'Sadaqah Fund' } },
      update: {},
      create: { tenantId: tenant.id, name: 'Sadaqah Fund', givingCategory: 'SADAQAH', isRestricted: false },
    }),
    prisma.fund.upsert({
      where: { tenantId_name: { tenantId: tenant.id, name: 'Lillah - Operations' } },
      update: {},
      create: { tenantId: tenant.id, name: 'Lillah - Operations', givingCategory: 'LILLAH', isRestricted: false },
    }),
    prisma.fund.upsert({
      where: { tenantId_name: { tenantId: tenant.id, name: 'Masjid Building Fund' } },
      update: {},
      create: { tenantId: tenant.id, name: 'Masjid Building Fund', givingCategory: 'SADAQAH_JARIYAH', isRestricted: true },
    }),
  ])

  // Create sample contacts
  const contacts = await Promise.all(
    [
      { firstName: 'Ahmed', lastName: 'Khan', email: 'ahmed.khan@example.com', postcode: 'E1 6AN' },
      { firstName: 'Fatima', lastName: 'Ali', email: 'fatima.ali@example.com', postcode: 'E2 0RH' },
      { firstName: 'Yusuf', lastName: 'Rahman', email: 'yusuf.r@example.com', postcode: 'E1 1BB' },
      { firstName: 'Aisha', lastName: 'Begum', email: 'aisha.b@example.com', postcode: 'E3 4TA' },
      { firstName: 'Omar', lastName: 'Hassan', email: 'omar.h@example.com', postcode: 'E1 5HJ' },
    ].map((c) =>
      prisma.contact.create({
        data: { ...c, tenantId: tenant.id },
      })
    )
  )

  // Create sample donations
  const categories = ['ZAKAT', 'SADAQAH', 'LILLAH', 'SADAQAH_JARIYAH'] as const
  const methods = ['CARD_ONLINE', 'BANK_TRANSFER', 'CASH', 'DIRECT_DEBIT'] as const

  for (let i = 0; i < 30; i++) {
    const contact = contacts[Math.floor(Math.random() * contacts.length)]
    const fund = funds[Math.floor(Math.random() * funds.length)]
    const daysAgo = Math.floor(Math.random() * 90)
    const date = new Date(Date.now() - daysAgo * 24 * 60 * 60 * 1000)

    await prisma.donation.create({
      data: {
        tenantId: tenant.id,
        contactId: contact.id,
        fundId: fund.id,
        amount: Math.round((10 + Math.random() * 490) * 100) / 100,
        givingCategory: fund.givingCategory,
        method: methods[Math.floor(Math.random() * methods.length)],
        giftAidEligible: Math.random() > 0.3,
        createdAt: date,
      },
    })
  }

  console.log('Seed complete: tenant, user, funds, contacts, donations created.')
}

main()
  .catch(console.error)
  .finally(() => prisma.$disconnect())
```

**Step 2: Add seed config to package.json**

```json
"prisma": {
  "seed": "npx tsx prisma/seed.ts"
}
```

**Step 3: Install tsx and run seed**

```bash
npm install -D tsx
npx prisma db seed
```

Expected: Seed data created. Login with `admin@demo-mosque.org` / `password123`.

**Step 4: Commit**

```bash
git add prisma/seed.ts package.json
git commit -m "feat: add seed script with demo mosque, funds, contacts, donations"
```

---

### Task 19: CSV Import from Beacon

**Files:**
- Create: `src/app/api/import/csv/route.ts`
- Create: `src/lib/csv-import.ts`
- Create: `src/app/(dashboard)/settings/import/page.tsx`
- Create: `src/components/settings/CsvImportForm.tsx`
- Test: `src/__tests__/lib/csv-import.test.ts`

**Step 1: Write failing test**

```typescript
// src/__tests__/lib/csv-import.test.ts
import { describe, it, expect } from 'vitest'
import { parseBeaconContactsCsv } from '@/lib/csv-import'

describe('parseBeaconContactsCsv', () => {
  it('parses Beacon-format CSV into contacts', () => {
    const csv = `First Name,Last Name,Email,Phone,Address Line 1,City,Postcode
Ahmed,Khan,ahmed@example.com,07700900000,123 High Street,London,E1 6AN
Fatima,Ali,fatima@example.com,,456 Park Road,London,E2 0RH`

    const result = parseBeaconContactsCsv(csv)
    expect(result).toHaveLength(2)
    expect(result[0].firstName).toBe('Ahmed')
    expect(result[0].postcode).toBe('E1 6AN')
    expect(result[1].phone).toBeUndefined()
  })
})
```

**Step 2: Run test to verify it fails**

```bash
npx vitest run src/__tests__/lib/csv-import.test.ts
```

**Step 3: Install csv-parse and write parser**

```bash
npm install csv-parse
```

```typescript
// src/lib/csv-import.ts
import { parse } from 'csv-parse/sync'
import { ContactCreate, contactCreateSchema } from '@/lib/validations/contact'

interface BeaconRow {
  'First Name': string
  'Last Name': string
  'Email': string
  'Phone': string
  'Address Line 1': string
  'City': string
  'Postcode': string
  [key: string]: string
}

export function parseBeaconContactsCsv(csvString: string): ContactCreate[] {
  const records = parse(csvString, {
    columns: true,
    skip_empty_lines: true,
    trim: true,
  }) as BeaconRow[]

  return records
    .map((row) => {
      const parsed = contactCreateSchema.safeParse({
        firstName: row['First Name'],
        lastName: row['Last Name'],
        email: row['Email'] || undefined,
        phone: row['Phone'] || undefined,
        address1: row['Address Line 1'] || undefined,
        city: row['City'] || undefined,
        postcode: row['Postcode'] || undefined,
      })
      return parsed.success ? parsed.data : null
    })
    .filter((c): c is ContactCreate => c !== null)
}
```

**Step 4: Run test to verify it passes**

```bash
npx vitest run src/__tests__/lib/csv-import.test.ts
```

Expected: PASS.

**Step 5:** Write the API route that accepts CSV upload and bulk-creates contacts. Write the UI page with file upload.

**Step 6: Commit**

```bash
git add src/lib/csv-import.ts src/__tests__/lib/csv-import.test.ts src/app/api/import/ src/app/\(dashboard\)/settings/import/ src/components/settings/ package.json
git commit -m "feat: add CSV import for Beacon contact migration"
```

---

## Phase 5: Reports & Polish (Tasks 20–22)

### Task 20: Islamic Giving Reports

**Files:**
- Create: `src/app/(dashboard)/reports/page.tsx`
- Create: `src/app/api/reports/giving-by-category/route.ts`
- Create: `src/app/api/reports/ramadan-comparison/route.ts`
- Create: `src/components/reports/GivingByCategoryChart.tsx`
- Create: `src/components/reports/RamadanComparison.tsx`

**Step 1:** Install a charting library:
```bash
npm install recharts
```

**Step 2:** Write API route that aggregates donations by Islamic giving category for a date range, returning data suitable for a pie/bar chart.

**Step 3:** Write API route that compares Ramadan giving across years (current vs previous Hijri year).

**Step 4:** Write chart components using Recharts.

**Step 5:** Write reports page with category breakdown and Ramadan year-over-year comparison.

**Step 6: Commit**

```bash
git add src/app/\(dashboard\)/reports/ src/app/api/reports/ src/components/reports/ package.json
git commit -m "feat: add Islamic giving reports with category breakdown and Ramadan comparison"
```

---

### Task 21: Donor Segmentation API

**Files:**
- Create: `src/app/api/segments/route.ts`
- Create: `src/lib/segments.ts`
- Test: `src/__tests__/lib/segments.test.ts`

Pre-built segments specific to mosque giving:
1. "Zakat donors who gave last Ramadan but not this year"
2. "Monthly recurring donors"
3. "Lapsed donors (no gift in 6+ months)"
4. "Top 10% donors by lifetime value"
5. "Gift Aid eligible but no declaration on file"

**Step 1: Write failing test for segment query builder**

**Step 2: Implement segment functions using Prisma queries**

**Step 3: Write API route returning segment results**

**Step 4: Commit**

```bash
git add src/lib/segments.ts src/__tests__/lib/segments.test.ts src/app/api/segments/
git commit -m "feat: add pre-built donor segments for Islamic giving patterns"
```

---

### Task 22: Settings Page & Tenant Configuration

**Files:**
- Create: `src/app/(dashboard)/settings/page.tsx`
- Create: `src/app/api/settings/tenant/route.ts`
- Create: `src/components/settings/TenantSettingsForm.tsx`

Settings page allows:
- Edit mosque name, charity number, HMRC reference
- Manage users (invite by email, set role)
- Configure default funds
- View API keys (future)

**Commit after completion.**

---

## Phase 6: Stripe Connect & Payments (Tasks 23–24)

### Task 23: Stripe Connect Onboarding

**Files:**
- Create: `src/lib/stripe.ts`
- Create: `src/app/api/stripe/connect/route.ts`
- Create: `src/app/api/stripe/webhook/route.ts`

**Step 1: Install Stripe**

```bash
npm install stripe
```

**Step 2: Write Stripe client singleton**

```typescript
// src/lib/stripe.ts
import Stripe from 'stripe'

export const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2024-12-18.acacia',
})
```

**Step 3:** Write Connect onboarding route that creates an Express connected account for the mosque and returns the onboarding URL.

**Step 4:** Write webhook handler for `account.updated`, `payment_intent.succeeded` events. On successful payment, auto-create a Donation record.

**Step 5: Commit**

```bash
git add src/lib/stripe.ts src/app/api/stripe/
git commit -m "feat: add Stripe Connect onboarding and payment webhook"
```

---

### Task 24: Online Donation Page (Public)

**Files:**
- Create: `src/app/donate/[slug]/page.tsx`
- Create: `src/components/donate/DonationWidget.tsx`

A public-facing page at `/donate/[mosque-slug]` where donors can:
1. Select giving category (Zakat/Sadaqah/Lillah/etc.)
2. Enter amount
3. Opt in to Gift Aid (with declaration text)
4. Pay via Stripe Checkout

**Step 1:** Write the public donation page as a Server Component that looks up the tenant by slug.

**Step 2:** Write the DonationWidget client component with category selection, amount input, Gift Aid checkbox, and Stripe Checkout redirect.

**Step 3: Commit**

```bash
git add src/app/donate/ src/components/donate/
git commit -m "feat: add public donation page with Islamic category selection and Stripe Checkout"
```

---

## Summary: 24 Tasks across 6 Phases

| Phase | Tasks | What it delivers |
|-------|-------|-----------------|
| 1. Foundation | 1–6 | Scaffolding, schema, auth, validation, Hijri utils |
| 2. CRUD APIs | 7–10 | Contacts, funds, donations, campaigns APIs |
| 3. UI Shell | 11–16 | Login, dashboard, contacts/donations/funds pages |
| 4. Gift Aid & Import | 17–19 | Gift Aid management, seed data, Beacon CSV import |
| 5. Reports & Segments | 20–22 | Islamic giving reports, donor segments, settings |
| 6. Payments | 23–24 | Stripe Connect, public donation page |

**After Phase 6 you have a working MVP** that your mosque can pilot: contacts, donations tagged by Islamic category, Gift Aid tracking, Hijri calendar awareness, Ramadan reports, CSV import from Beacon, and online donations via Stripe.

**Deferred to post-MVP:**
- Xero accounting sync
- HMRC Gift Aid API submission (XML)
- GoCardless Direct Debit
- Email automation (Resend/SendGrid)
- Mobile-responsive refinements
- Playwright e2e tests
- Production Docker deployment
- Multi-tenant onboarding flow
