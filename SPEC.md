# SPEC.md — "LedgerBook": a QuickBooks-style small-business accounting app

This document is the single source of truth for building the product. Every
decision below is **already made** — do not re-open decisions, do not add
alternatives, do not "improve" the scope. If something is genuinely missing or
contradictory, stop and ask the user; otherwise implement exactly what is
written here.

Target reader: a coding agent starting from an empty repository.

---

## 1. Product summary

LedgerBook is a web-based double-entry accounting application for small
businesses (1–20 employees), functionally modeled on QuickBooks Online. Core
value: track money in (invoices/payments), money out (bills/expenses), keep a
correct double-entry general ledger underneath everything, reconcile against
the bank, and produce the standard financial reports.

### 1.1 In scope (v1)

1. Multi-tenant companies with role-based users
2. Chart of Accounts
3. Customers, Vendors, Products & Services
4. Invoices (A/R) with PDF rendering and payment recording
5. Estimates (convertible to invoices)
6. Bills (A/P) and bill payments
7. Expenses (direct money-out transactions)
8. Manual journal entries
9. Bank accounts, CSV transaction import, transaction matching, reconciliation
10. Sales tax (flat-rate tax codes, tax liability report)
11. Reports: Profit & Loss, Balance Sheet, Cash Flow (indirect), Trial
    Balance, General Ledger, A/R Aging, A/P Aging, Sales Tax Liability
12. Dashboard (cash balance, P&L snapshot, overdue invoices, unpaid bills)
13. Audit log of all mutations

### 1.2 Explicitly out of scope (v1 — do not build)

- Payroll
- Inventory quantity tracking / COGS automation (products have prices only)
- Multi-currency (schema stores a currency code, UI is single-currency USD)
- Live bank feeds (Plaid etc.) — CSV import only, behind an interface
- Online payment collection (Stripe) — payments are recorded manually
- Recurring transactions, budgets, projects/classes/locations, time tracking
- Accountant tools (closing the books, adjusting periods) beyond a simple
  "books closed through DATE" lock
- Mobile apps; the web UI must be responsive but that's it

---

## 2. Tech stack (fixed decisions)

| Concern | Decision |
|---|---|
| Language | TypeScript, `strict: true` everywhere |
| Framework | Next.js 15, App Router, single app (no separate API service) |
| API layer | tRPC v11 (all mutations/queries go through tRPC routers) |
| Database | PostgreSQL 16 |
| ORM | Drizzle ORM + drizzle-kit migrations |
| Validation | Zod (shared schemas between tRPC and forms) |
| Auth | Auth.js (NextAuth v5), Credentials (email+password, bcrypt) + Google OAuth |
| UI | Tailwind CSS v4 + shadcn/ui components |
| Tables/forms | TanStack Table, React Hook Form + Zod resolver |
| PDF | `@react-pdf/renderer` for invoice/estimate PDFs, generated server-side |
| Email | Resend SDK; in dev, log to console via a `Mailer` interface |
| Money math | Integer **cents** (`bigint` in Postgres, `number` in TS — see §4.1) |
| Dates | `date` columns for business dates, `timestamptz` for audit timestamps; UI uses the company's timezone |
| IDs | UUIDv7 primary keys, generated in the app |
| Package manager | pnpm |
| Unit tests | Vitest |
| E2E tests | Playwright (Chromium only) |
| Lint/format | ESLint + Prettier, default configs, no bikeshedding |
| Local infra | `docker-compose.yml` with Postgres 16 only |
| Deployment target | Any Node host; provide `Dockerfile`. No vendor lock-in features. |

Repository layout (single Next.js app, not a monorepo):

```
/src
  /app                # Next.js routes (route groups: (auth), (app))
  /server
    /db               # drizzle schema, migrations, seed
    /ledger           # posting engine (pure functions + service)
    /services         # invoice, bill, payment, reconciliation, reports...
    /trpc             # routers, context, middleware
    /mail             # Mailer interface + Resend/console impls
  /components         # shared UI
  /lib                # utils (money, dates, invoice numbering)
/tests
  /unit
  /e2e
```

