# trading-architecture

System architecture and design diagrams for a low-latency Binance market-making engine — top-down from context to components.

The system is a single-venue Binance spot market-maker written in C++20. This repo documents its architecture at four levels of zoom, starting with the whole system in its environment and ending at internal component structure.

## Levels

| Level | Folder | Question it answers |
|-------|--------|---------------------|
| 1. Context     | [`00-context/`](00-context/)       | What is the system, what does it talk to, who runs it? |
| 2. Containers  | [`01-containers/`](01-containers/) | What are the top-level runtime processes and how do they connect? |
| 3. Components  | [`02-components/`](02-components/) | What's inside each container? |
| 4. Flows       | [`03-flows/`](03-flows/)           | Cross-cutting sequences: an order's lifecycle, a tick's path through the system, recovery after a WS gap. |

## Diagram format

All diagrams are Mermaid embedded in Markdown, rendered natively by GitHub. No build step; edit the `.md` files, push, view.

## Start here

- [System Context](00-context/README.md) — the whole thing as one box in the world.
