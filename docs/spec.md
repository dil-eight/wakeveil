# Wakeveil MVP — Implementation Specification

## Objective

Build an open-source iPhone and macOS application that lets a user authorize unlocking their paired Mac over the internet.

Devices are paired locally over Bluetooth once. Subsequent requests use an HTTPS API and an outbound WebSocket connection maintained by the Mac.

## Fixed Requirements

- iOS and macOS applications written in Swift.
- Relay backend written in Go.
- Initial deployment target: Linux VPS with 1 vCPU and 1 GB RAM.
- No user accounts.
- No Mac account passwords stored or submitted by Wakeveil.
- No SQL database, Redis, or WebRTC.
- Initial pairing requires Bluetooth proximity and confirmation on an unlocked Mac.
- Private keys and device trust remain on the devices.
- The relay cannot authorize an unlock or introduce trusted devices.
- No persistent command queue or automatic retries of unlock commands.
- Server upgrades and restarts must not require pairing again.

## 1. First Milestone: Validate macOS Unlocking

Before implementing the complete product, build an isolated proof of concept that unlocks an existing locked macOS user session without storing or submitting the account password.

Investigate the macOS authorization plug-in mechanism using current Apple documentation. Do not assume a general-purpose unlock API exists.

### Required validation

- Demonstrate unlocking on an explicitly declared macOS version.
- Preserve ordinary password-based login and unlock.
- Ensure a missing, unavailable, or crashing Wakeveil component does not block ordinary unlock.
- Test repeated lock/unlock cycles.
- Confirm the session actually becomes unlocked.
- Document installation, removal, and recovery.
- Require explicit administrator authorization for privileged installation.

Do not disable SIP or FileVault, weaken the standard authentication path, or implement a fake lock screen.

If the proof of concept cannot meet these requirements, stop and report the blocker before implementing the remaining product.

### MVP scope

Support an awake Mac with an existing locked user session.

The following are outside the MVP:

- Waking a sleeping Mac over the internet.
- FileVault preboot unlock.
- Initial login after restart.

## 2. Architecture

| Component | Responsibilities |
|---|---|
| iOS application | BLE pairing, paired Mac list, status display, user authentication, signed unlock authorization |
| macOS agent | Pairing, trusted device storage, WebSocket connection, challenge handling, session status |
| macOS authorization component | Minimal integration with the validated system unlock mechanism |
| Go relay | Connection authentication, message routing, bounded request correlation, rate limiting |

Keep networking and untrusted message parsing outside the privileged authorization component wherever possible.

Authenticate and authorize the local IPC boundary. An unrelated local process must not be able to authorize unlocking.

## 3. Device Identity and Storage

Each device has a cryptographic identity.

- Generate keys using platform cryptographic APIs.
- Protect private keys using platform storage.
- Use Secure Enclave keys where supported and appropriate.
- Specify a canonical public-key representation.
- Derive device IDs from a cryptographic hash of that representation.
- Persist trusted peer keys only after successful pairing.

The iPhone unlock-signing key must require system-enforced user authentication for each authorization operation.

Do not rely solely on a UI flag indicating that Face ID succeeded. Do not reuse an authenticated context in a way that silently authorizes a later unlock attempt.

Support Face ID, Touch ID, or device passcode according to the selected platform policy. Validate that the key access-control configuration supports the intended fallback.

A separate protected identity key may authenticate routine relay requests without prompting for biometrics. If separate keys are used, bind both keys to the device during pairing and document their purposes.

### Persistent storage

| Location | Stored data |
|---|---|
| Mac | Own keys, trusted iPhone keys, permissions, routing generation, settings |
| iPhone | Own keys, trusted Mac keys, paired-device metadata, routing grants |
| Relay | None required |

### Volatile storage

- Active connections.
- Pending API requests.
- Replay-protection and rate-limit state.
- Outstanding Mac challenges.

