# First Milestone — macOS Unlock Feasibility Report

| | |
| --- | --- |
| **Status** | Not started |
| **Recommendation** | _pending_ — GO / NO-GO |
| **Declared macOS version** | _e.g. macOS 27.0 (build), Apple silicon_ |
| **Test environment** | _disposable VM / spare Mac — fill in_ |
| **Last updated** | _YYYY-MM-DD_ |

> Goal ([spec §1](spec.md#1-first-milestone-validate-macos-unlocking)): unlock an existing, locked macOS user session on an awake Mac without storing or submitting the account password.
> Screen wake, screensaver dismissal, synthetic keyboard input, a fake lock screen, or authenticating inside our own app do **not** count.

## 1. Summary for the report

The spec's *First Action* asks for exactly these five items:

| Item | Answer |
| --- | --- |
| Supported macOS version | _TBD_ |
| Tested mechanism | _TBD_ |
| Observed limitations | _TBD_ |
| Recovery procedure | _TBD_ — see §6 |
| Evidence that ordinary password unlock still works | _TBD_ — see §5 |

## 2. Scope

| Scenario | In MVP |
| --- | --- |
| Awake Mac, existing locked user session | **Yes** |
| Waking a sleeping Mac over the internet | No |
| FileVault preboot unlock | No |
| Initial login after restart | No |

## 3. Research

Current Apple documentation only. Don't assume a general-purpose unlock API exists.

### 3.1 Authorization plug-in mechanism

_TBD_ — `AuthorizationPlugin` / mechanisms, SecurityAgent lifecycle, privileged vs non-privileged mechanisms, which rules govern screen unlock (`system.login.screensaver` and whatever it delegates to on the declared version).

### 3.2 Rule configuration and the standard path

_TBD_ — the current rule definition (`security authorizationdb read …`). How a Wakeveil mechanism can be added **without** removing or weakening the password mechanism.

### 3.3 Failure behaviour

_TBD_ — what the system does when the plug-in is missing, can't reach the agent, times out or crashes. It must fall back to ordinary password unlock.

### 3.4 Local IPC boundary

_TBD_ — how the authorization component authenticates the agent (code-signing requirement, XPC peer validation), so that an unrelated local process can't authorize unlocking ([spec §2](spec.md#2-architecture)).

### 3.5 Session-state detection

_TBD_ — a reliable locked / unlocked signal on the declared version ([spec §9](spec.md#9-device-status)). Display sleep, screensaver activity and an inactive session are not substitutes. When does it have to report `unknown`?

### 3.6 FileVault, Fast User Switching, multiple users

_TBD_ — which configurations are unsupported, and how the app detects and reports them.

## 4. Proof of concept

Code: [`poc/macos-unlock/`](../poc/macos-unlock/)

- **Mechanism:** _TBD_
- **Trigger used in the PoC:** _local only; no networking_
- **How "session actually unlocked" was confirmed:** _TBD (system signal, not UI observation)_
- **Repeated lock/unlock cycles:** _N cycles, results_
- **Evidence:** _logs (`log show` excerpts), screen recordings_

## 5. Ordinary unlock preserved

Each case is tested on the declared version.

| Case | Password unlock works? | Evidence |
| --- | --- | --- |
| Wakeveil installed, idle | | |
| Agent not running | | |
| Agent unreachable / hung | | |
| Authorization component crashes | | |
| Component removed (uninstalled) | | |
| Repeated remote unlocks, then password unlock | | |
| Restart → normal login | | |

## 6. Installation, removal, recovery

- **Install** (requires explicit administrator authorization): _steps; what is changed; backup of the original rule_
- **Uninstall:** _steps; how the original rule is restored; verification_
- **Recovery** if the lock screen misbehaves: _e.g. SSH, a second admin account, Recovery OS; exact commands_

Scripts: [`scripts/`](../scripts/)

## 7. Compatibility

| macOS | Build | Chip | FileVault | Result | Notes |
| --- | --- | --- | --- | --- | --- |
| | | | | | |

## 8. Risks

| Risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- |
| macOS update changes authorization behaviour | | | |
| Broken component affects the lock screen | | | |
| Session-state signal unreliable | | | |

## 9. Recommendation

**GO / NO-GO:** _pending_

_Reasoning. If NO-GO: the exact blocker, the evidence, and the simplest viable alternative. The rest of the product does not start._

## References

- _Apple documentation links_