---

## 3. Multi-tenancy, auth, roles

### 3.1 Tenancy model

- Tenant = **Company**. Every business row carries `company_id`.
- A User can belong to many Companies via `company_members`
  (`user_id, company_id, role`).
- The active company is stored in the session; a company switcher lives in
  the top nav.
- **Every** tRPC procedure (except auth/onboarding) runs through a middleware
  that (a) requires a session, (b) resolves active company, (c) verifies
  membership, (d) injects `{ companyId, role }` into context. Queries must
  always filter by `companyId` from context — never trust a client-sent
  company id.

### 3.2 Roles and permissions

Four roles, enforced in tRPC middleware per-procedure:

| Role | Can |
|---|---|
| `owner` | Everything, incl. company settings, members, delete company |
| `admin` | Everything except member management and company deletion |
| `accountant` | All accounting features; no company settings/members |
| `viewer` | Read-only: all lists and reports, no mutations |

No custom/granular permissions in v1.

### 3.3 Onboarding

Sign up → create user → "Create your company" form (name, industry free-text,
fiscal year start month, timezone, base currency fixed to USD) → seed the
default Chart of Accounts (§5.2) → land on dashboard.

---

## 4. The ledger core (most important section)

Everything financial in the app is a projection of an immutable double-entry
ledger. Build this first, test it hardest.

### 4.1 Money

- All amounts are integer cents. Postgres `bigint`, TS `number` (safe: max
  business amounts are far below 2^53). Never use floats for money.
- A single `formatMoney(cents, currency)` and `parseMoney(input)` in
  `/src/lib/money.ts`. All UI goes through these.
- Rounding rule everywhere: **half-up** (`Math.round` on non-negative values;
  implement a `roundHalfUp` that also handles negatives correctly).
- Tax and discount rounding: computed **per invoice line**, then summed.

### 4.2 Accounts (Chart of Accounts)

`accounts` table:

- `id, company_id, name, code (nullable, unique per company when set)`
- `type`: enum `asset | liability | equity | income | expense`
- `subtype`: enum, drives report grouping:
  - asset: `bank | accounts_receivable | current_asset | fixed_asset`
  - liability: `accounts_payable | credit_card | current_liability | long_term_liability | sales_tax_payable`
  - equity: `equity | retained_earnings | owner_equity`
  - income: `income | other_income`
  - expense: `expense | other_expense`
- `is_system boolean` — system accounts (A/R, A/P, Sales Tax Payable,
  Retained Earnings, Undeposited Funds, Opening Balance Equity) cannot be
  deleted or re-typed.
- `is_archived boolean` — archive instead of delete once an account has
  postings.

Normal balance: assets/expenses are debit-normal; liabilities/equity/income
are credit-normal. Account balance = `sum(debits) - sum(credits)`, displayed
sign-flipped for credit-normal accounts.

### 4.3 Journal

Two tables:

`journal_entries`:
- `id, company_id, entry_date (date), memo, source_type, source_id,
  created_by, created_at, reverses_entry_id (nullable)`
- `source_type`: enum `invoice | invoice_payment | bill | bill_payment |
  expense | manual | opening_balance | reconciliation_adjustment`
- `source_id`: id of the source document (nullable for `manual`).

`journal_lines`:
- `id, entry_id, account_id, debit bigint, credit bigint, description,
  customer_id (nullable), vendor_id (nullable)`
- Constraint per line: exactly one of debit/credit is > 0, the other is 0.
- Constraint per entry (enforced in the posting service AND by a deferred
  DB trigger): `sum(debit) = sum(credit)` and at least 2 lines.

**Immutability rule:** journal entries are never updated or deleted. Editing
a source document = post a full reversing entry (`reverses_entry_id` set) +
post a new entry. Voiding = reversing entry only. This keeps the audit trail
airtight and makes reports trivially consistent.

### 4.4 Posting engine

`/src/server/ledger/post.ts` exposes one function used by every service:

```ts
postEntry(tx: DbTransaction, input: {
  companyId, entryDate, memo, sourceType, sourceId?,
  lines: Array<{ accountId, debit?, credit?, description?, customerId?, vendorId? }>
}): Promise<JournalEntry>
```

