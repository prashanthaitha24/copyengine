# tradovate-copier

A low-latency trade copier for Tradovate-backed prop firm accounts, with a built-in risk engine and a real-time multi-account dashboard.

> Phase 1 scope: **Tradovate → Tradovate only**. NinjaTrader and Rithmic destinations land in later phases.

## Why

Most off-the-shelf copiers are either Windows-only, locked to NinjaTrader, or slow enough that ES/NQ slippage eats the edge. This one targets:

- **<150 ms master → follower fill latency** on the same Chicago-region VPS
- **Hard risk enforcement** before any order leaves the box (daily DD, trailing DD, max contracts, news pause, kill switch)
- **A UI you actually want to look at** — not a 2008 WinForms grid

## Features (target)

### Copier core
- One **master** account, N **follower** accounts
- Per-follower sizing: fixed lots, lot multiplier, or `% equity` Kelly-capped
- Symbol mapping: `ES → MES`, `NQ → MNQ`, configurable per follower
- Order types covered: market, limit, stop, stop-limit, OCO brackets
- Stop loss + trailing stop attached at master (mirrored to followers) or attached locally per follower
- Position reconciliation on reconnect / replay of missed fills
- Per-follower latency monitor + fill comparison view

### Risk engine (fires before order leaves the process)
- Daily drawdown ceiling
- Trailing drawdown ceiling (prop firm rules)
- Max open contracts (per account, per symbol)
- Max contracts per order
- News-window pause (configurable economic event list)
- Per-account kill switch (panic button + auto-trigger on DD breach)
- Flat-all-at-time-of-day enforcement

### UI
- Multi-account dashboard: live P&L, open positions, today's fills
- Per-account on/off toggle
- Real-time event log (orders sent, fills, errors, risk blocks)
- Trade journal with fill-by-fill master vs follower diff
- Encrypted credential vault (no plaintext API keys on disk)
- Light + dark theme

## Architecture (one-liner)

A long-running Node/TS **engine** process holds WebSocket sessions to Tradovate for master and every follower, runs the risk engine inline, and persists state to Postgres. A Next.js 16 **web** app reads live state via Server-Sent Events and posts control commands (kill, pause, config) back to the engine over a local API.

Full design: [`ARCHITECTURE.md`](./ARCHITECTURE.md).

## Status

**Pre-alpha.** Nothing here trades real money yet. See [`Plans.md`](./Plans.md) for the Phase 1 build plan.

## ⚠️ Disclaimers

- **Read your prop firm's Terms of Service before using this.** Copying among your own funded accounts is generally allowed at Apex / TradeDay / MyFundedFutures. FTMO, Topstep, and others have rules — verify first. Using this in violation of a prop firm's ToS can void your account.
- This software is provided **as-is**. Futures trading carries substantial risk of loss. The risk engine is a defense in depth, not a guarantee.
- The author is not responsible for blown accounts, lost funding, or violations of any third-party agreement.

## License

MIT — see [LICENSE](./LICENSE).
