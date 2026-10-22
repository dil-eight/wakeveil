## Summary

<!-- What changes and why. Link the spec section or ADR. -->

## Milestone

- [ ] 1 — Feasibility
- [ ] 2 — Secure device communication
- [ ] 3 — iOS application
- [ ] 4 — System integration
- [ ] 5 — Production hardening
- [ ] Repo / tooling

## Security impact

- [ ] Touches the authorization database, an authorization plugin, or LaunchDaemon/LaunchAgent config
- [ ] Touches key generation, signing, verification, nonces or pairing
- [ ] Changes the wire protocol
- [ ] None of the above

<!-- If any box is ticked: what could go wrong, and how local password unlock stays available. -->

## Testing

<!-- Commands run, devices/VMs used, macOS versions. -->

- [ ] Unit tests
- [ ] Integration tests
- [ ] System test on a disposable macOS VM / spare Mac
- [ ] Physical device

## Checklist

- [ ] No secrets, keys, or credentials in code, logs or fixtures
- [ ] Local password unlock still works if this component fails
- [ ] Install / uninstall / recovery docs updated (if applicable)
- [ ] `docs/` updated (spec, feasibility, ADR)
