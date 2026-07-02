# CLAUDE.md

LedgerBook: a QuickBooks-style double-entry accounting app.
**Read `SPEC.md` before writing any code.** It is the source of truth; all
product and stack decisions are already made there. Do not re-open them.

## Operating rules (Karpathy's 4)

### 1. Think before coding
State your assumptions and plan before implementing. If the spec is ambiguous
or contradicts itself, surface the confusion and ask — don't guess silently.
If you know a materially better approach than the one specified, say so and
wait; don't just build it.

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
Every task's success criteria: the feature behaves per SPEC.md, and
`pnpm lint && pnpm typecheck && pnpm test` pass (plus `pnpm test:e2e` for
user-facing flows). Loop autonomously — implement, run the checks, fix,
re-run — until green. Don't stop at "should work"; prove it.

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

- Follow the milestone order in SPEC.md §9; don't start M(n+1) before M(n)'s
  tests are green.
- Ledger/report logic must be pure functions with unit tests (SPEC.md §8)
  before any UI is wired to it.
- Commit per coherent unit of work with descriptive messages.
