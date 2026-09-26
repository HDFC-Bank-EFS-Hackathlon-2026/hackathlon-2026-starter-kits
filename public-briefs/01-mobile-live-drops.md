# Track 1 — Mobile: Live Drops, reserve with optimistic UI

**Stack:** Android (Kotlin / Jetpack Compose), iOS (Swift / SwiftUI), or Flutter — your choice.
**Sprint time:** 120–150 minutes (90 if the event is 2 hours).
**Audience you're building for:** 16–24 year-olds who have never waited for a spinner in their lives.
**Tools:** AI chatbots such as ChatGPT may be used only for assistance. Automated AI agents, autonomous systems, IDE-integrated AI chatbots/agents, and tools that generate substantial or complete solution code are restricted. You must be able to explain and modify your work live. A sealed constraint is released at 0:15.

---

## The story

The **hype-001** drop goes live at the top of the hour. Fifty units, thousands of people, one
button. When someone taps **Reserve**, they expect the app to react *instantly* — not after the
server has finished talking to the central authority, which today is having a bad afternoon.

Your screen must feel immediate, stay honest, and never lie about whether someone got the item.

---

## What you're given

In the starter kit under `mobile/`:

- `flutter/`, `android/`, `ios/` — three shells, each wired to the Reservation API (`:8080`) with
  a working **item list** screen and an **unfinished item detail** screen.
- A `ReservationRepository` (or equivalent) with `getItem(id)`, `reserve(itemId, qty,
  idempotencyKey)`, and `getReservation(id)` already implemented against the API contract.
- Design tokens (colours, type scale, spacing) and a small icon set. No pixel-perfect mock — the
  look is yours.
- Mock authority toggles so you can test `slow`, `down`, and `conflicting` from your machine.

---

## Core task (required)

Complete the **item detail screen** with a working **Reserve** flow.

### Acceptance criteria

1. **Instant feedback.** On tap, the UI reflects the reservation immediately (optimistic state) —
   button state, stock counter, and a visible "reserved" indicator all update before any network
   response arrives.
2. **Honest reconciliation.** When the server responds:
   - `201 confirmed` → the optimistic state becomes confirmed (subtle, satisfying).
   - `202 pending` → the UI clearly shows "holding your spot" and keeps polling
     `GET /reservations/{id}` until it becomes `confirmed`, `rejected`, or `reversed`.
   - `409 rejected` or later `reversed` → the optimistic state is **reverted** (stock counter
     returns, button re-enables) with a clear, human message — no raw error codes.
   - Timeout or network failure → the request is **not duplicated** on retry; the same
     `Idempotency-Key` is reused, and the user is told what's happening.
3. **Works when the authority is `slow`.** Toggle the authority to `slow` and the flow must still
   feel responsive; the pending state must be visible, and a slow confirmation must land correctly.
4. **No double-tap double-reserve.** Rapid taps produce exactly one reservation.
5. **Accessibility basics.** Every interactive element has a label; the flow is usable with
   TalkBack / VoiceOver / a screen reader; touch targets ≥ 44 pt; text scales with system settings.

### Constraints

- Reservation state must be explicit and testable. A pile of booleans will not pass the defense.
- Keep the UI off the main-thread work. No spinner longer than 300 ms without a message.
- You choose the architecture. Judges will ask you to change one transition, not name a pattern.

---

## Sealed constraint

At 0:15 you will receive one extra requirement that is **not** in this brief. Budget for it.
Assistants will not have seen it in advance.

## Stretch goals (pick one; required for a senior rating, only after core is done)

- **A. Live stock counter.** Poll or stream `GET /items/{id}` and animate the "available" number
  smoothly as it drops, including the moment it hits zero.
- **B. Resumable flow.** Kill the app mid-`pending`; on relaunch the detail screen recovers the
  pending reservation and resumes polling. (Hint: persist the idempotency key.)
- **C. Social confirmation.** After `confirmed`, show a shareable image ("I got hype-001 #17/50")
  with a system share sheet.

---

## Automated checks (part of your 60%)

`make check TRACK=mobile` runs:

- Build succeeds for your chosen platform.
- ViewModel / reducer unit tests you wrote pass (at least the state transitions:
  idle → optimistic → confirmed / pending → confirmed / pending → reversed / rejected).
- A UI test (skeleton provided) taps Reserve twice quickly and asserts one network call.

---

## Panel review (part of your 40%)

Judges will read your code and your six-line `SUBMISSION.md`, then run a 4-minute defense:
**they** toggle the authority mode while you drive the app, then ask one mutation.

They are looking for:

| Correctness | Craft | Judgment |
|---|---|---|
| All state transitions handled, including revert | Explicit state model, readable Compose/SwiftUI/Flutter, tested ViewModel | Sensible polling strategy, message wording, what you cut and why |

---

## Tips

- Outcomes matter more than the pattern name. The defense will ask you to change a transition.
- The `slow` mode is how you show pending. The `conflicting` mode is how you show reversed.
- Copy matters. “Something went wrong” is a fail.
