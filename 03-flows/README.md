# Level 4 — Cross-cutting flows

Sequence diagrams and behavioral flows for things that span multiple containers or components. Where component diagrams at Levels 2 and 3 answer *"what exists and how it connects,"* these answer *"what happens over time."*

## Flows

- [Order lifecycle](order-lifecycle.md) — flagship flow. Full path from strategy `submit()` through in-engine RiskManager, OrderManager allocation, RouterExchange, TCP to order_router, BinanceExchange, Binance, and the reverse path for ack + fill including PositionManager and Strategy `on_fill`. Covers happy path plus pre-trade reject, venue reject, partial fills, and late-fill-after-reconnect branches. Includes a rough timing-budget table for the hot path.
- [Tick flow](tick-flow.md) — one tick's path from Binance WS through market_feed, across the shared-memory ring, into a trading engine, and onto the strategy's `on_tick_impl`. Complements the order lifecycle. Covers sequence-gap on the market_feed side, slow-subscriber ring overwrite, multi-engine subscription independence, and the trade-tick reorder buffer branch.
- [Cancel-on-ACK race handling](cancel-on-ack.md) — the wire-layer fix inside `BinanceExchange` for the race where an engine cancels a still-in-flight order. Cancel is held until ACK arrives, then flushed in the same tick. Includes the naive-implementation illustration showing the orphan-order failure mode, the state machine inside BinanceExchange, edge cases, and rationale for placing the fix at the wire layer rather than in the engine.
- [KillSwitch trip cascade](killswitch-cascade.md) — the full sequence from trip source detection through deferred callback execution, cancel_all + halt, engine shutdown, ProcessManager alerting, operator-gated restart, and position recovery. Ties together most of the safety-critical flows in the system. Covers the deferred-trip rationale, the six stages, coordination with other flows, and internal failure modes.
- [Bootstrap protocol](bootstrap.md) — how a Connection Group transitions from BUFFERING to LIVE, ensuring the local order book is a correct projection of the venue before any tick reaches downstream. Parallel REST snapshot fetch + WS diff buffering, the specific sync condition on `U <= L+1 AND u >= L+1`, buffer overflow escalation, and the same-protocol reuse for mid-flight recovery.
- [WS gap recovery](ws-gap-recovery.md) — mid-flight recovery from a WS depth-stream discontinuity. Same protocol as bootstrap, different trigger. Covers the lock-free `recovery_mask_` design that lets the LWS thread signal without blocking, drop vs buffer modes during recovery, and the failure escalation path to `MARKET_DISCONNECT` if recovery drags out.
- [Backpressure escalation](backpressure-escalation.md) — how sustained order-SPSC push failures escalate through `check_order_queue_overflow` to `KillSwitch(RISK_VIOLATION)` and full engine restart. Includes the coordination table showing which queues escalate through which watchdogs, and rationale for using `RISK_VIOLATION` rather than a dedicated backpressure enum.
- [Position recovery after restart](position-recovery.md) — how a fresh engine reconstructs its `PositionManager` and `OrderManager` from QuestDB fill history + venue open-order reconciliation, including the guarantees and non-guarantees.

## Planned

*None outstanding.* All flows originally in scope for Level 4 are covered above.

Future additions worth considering if the system grows:

- Multi-symbol strategy coordination (if a strategy ever needs to span multiple symbols with cross-symbol constraints).
- Cross-venue arbitrage flow (only relevant if we ever add a second venue).
- Cold-start bootstrap with historical replay (currently we start from live only).
