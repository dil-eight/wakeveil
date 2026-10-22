# Design

UI and interaction design for the iPhone app (including **iPhone Duo**) and the Mac app. Behaviour and security rules come from [`spec.md`](spec.md); this document only describes how they look and feel. Where the two disagree, the spec wins.

## 1. Principles

1. **One action.** The app exists to unlock a Mac. On a Mac's screen, Unlock is the only prominent control.
2. **Honest status.** Success is shown only for a verified, signed result confirming the unlock ([spec §10](spec.md#10-results-and-failure-handling)). A sent request, an accepted authorization or a later "unlocked" status is not success. When the outcome is not known, say so.
3. **Connectivity and session are separate.** "Online" never implies "Locked" or "Unlocked". Never show stale session state as current ([spec §9](spec.md#9-device-status)).
4. **No silent retries.** Every attempt needs a fresh challenge and a new explicit tap plus system authentication. Nothing is resent after a timeout, a reconnect or a fold.
5. **Pairing happens in person.** Adding a device is always local, over Bluetooth, confirmed on both screens. No UI offers remote pairing.
6. **System first.** Use stock SwiftUI components, SF Symbols, Dynamic Type and system colors, so Liquid Glass, Dark Mode, accessibility and new form factors work without custom code.
7. **Authentication-agnostic.** Never hard-code "Face ID". Copy and symbols follow `LAContext.biometryType` and the selected policy: Face ID, Touch ID, or device passcode. iPhone Duo has Touch ID only.

## 2. Supported devices

| Device | User authentication | Layouts |
| --- | --- | --- |
| iPhone (Face ID models) | Face ID / passcode per policy | Compact |
| **iPhone Duo**, folded (outer display) | Touch ID (side button) / passcode per policy | Compact |
| **iPhone Duo**, unfolded (inner display) | Touch ID (side button) / passcode per policy | Regular width (two columns); compact when sharing the screen with another app |
| Mac | — (pairing confirmations happen on the unlocked Mac) | Window with sidebar |

iPhone Duo displays (from Apple's spec page): outer 5.4", 1398 × 2034 px; inner 7.6", 1878 × 2670 px. Both are 120 Hz with Always-On. Never branch on these numbers. Lay out by size class and the space you are given.

## 3. Foundations

### Color

System semantic colors only. Color never carries meaning on its own; every state also has a symbol and a text label.

| Role | Color |
| --- | --- |
| Locked | `.orange` |
| Unlocked; online dot | `.green` |
| In progress | `.accentColor` |
| Unavailable, unknown, expired | `.secondary` |
| Failure, rejected | `.red` |

### Type

SF Pro via text styles only (`.largeTitle`, `.title2`, `.headline`, `.body`, `.footnote`). Dynamic Type is supported up to accessibility sizes. At AX sizes, status rows stack vertically.

### Symbols

| Meaning | SF Symbol |
| --- | --- |
| Mac | `laptopcomputer` |
| iPhone | `iphone` |
| Locked / unlocked | `lock.fill` / `lock.open.fill` |
| Session unknown | `questionmark.circle` |
| Unavailable | `wifi.slash` |
| Authentication | `faceid` / `touchid` / `lock.circle` (passcode) |
| Success | `checkmark.circle.fill` |
| Rejected / not verified | `xmark.shield.fill` |
| Expired | `clock.badge.exclamationmark` |
| Unlock failed | `exclamationmark.triangle.fill` |
| Nearby pairing | `antenna.radiowaves.left.and.right` |

Verify the names in the SF Symbols app for the deployment target.

### Controls

- Primary action: full-width `Button` with `.buttonStyle(.glassProminent)` and `.controlSize(.large)`, minimum height 50 pt.
- All tap targets are at least 44 × 44 pt.
- Haptics: `.success` on confirmed unlock; `.error` on rejected or failed; `.warning` on expired, unavailable or outcome unknown. Nothing else buzzes.

## 4. Status display (iPhone)

Each Mac shows two independent indicators: **connectivity** and **session**.

| Connectivity | Session | Shown as | Unlock button |
| --- | --- | --- | --- |
| Checking… | — | Checking… | Enabled |
| Online | Locked | ● Online · Locked | Primary |
| Online | Unlocked | ● Online · Unlocked | Secondary |
| Online | Unknown | ● Online · Status unknown | Primary |
| Unavailable | — | Unavailable · Status unknown | Disabled |

Rules:

- A fresh signed status is requested every time the app comes to the foreground, and the screen shows "Checking…" until it arrives.
- Session state comes only from a verified status response. A session-change notification triggers a fresh status request; it is never displayed directly.
- When connectivity becomes unavailable, the session label switches to "Status unknown" at once. The last known state is never kept on screen.
- Status is information only. It never enables or skips any authorization step.

## 5. Unlock flow (iPhone)

The flow follows [spec §7](spec.md#7-unlock-flow). The user sees three phases:

| Phase | Spec steps | UI | Cancel |
| --- | --- | --- | --- |
| Contacting Mac | 2–9 (challenge requested, Mac signature and binding verified) | "Contacting Home Mac…" with spinner | Yes; nothing has been signed |
| Authenticate | 10 | System Face ID / Touch ID / passcode sheet | Yes, from the system sheet; returns to idle without a message |
| Unlocking | 11–17 | "Unlocking…" with spinner | No; the authorization is already signed and sent |

### Results

| Condition ([spec §10](spec.md#10-results-and-failure-handling)) | Title | Symbol | Tint | Button |
| --- | --- | --- | --- | --- |
| Unlock confirmed (signed, verified result) | Unlocked | `checkmark.circle.fill` | green | Done |
| Session already unlocked | Already unlocked | `lock.open.fill` | green | Done |
| Unknown, expired or consumed challenge | Request expired | `clock.badge.exclamationmark` | secondary | Try Again |
| Mac's signature doesn't verify on iPhone | Couldn't verify Home Mac | `xmark.shield.fill` | red | Details |
| iPhone's authorization rejected (invalid signature) | Request rejected | `xmark.shield.fill` | red | Details |
| Untrusted iPhone or routing grant no longer valid | This iPhone isn't paired with Home Mac | `xmark.shield.fill` | red | Pair Again |
| User cancels authentication | — (back to idle) | — | — | — |
| Mac disconnected | Home Mac is unavailable | `wifi.slash` | secondary | — |
| System unlock mechanism failed | Couldn't unlock Home Mac | `exclamationmark.triangle.fill` | red | Try Again |
| Authorization sent, response lost | Outcome unknown | `questionmark.circle` | secondary | (status refresh, see below) |

Rules:

- **Outcome unknown** immediately requests a fresh status. The result then reads "Home Mac is unlocked", "Home Mac is still locked" (with Try Again), or "Status unknown". A later "unlocked" is shown as the current state, never as "your unlock worked".
- **Try Again** starts over from step 2: new attempt ID, new challenge, new system authentication.
- The challenge lives 60 s. If the system sheet is left open past that, the Mac rejects the late authorization and the result is "Request expired".
- Results stay up until Done, or 4 s for success. Then the screen returns to the status display.

## 6. iPhone screens

### 6.1 Mac list

The root of navigation. Each row shows the Mac name, the connectivity dot and the session label. The toolbar has **+ Add Mac** and Settings. With exactly one paired Mac, the app opens straight to its detail on compact width. With none, the empty state reads "Pair your Mac. Keep your iPhone next to it and open Wakeveil on both." with **Add Mac**.

### 6.2 Mac detail — compact (iPhone, iPhone Duo folded, Split View)

```text
┌────────────────────────────┐
│ ‹ Macs                 (i) │
│                            │
│      [laptopcomputer]      │
│          Home Mac          │
│                            │
│   ● Online   [lock] Locked │
│                            │
│                            │
│                            │
│  ┌──────────────────────┐  │
│  │      Unlock Mac      │  │
│  └──────────────────────┘  │
│                            │
└────────────────────────────┘
```

- The Unlock button sits in the bottom third, within thumb reach. On iPhone Duo folded, that is also close to the side-button Touch ID sensor.
- (i) opens Mac details (§6.6).

### 6.3 Mac detail — regular width (iPhone Duo unfolded)

```text
┌──────────────────────────────┬┬──────────────────────────────┐
│ Macs                    +    ││                          (i) │
│                              ││       [laptopcomputer]       │
│ > Home Mac                   ││           Home Mac           │
│   ● Online · Locked          ││                              │
│                              ││    ● Online   [lock] Locked  │
│   Studio Mac                 ││                              │
│   Unavailable · Unknown      ││  ┌────────────────────────┐  │
│                              ││  │       Unlock Mac       │  │
│                              ││  └────────────────────────┘  │
│   Settings                   ││                              │
└──────────────────────────────┴┴──────────────────────────────┘
                               ▲ fold
```

- `NavigationSplitView`: the sidebar is the Mac list, the detail is the selected Mac. It collapses to a stack automatically on compact width.
- The column boundary sits on the fold where the system reports fold geometry. If it can't be aligned, the fold region stays free of controls.
- Switch layouts by `horizontalSizeClass`, not by orientation or screen size. The inner display ignores the app's supported orientations, so portrait and landscape both happen.
- Unfolded but sharing the screen with another app (compact width): use the compact layout from §6.2.

### 6.4 iPhone Duo: fold, posture and continuity

- **Nothing on the fold.** Centre the Unlock button in its column, not on the screen. Take the fold position from the system (reserved regions / safe area where available), never from hard-coded coordinates. Leave a gutter of at least 16 pt either side of the fold with no interactive content.
- **Partially folded.** Touch near the crease is unreliable. Keep controls inside a column.
- **Fold or unfold mid-flow.** The attempt belongs to a model object, not to a view. Moving between displays must:
  - keep the current phase and result exactly as they were;
  - never request a new challenge, resend an authorization, or re-trigger system authentication.
  If the system dismisses the authentication sheet during the transition, treat it as a cancel and return to idle.
- **Under-display camera** on the inner screen: respect the safe area; nothing tappable underneath.
- **Touch ID on the side button.** No custom fingerprint UI or arrows; the system sheet already points to the sensor. In-app copy says "Touch ID" and uses `touchid`.
- **Testing:** the iPhone Duo simulator (fold, unfold, rotate, partial fold) plus a physical device.

### 6.5 Authentication step

- The unlock-signing key requires system-enforced user authentication for every signature. The app has no "authenticated" state of its own and never reuses an `LAContext` across attempts.
- Reason string: **Unlock “Home Mac”** (the paired Mac's name). Face ID devices also need `NSFaceIDUsageDescription`: "Wakeveil uses Face ID to approve each unlock of your Mac."
- Authentication not available or not enrolled: the button is replaced with "Set up Face ID / Touch ID / a passcode to unlock your Mac", with a link to Settings. The text matches the policy.
- Key invalidated (for example, the biometric set changed under a `.biometryCurrentSet` policy): "Your Face ID / Touch ID changed. Pair this iPhone with Home Mac again." with **Pair Again**. Pairing again happens next to the Mac (§6.7).

### 6.6 Mac details

The Mac's name, paired date, short key fingerprint and supported macOS version.

**Remove from This iPhone** is destructive. Its confirmation reads: "This deletes Home Mac's pairing from this iPhone. To stop this iPhone from unlocking Home Mac, also remove it on the Mac." Removing on the iPhone does not revoke trust on the Mac ([spec §11](spec.md#11-trust-revocation)).

### 6.7 Pairing (iPhone side)

Works without internet. Follows [spec §4](spec.md#4-bluetooth-pairing).

1. **+ Add Mac.** The first time, the system asks for Bluetooth permission.
2. **Looking for your Mac.** "On your Mac, open Wakeveil and choose Add iPhone. Keep your iPhone nearby." The screen shows `antenna.radiowaves.left.and.right`.
3. **Code.** "Does this code match the one on Home Mac?", with the code in large monospaced digits (length set by the pairing protocol). Buttons: **Codes Match** and **Cancel**.
4. **Waiting.** "Confirm the code on your Mac."
5. **Paired.** Then Home Mac's detail screen with status loading.

Errors. Every error ends pairing, and nothing is saved:

| Situation | Text |
| --- | --- |
| Bluetooth off | Turn on Bluetooth to add a Mac. |
| Bluetooth permission denied | Allow Bluetooth for Wakeveil in Settings to add a Mac. |
| Codes don't match, or cancelled on either device | Pairing cancelled. Nothing was saved. |
| Abandoned / timed out | Pairing timed out. Start again on both devices. |
| Another pairing in progress on the Mac | Your Mac is busy pairing another iPhone. Try again in a moment. |

### 6.8 Settings

- Authentication method (read-only): Face ID / Touch ID / passcode, enrolled or not.
- Relay server (advanced; read-only in the MVP).
- About, version, licences, link to the source code.

## 7. Mac app

### 7.1 Main window

```text
┌───────────────┬──────────────────────────────────────────┐
│ Overview      │ Home Mac                                 │
│ iPhones       │                                          │
│ Security      │ Relay          ● Connected               │
│               │ Remote unlock  Ready                     │
│               │ macOS          27.0 (supported)          │
│               │                                          │
│               │ Paired iPhones                           │
│               │ iPhone Duo       Paired Oct 22           │
│               │ iPhone 17 Pro    Paired Oct 22           │
│               │                                          │
│ + Add iPhone  │                                          │
└───────────────┴──────────────────────────────────────────┘
```

- `NavigationSplitView`: **Overview**, **iPhones**, **Security**.
- **Relay** shows Connected, Reconnecting (with time to next attempt), or Unavailable.
- **Remote unlock** shows Ready, Not installed, Not supported on this macOS version, or Error, with a link to recovery docs.
- **macOS** shows the running version and whether it is the declared supported version ([spec §1](spec.md#1-first-milestone-validate-macos-unlocking)).

### 7.2 Add iPhone (pairing, Mac side)

Only possible on the unlocked Mac, in person.

1. **+ Add iPhone** opens a sheet: "On your iPhone, open Wakeveil and tap Add Mac. Keep it next to this Mac." The first time, the system asks for Bluetooth permission.
2. **Code.** "Does this code match the one on your iPhone?", with the same large monospaced code and the iPhone's name and model. Buttons: **Codes Match** and **Cancel**.
3. **Done.** The iPhone appears under **iPhones**.

The errors match the iPhone side (§6.7), with the wording reversed.

### 7.3 iPhones and revocation

A list of trusted iPhones: name, model, paired date.

**Remove** is destructive. Its confirmation reads: "“iPhone Duo” will no longer be able to unlock this Mac. Other iPhones stay paired." Removal takes effect at once and invalidates that phone's outstanding challenges. Remaining phones get updated relay access automatically; there is no extra UI unless it fails.

A footer links to "Lost an iPhone?", the recovery documentation.

### 7.4 Security

- The unlock component's status, plus **Install…** and **Uninstall…** actions. Each requires administrator authorization.
- Links to installation, removal and recovery docs.
- A plain statement: **Your password always works on the lock screen.**

## 8. Accessibility

- VoiceOver reads each status block as one element ("Home Mac, online, locked" / "Home Mac, unavailable, status unknown"). The Unlock button's label includes the Mac's name. Phase changes and results are announced with `AccessibilityNotification.Announcement`.
- The pairing code is read digit by digit.
- Reduce Motion: no pulsing or scaling while in progress; a static spinner only.
- Every state has text plus a symbol, so it never depends on color.
- Full Dynamic Type, including AX sizes.
- Mac: full keyboard navigation. Return triggers the default button, except in destructive confirmations and in the pairing code check. There, the user must choose explicitly.

## 9. Copy

| Situation | Text |
| --- | --- |
| Primary button | Unlock Mac |
| Requesting challenge | Contacting Home Mac… |
| Authorization sent | Unlocking… |
| Confirmed | Unlocked |
| Already unlocked | Already unlocked |
| Unavailable | Home Mac is unavailable. It will show as online when it reconnects. |
| Status unknown | Status unknown |
| Outcome unknown | Outcome unknown. Checking Home Mac… |
| Expired | Request expired. Try again. |
| Unlock failed | Couldn't unlock Home Mac. Your password still works on the Mac. |
| Not paired | This iPhone isn't paired with Home Mac. Pair again next to the Mac. |

Tone: short, literal, no exclamation marks, no blame.

## 10. Open questions

- Unlock history isn't in the MVP spec. If it is added, keep it local to each device; the relay stores nothing.
- Lock Screen widget / Control Center control for Unlock. Both would still require system authentication in the app; decide after the core flow ships.
- Whether the iPhone Duo outer display's Always-On view should show a Mac's status. Privacy review needed.