Server restarts must not invalidate device identities or require pairing again.

## 4. Bluetooth Pairing

Implement an application-level pairing protocol over BLE using Core Bluetooth.

BLE connectivity alone does not authenticate the peer.

### User flow

1. User opens Wakeveil on an unlocked Mac.
2. User selects **Add iPhone**.
3. User opens Wakeveil on iPhone and selects **Add Mac**.
4. Devices establish a BLE connection.
5. Devices execute an authenticated key exchange with a short authentication string.
6. Both devices display the verification code.
7. User confirms matching codes on both devices.
8. Devices persist peer identities and permissions after successful confirmation.

Pairing must work without internet access.

### Protocol requirements

- Select and document a reviewed pairing protocol before implementation.
- Use established cryptographic libraries.
- Do not substitute an improvised key exchange followed by a truncated hash comparison.
- Bind identities, roles, exchanged keys, ephemeral data, and confirmations to the pairing transcript.
- Expire abandoned attempts.
- Limit concurrent attempts.
- Require a new local pairing flow for a new iPhone.
- Do not expose an endpoint that remotely adds trusted keys.

If a suitable implementation cannot be identified, report the blocker.

## 5. Mac Relay Connection

The Mac maintains an outbound TLS-protected WebSocket connection.

### Authentication handshake

1. Relay issues a fresh random challenge.
2. Mac provides its public key and signs the challenge with connection context.
3. Relay verifies the signature.
4. Relay verifies that the public key hashes to the claimed Mac ID.
5. Relay associates the authenticated ID with the connection.

Connection challenges must be short-lived, bound to the connection, and accepted only once.

Maintain an in-memory mapping:

`Mac ID → authenticated WebSocket connection`

When replacing a connection, cleanup of an old connection must not remove its replacement.

The Mac advertises its current routing generation over the authenticated connection.

## 6. Routing Authorization

Knowing a Mac ID must not grant permission to query its status or forward messages.

During local pairing, the Mac issues a signed routing grant to the iPhone.

### Grant contents

- Protocol version and purpose.
- Mac ID.
- iPhone identity and request-authentication key binding.
- Allowed operations.
- Routing generation.

The iPhone includes the grant with authenticated API requests.

The relay verifies:

- Mac signature and identity.
- iPhone identity and request signature.
- Requested operation.
- Current routing generation.

Request signatures must bind the method, route, request content, request ID, and freshness data.

Define bounded replay protection and expiration for API authentication. This protection is separate from Mac-side unlock challenge consumption.

A routing grant permits communication only. The Mac independently checks local trust before accepting an unlock authorization.

Do not introduce a persistent server-side pairing database.

## 7. Unlock Flow

1. User taps **Unlock** on iPhone.
2. iPhone requests a challenge through the relay.
3. Relay forwards the authenticated request over the Mac's WebSocket.
4. Mac checks that the requesting iPhone is locally trusted.
5. Mac generates a cryptographically random nonce and stores the pending challenge in memory.
6. Mac signs the challenge.
7. Relay returns the challenge to iPhone.
8. iPhone verifies the signature using the Mac key saved during pairing.
9. iPhone verifies that the challenge matches the current attempt, device pair, and action.
10. iPhone requests system user authentication.
11. After successful authentication, iPhone signs an authorization bound to the complete challenge.
12. Relay forwards the authorization to Mac.
13. Mac verifies the trusted iPhone's signature, challenge binding, expiration, and unused state.
14. Mac atomically consumes the challenge.
15. Mac invokes the validated local unlock mechanism.
16. Mac returns a signed result.
17. iPhone verifies the result and displays the outcome.

### Signed challenge fields

- Protocol version.
- Message purpose.
- Attempt ID.
- Mac ID.
- iPhone ID.
- Action: `unlock`.
- Random nonce.
- Challenge lifetime.

The authorization must bind the complete challenge and intended action.

Use canonical encoding and distinct signing contexts for:

