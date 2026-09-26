# Track 5 — AI / Data Science: Ask the Drop

**Stack:** Python 3.11+ (stdlib is enough). pandas, scikit-learn, or an API-backed assistant
are fine. `make check` must pass **offline** — no network, no API keys.
**Sprint time:** 120–150 minutes (90 if the event is 2 hours).
**Tools:** AI chatbots such as ChatGPT may be used only for assistance. Automated AI agents, autonomous systems, IDE-integrated AI chatbots/agents, and tools that generate substantial or complete solution code are restricted. You must be able to explain and modify your work live. A sealed constraint is released at 0:15.
**Kit:** a launch-day event log, a question file, a labeled train set, and a short policy note.
There is no live model and no Reservation API in this zip.

---

## The story

Support is drowning. People keep asking the same things during the drop: *how many are left?*,
*did I get one?*, *why did my spot bounce back?* A well-meaning intern pasted the log into a
chat assistant. It invented a stock number that was never in the file. Three people thought
they had a reservation they did not have.

Your job is to build a small **Ask the Drop** service that answers only from the log — and a
**fair-queue ranker** that surfaces users who are gaming reservations (hold-then-release,
shared devices, bursts). A confident wrong number is worse than “I don’t know.”

---

## What you're given

Under `ai/`:

- `data/events.jsonl` — one launch-day log. Stock snapshots, reservation status changes,
  device and IP fields, authority mode changes.
- `data/questions.jsonl` — questions asked *during* the drop. Each row has `askedAt`, the
  asking `userId`, the `text`, and structured slots (`itemId`, `aboutUserId`) when they apply.
- `data/intent_train.jsonl` — labeled examples of the six intents. Use them. Do not assume
  the eval questions use the same wording.
- `data/users_train.jsonl` — a labeled sample of users (`gamer` | `legit`) for the ranker.
  This is **not** the full population. The holdout is unlabeled on purpose.
- `data/policy.md` — the published reservation rules. Policy questions are answered from
  this file, not from the log.
- `ask.py` — a skeleton that writes dummy output. Replace it.
- `run.sh` — what `make check` calls. Keep the output paths it already uses.

You do **not** get answers, eval intents, or holdout labels.

---

## Core task (required)

### Part A — Grounded answers

Implement `ai/out/answers.jsonl` with **one row per question**, same `questionId` order as
the input (any order is fine as long as every id is present).

```
{
  "questionId": "q-001",
  "intent": "stock_now",
  "answerable": true,
  "text": "12 left for hype-001.",
  "value": 12,
  "freshness": "current",
  "citations": ["e-1042", "e-1048"]
}
```

| Intent | When to use it | What `value` is |
|---|---|---|
| `stock_now` | remaining units for an item | integer remaining, or omit if unanswerable |
| `my_status` | the asking user’s reservation for an item | `confirmed` / `pending` / `rejected` / `reversed` / `released` / `none` |
| `why_reversed` | why their reservation was reversed | the `reason` string from the log |
| `count_pending` | how many reservations are pending for an item | integer count |
| `policy` | what the rules allow | a short sentence from `policy.md` |
| `other` | anything else, including another user’s data | `null` — and `answerable` must be `false` |

**Grounding rules (never break these):**

1. **No future facts.** You may only use events with `ts <= askedAt`. Citations must be
   event ids from that prefix (or `"policy"` for policy answers).
2. **No invented numbers.** A `stock_now` value must equal the stock reconstructed from the
   log at `askedAt` (see below). If you cannot reconstruct it, `answerable` is `false`.
3. **No other user’s reservations.** If `aboutUserId` is set and is not the asking user,
   `answerable` is `false`, intent is `other`.
4. **Unknown over confident-wrong.** “When is the next drop?”, items that never appear,
   reasons that are not in the log — `answerable` is `false`, `freshness` is `none`,
   `value` is `null`. Do not guess.
5. **Empty reservation is a fact.** If the asking user has no reservation for that item at
   `askedAt`, `my_status` is answerable and `value` is `"none"`.

**How to reconstruct stock at time T**

1. Take the latest `stock_snapshot` for the item with `ts <= T`. If none exists, the
   question is not answerable.
