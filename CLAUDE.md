# CLAUDE.md — tradovate-copier conventions

## What this project is

A low-latency Tradovate trade copier with a built-in risk engine and a Next.js dashboard. See `README.md`, `ARCHITECTURE.md`, `Plans.md`.

## Repo layout

```
apps/
  web/      Next.js 16 dashboard (App Router, shadcn, Tailwind)
  engine/   Long-running Node/TS process: WS sessions + risk engine + router
packages/
  shared/   Shared TS types: Order, Fill, RiskRule, Account, Mapping, etc.
infra/
  db/       Prisma schema + migrations
docs/
  prop-firm-tos.md  ToS notes per supported prop firm
```

(Subfolders created on demand — repo starts with docs only.)

## Build principles

- **Risk engine before copy logic.** Never wire a copier path without the risk gate already in place.
- **Latency is a feature.** Any change that adds >5 ms to the master-fill → submit hot path needs justification in the PR.
- **No plaintext credentials on disk, ever.** All Tradovate API tokens are envelope-encrypted with libsodium. Decrypted in engine memory only.
- **Persist everything for audit.** Orders, fills, risk blocks, kill events, config changes — all in Postgres with timestamps.
- **PR-for-every-change.** No commits straight to `main`. Even a typo fix goes through a PR.
- **Demo-grade UI bar.** shadcn components, light + dark theme, real data (no Lorem placeholders). React Flow only if/when we need a graph view of master→follower routing.

## Stack rules

- Next.js 16 App Router, Server Components default, Server Actions for mutations
- TypeScript strict mode everywhere; no `any` without a `// reason:` comment
- Prisma for DB schema + queries
- shadcn/ui — token-driven, no hex literals in `.tsx`
- Engine uses Fastify (or native http) — keep dependencies minimal in the hot path
- Test runner: Vitest

## Risk engine rules (non-negotiable)

1. Kill switch evaluated first, always
2. Every block writes to `risk_events`
3. No risk rule may make a network call — all evaluation is in-memory
4. Unit tests cover both pass and block paths for every rule

## What "done" means

A task is not done until:
- Code is committed via PR
- Tests pass locally
- For copier-path changes: tested end-to-end against Tradovate demo
- For UI changes: verified in browser, light + dark, no console errors

## What we don't do

- No mocking the Tradovate API in integration tests — use their demo environment
- No "I'll add the risk check later" PRs — risk gate ships with the path
- No untested order routing changes on a Friday
