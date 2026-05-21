# Plans.md

> Phase 1: **Tradovate → Tradovate copier, single-user, 1 master + up to 5 followers, with risk engine and live dashboard.**
> Target: 4–6 weeks of focused work in build-for-me mode. "Done" = paper-traded for 5 sessions without an order routing failure or risk-engine miss.

## Acceptance criteria (Phase 1)

A trade fired on the master Tradovate demo account is:
1. Mirrored to all active followers with correct symbol mapping and sizing
2. Blocked correctly by every risk rule (verified by unit + integration tests)
3. Visible in the dashboard within 1 s of the fill, with master vs follower diff
4. Reconciled correctly on engine restart (no duplicated, no missed orders)
5. Killable instantly via the dashboard panic button (all positions flat within 2 s)

End-to-end latency target: **<150 ms** master fill → follower fill confirmation on a Chicago-region VPS.

---

## Milestones

### M0 — Foundations (week 1)
- [ ] **T01** Scaffold Next.js 16 (App Router, TS strict) in `apps/web` with shadcn + Tailwind, light/dark theme tokens
- [ ] **T02** Scaffold `apps/engine` (Node/TS, tsx for dev, esbuild for prod) with HTTP server (Fastify or native) and SSE endpoint stub
- [ ] **T03** Add Postgres via Neon, Prisma schema for `accounts`, `mappings`, `risk_rules`, `orders`, `fills`, `risk_events`, `audit_log`
- [ ] **T04** Encrypted credential vault: KMS-style envelope encryption (libsodium sealed box, key in env var for v0)
- [ ] **T05** Local dev: docker-compose or pnpm scripts to run web + engine + postgres together

### M1 — Tradovate connectivity (week 2)
- [ ] **T06** Tradovate OAuth flow, refresh-token handling, store encrypted per-account in DB
- [ ] **T07** WS session manager: one persistent session per account, auto-reconnect with exponential backoff, heartbeat monitor
- [ ] **T08** Parse Tradovate events into typed `MarketEvent` / `OrderEvent` / `FillEvent`
- [ ] **T09** REST order submit wrapper (market, limit, stop, OCO bracket)
- [ ] **T10** Latency telemetry: every order tagged with t0 (master fill), t1 (submit sent), t2 (follower fill) → persisted

### M2 — Risk engine (week 3)
- [ ] **T11** In-memory `Book` per account (positions, working orders, realized P&L today)
- [ ] **T12** Risk rules engine with rule registry (kill, daily DD, trailing DD, max open, max per-order, symbol allowlist, news window, flat-all-at-time)
- [ ] **T13** Each rule has unit tests covering pass + block, edge cases (exactly at limit, crossing limit mid-bar)
- [ ] **T14** Risk evaluation order: kill → account → order. Persist every block to `risk_events`.
- [ ] **T15** Kill switch: HTTP endpoint, in-memory flag, also persisted so it survives restart

### M3 — Copy logic (week 4)
- [ ] **T16** Master fill subscriber → fan-out to followers
- [ ] **T17** Per-follower sizing: fixed lots, multiplier, %-equity (with half-Kelly cap of 2%)
- [ ] **T18** Symbol mapping resolver (ES → MES, NQ → MNQ, etc.) per follower
- [ ] **T19** Bracket order replication: SL + TP attached at master mirror to followers
- [ ] **T20** Trailing stop logic (server-side on master if Tradovate doesn't support; mirrored)
- [ ] **T21** Position reconciliation on engine restart: read open positions from Tradovate, replay/diff against DB

### M4 — Dashboard (week 5)
- [ ] **T22** `/dashboard` — live multi-account grid: P&L today, open positions, working orders, status pill
- [ ] **T23** Per-account on/off toggle (writes to engine via Server Action → control API)
- [ ] **T24** Panic button (red, confirms with type-to-confirm). Wired to engine kill endpoint.
- [ ] **T25** Live event feed (SSE) — orders, fills, risk blocks, errors
- [ ] **T26** `/journal` — per-day fill list with master vs follower diff (price slippage, fill time delta, lot delta)
- [ ] **T27** `/accounts` `/mappings` `/risk-rules` CRUD pages
- [ ] **T28** Auth: email magic link (Resend), single-user for v0

### M5 — Hardening + paper test (week 6)
- [ ] **T29** End-to-end test harness: scripted Tradovate demo orders → assert mirror + journal output
- [ ] **T30** Latency budget verification: 95th percentile fill→submit <20 ms on Chicago VPS
- [ ] **T31** Chaos tests: WS disconnect mid-fill, DB unavailable, follower account rejected order
- [ ] **T32** Deploy: engine to Chicago VPS, web to Vercel, Neon Postgres
- [ ] **T33** Paper-trade 5 full RTH sessions with master on real demo + 2 followers. Zero routing failures, zero risk-engine misses required to declare Phase 1 done.

---

## Risks + mitigations

| Risk | Mitigation |
|---|---|
| Tradovate API rate limits across N followers | Benchmark in M1; if hit, parallelize with bounded concurrency + backoff |
| Latency >150 ms | If exceeded, profile in M5; engine→Tradovate proximity is the lever, not code |
| Prop firm ToS forbids copying | Document ToS check per supported prop firm in `/docs/prop-firm-tos.md` before Phase 1 done. Block usage flag if ToS unclear. |
| Lost fills on reconnect | T21 is the gate; if reconciliation isn't bulletproof, Phase 1 is not done |
| Risk engine bug → blown account | T13 unit tests + 5 paper sessions in T33 before any live capital |

## Out of scope for Phase 1

- NinjaTrader source (Phase 2)
- Rithmic destinations (Phase 3)
- Multi-tenant / multi-user (later)
- Mobile/tablet view (later)
- Trade Copier marketplace listing (way later)
- News API integration — Phase 1 ships with manual news-window config; auto-pause on economic events is Phase 2

## Phase 2+ teaser

- Phase 2: NT8 source via Windows bridge + auto news pause + per-follower custom risk overlays
- Phase 3: Rithmic destinations (gated on R|API license)
- Phase 4: Multi-tenant SaaS if we decide to sell it