It validates balance, validates accounts belong to the company and aren't
archived, checks the books-closed lock (§4.6), writes entry + lines, and
writes an audit log row — all inside the caller's DB transaction.

There is **no** stored `balance` column anywhere. Balances are always
`SUM` over journal_lines (with indexes on `(account_id)`, and on
`journal_entries (company_id, entry_date)`). If reports become slow later,
that is a later problem — do not add caching/materialized balances in v1.

### 4.5 Posting recipes (exact debits/credits)

| Event | Debit | Credit |
|---|---|---|
| Invoice issued | Accounts Receivable (total incl. tax) | Income account per line (net) + Sales Tax Payable (tax) |
| Invoice payment received | Bank account (or Undeposited Funds) | Accounts Receivable |
| Bill received | Expense account per line (net) + Sales Tax Payable if tax on purchases | Accounts Payable |
| Bill payment | Accounts Payable | Bank account |
| Expense (direct) | Expense account per line | Bank/Credit-card account |
| Customer overpayment | Bank | A/R (creates customer credit = negative A/R balance for that customer) |
| Opening balance on bank account | Bank | Opening Balance Equity |
| Reconciliation adjustment | Bank or Reconciliation Discrepancies (expense) | the other one |

Accrual basis only in v1. Cash-basis reports are out of scope.

### 4.6 Books-closed lock

`companies.books_closed_through date NULL`. `postEntry` rejects any entry with
`entry_date <= books_closed_through` (error: "Period is closed"). Only
owner/admin can change the lock date, from company settings.

---

## 5. Feature specs

### 5.1 Company settings

Name, legal name, address, phone, email, logo upload (stored as bytea, max
1 MB, png/jpg), fiscal year start month (default January), timezone, invoice
numbering prefix (default `INV-`), estimate prefix (`EST-`), default payment
terms (enum: `due_on_receipt | net_15 | net_30 | net_60`, default net_30),
books-closed-through date. Only owner/admin can edit.

### 5.2 Default Chart of Accounts (seeded per company)

Seed exactly these accounts on company creation (code, name, type/subtype):

- 1000 Checking — asset/bank
- 1050 Undeposited Funds — asset/current_asset, system
- 1100 Accounts Receivable — asset/accounts_receivable, system
- 1500 Fixed Assets — asset/fixed_asset
- 2000 Accounts Payable — liability/accounts_payable, system
- 2100 Credit Card — liability/credit_card
- 2200 Sales Tax Payable — liability/sales_tax_payable, system
- 3000 Owner's Equity — equity/owner_equity
- 3100 Opening Balance Equity — equity/equity, system
- 3900 Retained Earnings — equity/retained_earnings, system
- 4000 Sales — income/income
- 4100 Service Revenue — income/income
- 5000 Cost of Goods Sold — expense/expense
- 6000–6900 common expenses: Advertising, Bank Fees, Insurance, Legal &
  Professional, Meals, Office Supplies, Rent, Software, Travel, Utilities,
  Payroll (plain expense), Reconciliation Discrepancies (system)

Users can add/edit/archive accounts (CRUD screen with type/subtype pickers);
system accounts are locked as described in §4.2.

### 5.3 Contacts

**Customers**: name (required), company name, email, phone, billing address,
notes, archived flag. **Vendors**: same shape. Simple CRUD, searchable list,
detail page showing their transactions and open balance (computed from A/R
or A/P lines tagged with the contact).

### 5.4 Products & Services

`items`: name, type (`service | product`), description, unit price (cents),
income account (default 4000 Sales), tax code (nullable → non-taxable),
archived flag. Used to prefill invoice/estimate lines; lines remain freely
editable after insertion (no link-back required beyond `item_id` for
reporting).

### 5.5 Sales tax

