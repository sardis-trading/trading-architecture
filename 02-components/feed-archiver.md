# feed_archiver — Components

Zooms into the `feed_archiver` binary from [Level 2](../01-containers/feed-archiver.md) to show the internal components that turn a WebSocket stream + REST snapshot into ordered ticks on disk.

Three diagrams, each doing one job: a structural overview, a zoom into a single Connection Group's pipeline, and the bootstrap flow that gets each group from `BUFFERING` to `LIVE`.

## 1. Component overview

Top-level structure with Connection Groups collapsed. Shows what the orchestrator owns and how the pieces relate.

```mermaid
flowchart TB
    binance{{"Binance Spot"}}

    binance -->|WS + REST| groups

    subgraph binary["feed_archiver binary"]
        manager["BinanceFeedManager<br/>orchestrator"]

        groups["Connection Group × N<br/>(one per config entry)"]

        bootstrap["Bootstrap coordinator<br/>parallel REST + diff buffer<br/>+ sync condition"]

        recorder["SymbolRecorder<br/>per-symbol disk writer"]

        context["FeedContext<br/>shared state per group"]
    end

    localticks[("Local Tick Archive")]

    manager -->|spawns / joins| groups
    manager -->|owns| bootstrap
    manager -->|owns| context
    manager -->|owns| recorder

    groups -->|normalized events| recorder
    bootstrap -.->|controls phase| groups
    bootstrap -.->|reads / writes| context

    recorder -->|append| localticks
```

## 2. Inside a Connection Group

One group's internal pipeline. There are N of these in the binary, isolated from each other — failure in one doesn't stop others.

```mermaid
flowchart LR
    binance{{"Binance Spot<br/>(WS)"}}

    binance -->|raw JSON| lws

    subgraph group["Connection Group"]
        lws["LWS client<br/>libwebsockets thread"]
        queue["MessageQueue<br/>SPSC ring buffer"]
        worker["Worker thread<br/>consumes queue"]
        parser["BinanceParser<br/>JSON → events"]
        seqcheck["Sequence gap detector<br/>checks update_id continuity"]
        reorder["Trade reorder buffer<br/>10ms window<br/>(trades only)"]
    end

    recorder["SymbolRecorder"]
    bootstrap["Bootstrap coordinator<br/>(re-bootstrap on gap)"]

    lws -->|push| queue
    worker -->|pop| queue
    worker -->|parse| parser
    parser -->|depth event| seqcheck
    parser -->|trade event| reorder
    seqcheck -->|in-sequence| recorder
    seqcheck -.->|gap detected| bootstrap
    reorder -->|ordered trade| recorder
```

## 3. Bootstrap phase (BUFFERING → LIVE) and mid-flight gap recovery

Same mechanism is used in two situations:
- **On group start**, every symbol begins in `BUFFERING`.
- **Mid-flight**, when the sequence gap detector catches a discontinuity in `update_id`, the affected symbol is re-entered into `BUFFERING`, its WS diffs buffered, a fresh REST snapshot fetched, and the sync condition re-checked before returning to `LIVE`.

In both cases WS diffs stream into a per-symbol buffer while a REST snapshot fetch runs in parallel on the ThreadPool. When the snapshot arrives, the sync condition is checked; on success, the phase flips to `LIVE` and buffered diffs are drained.

```mermaid
flowchart TB
    binance{{"Binance Spot"}}

    binance -->|WS diffs| lws
    binance -->|REST snapshot response| pool

    subgraph binary["feed_archiver binary"]
        lws["LWS client<br/>(streaming)"]
        pool["ThreadPool<br/>parallel REST fetch"]
        diffbuf[("Per-symbol diff buffer<br/>(bounded, in-memory)")]
        bootstrap["Bootstrap coordinator<br/>checks sync condition,<br/>flips phase"]
        context["FeedContext<br/>phase = BUFFERING or LIVE"]
        worker["Worker thread<br/>gated by phase"]
        recorder["SymbolRecorder"]
    end

    lws -->|diffs during BUFFERING| diffbuf
    pool -->|snapshot| bootstrap
    diffbuf -->|read on flush| bootstrap
    bootstrap -->|writes phase transition| context
    bootstrap -->|releases buffered diffs in order| worker
    worker -->|events| recorder

    lws -.->|diffs during LIVE go straight to worker via queue| worker
```

**Sync condition:** for the depth stream, a snapshot with `lastUpdateId = L` is compatible with a WS diff `[U, u]` if `U ≤ L+1 AND u ≥ L+1`. Bootstrap holds diffs in the per-symbol buffer until this is satisfied. If the snapshot fetch fails, the coordinator escalates to a kill signal for the group.

## Components

