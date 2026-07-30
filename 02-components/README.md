# Level 3 — Components

Zooms into a single container from [Level 2](../01-containers/) and shows its internal components — the named parts (classes, modules, subsystems, threads) that structure the code inside one process.

**Component** here means what it means in the C4 model — a grouping of related functionality inside a single container. Not a UML class, not a single object; more like a "subsystem" or "module."

## Per-container zooms

- [feed_archiver](feed-archiver.md) — internals of the WS collection binary: BinanceFeedManager, per-group pipeline (WS client → SPSC queue → parser → worker), bootstrap coordinator, sequence gap detector, trade reorder buffer, SymbolRecorder.
- [backtest_app](backtest.md) — replay engine, single-run tick flow, grid search / walk-forward orchestration. CRTP strategy dispatch, deterministic virtual time, mmap tick reader, bounded order book, fill simulator, metrics collector.
- [risk_manager](risk-manager.md) — small shared service between engines and order_router. Ingest queue, check engine, limits registry, KillSwitch state reader, reject dispatcher.
- trading_system — TBD (the trading engine binary — MarketFeed consumer, OMS, PositionManager, KillSwitch, ParameterManager, QuestDBWriter, CRTP strategies).
- order_router — TBD.
- feed_router — TBD.

## Not shown at this level

- Line-level code, class hierarchies, exact function signatures — that's what the source code is for.
- Cross-container flows (a full order's lifecycle spans engine → risk_manager → order_router → venue) — see [Level 4](../03-flows/).
