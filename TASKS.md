# TASKS.md — build plan, progress state, and decision log

This file is the working memory of the build. `SPEC.md` says *what* to
build; `CLAUDE.md` says *how to behave*; this file says *what's done, what's
next, and how each task is proven done*.

## Rules for using this file

1. Work tasks **in order** within a milestone; finish a milestone (all boxes
   checked, its Exit criteria met) before starting the next.
2. When you complete a task: check its box, run its Verify command, and
   update the **Current state** block below — in the same commit as the work.
3. When you make ANY decision not already written in SPEC.md (or discover an
   ambiguity you resolved), append one row to the **Decision log**. Never
   edit or delete existing rows.
4. A task is done only when its Acceptance criteria are true and its Verify
   command passes. "Should work" doesn't count.
5. Do not add, remove, or reorder tasks without the user's approval — but DO
   split a task into sub-checkboxes underneath it if that helps tracking.

---

## Current state

- **Active milestone**: M0 (not started)
- **Last completed task**: — (repository contains only spec documents)
- **Notes for next session**: No code exists yet. Start at M0.1.

---

## Decision log (append-only)

| Date | Decision | Rationale |
|---|---|---|
| 2026-07-02 | All product/stack decisions consolidated into SPEC.md v2 (appendices A–D added: full schema, API contract, state machines, deterministic seed) | Make the spec implementable by a smaller model with zero design decisions left open |
| 2026-07-02 | Discounts are per-line, percentage (bps), invoices+estimates only; reduce income directly, no contra account | §5.7.2 / §1.2 |
| 2026-07-02 | Overpayments allowed on both A/R and A/P; credits are subledger-only (no extra journal entries) | §5.7.4 / §4.5 |

---

## M0 — Scaffold

- [ ] **M0.1 Project init**: Next.js 15 (App Router) + TypeScript strict +
  Tailwind v4 + shadcn/ui + ESLint/Prettier + pnpm scripts (`dev`, `build`,
  `lint`, `typecheck`, `test`, `test:e2e`, `db:migrate`, `db:seed`).
  *Acceptance*: `pnpm dev` serves a placeholder page; `pnpm lint` and
  `pnpm typecheck` pass on a clean tree.
  *Verify*: `pnpm lint && pnpm typecheck && pnpm build`
- [ ] **M0.2 Database plumbing**: `docker-compose.yml` (Postgres 16),
  Drizzle + drizzle-kit configured, empty initial migration applies,
  `src/server/db/` skeleton.
  *Acceptance*: `docker compose up -d && pnpm db:migrate` succeeds from
  scratch. *Verify*: fresh-container migrate run
- [ ] **M0.3 tRPC wiring**: tRPC v11 app router + React client + a
  `health.ping` procedure; error formatter that never leaks internals.
  *Acceptance*: page renders the ping response.
  *Verify*: `pnpm test` (one integration test hitting ping)
- [ ] **M0.4 Auth.js skeleton**: NextAuth v5 with database sessions,
  Credentials + Google providers configured (Google behind env vars),
  signin/signup pages render, `(auth)`/`(app)` route groups + redirect
  middleware. *Acceptance*: unauthenticated visit to `/dashboard`
  redirects to `/signin`. *Verify*: Playwright smoke test
- [ ] **M0.5 App shell**: sidebar nav (all §7 sections, links stubbed),
  top bar with "+ New" menu (stub) and account menu.
  *Acceptance*: shell renders on `/dashboard` for a signed-in user.
- [ ] **M0.6 CI**: GitHub Actions workflow — lint, typecheck, unit tests,
  e2e with a Postgres service container. Dockerfile for the app.
  *Acceptance*: workflow green on push.

**Exit criteria**: all above; CI green; `README.md` has run instructions.

## M1 — Tenancy & auth

- [ ] **M1.1 Schema: auth + tenancy tables** (Appendix A.2 exactly:
  users, oauth_accounts, sessions, verification_tokens, password_resets,
  companies, company_members, invitations, document_counters) + migration.
  *Verify*: `pnpm db:migrate` from scratch; drizzle schema typechecks
- [ ] **M1.2 Signup + email verification**: `auth.signup`, hashed tokens,
  `Mailer` interface (console impl), `/verify-email`, resend +
  rate limits per §3.4, non-blocking banner.
  *Acceptance*: full flow works with the console mailer (token printed).
  *Verify*: service tests for token expiry/single-use/hashing
