# Level 2 — Containers

Zooms into each of the three top-level systems from [Level 1](../00-context/) and shows the runtime processes that make them up, and how those processes coordinate.

**Container** here means what it means in the C4 model — a separately runnable/deployable unit (a process, a database, a static site, a queue), not a Docker container specifically. Some of these processes happen to run in Docker; others are bare binaries or systemd units.

## Per-system zooms

- [Live Trading](live-trading.md) — supervisor + N trading engines + param control + admin UI + observability stack.
- Feed Archiver — TBD.
- Backtest — TBD.

## Not shown at this level

- Internal structure of any single container (that's [Level 3](../02-components/)).
- Where each container physically runs (host / region / VM / container tech) — that belongs in a separate deployment doc.
