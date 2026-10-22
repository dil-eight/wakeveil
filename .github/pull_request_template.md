## Summary

<!-- What changes and why. Link the spec section, design section or ADR. -->

## Spec step

<!-- docs/spec.md §15 -->

- [ ] 1 — Validate macOS unlock mechanism and recovery path
- [ ] 2 — Cryptographic formats and pairing protocol
- [ ] 3 — BLE pairing and protected device storage
- [ ] 4 — Relay connection authentication and routing grants
- [ ] 5 — Signed challenges and authenticated iPhone signing
- [ ] 6 — Local unlock component integration
- [ ] 7 — Status and uncertain-outcome handling
- [ ] 8 — Revocation and recovery
- [ ] 9 — Protocol, failure-path and load tests
- [ ] 10 — Installation, deployment and upgrade docs
- [ ] Repo / docs / tooling

## Security impact

- [ ] Touches the authorization database, the authorization component, or privileged installation
- [ ] Touches keys, signing contexts, canonical encoding, or the wire protocol
- [ ] Touches pairing, trust storage, routing grants, or revocation
- [ ] Touches challenge lifecycle, replay protection, or result reporting
- [ ] Touches the local IPC boundary
- [ ] None of the above

<!-- If any box is ticked: what a compromised relay or a local process could do, and why ordinary password unlock stays available. -->

## Testing

<!-- Commands run, devices/VMs used, macOS/iOS versions. -->

- [ ] Unit / protocol tests
- [ ] Failure-path tests
- [ ] System test on a disposable macOS VM / spare Mac
- [ ] Physical devices (iPhone / iPhone Duo / Mac)
- [ ] Relay load test (connections, RSS, CPU)

## Checklist

- [ ] No secrets, keys, payloads or credentials in code, logs or fixtures
- [ ] Ordinary password unlock still works if this component is missing or crashes
- [ ] Nothing persists pending operations or retries an unlock automatically
- [ ] Queues, buffers and state added here are bounded
- [ ] Docs updated (spec, design, feasibility, ADR, install/recovery)