`tax_codes`: name (e.g. "CA Sales Tax"), rate in basis points (e.g. 725 =
7.25%), agency name, archived flag. One tax code per invoice line (nullable).
Tax per line = `roundHalfUp(net * rate / 10000)`. All collected tax credits
the single system account 2200 Sales Tax Payable. The Sales Tax Liability
report groups collected tax by tax code over a date range. "Recording a tax
payment" is just an Expense posted against 2200. No filing workflows.

### 5.6 Estimates

Statuses: `draft → sent → accepted | declined` (+ `converted` once turned
into an invoice). Same line-item editor as invoices. **No journal postings
ever** — estimates are non-posting documents. "Convert to invoice" copies all
lines into a new draft invoice and marks the estimate converted. PDF + email
sending like invoices.

### 5.7 Invoices

- Fields: customer (required), issue date (default today), due date (derived
  from terms, editable), invoice number (auto: prefix + zero-padded sequence
  per company, e.g. `INV-000042`; editable but unique per company), memo,
  lines.
- Line: item (optional), description, quantity (3 decimal places max, stored
  as integer thousandths), unit price (cents), tax code. Line net =
  `roundHalfUp(qty_thousandths * unit_price / 1000)`.
- Statuses: `draft → open → paid` with `partial` (open + some payment),
  `overdue` derived (open/partial + past due date, computed at read time, not
  stored), `voided`.
- **Draft** invoices have no journal entry. Moving to **open** ("Approve" or
  "Send") posts the §4.5 invoice entry. Editing an open invoice reverses and
  reposts. Voiding reverses; voiding is blocked if payments are applied
  (unapply first). Deleting is allowed only for drafts.
- Send = generate PDF, email to customer with PDF attached (Resend), mark
  sent. PDF also downloadable.
- Payments: "Receive payment" flow — pick customer, see open invoices, enter
  payment date/amount/deposit-to account (bank or Undeposited Funds), apply
  across invoices (auto-fill oldest-first, user can adjust). One payment can
  cover many invoices (`payments` + `payment_applications` tables). Posts per
  §4.5. Unapplying/deleting a payment reverses its entry.

### 5.8 Bills & bill payments

Mirror image of invoices with vendors and A/P: bill number (vendor's ref,
free text), issue/due dates, lines hitting **expense accounts** directly
(account picker per line, not items), optional tax code per line. Statuses:
`open → paid` (+ `partial`, derived `overdue`). Bills post on creation (no
draft state). "Pay bills" flow mirrors "Receive payment" (pick vendor, apply
payment from a bank account across open bills).

### 5.9 Expenses

One-step money-out: payee (vendor optional), payment account (bank or credit
card), date, lines (expense account, description, amount, tax code optional),
memo. Posts immediately per §4.5. Edit = reverse + repost. This is also how
sales-tax payments and owner draws are recorded (pick the right accounts).

### 5.10 Manual journal entries

Full editor: date, memo, N lines with account / debit / credit / description /
optional customer-vendor tag. Client-side and server-side balance check.
Accountant+ roles only. Edit = reverse + repost (the UI shows entry history).

### 5.11 Banking: import, matching, reconciliation

`bank_transactions` (staging area, NOT ledger data):
- `id, company_id, account_id (must be bank/credit_card subtype),
  txn_date, amount bigint (signed: positive = money in), description,
  import_batch_id, status: unmatched | matched | excluded,
  matched_entry_id (nullable), fingerprint`

**CSV import**: user uploads CSV, maps columns (date, description, amount —
or debit/credit pair) with a preview, picks date format from
(`MM/DD/YYYY | DD/MM/YYYY | YYYY-MM-DD`). Duplicate protection via
`fingerprint = sha256(accountId|date|amount|normalized description)`;
duplicates are skipped and reported. Behind a `BankFeedSource` interface so a
Plaid implementation can slot in later — but do not build Plaid.

**Matching screen** (per bank account): list unmatched transactions, for each
offer:
1. *Match* — suggested existing ledger candidates: journal lines on that bank
   account within ±7 days and exact amount; user confirms → status matched.
2. *Categorize* — create an Expense (money out) or a simple deposit entry
   (money in: debit bank, credit chosen income/other account) directly from
   the row, then auto-match to it.