- Connection authentication.
- API authentication.
- Routing grants.
- Unlock challenges.
- Unlock authorizations.
- Status responses.
- Operation results.

Document exact encoding, algorithms, key formats, and signature formats before implementing clients independently.

### Challenge lifecycle

- Initial configurable lifetime: 60 seconds.
- Enforce expiration on Mac using a local monotonic deadline.
- Do not extend expiration when an authorization arrives late.
- Consume atomically before attempting to unlock.
- Reject unknown, expired, consumed, or mismatched challenges.
- Bound outstanding challenges per phone and globally.
- Invalidate affected challenges when trust is revoked.
- Invalidate all outstanding challenges when the Mac agent restarts.

A failed attempt requires a fresh challenge and another explicit user action.

## 8. API Outline

| Endpoint | Purpose |
|---|---|
| `WSS /v1/mac/connect` | Authenticate and connect a Mac |
| `POST /v1/status` | Request fresh signed status |
| `POST /v1/unlock/challenge` | Request a signed unlock challenge |
| `POST /v1/unlock/authorize` | Submit signed unlock authorization |

Use HTTPS for every API request.

Validate request authentication and routing permission before forwarding.

Pending HTTP requests may wait for a bounded period for a correlated WebSocket response.

Bound:

- Request body and frame sizes.
- Concurrent pending requests.
- Request duration.
- Outgoing queues.
- Replay-protection state.
- Memory use per connection and device.

Do not persist pending operations or deliver them after reconnection.

## 9. Device Status

Represent connectivity and session state separately.

| Property | Values |
|---|---|
| Connectivity | Online, unavailable |
| Session | Locked, unlocked, unknown |

A live WebSocket establishes relay connectivity. It does not independently prove the current session state.

### Behavior

- Request fresh signed status when the iOS app opens.
- Bind the response to a fresh iPhone request nonce and device identities.
- Send session-change notifications through the existing WebSocket.
- Treat notifications as refresh hints when necessary.
- Mark connectivity unavailable after connection loss.
- Do not present stale session state as current.
- Refresh status after an unlock attempt with an uncertain result.

Validate session-state detection on the supported macOS version.

Display sleep, screensaver activity, and an inactive session are not sufficient substitutes for a reliable lock-state check.

Return `unknown` when the state cannot be established reliably.

Status must never substitute for authorization checks.

## 10. Results and Failure Handling

| Condition | Behavior |
|---|---|
| Unlock confirmed | Display success |
| Session already unlocked | Return an explicit already-unlocked result |
| Invalid signature | Reject |
| Untrusted iPhone | Reject |
| Unknown, expired, or consumed challenge | Reject |
| User cancels authentication | Stop the attempt |
| Mac disconnected | Return unavailable |
| System unlock mechanism fails | Return failure |
| Authorization submitted but response lost | Display outcome unknown and refresh status |

Bind signed results to the attempt ID and device identities.

Never automatically resend an unlock authorization after a timeout.

A later unlocked status establishes current state; it does not prove which action caused it.

An accepted request or authorization decision alone is insufficient to report successful unlocking.

## 11. Trust Revocation

Provide a Mac interface for removing trusted iPhones.

When a phone is removed:

- Reject subsequent unlock authorizations from its key.
- Invalidate its outstanding challenges.
- Update routing authorization when immediate relay-access revocation is required.

Incrementing the Mac's routing generation invalidates previous routing grants. Provide replacement grants to remaining trusted phones through an authenticated mechanism.

Deleting the iPhone app does not itself revoke trust on Mac.

Document recovery when a phone is lost or its keys become unavailable.

## 12. Reconnection and Heartbeats

- Use exponential backoff with fresh random jitter.
- Initial reconnect window: 0–10 seconds.
- Expand retry windows to 20, 40, then 60 seconds.
- Reset backoff after a stable connection.
- Send heartbeats approximately once per minute with staggered timing.
- Detect and close dead connections.
- Limit concurrent authentication handshakes and signature checks.

