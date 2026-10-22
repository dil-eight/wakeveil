# Phase 0 — macOS Unlock Feasibility

| | |
| --- | --- |
| **Status** | Not started |
| **Recommendation** | _pending_ — GO / NO-GO |
| **Target macOS** | 26, 27 |
| **Test environment** | _disposable VM / spare Mac — fill in_ |
| **Last updated** | _YYYY-MM-DD_ |

> Only a real, verified unlock of an already-logged-in, locked graphical session counts.
> Screen wake, screensaver dismissal, synthetic keyboard input, or authenticating inside our own app do **not**.

## 1. Question

Is there a supported, reliable mechanism that lets a background component on the Mac complete a screen-unlock transaction for the existing user session, without the user's password being typed or transmitted?

## 2. Scope

| # | Scenario | In MVP |
| --- | --- | --- |
| 1 | Authenticating inside the application | No (baseline only) |
| 2 | Unlocking an existing, locked macOS graphical session | **Yes** |
| 3 | Logging into macOS after a reboot | No |
| 4 | Unlocking a FileVault-protected startup volume | No |

## 3. Research findings

For each item: what Apple documents, what was observed, and the source.

### 3.1 Authorization Services

_TBD_

### 3.2 `Security.AuthorizationPlugin` and SecurityAgent plugin lifecycle

_TBD_

### 3.3 `system.login.screensaver` rule

_TBD_ — current rule definition (`security authorizationdb read system.login.screensaver`), mechanisms, and how it changed across macOS versions.

### 3.4 `SFAuthorizationPluginView`

_TBD_

### 3.5 Screen-unlock policy configuration

_TBD_

### 3.6 FileVault restrictions

_TBD_

### 3.7 Fast User Switching and multiple users

_TBD_

### 3.8 Sleep and wake behavior

_TBD_

### 3.9 Process lifecycle while locked

_TBD_ — which of LaunchAgent / LaunchDaemon / plugin stays alive and reachable at the lock screen.

### 3.10 Credential fallback (only if passwordless is infeasible)

_TBD_ — can a Keychain item be read while the session is locked, and is there a supported path to hand it to the system?

## 4. Proof of concept

Code: [`poc/macos-unlock/`](../poc/macos-unlock/)

- **Approach:** _TBD_
- **How the unlock was verified (system signal):** _TBD_
- **Result:** _TBD_
- **Evidence:** _logs, screen recordings, `log show` excerpts_

## 5. Compatibility matrix

| macOS | Build | Apple silicon / Intel | FileVault | Result | Notes |
| --- | --- | --- | --- | --- | --- |
| 26.x | | | | | |
| 27.x | | | | | |

## 6. Safety and recovery

- Backup of the original authorization rule: _TBD_
- Install procedure (requires explicit admin approval): _TBD_
- Uninstall procedure: _TBD_
- Emergency recovery (e.g. SSH, Recovery OS, second admin account): _TBD_

Recovery is verified by the system tests in [`spec.md` §11](spec.md#11-tests).

## 7. Risks

| Risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- |
| macOS update changes authorization behavior | | | |
| Broken plugin blocks the lock screen | | | |
| | | | |

## 8. Recommendation

**GO / NO-GO:** _pending_

_Reasoning, blockers with evidence, and the simplest viable alternative if NO-GO._

## References

- _Apple documentation links_
