# Relay (Go)

Minimal relay that routes signed commands between paired devices. It authenticates connections, tracks presence, issues short-lived challenges and enforces expiry and rate limits. It never holds private keys or passwords and cannot forge an unlock.

**Milestone 2** — not started. See [`docs/spec.md` §8](../docs/spec.md#8-backend).

Planned deliverables: Go service, `Dockerfile`, `docker-compose.yml`, `.env.example`, health endpoint, structured logging, graceful shutdown, ops docs.
