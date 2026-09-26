# Launch Day Hackathon — Overview & Rules

Welcome. Over the next three hours you will build for the hardest moment any consumer product
faces: **launch day**. A limited drop goes live, thousands of people tap "Reserve" at once, and
somewhere in the middle of it the central inventory authority slows down, then disappears for a
minute, then comes back with a different opinion about how much stock is left.

Your job depends on your track — but every track is judged on the same three things:

1. **Correctness** — does it do the right thing, every time, including under failure?
2. **Craft** — is the code/test/config clear, structured, and something a teammate could change?
3. **Judgment** — did you make sensible trade-offs, and can you change the behaviour when the constraint changes?

AI chatbots such as ChatGPT may be used only for assistance, explanation, debugging guidance, and review.
Automated AI agents, autonomous systems, IDE-integrated AI chatbots/agents, and tools that generate
substantial or complete solution code are restricted. You must be able to explain and modify your
implementation during the live defense.

---

## Tracks

| # | Track | Title | Stack |
|---|---|---|---|
| 1 | Mobile | **Live Drops: reserve with optimistic UI** | Android (Kotlin/Compose), iOS (Swift/SwiftUI), or Flutter |
| 2 | Backend | **Sell-out safe: make it idempotent and outage-safe** | Go, Java, or Kotlin |
| 3 | Platform / SRE | **Survive the spike: make the outage visible and safe** | Docker, OpenTelemetry, Grafana, any IaC/scripting |
| 4 | Test Engineering | **Prove it never oversells** | Go, Java, or Kotlin test frameworks |
| 5 | AI / Data Science | **Ask the Drop: grounded answers + fair-queue ranker** | Python 3.11+ (offline) |

You compete **individually** within one track. Read your track brief after this document.

---

## Schedule (3-hour block)

| Time | Activity |
|---|---|
| 0:00 – 0:15 | Kickoff, rules, starter kit walkthrough, environment check |
| 0:15 | **Sealed constraint** released (one extra requirement per track, not in this brief) |
| 0:15 – 2:30 | Hands-on sprint. Mentors available on the floor / Discord |
| 2:30 – 2:50 | **Submission freeze.** Automated checks run against all submissions |
| 2:50 – 3:25 | **Live defense** (4 minutes each, in parallel per track): judge toggles a mode or an Ask-the-Drop rule, then asks one mutation |
| 3:25 – 3:30 | Wrap-up and next steps |

If the event is shortened to 2 hours, the sprint is 90 minutes and the sealed constraint is
released at kickoff. Defense stays. Stretch goals are required only for a senior rating.

You receive **only your track’s kit**. It does not contain other tracks’ solutions.

---

## The starter kit

Download **only your assigned track archive** from the official public starter-kit repository,
extract it locally, and work inside that folder. Do not fork or publish the kit to a public repository.

Typical commands are:

```
make up
make check
make authority MODE=down   # where applicable; see your track brief
```

Track-specific README files take precedence for exact commands.

What comes up:

| Component | Port | Notes |
|---|---|---|
| **Central Authority** (mock) | 9000 | Source of truth for item stock. Can be toggled into failure modes. |
| **Reservation API** | 8080 | What the mobile app talks to. Backend extends this; other tracks consume a provided build. |
| **Track-specific skeleton** | — | Only the files for *your* track (mobile shells, student API, platform stack, test harness, or Ask-the-Drop log). |
| **Observability (Platform kit)** | 3000 | Grafana `admin/admin`. Empty dashboards. Collector pipeline is unfinished. |
| **Ask the Drop log (AI kit)** | — | Offline event log, questions, labeled train users, policy note. No live model. |
| **Seed data** | — | 5 items with limited stock, 200 users, 1 "hyped" item with stock 50. |

### Central Authority modes

| Mode | Behaviour |
|---|---|
| `healthy` | Responds in < 20 ms. |
| `slow` | Responds in 3–8 seconds. |
| `down` | Refuses connections. |
| `conflicting` | Responds, but reports **lower** available stock than the Reservation API believes. |

Toggle: `POST http://localhost:9000/admin/mode` with `{"mode": "down"}`.

---

## API contracts

### Central Authority (mock, port 9000) — you do not modify this

```
GET  /health
     200 {"status":"ok"} | 503

GET  /items/{itemId}
     200 {"itemId":"hype-001","available":50}

POST /reservations
     body {"reservationId":"<uuid>","itemId":"hype-001","userId":"u-17","qty":1}
     201 {"reservationId":"...","status":"confirmed","available":49}
     409 {"reservationId":"...","status":"rejected","reason":"insufficient_stock","available":0}
     Replaying the same reservationId returns the original result (the authority is idempotent).

POST /releases
     body {"reservationId":"<uuid>"}
     200 {"reservationId":"...","status":"released","available":50}

POST /admin/mode
     body {"mode":"healthy|slow|down|conflicting"}
```

