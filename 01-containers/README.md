# Level 2 — Containers

Zooms into each of the three top-level systems from [Level 1](../00-context/) and shows the runtime processes that make them up, and how those processes coordinate.

**Container** here means what it means in the C4 model — a separately runnable/deployable unit (a process, a database, a static site, a queue), not a Docker container specifically. Some of these processes happen to run in Docker; others are bare binaries or systemd units.

## Per-system zooms

- [Live Trading](live-trading.md) — control plane (Caddy + Control Hub SPA + ProcessManager + ParamGateway) and data plane (feed_router + risk_manager + order_router + N engines + observability).
- [Feed Archiver](feed-archiver.md) — WS collection binary + tick_packer + rclone cloud sync, writing to Local Tick Archive.
- [Backtest](backtest.md) — backtest_app + grid_search + data_manager for local/cloud tick resolution, results as CSV + equity curves.

## Not shown at this level

- Internal structure of any single container (that's [Level 3](../02-components/)).
- Where each container physically runs (host / region / VM / container tech) — that belongs in a separate deployment doc.
