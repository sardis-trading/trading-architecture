# Level 4 — Cross-cutting flows

Sequence diagrams and behavioral flows for things that span multiple containers or components. Where component diagrams at Levels 2 and 3 answer *"what exists and how it connects,"* these answer *"what happens over time."*

## Flows

- [Position recovery after restart](position-recovery.md) — how a fresh engine reconstructs its `PositionManager` and `OrderManager` from QuestDB fill history + venue open-order reconciliation, including the guarantees and non-guarantees.

## Planned

- Order lifecycle — full path from strategy `submit()` through in-engine RiskManager, OrderEngine, RouterExchange, TCP to order_router, BinanceExchange, Binance venue, and the reverse path for ack + fill including PositionManager and Strategy `on_fill`.
- Tick flow — Binance WS to market_feed to shm ring to trading_system's ShmTickSubscriber to MDM to Strategy.
- WS gap recovery — gap detected → recovery mask bit set → main loop notices → REST snapshot fetch → book re-sync → mask cleared. Covers both feed_archiver and market_feed variants.
- Bootstrap protocol — feed_archiver startup, BUFFERING → LIVE transition with parallel REST + WS diff buffering, sync condition check.
- KillSwitch trip → cascade — condition detected → trip() → deferred callbacks → cancel_all + halt → engine exits → ProcessManager restart → position recovery (linked to the position-recovery flow above).
- Cancel-on-ACK race handling — engine cancels a not-yet-acked order → order_router holds the cancel until ack arrives → flushes both in the same tick.
- Backpressure escalation — order queue push failures accumulating → check_order_queue_overflow → KillSwitch(RISK_VIOLATION) → cascade to restart.
