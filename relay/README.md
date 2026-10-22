# Relay (Go)

A stateless relay that authenticates Mac connections and iPhone requests, checks routing grants, and correlates HTTPS requests with the Mac's WebSocket. It can't authorize an unlock or introduce trusted devices, and it stores nothing persistently: no database, no Redis, no pairing records. Restarts and upgrades never require pairing again.

Not started (spec steps 4, 9, 10). See [`docs/spec.md` §5–§8, §12–§13](../docs/spec.md#5-mac-relay-connection).

| Endpoint | Purpose |
| --- | --- |
| `WSS /v1/mac/connect` | Authenticate and connect a Mac |
| `POST /v1/status` | Request fresh signed status |
| `POST /v1/unlock/challenge` | Request a signed unlock challenge |
| `POST /v1/unlock/authorize` | Submit signed unlock authorization |

Target: one systemd service on a Linux VPS with 1 vCPU / 1 GB RAM. WebSocket compression off; every queue, buffer and state table is bounded. Load tests cover 1,000 and 5,000 real TLS/WebSocket connections.