2. Then apply later events up to T: `reservation_confirmed` decreases `available` by `qty`;
   `reservation_released` and `reservation_reversed` increase it by `qty`.
3. Using only the snapshot, and ignoring later confirms, will be wrong on some questions.

**How to reconstruct a reservation’s status at time T**

Walk that `reservationId`’s events with `ts <= T`. Latest type wins:
`reservation_pending` / `_created` → `pending`; `_confirmed` → `confirmed`;
`_rejected` → `rejected`; `_reversed` → `reversed`; `_released` → `released`.

`count_pending` is the number of reservations whose status at T is `pending` for that item.

### Part B — Fair-queue ranker

Implement `ai/out/scores.jsonl` with **one row per user who appears in the event log**:

```
{"userId": "u-17", "score": 0.91, "reasons": ["rapid_release", "shared_device"]}
```

`score` is in `[0, 1]`. **1.0 = most likely gaming the queue.** `reasons` are short tags
you define; they are for the defense, not the gate.

Train labels are a hint, not a complete taxonomy. Obvious patterns in the log include
rapid reserve-then-release, many users on one `deviceId`, and tight bursts of attempts.
Your ranker must **separate** those users from people who reserved once on their own
device and kept it. A constant `0.5` fails.

Do **not** train on the eval questions or on events after a question’s `askedAt` when
answering that question. The ranker may use the full log — it is an offline batch job
after the drop.

### Requirements

- **Offline and deterministic.** `ai/run.sh` must produce byte-identical output if run
  twice. No live model calls inside `run.sh`.
- **Do not modify** `checks/` or `data/events.jsonl`. You may add features, models, and
  notes under `ai/`.
- **Runtime** under 60 seconds on the provided log.

### Acceptance criteria

- Every question has an answer row with a published intent.
- Grounding rules 1–5 hold on the whole question file.
- The ranker scores every user in the log and separates high-signal gaming from clean
  one-and-done users (the runner’s separation check).
- `make check TRACK=ai` is green, twice.

---

## Sealed constraint

At 0:15 you will receive one extra grounding rule that is **not** in this brief. It
changes when a stock number may be treated as current. Budget for it.

## Stretch goals (pick one; required for a senior rating)

- **A. Slots from text.** Ignore `itemId` / `aboutUserId` on the question file and recover
  them from `text`. `make check` still uses the structured file; show your text-only
  answers in `ai/out/answers_from_text.jsonl` and a 10-row error analysis.
- **B. Feature note.** `FEATURES.md`: each ranker feature, how you blocked leakage, and
  one feature you refused to use (and why).
- **C. Two heads.** A second score `no_show` (likely to release after confirm) in addition
  to `gamer`. Keep both in `scores.jsonl` (`noShow` field). Be ready to defend the labels
  you invented for it.

---

## Automated checks (part of your 60%)

`make check TRACK=ai` runs `ai/run.sh`, then:

| Check | What it does |
|---|---|
| `answers_complete` | One answer per question; required fields present. |
| `intents_obvious` | On clearly worded questions, `intent` matches the taxonomy. |
| `no_future_leak` | Every citation event has `ts <= askedAt`. |
| `no_invented_stock` | Every answerable `stock_now` value matches reconstructed stock. |
| `unknown_when_absent` | Next-drop / other-user / missing-item / missing-reason → not answerable. |
| `status_and_pending` | `my_status` and `count_pending` match the log at `askedAt`. |
| `ranker_complete` | Every log user has a score in `[0, 1]`. |
| `ranker_separates` | High-signal gaming users score higher than clean one-and-done users. |
| `deterministic` | A second run writes the same answers and scores. |

You may run the suite as often as you like.

---

## Panel review (part of your 40%)

| Correctness | Craft | Judgment |
|---|---|---|
| Numbers come from the log; unknown is used; ranker separates real patterns | Clear features or rules; reproducible; no hidden network calls | What you would not let an assistant answer in production, and why |

---

## Tips

- Reconstruct, then phrase. Do not let the assistant write the number first.
- A keyword intent model will clear `intents_obvious`. The hard part is stock and status.
- If a library needs a download, do not use it — the freeze machine is offline.
- The defense will change one rule (a window, a feature, or a refusal) and ask you to apply it.
