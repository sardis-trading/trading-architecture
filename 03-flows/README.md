# Cross-cutting flows

Placeholder. Sequence diagrams for behaviours that span multiple containers or components.

Candidates:
- Order lifecycle (submit → risk → wire → ack → fill → position update)
- Tick flow (WS → parser → OrderBook → strategy → order)
- WS gap recovery (gap detected → recovery mask set → REST snapshot → resync → clear mask)
- Bootstrap protocol for feed archiver
- Kill switch trip → clean shutdown → supervisor restart → position recovery

TBD.