### Reservation API (port 8080) — Backend track extends this; others consume it

```
GET  /health
     200 {"status":"ok","authority":"healthy|degraded|unreachable","mode":"live|standin"}

GET  /items/{itemId}
     200 {"itemId":"hype-001","available":50,"shadowAvailable":50,"mode":"live"}

POST /reservations
     headers Idempotency-Key: <client-generated uuid>
     body {"itemId":"hype-001","userId":"u-17","qty":1}
     201 {"reservationId":"...","status":"confirmed","mode":"live"}
     202 {"reservationId":"...","status":"pending","mode":"standin"}      # accepted locally, awaiting authority
     409 {"reservationId":"...","status":"rejected","reason":"insufficient_stock"}
     Same Idempotency-Key must always return the same reservationId and final status.

GET  /reservations/{reservationId}
     200 {"reservationId":"...","status":"confirmed|pending|rejected|reversed","mode":"live|standin"}

GET  /reservations?userId=u-17
     200 [ ... ]

POST /admin/reconcile          # force a reconciliation pass (also runs automatically)
     200 {"replayed":12,"confirmed":11,"reversed":1}
```

Status meanings: `confirmed` — the authority accepted it; `pending` — accepted locally during an
outage, not yet replayed; `rejected` — refused; `reversed` — accepted locally, later refused by
the authority and compensated.

---

## Rules

1. Work individually. You may talk to mentors and other participants, but submit your own work.
   Another person may not drive your session.
2. You may use documentation and normal development libraries/tools. AI chatbots such as ChatGPT
   may be used only for assistance. Automated AI agents, autonomous systems, IDE-integrated AI
   chatbots/agents, and tools that generate substantial or complete solution code are restricted
   and may result in disqualification. You must be able to explain and modify your work live.
3. Do not modify the Central Authority mock or the automated check runner. Everything else in your
   track’s kit is yours.
4. Do not upload the starter kit to a public repository or a public assistant project. Private
   local tools are fine.
5. Submissions must run with `make up` and `make check` on a clean kit. If it does not run, it
   is not judged.
6. Submission freeze is hard. Your files must be received through the official submission channel
   before the cutoff; the organizer's receipt timestamp is authoritative.
7. Respect the code of conduct. Be kind; help people boot their kit.

---

## Submission

No GitHub account is required for participants.

Before the submission freeze:

1. Finish `SUBMISSION.md` with **exactly six lines**:
   - What you changed.
   - How to see it work (one command or one tap sequence).
   - What you would do next with one more week.
   - One risk or weakness you see in your own solution.
   - One thing in the starter kit you would fix if you owned it.
   - If you used permitted chatbot assistance, one thing it got wrong and how you caught it;
     otherwise write **I did not use an assistant**.
2. Run the track's required `make check` command.
3. Make a final local Git commit:
   ```
   git add .
   git commit -m "FINAL SUBMISSION"
   git rev-parse HEAD > FINAL_COMMIT.txt
   ```
4. Create a Git bundle and a ZIP using your Participant ID and track:
   ```
   git bundle create <ParticipantId>-<track>.bundle --all
   ```
   Create `<ParticipantId>-<track>.zip` containing your completed solution, `SUBMISSION.md`,
   and `FINAL_COMMIT.txt`.
5. Upload both the `.zip` and `.bundle` to the **official submission channel announced by the organizers**
   before the freeze.

The uploaded artifacts and organizer receipt timestamp constitute the official submission.

---

## Judging

| Weight | Component | How |
|---|---|---|
| 60% | **Automated gate** | `make check` invariant suite, build/test pass, and track-specific automated checks. Runs during the freeze. |
| 40% | **Live defense** | Six lines, then 4 minutes: the judge toggles an authority mode and asks one mutation. Scored on Correctness · Craft · Judgment. Permitted chatbot assistance is not a penalty; prohibited automation is handled under the event rules. |

Defense runs in parallel rooms, one per track. Senior rating additionally requires **one stretch
goal** (or a sealed-constraint solution that is clearly above the core bar).

Shortlisted participants are contacted within 24 hours for follow-up conversations.

---

## Mentor guidance

Mentors can help you boot the kit, read an error, or clarify a requirement. They will not tell you
how to solve the problem, which trade-off to make, or what to paste into an assistant. They will
not point you at another track’s files. That part is the interview.