3. *Exclude* — mark excluded (e.g. transfers already recorded).

**Reconciliation**: pick account, statement end date, statement ending
balance (cents). Screen lists all ledger lines on the account up to that date
that are not yet reconciled; user ticks them; running "cleared balance" shown;
finish enabled only when difference = 0, or user posts a reconciliation
adjustment (§4.5) for the residual. Completing writes a `reconciliations` row
and stamps `reconciled_reconciliation_id` on the ticked journal lines.
Reconciled lines' entries cannot be reversed without a warning + confirmation.

### 5.12 Reports

All reports: date-range picker (presets: this month, last month, this
quarter, this year, last year, custom; default = current fiscal year to
date), on-screen HTML table, CSV export button. Company timezone applies.
Comparisons ("vs previous period") are out of scope.

1. **Profit & Loss** — income accounts (grouped income/other_income), minus
   expenses (expense/other_expense), sections COGS separated, Net Income line.
   Rows = accounts with nonzero activity; click-through to General Ledger
   filtered to that account+range.
2. **Balance Sheet** — as of a single date. Assets / Liabilities / Equity by
   subtype. Equity includes computed Retained Earnings = all-time net income
   through the report date minus what's already in 3900, plus current-year
   Net Income as its own line. Must always balance; render a red error banner
   if it doesn't (that's a bug).
3. **Cash Flow (indirect)** — Net income, adjust by deltas in non-cash
   balance-sheet accounts over the range, grouped Operating (A/R, A/P, tax
   payable, other current), Investing (fixed assets), Financing (long-term
   liabilities, equity). Ending cash must equal sum of bank+credit-card
   account balances delta; assert and show error banner if not.
4. **Trial Balance** — every account, debit/credit columns, totals equal.
5. **General Ledger** — per-account listing of journal lines with running
   balance; filter by account and date range.
6. **A/R Aging** / **A/P Aging** — open invoice/bill balances per
   customer/vendor bucketed: Current, 1–30, 31–60, 61–90, 90+ (by days past
   due at report date).
7. **Sales Tax Liability** — per tax code: taxable sales, tax collected,
   payments applied to 2200, balance owed, over a range.

Report engine: one shared query helper that returns
`accountId → { debits, credits }` for a date range, and one for "as of"
balances. Reports are pure functions over those maps — unit-test them
heavily with a seeded fixture company.

### 5.13 Dashboard

Cards: total cash (sum of bank subtypes), P&L this-month snapshot (income,
expenses, net), overdue invoices (count + total, link to filtered list),
unpaid bills (count + total), recent activity (last 10 audit events). No
charts in v1 beyond a simple 6-month income-vs-expense bar chart (Recharts).

### 5.14 Audit log

`audit_logs`: `id, company_id, user_id, action (string, e.g. invoice.created),
entity_type, entity_id, payload jsonb (diff or snapshot), created_at`.
Written by every mutating service in the same transaction. Read-only screen
for owner/admin with filters by user/entity/date.

---

## 6. Data model — table inventory

All tables have `id uuid pk`, `created_at`, `updated_at`; business tables have
`company_id` FK + index. Inventory (columns beyond §4/§5 are obvious):

`users`, `sessions/accounts` (Auth.js tables), `companies`,
`company_members`, `accounts`, `journal_entries`, `journal_lines`,
`customers`, `vendors`, `items`, `tax_codes`,
`estimates`, `estimate_lines`,
`invoices`, `invoice_lines`, `payments`, `payment_applications`,
`bills`, `bill_lines`, `bill_payments`, `bill_payment_applications`,
`expenses`, `expense_lines`,
`bank_transactions`, `import_batches`, `reconciliations`,
`invoice_counters` (per-company sequence row, incremented with
`SELECT ... FOR UPDATE`), `audit_logs`.

Document tables store denormalized totals (`subtotal, tax_total, total,
amount_paid` on invoices/bills) — these are **document** state for lists/UX;
the ledger remains the accounting truth, and a consistency unit test asserts
document totals equal their journal postings.

Deletion policy: hard-delete only drafts and unposted staging data; everything
else is void/archive.

---

