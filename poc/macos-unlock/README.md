# macOS screen-unlock prototype (Phase 0)

Smallest possible, isolated prototype that answers one question: can an already-logged-in, locked macOS session be unlocked through a legitimate authorization mechanism, triggered locally by our own code?

Findings go to [`docs/feasibility.md`](../../docs/feasibility.md).

## Rules

- Run only on a disposable macOS VM or spare Mac.
- Never modify the authorization database automatically; every change is a manual, documented, reversible step.
- Back up the original `system.login.screensaver` rule before any change.
- Keep the built-in password mechanism in the rule at all times.
- A result counts only if the real lock screen is dismissed and the desktop session is usable. Screen wake, screensaver dismissal and synthetic input do not count.

## Layout

_Empty until the approach is chosen in an ADR._
