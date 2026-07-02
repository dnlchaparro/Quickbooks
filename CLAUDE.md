# CLAUDE.md

LedgerBook: a QuickBooks-style double-entry accounting app.

Three documents run this project — read the first two before writing any code:
- **`SPEC.md`** — WHAT to build. Source of truth; every product, stack, schema
  (Appendix A), API (Appendix B), and formula (Appendix C) decision is already
  made there. Do not re-open them.
- **`TASKS.md`** — WHAT'S NEXT. Task order, acceptance criteria, verify
  commands, current progress, decision log. Start every session by reading its
  "Current state" block.
- **`CLAUDE.md`** (this file) — HOW to behave.

## Operating rules (Karpathy's 4)

### 1. Think before coding
State your assumptions and plan before implementing. If the spec is ambiguous
or contradicts itself, surface the confusion and ask — don't guess silently.
If you know a materially better approach than the one specified, say so and
wait; don't just build it. Record every resolved assumption as a row in the
TASKS.md decision log so it survives the session.

### 2. Simplicity first
Build exactly what SPEC.md asks — no extra features, abstractions, config
options, or "future-proofing" beyond what's written. §1.2 lists explicit
non-goals; treat them as forbidden, not optional. Prefer the boring, direct
implementation.

### 3. Surgical changes
Touch only the files the task requires. Match the existing style, naming, and
patterns of neighboring code. No drive-by refactors, renames, dependency
bumps, or formatting churn mixed into feature work.

### 4. Goal-driven execution
Every task in TASKS.md carries its own acceptance criteria and verify
command — those, plus `pnpm lint && pnpm typecheck && pnpm test` (and
`pnpm test:e2e` for user-facing flows), define done. Loop autonomously —
implement, run the checks, fix, re-run — until green. Don't stop at
"should work"; prove it, then check the box and update TASKS.md's
"Current state" in the same commit.

## Project invariants (never violate)

- Money is integer cents (`bigint` in Postgres). Never floats. All formatting
  through `src/lib/money.ts`; rounding is half-up, per line.
- Every journal entry balances: `sum(debits) == sum(credits)`. All postings go
  through `postEntry()` — never insert journal rows directly.
- Journal entries are immutable: edits and voids are reverse + repost, never
  UPDATE/DELETE.
- Every query is scoped by `companyId` from tRPC context — never from client
  input.
- Multi-write operations run in one DB transaction; mutations write an audit
  log row in that same transaction.

## Commands

```bash
pnpm dev           # start app (needs `docker compose up -d` for Postgres)
pnpm db:migrate    # apply drizzle migrations
pnpm db:seed       # demo company with 6 months of data
pnpm lint && pnpm typecheck
pnpm test          # vitest unit/service tests
pnpm test:e2e      # playwright
```

## Workflow

- Work TASKS.md top to bottom; don't start milestone M(n+1) before M(n)'s
  exit criteria are met and its boxes are checked.
- Ledger/report logic must be pure functions with unit tests (SPEC.md §8)
  before any UI is wired to it.
- The database schema is Appendix A of SPEC.md, transcribed — do not invent
  columns, names, or relationships; the API surface is Appendix B likewise.
- Commit per coherent unit of work with descriptive messages; TASKS.md
  bookkeeping (checkbox, current state, decision log) rides in the same
  commit as the work it describes.
