# Wakeveil

Open-source iPhone and Mac apps that let you approve unlocking your locked Mac from your iPhone, over the internet. Works on iPhone and iPhone Duo.

You pair the iPhone and the Mac once, in person, over Bluetooth. After that, the iPhone talks to a small Go relay over HTTPS, and the Mac keeps an outbound WebSocket to the same relay. Every unlock is a challenge signed by the Mac and an authorization signed by the iPhone after Face ID, Touch ID or passcode. The relay only routes messages: it can't add devices, forge an unlock, or replay one. Wakeveil never stores or submits your Mac password, and the normal password unlock always keeps working.

> [!WARNING]
> **Status: first milestone — validating macOS unlocking.** Nothing here can unlock a Mac yet. Whether a safe, passwordless mechanism exists on a declared macOS version is still being verified. See [`docs/feasibility.md`](docs/feasibility.md).

## How it works

```text
Pairing (once, offline):   iPhone ◀──── BLE + verification code ────▶ Mac

Unlock:  iPhone ──HTTPS──▶ Relay (Go) ◀──WSS (outbound)── Mac agent ──IPC──▶ authorization component
         1. request challenge            ─────▶  Mac signs challenge
         2. Face ID / Touch ID / passcode, then sign authorization
         3. submit authorization         ─────▶  Mac verifies, consumes, unlocks, signs result
```

| Component | Responsibilities |
| --- | --- |
| iOS app (Swift) | BLE pairing, paired Mac list, status, user authentication, signed unlock authorization |
| macOS agent (Swift) | Pairing, trusted device storage, WebSocket connection, challenges, session status |
| macOS authorization component (Swift) | Minimal integration with the validated system unlock mechanism |
| Relay (Go) | Connection authentication, routing, bounded request correlation, rate limiting. No database; runs as one systemd service on a 1 vCPU / 1 GB VPS |

Documents:

- [`docs/spec.md`](docs/spec.md) — implementation specification
- [`docs/design.md`](docs/design.md) — UI and interaction design (iPhone, iPhone Duo, Mac)
- [`docs/feasibility.md`](docs/feasibility.md) — first-milestone report

## Repository layout

| Path | Contents | Spec step |
| --- | --- | --- |
| [`docs/`](docs/) | Spec, design, feasibility report, ADRs; later the wire protocol, threat model and deployment docs | all |
| [`poc/macos-unlock/`](poc/macos-unlock/) | Isolated macOS unlock proof of concept | 1 |
| [`apps/macos/`](apps/macos/) | macOS agent and authorization component | 3–8 |
| [`apps/ios/`](apps/ios/) | iPhone app | 3, 5, 7 |
| [`relay/`](relay/) | Go relay service | 4, 9, 10 |
| [`scripts/`](scripts/) | Install, uninstall and recovery scripts | 1, 6, 10 |

## Roadmap

Per [spec §15](docs/spec.md#15-implementation-order):

1. Validate the macOS unlock mechanism and recovery path.
2. Finalize cryptographic formats and pairing protocol.
3. Implement BLE pairing and protected device storage.
4. Implement relay connection authentication and routing grants.
5. Implement signed challenges and authenticated iPhone signing.
6. Integrate the validated local unlock component.
7. Implement status and uncertain-outcome handling.
8. Implement revocation and recovery.
9. Run protocol, failure-path, and load tests.
10. Document installation, deployment, and upgrades.

Nothing past step 1 starts until safe passwordless session unlocking is proven.

## Safety rules

- Never disable SIP or FileVault, weaken the standard authentication path, or build a fake lock screen.
- Privileged installation always requires explicit administrator authorization.
- A missing, unavailable or crashing Wakeveil component must never block the normal password unlock.
- Test anything that touches authorization on a disposable macOS VM or spare Mac first.
- Screen wake, screensaver dismissal or synthetic key presses do **not** count as an unlock.
