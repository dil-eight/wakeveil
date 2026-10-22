# macOS app and agent

SwiftUI app plus background agent. The agent keeps an outbound connection to the relay, verifies incoming signed requests (signature, nonce, freshness, pairing) and hands verified requests to the authorization component proven in Phase 0.

**Milestones 2 and 4** — not started. See [`docs/spec.md` §6](../../docs/spec.md#6-macos-application).
