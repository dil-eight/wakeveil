# macOS agent and authorization component

- **Agent:**
  - BLE pairing (Add iPhone) and trusted-device storage;
  - routing grants and routing generation;
  - the outbound WebSocket to the relay;
  - signed challenges, with in-memory, atomically consumed state;
  - session status and the Mac UI.
- **Authorization component:** minimal and privileged. It integrates with the system unlock mechanism validated in the first milestone. No networking and no untrusted parsing; it accepts requests only from the authenticated agent over local IPC.

Not started (spec steps 3–8). See [`docs/spec.md` §2–§7](../../docs/spec.md#2-architecture) and [`docs/design.md` §7](../../docs/design.md#7-mac-app).
