# Design

UI and interaction design for the iPhone app (including **iPhone Duo**) and the Mac app. Behaviour and security rules come from [`spec.md`](spec.md); this document only describes how they look and feel.

## 1. Principles

1. **One action.** The app exists to unlock one Mac. The Unlock button is the only prominent control on the home screen.
2. **Honest status.** Show "Unlocked" only after the Mac reports `unlocked`, which it does only after verifying the session is actually unlocked. Never show optimistic success. While waiting, say what is actually happening.
3. **No silent retries.** A failed, expired or unconfirmed request is never resent automatically. Trying again means a new request and a new biometric check.
4. **System first.** Use stock SwiftUI components, SF Symbols, Dynamic Type and system colors so Liquid Glass, Dark Mode, accessibility and new form factors work without custom code.
5. **Biometry-agnostic.** Never hard-code "Face ID". Read `LAContext.biometryType` and use the matching name, symbol and copy. iPhone Duo has Touch ID only.

## 2. Supported devices

| Device | Biometry | Layouts |
| --- | --- | --- |
| iPhone (Face ID models) | Face ID | Compact |
| **iPhone Duo**, folded (outer display) | Touch ID, side button | Compact |
| **iPhone Duo**, unfolded (inner display) | Touch ID, side button | Regular width (two panes); compact when sharing the screen with another app |
| Mac | Touch ID / password, for local confirmations only | Window with sidebar |

