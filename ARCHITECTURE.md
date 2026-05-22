# Architecture

## Two processes, one repo

```
┌─────────────────────────────┐         ┌──────────────────────────────────┐
│   Next.js 16 web (apps/web) │         │   Trading engine (apps/engine)   │
│                             │   SSE   │                                  │
│  - shadcn dashboard         │ ◄────── │  - WS to Tradovate (1 + N)       │
│  - control API (POST)       │ ──────► │  - risk engine                   │
│  - reads engine state via   │  HTTP   │  - order router                  │
│    Postgres + SSE bridge    │         │  - reconciliation loop           │
└─────────────────────────────┘         └──────────────────────────────────┘
              │                                       │
              └───────────────┬───────────────────────┘
                              ▼
                  ┌────────────────────────┐
                  │   Postgres (Neon)       │
                  │  - accounts             │
                  │  - mappings             │
                  │  - fills, orders        │
                  │  - risk_events          │
                  │  - audit_log            │
                  └────────────────────────┘
```

### Why two processes (and not just Next.js Server Actions)

Next.js serverless functions cannot hold persistent WebSocket sessions to Tradovate. Order routing must complete in <150 ms with no cold start, and the risk engine must own in-memory position state to enforce rules synchronously. A long-running Node/TS process is the right shape; Next.js is the wrong shape.

So:
- **engine** = always-on Node/TS process (deployed to Chicago-region VPS, runs under systemd or PM2)
- **web** = Next.js 16 on Vercel for the dashboard. Reads Postgres directly for history; subscribes to a local SSE endpoint on the engine for live state.

## Process responsibilities

### `apps/engine`
- Opens one Tradovate WS session per configured account (primary + linked)
- Maintains in-memory `Book` per account: open positions, working orders, today's realized P&L
- On primary fill → builds linked-account orders → runs risk engine → submits in parallel
- Persists every order + fill + risk event to Postgres for audit
- Exposes `/sse/state` (live), `/api/control/{kill,pause,resume,config}` (commands), `/healthz`
- Reconnect loop with state reconciliation on restart

### `apps/web`
- Next.js 16 App Router, Server Components by default
- shadcn/ui + Tailwind, light/dark theme tokens
- Pages: `/dashboard` (live), `/accounts`, `/mappings`, `/risk-rules`, `/journal`, `/events`
- Auth: simple email-magic-link via Resend (single-user for v0; multi-user later)
- All control actions go through Server Actions → engine HTTP

## Data flow: a single propagated trade

```
Primary fills 2 ES @ 4523.50
      │
      ▼
  [engine] WS fill event received  (t = 0 ms)
      │
      ▼
  resolve linked accounts active for this primary + symbol
      │
      ▼
  for each linked account:
     map symbol (ES → MES if configured)         ← 0.1 ms
     compute size (multiplier / fixed / %eq)     ← 0.1 ms
     risk gate (DD, max contracts, news, kill)   ← 0.5 ms
     submit order via Tradovate REST             ← network ~30–80 ms
      │
      ▼
  fills come back on linked-account WS           ← ~50–120 ms after submit
      │
      ▼
  persist + emit SSE event to dashboard
```

Target: primary fill → all linked-account submissions in flight within **20 ms** of the primary fill event. End-to-end fill confirmation: **<150 ms** on Chicago VPS.

## Risk engine

Three layers, evaluated in order:

1. **Kill switch** — if active, reject all orders immediately
2. **Account-level limits** — daily DD, trailing DD, max open contracts, flat-all time
3. **Order-level limits** — max contracts per order, symbol allow-list, news window

Every block is persisted to `risk_events` with the rule that fired and the order that was blocked.

The risk engine runs **inline** in the engine process — never as a separate microservice. A network hop here can cost the trade.

## Storage

- **Postgres (Neon)** — accounts, mappings, risk rules, orders, fills, events, audit log
- **Credential storage** — Tradovate API tokens are envelope-encrypted with a per-deploy KMS key. Plaintext never lands on disk. Decrypted only in engine memory.
- **No Redis in v0.** If we need cross-process pub/sub later (e.g., separate web and engine hosts), Postgres `LISTEN/NOTIFY` first; Redis only if measured load demands it.

## Deployment

| Component | Host | Why |
|---|---|---|
| `apps/engine` | Chicago-region VPS (Vultr CHI or OVH Chicago) | Co-location with CME → lowest latency. systemd or PM2 supervisor. |
| `apps/web` | Vercel | Standard Next.js deploy. Connects to same Neon Postgres. |
| `Postgres` | Neon (US-East or Chicago region) | Managed, branch-able for dev. |

## Open questions (resolve before Phase 1 starts)

- Does Tradovate REST + WS satisfy <150 ms for both order submit and fill notification? Need to benchmark from Chicago VPS before locking design.
- Tradovate API rate limits across N linked accounts — confirm whether per-account or per-app.
- Tradovate OAuth flow for multi-account auth: one user with N accounts vs N user logins?
