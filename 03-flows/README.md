# Level 4 — Cross-cutting flows

Sequence diagrams and behavioral flows for things that span multiple containers or components. Where component diagrams at Levels 2 and 3 answer *"what exists and how it connects,"* these answer *"what happens over time."*

## Flows

- [Order lifecycle](order-lifecycle.md) — flagship flow. Full path from strategy `submit()` through in-engine RiskManager, OrderManager allocation, RouterExchange, TCP to order_router, BinanceExchange, Binance, and the reverse path for ack + fill including PositionManager and Strategy `on_fill`. Covers happy path plus pre-trade reject, venue reject, partial fills, and late-fill-after-reconnect branches. Includes a rough timing-budget table for the hot path.
- [Tick flow](tick-flow.md) — one tick's path from Binance WS through market_feed, across the shared-memory ring, into a trading engine, and onto the strategy's `on_tick_impl`. Complements the order lifecycle. Covers sequence-gap on the market_feed side, slow-subscriber ring overwrite, multi-engine subscription independence, and the trade-tick reorder buffer branch.
- [Cancel-on-ACK race handling](cancel-on-ack.md) — the wire-layer fix inside `BinanceExchange` for the race where an engine cancels a still-in-flight order. Cancel is held until ACK arrives, then flushed in the same tick. Includes the naive-implementation illustration showing the orphan-order failure mode, the state machine inside BinanceExchange, edge cases, and rationale for placing the fix at the wire layer rather than in the engine.
- [KillSwitch trip cascade](killswitch-cascade.md) — the full sequence from trip source detection through deferred callback execution, cancel_all + halt, engine shutdown, ProcessManager alerting, operator-gated restart, and position recovery. Ties together most of the safety-critical flows in the system. Covers the deferred-trip rationale, the six stages, coordination with other flows, and internal failure modes.
- [Position recovery after restart](position-recovery.md) — how a fresh engine reconstructs its `PositionManager` and `OrderManager` from QuestDB fill history + venue open-order reconciliation, including the guarantees and non-guarantees.

## Planned

- WS gap recovery — gap detected → recovery mask bit set → main loop notices → REST snapshot fetch → book re-sync → mask cleared. Covers both feed_archiver and market_feed variants.
- Bootstrap protocol — feed_archiver startup, BUFFERING → LIVE transition with parallel REST + WS diff buffering, sync condition check.
- Backpressure escalation — order queue push failures accumulating → check_order_queue_overflow → KillSwitch(RISK_VIOLATION) → cascade to restart (this one is now largely a pointer at the KillSwitch cascade above, from the backpressure angle).
