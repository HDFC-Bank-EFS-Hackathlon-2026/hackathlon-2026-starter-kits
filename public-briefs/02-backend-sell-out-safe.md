# Track 2 — Backend: Sell-out safe, make it idempotent and outage-safe

**Stack:** Go, Java (21+), or Kotlin — your kit ships all three.
**Sprint time:** 120–150 minutes (90 if the event is 2 hours).
**Tools:** AI chatbots such as ChatGPT may be used only for assistance. Automated AI agents, autonomous systems, IDE-integrated AI chatbots/agents, and tools that generate substantial or complete solution code are restricted. You must be able to explain and modify your work live. A sealed constraint is released at 0:15.
**Kit:** you receive only the student API and the authority. There is no reference implementation in your tree.

---

## The story

The Reservation API sits between the mobile app and the **Central Authority**, which owns the
real stock count. Today the API is a thin proxy — and it has two problems that will ruin launch
day:

1. When a client retries (bad network, double tap, load balancer hiccup), the API **reserves
   twice**.
2. When the authority goes `slow` or `down`, the API **fails every request**, and the drop dies.

Your job: make the Reservation API keep its promises when nothing else does.

---

## What you're given

Under `backend/<go|java|kotlin>/`:

- A running Reservation API that implements the contract in `00-overview-and-rules.md` — but with
  the two bugs above.
- A Postgres schema with `items` and `reservations` tables (no idempotency column, no queue).
- An `AuthorityClient` with a 2-second timeout and no retry, no breaker.
- A `make check` suite (written by the organisers) that currently **fails** — your goal is to make
  it pass without modifying it.

---

## Core task (required)

### Part A — Idempotency

Implement `Idempotency-Key` handling on `POST /reservations`:

- The same key from the same user **always** returns the same `reservationId` and the same final
  status, no matter how many times it's replayed — including concurrent replays.
- A key reused with a **different** body returns `422 idempotency_key_reused`.
- Keys are scoped to the user; two users may use the same key string.

### Part B — Stand-in mode

When the authority is unreachable or too slow, the API must keep accepting reservations locally:

- Maintain a **shadow available count** per item (starts equal to the authority's last known
  count).
- If the authority is `healthy`: pass through as today, update the shadow count from the response.
- If the authority is `slow` or `down` (health check failing or request timing out): switch to
  **stand-in** mode. Authorize against the shadow count, subject to a **stand-in limit** per item
  (config: `STANDIN_MAX_PER_ITEM`, default 10), return `202 pending`, and enqueue the reservation
  **durably** (a DB table is fine).
- If the shadow count or stand-in limit is exhausted: `409 rejected` with reason
  `standin_limit_reached` or `insufficient_stock`.
- When the authority returns to `healthy`, **replay** the queue in order. Each queued reservation
  becomes `confirmed` or — if the authority now refuses it — `reversed`. Replaying must itself be
  idempotent (use the same `reservationId` the authority already knows).
- `GET /health` reports `authority` and `mode` truthfully.
- Mode switching must be **automatic** (breaker or health-poll driven), and the switch back to
  `live` must not lose or duplicate anything that was queued.

### Invariants you must never break

```
I1  For every Idempotency-Key + user: exactly one reservation ever exists.
I2  confirmed(item) + pending(item) ≤ authority_available_at_last_sync(item) + 0   (never oversell locally)
I3  After replay completes: every pending reservation is confirmed or reversed. None remain pending, none are lost.
I4  A reservation's status only moves forward: pending → confirmed | reversed; confirmed and rejected are terminal.
I5  Under concurrent requests for the same item, I1–I4 still hold (no lost updates).
```

---

## Sealed constraint

At 0:15 you will receive one extra requirement that is **not** in this brief. It will change how
stand-in authorisation is limited. Budget for it.

## Stretch goal (required for a senior rating; only if core passes `make check`)

**Conflict resolution.** Put the authority in `conflicting` mode (it reports fewer units than your
shadow count). During replay, when a reservation is refused:

- Mark it `reversed`, call `POST /releases` on any partial hold if you created one, and write an
  **audit record** (`reservation_events` table) with who/what/when/why.
- Re-sync the shadow count from the authority's answer.
- Expose `GET /admin/audit?reservationId=` to show the trail.

---

## Automated checks (part of your 60%)

`make check TRACK=backend` runs, against your service with the mock authority toggled by the
runner:

| Check | What it does |
|---|---|
| `idem_replay` | Sends the same key 50× sequentially and 50× concurrently. Expects one reservation. |
| `idem_body_mismatch` | Same key, different body. Expects 422. |
| `race_single_item` | 200 concurrent reservations for an item with stock 50. Expects exactly 50 confirmed, 150 rejected, `available == 0`, never negative. |
| `standin_accept` | Authority `down`. Expects `202 pending` up to `STANDIN_MAX_PER_ITEM`, then `409`. |
| `standin_replay` | Authority `down` → 10 pendings → `healthy` → wait. Expects all 10 terminal; count of confirmed matches authority stock. |
| `standin_no_loss` | Kills the API process mid-queue and restarts it. Expects nothing lost or duplicated after replay. |
| `health_truthful` | Toggles modes; expects `/health` to reflect within 5 s. |
| `stretch_conflict` | (stretch) Authority `conflicting`. Expects `reversed` records and audit entries. |

You may run the suite as often as you like.

---

## Panel review (part of your 40%)

| Correctness | Craft | Judgment |
|---|---|---|
| Invariants hold under the runner; transactions/locking are right | Clear separation (handler / service / repository / authority client), named errors, tests you added | Why this breaker threshold, why this queue design, what you'd change for 100× traffic |

---

## Tips

- The checks assert **properties** (exactly one effect per key, never oversell, complete replay).
  How you get there is yours. The defense will change one constraint and ask you to adapt.
- `slow` and `down` are the same from the API’s point of view: the authority did not answer in time.
- If work is only in memory, `standin_no_loss` will fail.
