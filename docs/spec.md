# Remote Mac Unlock — Implementation Specification

## 1. Objective

Build a native iOS + macOS application that allows a user to remotely unlock their personal Mac from an iPhone over the internet.

**Primary use case:** the user is at work with their iPhone. Their personal Mac is at home, connected to the internet, and currently locked. The user opens the iPhone app, authenticates with Face ID, taps **Unlock**, and the home Mac unlocks its existing macOS user session.

No Bluetooth, shared Wi-Fi network, physical proximity, remote desktop interaction, or manual password entry should be required for the successful remote-unlock flow.

### Core principles

- Native Apple applications.
- End-to-end authenticated commands.
- No account password transmission.
- No cloud storage of macOS credentials.
- Minimal infrastructure.
- No dependency on accessibility-based password typing.
- Preserve standard macOS password unlock.
- Fail closed on unauthorized commands.
- Never introduce an unrecoverable lockout condition.

## 2. Technical Architecture

### Components

**iOS Application**

- Swift 6
- SwiftUI
- LocalAuthentication
- CryptoKit
- Keychain / Secure Enclave
- URLSession WebSocket or HTTPS

**macOS Application**

- Swift 6
- SwiftUI
- Background agent
- LaunchDaemon or LaunchAgent, depending on verified requirements
- CryptoKit
- Keychain
- Security.framework
- Optional Authorization Plugin

**Relay Backend**

- Go
- HTTPS / WebSocket
- Lightweight deployment via Docker
- SQLite for MVP metadata, if persistence is necessary

### Communication flow

```text
iPhone
  |
  | Face ID authentication
  |
  | Signed unlock request
  v
Relay Server (Internet)
  |
  | Authenticated encrypted channel
  v
macOS Background Agent
  |
  | Verify device signature,
  | nonce, freshness, permissions
  v
macOS Authorization Integration
  |
  | Complete an authorized
  | screen-unlock transaction
  v
Existing macOS session unlocked
```

The backend must never possess enough key material to impersonate the authorized iPhone.

## 3. Phase 0 — macOS Unlock Feasibility

**Priority: Critical**

Before developing the complete product, investigate whether a supported and reliable mechanism exists for remotely authorizing a screen unlock on the target macOS version.

### Research

Investigate:

- Apple Authorization Services
- `Security.AuthorizationPlugin`
- `system.login.screensaver`
- `SFAuthorizationPluginView`
- SecurityAgent plugin lifecycle
- Screen-unlock policy configuration
- FileVault restrictions
- Fast User Switching
- Sleep and wake behavior
- macOS 26 and macOS 27 compatibility

Use Apple's official documentation as the primary reference.

### Proof of concept

Build an isolated local prototype capable of receiving an authorization event and evaluating whether an already-logged-in, locked macOS session can be unlocked through a legitimate authorization mechanism.

The prototype must distinguish between:

1. Authenticating inside the application.
2. Unlocking an existing macOS graphical session.
3. Logging into macOS after a reboot.
4. Unlocking a FileVault-protected startup volume.

Only item 2 is required for the MVP.

### Safety requirements

Do not automatically modify the system authorization database during development.

If a plugin is necessary:

- Require explicit local administrator approval.
- Back up the original authorization rule.
- Provide installation and uninstall scripts.
- Preserve the built-in password authentication mechanism.
- Document emergency recovery procedures.
- Never replace the only working authentication mechanism.
- Test on a disposable macOS installation or VM before installing on the primary computer.
- Do not alter SIP settings or protected system components.

The local proof of concept must demonstrate a real screen unlock.

Do not treat screen wake, screensaver dismissal, simulated keyboard input, or application authentication as successful system unlock.

### Deliverables

- [`docs/feasibility.md`](feasibility.md)
- Technical decision record ([`docs/adr/`](adr/))
- Minimal unlock prototype ([`poc/macos-unlock/`](../poc/macos-unlock/))
- Compatibility findings
- Installation and recovery documentation
- Explicit **GO / NO-GO** recommendation

If reliable native screen unlocking is not feasible, document the blockers and stop implementation of that portion rather than presenting a simulated result.

## 4. Device Pairing

Pair one Mac with one iPhone initially.

### Pairing process

1. Install and launch the macOS application.
2. Generate a unique Mac device identity.
3. Generate an iPhone authentication key.
4. Display a one-time pairing QR code on Mac.
5. Scan the QR code using the iPhone app.
6. Require local confirmation on the Mac.
7. Exchange public keys securely.
8. Store the trusted device identities locally.
9. Register routing identifiers with the relay.