iPhone Duo displays (from Apple's spec page): outer 5.4", 1398 × 2034 px; inner 7.6", 1878 × 2670 px. Both are 120 Hz with Always-On. Never branch on these numbers. Lay out by size class and the space you are given.

## 3. Foundations

### Color

System semantic colors only, so light/dark and Increase Contrast work for free. Color never carries meaning on its own; every state also has a symbol and a text label.

| Role | Color |
| --- | --- |
| Locked | `.orange` |
| Unlocked, online | `.green` |
| Pending | `.accentColor` |
| Offline, expired, unknown | `.secondary` |
| Failure | `.red` |

### Type

SF Pro via text styles only (`.largeTitle`, `.title2`, `.headline`, `.body`, `.footnote`). Dynamic Type is supported up to accessibility sizes. At AX sizes the home screen stacks vertically and the history row moves its time under the status.

### Symbols

| Meaning | SF Symbol |
| --- | --- |
| Mac | `laptopcomputer` |
| iPhone | `iphone` |
| Locked / unlocked | `lock.fill` / `lock.open.fill` |
| Biometry | `faceid` / `touchid` (from `biometryType`) |
| Offline | `wifi.slash` |
| Success | `checkmark.circle.fill` |
| Not authorized | `xmark.shield.fill` |
| Expired | `clock.badge.exclamationmark` |
| Rejected | `hand.raised.fill` |
| Not supported | `exclamationmark.triangle.fill` |
| No confirmation | `questionmark.circle` |
| Pair | `qrcode.viewfinder` |

### Controls

- Primary action: full-width `Button` with `.buttonStyle(.glassProminent)` and `.controlSize(.large)`, minimum height 50 pt.
- All tap targets are at least 44 × 44 pt.
- Haptics: `.success` on `unlocked`, `.error` on `authorization_failed` / `rejected`, `.warning` on `expired` / no confirmation. Nothing else buzzes.

## 4. Unlock states (iPhone)

The home screen is a state machine. Every protocol status from [`spec.md` §5](spec.md#5-remote-unlock-protocol) maps to exactly one visual state.

| State | Source | Title | Symbol | Tint | Button |
| --- | --- | --- | --- | --- | --- |
| Locked | Mac presence | Locked | `lock.fill` | orange | **Unlock Mac** |
| Unlocked | Mac presence | Unlocked | `lock.open.fill` | green | Unlock Mac (secondary) |
| Offline | Mac presence / `mac_offline` | Mac is offline | `wifi.slash` | secondary | disabled |
| Authenticating | local | system biometric sheet | — | — | hidden |
| Sending | local | Sending… | spinner | accent | disabled |
| `received` | Mac | Mac received request | spinner | accent | disabled |
| `authorized` | Mac | Unlocking… | spinner | accent | disabled |
| `unlocked` | Mac | Unlocked | `checkmark.circle.fill` | green | Done |
| `already_unlocked` | Mac | Already unlocked | `lock.open.fill` | green | Done |
| `expired` | Mac | Request expired | `clock.badge.exclamationmark` | secondary | Try Again |
| No confirmation | local, past `expiresAt` | No confirmation. Your Mac may still be locked. | `questionmark.circle` | secondary | Try Again |
| `authorization_failed` | Mac | Not authorized | `xmark.shield.fill` | red | Details |
| `rejected` | Mac | Request rejected | `hand.raised.fill` | red | Details |
| `not_supported` | Mac | This Mac can't be unlocked remotely | `exclamationmark.triangle.fill` | red | Details |

Rules:

- Pending states can't be cancelled after sending: the request is already signed and on its way. Before sending (biometric sheet showing), cancelling just returns to idle.
- Result states stay up for 4 s, or until the user taps Done, then the home screen returns to presence state.
- "Try Again" always starts from the biometric step with a new request ID and nonce.

## 5. iPhone screens

### 5.1 Home — compact (iPhone, iPhone Duo folded, Split View)

```text
┌────────────────────────────┐
│ Wakeveil                ⚙  │
│                            │
│       [laptopcomputer]     │
│          Home Mac          │
│          ● Online          │
│                            │
│       [lock.fill]          │
│          Locked            │
│                            │
│  ┌──────────────────────┐  │
│  │      Unlock Mac      │  │
│  └──────────────────────┘  │
│    Last unlock 10:42 AM    │
│                            │
├────────────────────────────┤
│  Unlock   History  Devices │
└────────────────────────────┘
```

- The Unlock button sits in the bottom third, within thumb reach. On iPhone Duo folded, that is also close to the side-button Touch ID sensor.
- Tabs: **Unlock**, **History**, **Devices**. Settings is the gear in the toolbar.

### 5.2 Home — regular width (iPhone Duo unfolded)

```text
┌──────────────────────────────┬┬──────────────────────────────┐
│ Wakeveil                     ││                           ⚙  │
│                              ││ Recent attempts              │
│      [laptopcomputer]        ││                              │
│         Home Mac             ││ 10:42  Unlocked          ✓   │
│         ● Online             ││ 09:15  Request expired   !   │
│                              ││ Yesterday  Unlocked      ✓   │
│      [lock.fill] Locked      ││                              │
│                              ││ Home Mac                     │
│  ┌────────────────────────┐  ││ macOS 27.0 · Relay connected │
│  │       Unlock Mac       │  ││ Paired with this iPhone Duo  │
│  └────────────────────────┘  ││                              │
│    Last unlock 10:42 AM      ││                    See All › │
└──────────────────────────────┴┴──────────────────────────────┘
                               ▲ fold
```

- Two panes split **on the fold**: the primary pane holds status and Unlock, the secondary pane holds recent attempts and device details. With one Mac there is nothing to put in a list column, so this is a dashboard, not a master-detail layout.
- Navigation uses `TabView` with `.tabViewStyle(.sidebarAdaptable)`: a tab bar when compact, a sidebar (hidden by default) when regular.
- Switch layouts by `horizontalSizeClass`, not by orientation or screen size. The inner display ignores the app's supported orientations, so portrait and landscape both happen. Use `AnyLayout` (`HStackLayout` / `VStackLayout`) so the change animates.
- Unfolded but sharing the screen with another app (compact width): use the compact layout from 5.1.

### 5.3 iPhone Duo: fold, posture and continuity

- **Nothing on the fold.** Don't centre the Unlock button on the screen; centre it in its pane. Take the fold position from the system (reserved regions / safe area where available), never from hard-coded coordinates. Leave a gutter of at least 16 pt either side of the fold with no interactive content.
- **Partially folded.** Touch near the crease is unreliable. Keep the button inside a pane, and keep the pane divider on the fold.
- **Fold or unfold mid-flow.** The request lifecycle belongs to a model object, not to a view. Moving between displays must:
  - keep the current state (pending, result) exactly as it was;
  - never resend, duplicate or cancel an in-flight request;
  - never re-trigger the biometric prompt.
  If the system dismisses the biometric sheet during the transition, return to idle; the user taps Unlock again.
- **Under-display camera** on the inner screen: respect the safe area; nothing tappable underneath.
- **Touch ID on the side button.** Don't draw a custom fingerprint UI or arrows. The system biometric sheet already points to the sensor. In-app copy says "Touch ID" and uses the `touchid` symbol.
- **Testing:** the iPhone Duo simulator (fold, unfold, rotate, partial fold) plus a physical device before release.

### 5.4 Biometric step

- Reason string (`localizedReason`): **Unlock “Home Mac”** (the paired Mac's name).
- Face ID devices: `NSFaceIDUsageDescription` = "Wakeveil uses Face ID to approve each unlock of your Mac."
- No in-app reuse: every unlock creates a new `LAContext` and evaluates again.
- Biometry not available or not enrolled: the home screen replaces the button with "Set up Face ID / Touch ID to use Wakeveil" and a link to Settings.
- Biometric set changed (the key is invalidated by `.biometryCurrentSet`): show "Your Face ID / Touch ID changed. Pair this iPhone again to keep unlocking Home Mac.", with **Pair Again**.

### 5.5 Pairing

1. **Devices › Pair a Mac** → camera view with a `qrcode.viewfinder` frame and "Scan the code shown on your Mac."
2. After a scan: "Pair with “Home Mac”?" with a 6-digit verification code, also shown on the Mac. Primary: **Continue**.
3. Waiting: "Confirm on your Mac" with a spinner. The Mac must approve locally.
4. Done: "Paired" with **Unlock Mac** as the next action.

An expired or reused code reads "This code has expired. Show a new one on your Mac."

### 5.6 History

A grouped list (Today / Yesterday / date) of rows: time · result symbol + label · Mac name. Tapping a row shows request ID, issued/expires times, and the final status. It is read-only; nothing in it can be retried.

### 5.7 Devices and device details

- Paired Macs list (one in the MVP) → details: name, macOS version, relay status, paired date, key fingerprint (short form).
- **Remove This Mac**: destructive, with a confirmation alert.

### 5.8 Settings

- Biometry (read-only status: Face ID / Touch ID, enrolled or not).
- Passcode fallback: shown only if the security policy allows it; off by default.
- Relay server (advanced, read-only in the MVP).
- About, version, licences.

## 6. Mac app

### 6.1 Main window

```text
┌───────────────┬──────────────────────────────────────────┐
│ Overview      │ Home Mac                                 │
│ iPhones       │                                          │
│ History       │ Relay          ● Connected               │
│ Security      │ Screen         Unlocked                  │
│               │ Remote unlock  Ready                     │
│               │                                          │
│               │ Recent attempts                          │
│               │ 10:42  iPhone Duo   Unlocked          ✓  │
│               │ 09:15  iPhone Duo   Request expired   !  │
│               │                                          │
│ + Pair iPhone │                                          │
└───────────────┴──────────────────────────────────────────┘
```

- `NavigationSplitView`: **Overview**, **iPhones**, **History**, **Security**.
- "Screen" is shown only when the lock state is reliably known; otherwise the row is hidden, not guessed.
- "Remote unlock" shows `Ready`, `Not installed`, `Not supported on this configuration` (with a reason, e.g. FileVault), or `Error` with a link to recovery docs.

### 6.2 Pairing (Mac side)

1. **Pair iPhone** → sheet with a QR code, a countdown (the session is short-lived), and "Scan with Wakeveil on your iPhone."
2. After the iPhone scans: "Allow “iPhone Duo” to unlock this Mac?", with the same 6-digit code shown on the iPhone, the device model, and **Allow** / **Don't Allow**. **Allow** requires local authentication on the Mac (Touch ID or password).
3. Done → the iPhone appears under **iPhones**.

### 6.3 iPhones and revocation

A list of paired iPhones (name, model, paired date, last used). **Revoke** is destructive and asks for confirmation: "“iPhone Duo” will no longer be able to unlock this Mac." Revocation takes effect immediately, including for requests in flight.

### 6.4 Security

Shows the authorization integration status and the install / uninstall actions, each requiring administrator approval. It also links to the recovery documentation and says plainly that **your password always works on the lock screen**.

## 7. Accessibility

- VoiceOver: the status block reads as one element ("Home Mac, online, locked"). The Unlock button's label includes the Mac's name. State changes are announced with `AccessibilityNotification.Announcement`.
- Reduce Motion: no pulsing or scaling while pending; a static spinner only.
- Every state has text plus a symbol, so it never depends on color.
- Full Dynamic Type, including AX sizes (see §3).
- Mac: full keyboard navigation; Return triggers the default button in sheets, except destructive ones.

## 8. Copy

| Situation | Text |
| --- | --- |
| Primary button | Unlock Mac |
| Sent | Sending… |
| Mac acknowledged | Mac received request |
| Mac verified | Unlocking… |
| Success | Unlocked |
| Offline | Mac is offline. It will appear here when it reconnects. |
| No confirmation | No confirmation. Your Mac may still be locked. |
| Expired | Request expired. Try again. |
| Not authorized | This iPhone isn't authorized for this Mac. |
| Not supported | This Mac can't be unlocked remotely. See details. |

Tone: short, literal, no exclamation marks, no blame.

## 9. Open questions

- Lock Screen widget / Control Center control for Unlock. Both would still need to open the app for the biometric step; decide after Milestone 3.
- Multiple Macs: the regular-width layout would gain a Mac list column. Out of scope for the MVP.
- Whether the iPhone Duo outer display's Always-On view should show the Mac's presence. Privacy review needed.
