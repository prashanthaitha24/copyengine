# copyengine

A multi-account futures trading dashboard for Tradovate, with a real-time risk engine, position oversight across accounts, and a unified control panel.

> Phase 1: Tradovate API only. Additional broker APIs come in later phases.

## Why

Managing several Tradovate accounts from the native UI means juggling browser tabs, missing risk signals, and reconciling P&L by hand. This dashboard targets:

- **<150 ms event-to-action loop** on a Chicago-region VPS, co-located with CME
- **Hard risk enforcement** before any order leaves the box (daily DD, trailing DD, max contracts, news pause, kill switch)
- **A UI you actually want to look at** — not a 2008 WinForms grid

## Features (target)

### Account oversight
- One primary account + N linked accounts under a single dashboard
- Per-account configuration: position sizing rules, symbol substitutions (`ES ↔ MES`), order-type preferences
- Order types covered: market, limit, stop, stop-limit, OCO brackets
- Stop loss + trailing stop management with per-account overrides
- Position reconciliation on reconnect
- Per-account latency monitor and execution diagnostics

### Risk engine (fires before any order leaves the process)
- Daily drawdown ceiling
- Trailing drawdown ceiling
- Max open contracts (per account, per symbol)
- Max contracts per order
- News-window pause (configurable economic event list)
- Per-account kill switch (panic button + auto-trigger on DD breach)
- Flat-all-at-time-of-day enforcement

### UI
- Multi-account dashboard: live P&L, open positions, today's fills
- Per-account on/off toggle
- Real-time event log (orders sent, fills, errors, risk blocks)
- Per-day trade journal across accounts
- Encrypted credential vault (no plaintext API keys on disk)
- Light + dark theme

## Architecture (one-liner)

A long-running Node/TS **engine** process holds WebSocket sessions to Tradovate for every configured account, runs the risk engine inline, and persists state to Postgres. A Next.js 16 **web** app reads live state via Server-Sent Events and posts control commands (kill, pause, config) back to the engine.

Full design: [ARCHITECTURE.md](./ARCHITECTURE.md).

## Status

**Pre-alpha.** Nothing here trades real money yet. See [Plans.md](./Plans.md) for the Phase 1 build plan.

## License

MIT — see [LICENSE](./LICENSE).