- [ ] **M1.3 Password reset**: request + reset pages, session invalidation
  on reset, no-enumeration response. *Verify*: service tests incl. reused
  token rejection
- [ ] **M1.4 Company onboarding**: `company.create` seeding the §5.2
  default CoA + both counters; switcher (`listMine`/`switchActive`);
  active company in session. *Acceptance*: new user lands on onboarding,
  creates company, sees dashboard; CoA page shows all §5.2 accounts.
  *Verify*: service test asserts exact seeded account set
- [ ] **M1.5 tRPC tenant middleware**: session → membership → `{companyId,
  role}` context; role-gate helper (`viewer<accountant<admin<owner`).
  *Acceptance*: procedures reject non-members with NOT_FOUND and
  under-role with FORBIDDEN. *Verify*: tenant-isolation tests (two
  companies, cross access blocked)
- [ ] **M1.6 Members & invitations**: settings/members page, invite/revoke/
  resend/updateRole/remove, `/accept-invitation` + prefilled signup,
  invite email. *Verify*: service tests: expiry, email match, single
  pending invite per email, role guards (owner-only)
- [ ] **M1.7 Company settings page** incl. logo upload (1 MB, png/jpg),
  books-closed date field (plumbing only — enforcement lands in M2),
  company delete with name confirmation. *Verify*: upload rejection tests
- [ ] **M1.8 Audit log plumbing**: `audit_logs` table, `writeAudit(tx, ...)`
  helper, wired into every M1 mutation; settings/audit-log screen with
  §7.1 filters. *Verify*: audit rows asserted in M1 service tests

**Exit criteria**: two users in two companies cannot see each other's data
(tested); all M1 flows audit-logged; CI green.

## M2 — Ledger core

- [ ] **M2.1 Schema: accounts + journal** (Appendix A.3, incl. the deferred
  balance trigger and all CHECKs). *Verify*: migration applies; direct
  unbalanced INSERT fails at the DB level
- [ ] **M2.2 Money & date libs**: `src/lib/money.ts` (`roundHalfUp` in
  integer math, `formatMoney`, `parseMoney`, line-math functions of
  §5.7.2/C.3) and `src/lib/dates.ts` (company-tz today, fiscalYearStart
  per C.4, terms → due date). *Verify*: exhaustive unit tests (§8.1
  cases) — this is the highest-value test file in the repo
- [ ] **M2.3 Posting engine**: `postEntry` + `reverseEntry` per §4.4 with
  books-closed enforcement (§4.6) incl. the closed-period reversal-date
  rule. *Verify*: §8.1 invariant tests (balance, mirror reversal,
  archived-account rejection, closed-period rejection + shifted reversal)
- [ ] **M2.4 Chart of Accounts UI + router** (`coa.*` per Appendix B):
  list with balances, create/edit/archive/delete, system-account locks,
  subtype-belongs-to-type validation. *Verify*: service tests for each
  lock rule
- [ ] **M2.5 Manual journal entries**: editor with live balance indicator,
  A/R-A/P tag rule (§5.10), list/detail with reversal chain,
  `journal.create/editManual/reverse`. *Verify*: service tests incl. tag
  rule and edit-as-reverse-and-repost
- [ ] **M2.6 Reports foundation + Trial Balance + General Ledger**: the
  two shared query helpers (§5.12), report page scaffold with fiscal
  date-range picker + CSV export, both reports per §5.12.4/.5.
  *Verify*: unit tests on helper maps; trial balance sums equal after a
  scripted posting sequence

**Exit criteria**: can post arbitrary balanced entries via UI, trial
balance balances, general ledger running balances correct; CI green.

## M3 — Sales

- [ ] **M3.1 Schema: contacts, items, tax codes, sales docs** (A.4, A.5).
- [ ] **M3.2 Customers + vendors CRUD** (§5.3, §7.1 lists; vendor screens
  live under Purchases nav but ship now — same component).
  *Verify*: archive-vs-delete rules tested
- [ ] **M3.3 Items + tax codes** (§5.4, §5.5) incl. settings pages.
  *Verify*: item archive doesn't touch documents (test)
- [ ] **M3.4 Invoice create/edit (draft)**: line editor (item prefill, qty/
  price/discount/tax per line, live totals via §5.7.2 math), auto
  numbering via counters, draft save/delete.
  *Verify*: line-math unit tests already exist (M2.2); service test:
  server ignores client totals; concurrency test on numbering
