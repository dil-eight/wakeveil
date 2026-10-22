# macOS unlock proof of concept (first milestone)

An isolated prototype that answers one question: can an existing, locked macOS user session on an awake Mac be unlocked through the authorization plug-in mechanism, without storing or submitting the account password?

No networking. The trigger is local only. Findings go to [`docs/feasibility.md`](../../docs/feasibility.md).

## Rules

- Run only on a disposable macOS VM or spare Mac, on an explicitly declared macOS version.
- Privileged installation requires explicit administrator authorization. Nothing changes the authorization database automatically.
- Back up the original rule before any change. Never remove or weaken the password mechanism.
- Never disable SIP or FileVault. Never build a fake lock screen.
- A missing, hung or crashing component must leave ordinary password unlock working.
- A result counts only if the session is confirmed unlocked by a system signal. Screen wake, screensaver dismissal and synthetic input do not count.

## Layout

_Empty until the mechanism is chosen in an ADR._