## 7. UX map (routes)

```
/(auth)/signin, /signup
/(app)/dashboard
/(app)/sales/invoices, /invoices/new, /invoices/[id], /invoices/[id]/edit
/(app)/sales/estimates (same pattern)
/(app)/sales/payments/new        # receive payment
/(app)/sales/customers, /customers/[id]
/(app)/purchases/bills (same pattern), /purchases/expenses, /purchases/vendors
/(app)/banking/[accountId]       # matching screen
/(app)/banking/[accountId]/import
/(app)/banking/[accountId]/reconcile
/(app)/accounting/chart-of-accounts
/(app)/accounting/journal-entries, /journal-entries/new
/(app)/reports, /reports/[reportKey]
/(app)/settings/{company,members,taxes,items,audit-log}
```

Left sidebar nav: Dashboard, Sales, Purchases, Banking, Accounting, Reports,
Settings. Global "+ New" button (invoice/expense/bill/journal entry/etc.).
Keep UI plain shadcn/ui defaults — no custom design system work.

---

## 8. Quality bar & testing

1. **Ledger invariants (unit, Vitest)** — the non-negotiables:
   - every posted entry balances; unbalanced input throws
   - reversal produces exact mirror; net effect zero
   - trial balance always sums to zero for any seeded scenario
   - balance sheet equation holds after every scripted operation sequence
   - money/rounding functions: exhaustive edge cases (negatives, thousandth
     quantities, 1-cent rounding)
2. **Service tests** — invoice lifecycle (draft→open→partial→paid→void),
   bill lifecycle, payment over/under-application rejection, books-closed
   rejection, tenant isolation (company A cannot read/post company B data —
   test at the tRPC layer).
3. **E2E (Playwright)** — one happy path: sign up → onboard → create
   customer → create+approve invoice → receive payment → import bank CSV →
   match payment → run P&L and Balance Sheet and assert on-screen numbers.
4. **Seed script** — `pnpm db:seed` creates a demo company with ~6 months of
   realistic data (30 invoices, 20 bills, 40 expenses, bank CSV fixture) so
   every report has content. E2E and manual QA use it.
5. CI: GitHub Actions — lint, typecheck, unit tests, e2e against dockerized
   Postgres. All green before any PR merges.

Definition of done for any feature: implemented per this spec, covered by the
tests named above, seed data exercises it, and the relevant report reflects it
correctly.

---

## 9. Build order (milestones)

Build strictly in this order; each milestone ends with green tests.

- **M0 Scaffold**: Next.js + tRPC + Drizzle + Auth.js + docker-compose,
  CI, empty shell with nav.
- **M1 Tenancy**: signup, company onboarding, members/roles, middleware,
  settings page, audit log plumbing.
- **M2 Ledger core**: chart of accounts (+ seed), posting engine,
  manual journal entries, trial balance + general ledger reports.
- **M3 Sales**: customers, items, tax codes, invoices (full lifecycle),
  estimates, receive payment, PDF + email, A/R aging.
- **M4 Purchases**: vendors, bills, bill payments, expenses, A/P aging.
- **M5 Banking**: CSV import, matching, reconciliation.
- **M6 Reporting & polish**: P&L, balance sheet, cash flow, sales tax
  liability, dashboard, seed script, full E2E, README.

---

## 10. Error handling & conventions

- All tRPC errors are typed `TRPCError` with user-safe messages; the UI shows
  them in toasts. Never leak SQL or stack traces to the client.
- Every multi-write operation runs in a single Postgres transaction.
- Concurrency: invoice numbering via row lock (§6); posting engine relies on
  transaction isolation `read committed` + explicit locks where noted; no
  optimistic-locking columns in v1.
- Timezone: store business dates as plain `date`; "today" is computed in the
  company timezone.
- Accessibility: use shadcn/ui semantics, label every form field; no further
  a11y work in v1.
- Security: bcrypt (cost 12), session cookies httpOnly/secure, CSRF handled
  by Auth.js, all uploads size/type-checked, rate-limit auth endpoints
  (simple in-memory limiter is fine for v1).