Private keys must never leave their originating devices.

### Pairing security

- Short-lived pairing sessions.
- Single-use tokens.
- Explicit Mac-side approval.
- Public-key verification.
- Replay protection.
- Ability to revoke paired devices.
- No pairing through an unauthenticated remote request.

## 5. Remote Unlock Protocol

### Request

An unlock request should contain:

```json
{
  "protocolVersion": 1,
  "requestId": "uuid",
  "deviceId": "mac-id",
  "senderId": "iphone-id",
  "action": "unlock",
  "nonce": "random-value",
  "issuedAt": "ISO-8601",
  "expiresAt": "ISO-8601",
  "signature": "base64-signature"
}
```

Use a canonical binary encoding or canonical serialization for the exact signed payload. The signature must cover every security-relevant field.

### Security

- Require fresh Face ID authentication before each unlock.
- Use a Secure Enclave-backed private key when supported.
- Use a cryptographic challenge-response protocol.
- Bind requests to the paired Mac identity.
- Use short request expiration windows.
- Reject duplicated nonces and request IDs.
- Validate timestamps with reasonable clock-skew handling.
- Prevent cross-device command replay.
- Use authenticated HTTPS/WSS connections.
- Rate-limit requests.
- Maintain an auditable unlock-attempt history.

Successful Face ID authentication only authorizes access to the device's signing key. The Mac must independently verify the cryptographic authorization.

### Response

```json
{
  "requestId": "uuid",
  "status": "unlocked",
  "deviceId": "mac-id"
}
```

Possible statuses:

- `received`
- `authorized`
- `unlocked`
- `already_unlocked`
- `mac_offline`
- `not_supported`
- `authorization_failed`
- `expired`
- `rejected`

Only return `unlocked` when successful macOS screen unlock has actually been verified through an appropriate system signal.

## 6. macOS Application

### Main window

Display:

- Mac device name.
- Relay connection status.
- Paired iPhones.
- Screen lock state, when reliably available.
- Recent unlock attempts.
- Security settings.
- Revoke device action.

### Background agent

The background component must:

- Start automatically when required and permitted.
- Maintain a secure outbound connection.
- Reconnect after internet interruptions.
- Verify incoming requests.
- Apply replay protection.
- Coordinate with the authorization component.
- Report outcomes.
- Avoid exposing a public inbound port.

Do not assume the user-facing LaunchAgent can continue running in every lock-screen or pre-login state. Select the appropriate process architecture based on actual system lifecycle requirements.

### Reliability

Handle:

- Locked session.
- Already-unlocked session.
- Network disconnection.
- Relay restart.
- Mac sleep.
- User logout.
- Mac restart.
- Multiple macOS users.
- Unsupported security configuration.

No unlock request may be automatically replayed after expiration or after a reconnect.

## 7. iPhone Application

### Main screen

Minimal interface:

```text
Remote Unlock
Home Mac
● Online
Status: Locked
[ Unlock Mac ]
Last unlock: 10:42 AM
```

### Unlock interaction

1. Tap **Unlock**.
2. Perform Face ID authentication.
3. Obtain a fresh challenge.
4. Sign the authorization request.
5. Send it through the relay.
6. Display pending state.
7. Wait for confirmed completion.
8. Display the result.

### Additional screens

- Pair a Mac.
- Paired devices.
- Device details.
- Unlock history.
- Security settings.

Support Touch ID and device passcode fallback only if explicitly permitted by the application's security policy.

## 8. Backend

Implement a minimal relay service.

### Responsibilities

- Route commands between paired devices.
- Authenticate connections.
- Track device presence.
- Issue short-lived challenges.
- Enforce expiration and rate limits.
- Forward command acknowledgments.
- Prevent unauthorized device enumeration.

### Restrictions

The backend must not:

- Store macOS passwords.
- Store private signing keys.
- Generate trusted unlock requests on behalf of an iPhone.
- Grant unlock permission without the device-level cryptographic authorization.
- Store sensitive credentials in logs.

### Deployment

Provide:

- Go service.
- Dockerfile.
- Docker Compose configuration.
- Environment configuration template.
- Health endpoint.
- Structured logging.
- Graceful shutdown.
- Basic operational documentation.

## 9. Credential Handling

**Preferred model:** passwordless remote authorization through a verified macOS authorization extension.

The user's macOS password must not be required for each remote-unlock operation if the native authorization mechanism can safely support the intended workflow.

### Alternative: locally stored macOS password

Investigate this only if passwordless integration is infeasible.

Requirements:

- Password must remain on the Mac.
- Never transmit it to the iPhone or relay.
- Never store it in plaintext.
- Use Keychain access restrictions.
- Do not log credential contents.
- Do not use AppleScript, UI automation, synthetic keyboard events, or unsupported password injection as an assumed solution.
- Document whether Keychain accessibility permits the password to be retrieved while the Mac session is locked.

Do not implement this fallback unless both credential accessibility and a supported system authentication path have been demonstrated.

## 10. Security Threat Model

Consider at minimum:

- Compromised relay.
- Stolen iPhone.
- Stolen or compromised Mac.
- Replay attacks.
- Spoofed device identities.
- Pairing hijacking.
- Unauthorized device enrollment.
- Relay message tampering.
- Session state confusion.
- Race conditions.
- Lost connectivity.
- Malicious repeated unlock requests.
- Broken authorization-plugin installation.
- Partial uninstall.
- macOS updates changing authorization behavior.

Security controls must fail closed without disabling ordinary local password access.

## 11. Tests

### Unit tests

- Signature verification.
- Challenge validation.
- Request expiration.
- Nonce uniqueness.
- Paired-device validation.
- Revocation.
- State transitions.
- Incorrect authentication attempts.

### Integration tests

- iPhone to relay.
- Relay to Mac.
- Connection recovery.
- Offline device handling.
- Expired command rejection.
- Concurrent request handling.

### System tests

Test the following on a disposable macOS environment:

- Lock → remote unlock.
- Unlock → lock → unlock repeatedly.
- Incorrect command → remain locked.
- Relay offline → local password still works.
- Agent crashed → local password still works.
- Plugin crashed → local password still works.
- Network unavailable → local password still works.
- Plugin uninstall → standard behavior restored.
- Mac restart → normal login remains functional.

Run physical-device validation before claiming success.

## 12. Implementation Order

### Milestone 1 — Feasibility

- Research macOS authorization.
- Implement isolated local unlock proof of concept.
- Validate a real screen unlock.
- Document risks and compatibility.
- Decide whether to continue.

### Milestone 2 — Secure device communication

- Build the macOS agent.
- Implement local device identity.
- Implement signing and verification.
- Build the Go relay.
- Establish authenticated internet communication.
- Test on separate networks.

### Milestone 3 — iOS application

- Build SwiftUI screens.
- Implement QR pairing.
- Implement Face ID authorization.
- Implement signed unlock requests.
- Implement status updates.

### Milestone 4 — System integration

- Connect the relay command to the proven local unlock mechanism.
- Implement result verification.
- Add explicit installation approval.
- Provide rollback and recovery.
- Test locked-session behavior.

### Milestone 5 — Production hardening

- Security review.
- Connection recovery.
- Logging and observability.
- Error handling.
- Code signing and notarization.
- Installation and update workflows.
- Compatibility matrix.

## 13. Acceptance Criteria

The MVP is complete only when:

- [ ] iPhone and Mac work from different networks.
- [ ] No Bluetooth connection is involved.
- [ ] The Mac does not require an open inbound router port.
- [ ] The iPhone user authenticates with Face ID.
- [ ] The remote command is cryptographically authenticated.
- [ ] The relay cannot forge an authorized unlock.
- [ ] The actual macOS lock screen is dismissed.
- [ ] The existing desktop session becomes usable.
- [ ] No macOS password is transmitted.
- [ ] The Mac can still be unlocked normally using its password.
- [ ] Failed or interrupted requests cannot unlock the Mac.
- [ ] Device revocation works.
- [ ] Agent or relay failures do not block local login.
- [ ] Installation and removal procedures are documented.

## 14. Execution Instructions

Act as a senior macOS security engineer, iOS engineer, and Go backend engineer.

Work autonomously and prioritize a functioning, verifiable MVP.

Start with Phase 0. Inspect Apple's security architecture, establish an evidence-based technical approach, and implement the smallest possible proof of concept.

Do not assume that a successful authorization callback automatically implies an unlocked macOS session.

Do not install a custom system authorization mechanism or modify machine-wide authentication policy without explicit permission.

Do not bypass SIP or weaken platform security protections.

Do not claim that a remote unlock is working unless it has passed a real system-level test.

Once feasibility is established, build the remaining components in the specified milestone order.

Prefer straightforward implementations over unnecessary abstractions.

Maintain clear technical documentation and automated tests.

If blocked by macOS platform restrictions, report the exact limitation, supporting evidence, and the simplest technically viable alternative.

The primary success metric is a real, secure, remotely triggered macOS screen unlock — not merely successful network communication.