- [ ] **M3.5 Invoice lifecycle**: approve/void/edit-open per C.1 (reverse+
  repost), derived partial/overdue in lists and detail.
  *Verify*: full state-machine service test incl. every ✗ transition
- [ ] **M3.6 Receive payment + credits**: §5.7.4 flow in full (multi-invoice
  application, oldest-first autofill, overpayment→credit, later credit
  application, $0-payment credit application, edit/delete semantics).
  *Verify*: service tests for each §8.2 payment case
- [ ] **M3.7 Estimates**: full lifecycle per C.1 incl. conversion; own
  counter. *Verify*: conversion copies lines verbatim (test)
- [ ] **M3.8 PDF + email send**: §5.7.5 PDF for invoices+estimates,
  §5.15 templates, send flows stamping sent_at.
  *Verify*: PDF snapshot test (render → assert text content), mailer
  interface test
- [ ] **M3.9 A/R aging report** (C.6). *Verify*: unit test with crafted
  due dates spanning all buckets + a credit row

**Exit criteria**: E2E slice — create customer → discounted invoice →
approve → partial payment → overpay → credit visible → apply credit; A/R
aging and trial balance agree with expectations; CI green.

## M4 — Purchases

- [ ] **M4.1 Schema: purchase docs** (A.6).
- [ ] **M4.2 Bills** (§5.8): create(posts)/edit/void, amount-based lines
  with account picker rule, derived statuses. *Verify*: state-machine
  tests mirror M3.5
- [ ] **M4.3 Pay bills + vendor credits**: mirror of M3.6.
  *Verify*: mirrored service tests incl. overpayment
- [ ] **M4.4 Expenses** (§5.9): create/edit/void, tax lines, works as
  sales-tax-payment and owner-draw vehicle. *Verify*: posting recipe tests
- [ ] **M4.5 A/P aging report**. *Verify*: bucket unit tests

**Exit criteria**: purchases mirror sales end-to-end; balance sheet
(manual check via trial balance until M6) consistent; CI green.

## M5 — Banking

- [ ] **M5.1 Schema: banking tables** (A.7).
- [ ] **M5.2 CSV import wizard**: 3 steps per §5.11, `BankFeedSource`
  interface, fingerprint dedup, error-row reporting.
  *Verify*: unit tests: both mapping modes, all 3 date formats, dup skip,
  unparseable rows surfaced
- [ ] **M5.3 Matching screen**: tabs, candidate finder (±7 days, exact
  amount, unclaimed), match/unmatch/exclude/restore per C.2.
  *Verify*: candidate-query unit tests (boundary dates, sign rules)
- [ ] **M5.4 Categorize**: expense (money-out) and bank_deposit (money-in)
  creation + automatch. *Verify*: posting recipe tests
- [ ] **M5.5 Reconciliation**: state query + atomic completion + adjustment
  path + reconciled-warning on reverse/edit/delete of stamped entries +
  history list. *Verify*: atomicity test (failure mid-completion stamps
  nothing); cleared-balance math test; adjustment posting test

**Exit criteria**: import → match → reconcile a month against fixture CSVs
by hand in dev; CI green.

## M6 — Reporting & polish

- [ ] **M6.1 P&L** (§5.12.1) with GL click-through.
- [ ] **M6.2 Balance Sheet** (§5.12.2, C.4) incl. balanced assertion
  banner. *Verify*: equity-rows unit tests across fiscal-year starts
  (Jan and Jul companies)
- [ ] **M6.3 Cash Flow** (C.5) incl. self-check. *Verify*: identity test
  on fixture data
- [ ] **M6.4 Sales Tax Liability** (§5.12.7) + record-payment shortcut.
- [ ] **M6.5 Dashboard** (§5.13) incl. Recharts bar chart.
- [ ] **M6.6 Seed script** (Appendix D, exactly): fixture file, bank CSVs,
  pinned "today", January reconciliation, < 30 s, idempotent.
  *Verify*: seed twice in a row succeeds; fixture-vs-ledger cross-check
  tests pass
- [ ] **M6.7 Report correctness suite**: every report unit-tested against
  Appendix D expectations derived from fixture objects.
- [ ] **M6.8 Full E2E happy path** (§8.3) on a fresh database.
- [ ] **M6.9 README**: setup, env vars (incl. Resend/Google — both
  optional in dev), architecture overview (1 page), links to SPEC/TASKS.

**Exit criteria**: all §8 tests green in CI; seeded balance sheet balances;
cash-flow self-check passes; E2E green. **v1 done.**
