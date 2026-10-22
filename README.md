# Wakeveil

Remotely unlock your locked Mac from your iPhone, over the internet, with Face ID.

Wakeveil is a native iOS + macOS app with a minimal Go relay. The iPhone signs every unlock request with a Secure Enclave key, the relay only routes messages, and the Mac verifies the signature itself before completing a legitimate macOS screen-unlock transaction. No password ever leaves the Mac, and the standard password unlock always keeps working.

> [!WARNING]
> **Status: Phase 0 — feasibility.** Nothing here can unlock a Mac yet. Whether a supported macOS mechanism for remote screen unlock exists is still being verified. See [`docs/feasibility.md`](docs/feasibility.md).

## How it works

```text
iPhone ──Face ID──▶ signed request ──▶ Relay (Go) ──WSS──▶ Mac agent ──▶ macOS authorization ──▶ session unlocked
```

- **iPhone app** — SwiftUI, LocalAuthentication, CryptoKit, Secure Enclave.
- **Mac app + agent** — SwiftUI, background agent, Security.framework, optional Authorization Plugin.
- **Relay** — Go, HTTPS/WebSocket, Docker. Routes commands; cannot forge them.

The full specification is in [`docs/spec.md`](docs/spec.md).

## Repository layout

| Path | Contents | Milestone |
| --- | --- | --- |
| [`docs/spec.md`](docs/spec.md) | Implementation specification | — |
| [`docs/feasibility.md`](docs/feasibility.md) | Phase 0 research, findings, GO / NO-GO | 1 |
| [`docs/adr/`](docs/adr/) | Architecture decision records | all |
| [`poc/macos-unlock/`](poc/macos-unlock/) | Isolated local screen-unlock prototype | 1 |
| [`relay/`](relay/) | Go relay service | 2 |
| [`apps/macos/`](apps/macos/) | macOS app and background agent | 2, 4 |
| [`apps/ios/`](apps/ios/) | iPhone app | 3 |
| [`scripts/`](scripts/) | Install, uninstall and recovery scripts | 4 |

## Roadmap

1. **Feasibility** — research macOS authorization, prove a real screen unlock locally, decide GO / NO-GO.
2. **Secure device communication** — Mac agent, device identity, signing, Go relay.
3. **iOS application** — SwiftUI screens, QR pairing, Face ID, signed requests.
4. **System integration** — wire the relay command to the proven unlock mechanism, install approval, rollback.
5. **Production hardening** — security review, observability, signing and notarization, compatibility matrix.

Each milestone starts only after the previous one is done.

## Safety rules

- Never modify the system authorization database without explicit local administrator approval.
- Always back up the original authorization rule and keep the built-in password mechanism in place.
- Test anything that touches authorization on a disposable macOS VM first.
- Never disable SIP or touch protected system components.
- Screen wake, screensaver dismissal or synthetic key presses do **not** count as an unlock.