| Component | What it does |
|-----------|--------------|
| **BinanceFeedManager** | Top-level orchestrator. Reads `[[archiver.connections]]` from config, spawns one Connection Group per entry, owns Bootstrap coordinator, SymbolRecorder, and FeedContext. On shutdown, joins worker threads first (drain queues), then disconnects WS clients — ordering avoids WS callbacks firing into joined workers. |
| **Connection Group** | Isolated pipeline for one WebSocket connection. N of these, one per configured group. Each holds its own LWS client, MessageQueue, worker thread, and BinanceParser. Failure in one doesn't affect others. |
| **LWS client** | Wraps libwebsockets. Runs on its own thread (per-group `binance-ws-N`). Receives raw WS messages from Binance and pushes into MessageQueue via lock-free SPSC. Never blocks. LwsAnchor → LwsService → LwsConnection ownership model (see `include/network/CLAUDE.md`). Service timeout tuned to 1 ms per group to avoid update pileup during bursts. |
| **MessageQueue (SPSC)** | Bounded single-producer single-consumer ring between the LWS thread (producer) and the worker thread (consumer). Non-blocking on both sides. |
| **Worker thread** | Consumes messages, calls BinanceParser, routes normalized events downstream. One per group. During BUFFERING the worker is gated by phase — events flow into the diff buffer instead of straight to SymbolRecorder. |
| **BinanceParser** | Stateless JSON → normalized-event translator. Depth-diff messages produce one event per level, so the consumer must handle the emitted buffer. |
| **Trade reorder buffer** | Bounded 10 ms delay window for `@trade` events. Binance's matching engine is sharded, so trade IDs can arrive slightly out of order; the buffer holds and re-orders. Not applied to depth (depth uses `update_id` for sequencing). |
| **Sequence gap detector** | Runs on the depth path only. Checks `update_id` continuity across events. On a valid sequence, forwards straight to SymbolRecorder. On a gap, signals Bootstrap coordinator to re-enter BUFFERING for that symbol — same mechanism as initial bootstrap. |
| **Bootstrap coordinator** | Runs the REST snapshot fetch in parallel with the WS diff buffering, checks the sync condition on snapshot arrival, transitions phase from BUFFERING to LIVE, flushes buffered diffs in order. Kills the group on fetch failure. |
| **SymbolRecorder** | Per-symbol writer. Formats normalized events into the binary tick file layout (64B-aligned header, mmap-friendly), rotates files, appends to `Local Tick Archive`. Same on-disk format read by Backtest via `data_manager`. |
| **FeedContext** | Shared per-group state — parsed config, current phase, sequence tracking for gap detection. Passed by reference. |
| **ThreadPool** | General-purpose pool used by Bootstrap for parallel snapshot fetches. Not per-group; shared across all groups. |

## Backpressure and overflow behavior

Every bounded queue in this container has a defined behavior when it fills. The overarching principle: the LWS thread never blocks. If any consumer downstream can't keep up, upstream data is dropped or the group is escalated to a kill signal — never held.

| Queue / buffer | Bound | Behavior on full | Rationale |
|----------------|-------|-------------------|-----------|
| **MessageQueue (SPSC)** | Config-defined ring capacity (~ 4096–65536) | LWS thread drops the message and increments a drop counter. Never blocks. | Blocking here would back-pressure into libwebsockets and starve other groups; dropped messages are a smaller harm than a stalled service loop. |
| **Per-symbol diff buffer** (during BUFFERING) | Bounded per-symbol slot count | Oldest buffered diff is discarded; if the buffer overflows before sync condition is reached, Bootstrap escalates to kill. | During bootstrap the buffer size sets an upper bound on how long the REST snapshot can take before recovery is impossible. |
| **Trade reorder buffer** | 10 ms window (time-bounded, not count) | Events past the window flush regardless — never a "full" condition. | Time-bounded resolves out-of-order without letting the buffer grow. |
| **ThreadPool queue** (Bootstrap snapshot fetches) | Bounded task queue | Fetch requests block briefly at enqueue if the pool is saturated. Snapshots aren't hot-path so blocking is acceptable here. | Bootstrap runs during BUFFERING; slight enqueue latency doesn't affect LIVE path. |

**Kill-on-overflow:** any repeated MessageQueue drops in a single group or a diff-buffer overflow during bootstrap escalates to killing that group. `BinanceFeedManager` sees the signal, joins the worker, disconnects the LWS client. Other groups continue running unaffected.

## Deliberately not here

- Binance JSON schema and per-message field details — a venue concern, lives near `BinanceParser` in code.
- libwebsockets internals (event loop, SSL, poll behavior) — see `include/network/CLAUDE.md`.

## Not shown at this level

- Class-level structure (constructors, member layouts, exact signatures). Read the source for that.
- The reader side that consumes the produced tick files — see [Level 2 backtest](../01-containers/backtest.md) and backtest components (TBD).

## Next zooms

- trading_system components — TBD.
- [Level 4 — Flows](../03-flows/) — a full gap-recovery sequence would live here, spanning `BUFFERING → LIVE` and a mid-flight WS gap.
