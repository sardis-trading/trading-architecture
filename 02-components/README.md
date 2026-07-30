# Level 3 — Components

Zooms into a single container from [Level 2](../01-containers/) and shows its internal components — the named parts (classes, modules, subsystems, threads) that structure the code inside one process.

**Component** here means what it means in the C4 model — a grouping of related functionality inside a single container. Not a UML class, not a single object; more like a "subsystem" or "module."

## Per-container zooms

- [feed_archiver](feed-archiver.md) — internals of the WS collection binary: BinanceFeedManager, per-group pipeline (WS client → SPSC queue → parser → worker), bootstrap coordinator, sequence gap detector, trade reorder buffer, SymbolRecorder.
- [backtest_app](backtest.md) — replay engine, single-run tick flow, grid search / walk-forward orchestration. CRTP strategy dispatch, deterministic virtual time, mmap tick reader, bounded order book, fill simulator, metrics collector.
- [order_router](order-router.md) — TCP server for engine connections, `client_order_id` remapping (`global_coid = (engine_id << 32) \| orig_coid`), BinanceExchange wire layer, routing tables guarded by mutex, per-account rate limiter.
- [market_feed](market-feed.md) — WS collection + book maintenance + per-symbol shm publishing. N connection groups (LwsService + BinanceClient), MarketDataManager, ShmTickPublisher per symbol, health hooks.
- trading_system — TBD (the trading engine binary — MarketFeed consumer, OMS, PositionManager, in-process **RiskManager**, KillSwitch, ParameterManager, QuestDBWriter, CRTP strategies).

**Note on RiskManager:** risk checks live inside the trading engine, not as a separate container. Target <100ns per check makes IPC-based risk unworkable. See [Level 2 note](../01-containers/live-trading.md) and the RiskManager component under `trading_system` (TBD).

## Not shown at this level

- Line-level code, class hierarchies, exact function signatures — that's what the source code is for.
- Cross-container flows (a full order's lifecycle spans engine → risk_manager → order_router → venue) — see [Level 4](../03-flows/).
