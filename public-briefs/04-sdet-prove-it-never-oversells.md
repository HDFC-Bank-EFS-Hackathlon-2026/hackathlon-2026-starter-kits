# Track 4 — Test Engineering: Prove it never oversells

**Stack:** Go (`testing` + `testify`/`gopter`), Java (JUnit 5 + `jqwik`/`awaitility`), or Kotlin
(`kotest` + coroutines) — the starter kit ships skeletons for all three. Any HTTP client and load
tool is fine.
**Sprint time:** 120–150 minutes (90 if the event is 2 hours).
**Tools:** AI chatbots such as ChatGPT may be used only for assistance. Automated AI agents, autonomous systems, IDE-integrated AI chatbots/agents, and tools that generate substantial or complete solution code are restricted. You must be able to explain and modify your work live. A sealed constraint is released at 0:15.
**Kit:** `reference` and `buggy` are provided as **binaries / images**. You do not get their source.

---

## The story

Every backend engineer in the building says their Reservation API "can't oversell." Your job is
to be the person who proves it — or breaks it. Your suite will be run against **real
submissions** from the Backend track at the freeze, and it's the thing that decides whether their
claim is true.

Write tests that assert **invariants**, not screen text. Write tests that are **fast**, **not
flaky**, and produce a report a non-engineer could read.

---

## What you're given

Under `tests/<go|java|kotlin>/`:

- A skeleton project with an `ApiClient` (Reservation API `:8080`) and `AuthorityAdmin`
  (`:9000/admin/mode`) already implemented.
- Two target services to test against:
  - `reference` — the organisers' correct Reservation API.
  - `buggy` — a deliberately broken build with **at least four** hidden defects. Your suite
    should catch them.
- A `make check` command that runs your suite against both and emits `report/invariants.md`.
- Seed data: item `hype-001` with stock 50, 200 users.

---

## Core task (required)

Write a suite that asserts the following invariants against the Reservation API. Each invariant
must be a **separately reported** test with a clear name.

| ID | Invariant | Minimum scenario |
|---|---|---|
| **I1** | Same `Idempotency-Key` + user → exactly one reservation, same `reservationId`, same final status | 50 sequential replays **and** 50 concurrent replays |
| **I2** | Never oversell: `confirmed + pending ≤ stock` for an item, `available` never negative | 200 concurrent reservations for stock 50; assert exactly 50 confirmed, 150 rejected |
| **I3** | Reconciliation completeness: after an outage ends, **no** reservation remains `pending`, none are lost, none duplicated | Authority `down` → N pendings → `healthy` → poll until settled (bounded wait) |
| **I4** | Status monotonicity: `pending → confirmed \| reversed` only; `confirmed`/`rejected` never change | Observe every reservation from I3 over time |
| **I5** | Key mismatch: same key, different body → `422` and **no** new reservation | One test |
| **I6** | Health honesty: `/health.mode` reflects the authority within 5 s of a toggle | `healthy → down → healthy` |

### Requirements

- **Deterministic waits.** No `sleep(5)` and hope. Use bounded polling with a clear timeout and a
  failure message that says what was still pending.
- **Isolation.** Each test resets state (the kit provides `POST /admin/reset` on both services) and
  uses fresh users/keys. Tests must pass in any order and in parallel.
- **Authority always restored.** Every test that toggles the authority leaves it `healthy`, even on
  failure.
- **Readable report.** `report/invariants.md` lists each invariant, PASS/FAIL per target
  (`reference`, `buggy`), and for failures: the observed vs. expected numbers.
- **Zero flakes.** `make check` will be run **three times** at the freeze. Any test that gives
  different results across runs on `reference` counts against you.

### Acceptance criteria

- All I1–I6 pass on `reference`.
- Your suite finds **at least 3 of the 4** hidden defects in `buggy`, each as a distinct failing
  invariant.
- Total runtime under **4 minutes** per target.

---

## Sealed constraint

At 0:15 you will receive one extra invariant that is **not** in this brief. Budget for it.

## Stretch goals (pick one; required for a senior rating)

- **A. Property-based sequences.** Generate random sequences of `{reserve, replay-key, toggle
  authority, reconcile}` actions and assert I1–I4 after every step, with **shrinking** to the
  smallest failing sequence. (`gopter`, `jqwik`, or `kotest` property testing.)
- **B. Contract test.** Using the mobile skeleton's `ReservationRepository` as the consumer, write
  a consumer-driven contract (Pact or hand-rolled schema assertions) and run it against both
  targets. Report any field the API returns that the client doesn't handle, and vice versa.
- **C. Crash safety.** Kill the API container mid-queue (`docker kill`) during I3, restart it, and
  assert I3 still holds. Turn it into a repeatable scenario.

---

## Automated checks (part of your 60%)

`make check TRACK=sdet` runs:

| Check | What it does |
|---|---|
| `passes_reference` | Your suite passes I1–I6 on `reference`, three consecutive runs. |
| `catches_buggy` | Counts distinct hidden defects your suite exposes on `buggy` (0–4). |
| `runtime_budget` | Under 4 minutes per target. |
| `restores_authority` | After your suite, authority mode is `healthy`. |
| `report_present` | `report/invariants.md` exists and lists all six invariants. |

At the freeze, your suite is **also** run against anonymised Backend-track submissions. Defects
you find there are reported to those participants' judges — and credited to you.

---

## Panel review (part of your 40%)

| Correctness | Craft | Judgment |
|---|---|---|
| Invariants are asserted precisely; concurrency is real, not sequential | Test names read like specs, helpers are reusable, no magic sleeps | What you'd add for a real launch, which invariant you'd trust least and why |

---

## Tips

- Assert **properties** (never oversell, exactly one key, complete replay). Screen text is not an invariant.
- A flaky test is a bug in the suite. The freeze runs your suite three times.
- Think about what a tired engineer would get wrong about retries, transactions and restarts —
  without reading a second implementation. You will not be given one.