After reconnecting, refresh state. Never replay an earlier unlock command.

Test mass reconnection after relay restart.

## 13. Resource Budget

Target a Linux VPS with 1 vCPU and 1 GB RAM.

- Run one Go service under systemd.
- Avoid unnecessary background services.
- Disable WebSocket compression.
- Use small buffers and pooled write buffers where appropriate.
- Bound queues and message sizes.
- Disconnect slow consumers.
- Avoid busy polling.
- Avoid payload and heartbeat logging.
- Start with default GC behavior and tune from measurements.
- Evaluate an initial Go soft memory limit around 600 MiB while preserving headroom for the OS and networking.

The Go soft memory limit does not cap all process or kernel memory.

### Required measurements

- Process RSS.
- Total system memory, including socket overhead.
- Idle CPU.
- Memory growth per connection.
- Request latency.
- TLS handshake load.
- File descriptor usage.
- Recovery under connection loss and reconnect storms.

Test 1,000 and 5,000 real TLS/WebSocket connections using a separate load generator.

Five thousand connections is a test target, not a promised capacity.

## 14. Security Model

Assume the relay can be compromised.

A compromised relay may interrupt service or observe metadata, but must not be able to:

- Add a trusted iPhone.
- Replace a pinned peer key.
- Forge an unlock authorization.
- Successfully replay an authorization.
- Redirect authorization to a different Mac or action.
- Forge fresh, verified Mac status.
- Forge a successful operation result.

Use TLS and signed device messages.

Signatures authenticate messages; they do not hide payloads from the relay. If payload confidentiality is required, select an established end-to-end encryption scheme rather than inventing one.

Do not store the following on the relay:

- Account passwords.
- Biometric data.
- Private keys.
- Challenge or authorization payloads in logs.
- IP history in application storage.

Review reverse-proxy and infrastructure access logging separately.

## 15. Implementation Order

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

Do not proceed past the first milestone if safe passwordless session unlocking remains unproven.

## 16. Acceptance Criteria

- [ ] Passwordless unlock works on the declared supported macOS version.
- [ ] Ordinary local unlock works when Wakeveil is unavailable or crashes.
- [ ] Installation, removal, and recovery are documented and tested.
- [ ] BLE pairing works without internet.
- [ ] Mismatched verification codes abort pairing.
- [ ] Peer keys are persisted only after successful pairing.
- [ ] Device trust survives application and relay restarts.
- [ ] Unlock signing requires system-enforced user authentication.
- [ ] Forged, wrong-device, expired, and replayed authorizations are rejected.
- [ ] Concurrent copies cause at most one unlock execution attempt.
- [ ] Removing a trusted phone prevents further unlock authorization.
- [ ] Relay compromise cannot establish new device trust.
- [ ] Status distinguishes unavailable from unknown.
- [ ] Lost responses produce an uncertain outcome instead of false success.
- [ ] Unlock authorizations are never automatically retried.
- [ ] Relay restart requires no manual pairing.
- [ ] Requests, queues, challenges, and replay state remain bounded.
- [ ] Capacity and reconnection behavior are documented from measurements.

## Deliverables

- macOS feasibility prototype and compatibility report.
- Swift iOS application.
- Swift macOS agent and validated authorization integration.
- Go relay service.
- Wire protocol and cryptographic format documentation.
- Device trust and threat model documentation.
- Automated protocol and failure-path tests.
- VPS deployment and upgrade instructions.
- macOS installation, uninstall, and recovery instructions.
- Load-test scripts and measured resource-use report.

## First Action for the Implementation Agent

Start with the isolated macOS unlocking proof of concept.

Report the supported macOS version, tested mechanism, observed limitations, recovery procedure, and evidence that ordinary password unlock remains functional before implementing the rest of the product.
