# Track 3 — Platform / SRE: Survive the spike, make the outage visible and safe

**Stack:** Docker Compose (or kind), OpenTelemetry, Grafana/Prometheus, any scripting or IaC.
**Sprint time:** 120–150 minutes (90 if the event is 2 hours).
**Tools:** AI chatbots such as ChatGPT may be used only for assistance. Automated AI agents, autonomous systems, IDE-integrated AI chatbots/agents, and tools that generate substantial or complete solution code are restricted. You must be able to explain and modify your work live. A sealed constraint is released at 0:15.
**Kit:** you get a running Reservation API as a **binary / image**, not its source.

---

## The story

At 00:12 the Central Authority stalls. Nobody notices for four minutes because the only signal is
"the app feels slow." By the time someone opens a dashboard, the drop is over and support has
300 tickets asking "did I get it or not?"

Your job: make the outage **visible in seconds**, make the system's response to it **automatic and
safe**, and prove it with a repeatable demo.

---

## What you're given

Under `platform/`:

- `docker-compose.yml` bringing up the mock authority, a **reference Reservation API** (the
  organisers' working implementation of the Backend track, with stand-in mode already built),
  Postgres, an OTel Collector, Prometheus, and Grafana with no dashboards.
- The Reservation API already emits OTel **traces** and a handful of **metrics** —
  `reservations_total{status,mode}`, `authority_request_duration_seconds`,
  `standin_queue_depth`, `reconciliation_lag_seconds` — but the collector pipeline is misconfigured
  and nothing reaches Prometheus.
- A `load/` folder with a `k6` script that simulates a 2-minute drop with a mid-run authority
  outage.
- An empty `chaos/` folder.

---

## Core task (required)

### Part A — See it

1. Fix the collector pipeline so traces and metrics flow end-to-end.
2. Build **one Grafana dashboard** ("Launch Day") with, at minimum:
   - Reservation rate split by `status` and `mode` (live vs stand-in).
   - Authority request latency p50 / p95 / p99.
   - `standin_queue_depth` and `reconciliation_lag_seconds`.
   - Current mode indicator (live / stand-in) and authority health.
3. Define **two SLOs** as recording rules and show burn on the dashboard:
   - Availability: proportion of `POST /reservations` that return 201 or 202 (not 5xx).
   - Latency: p95 of `POST /reservations` under 500 ms.

### Part B — React to it

4. Write **one alert rule** that fires when the system is in stand-in mode for more than 60 s
   **or** `reconciliation_lag_seconds` exceeds 120. Route it somewhere visible (Grafana alert
   list is fine; webhook to a local endpoint is better).
5. Configure the Reservation API's breaker via environment (`AUTHORITY_BREAKER_*`) so the switch to
   stand-in happens within **5 seconds** of the authority going `down`, and the switch back
   happens only after **3 consecutive healthy checks**. Show these transitions on the dashboard.

### Part C — Prove it

6. Write `chaos/outage-demo.sh` that, in one command:
   - Starts the k6 load.
   - Toggles the authority `slow` at 30 s, `down` at 60 s, `healthy` at 120 s.
   - Prints the mode transitions, alert firing time, and final reconciliation result.
7. The dashboard must tell the story without explanation: a viewer should see the degradation,
   the mode switch, the queue building, and the drain.

### Acceptance criteria

- `make up` brings the stack to green; `make demo` runs the outage demo end-to-end.
- Dashboards and alert rules are **provisioned as code** (files in the repo), not clicked into
  Grafana.
- No secrets in images or the repo; anything sensitive comes from `.env.example` → `.env`.
- The alert fires during the demo and clears after recovery.

---

## Sealed constraint

At 0:15 you will receive one extra requirement that is **not** in this brief. It will change when
an alert is allowed to fire. Budget for it.

## Stretch goals (pick one; required for a senior rating)

- **A. Canary with auto-rollback.** Run two versions of the Reservation API behind a local proxy
  (Traefik/Envoy/nginx), shift 10% of traffic to v2, and roll back automatically when v2's SLO
  burn exceeds a threshold. Provide a broken `v2` image to show the rollback.
- **B. Chaos toolkit.** Extend `chaos/` with network-level faults (latency, packet loss via `tc` or
  Toxiproxy) between the API and the authority, and add them as scenarios to the demo.
- **C. Cost & capacity note.** Add `CAPACITY.md`: from the k6 results, estimate the request rate at
  which the single API instance saturates and what you'd autoscale on.

---

## Automated checks (part of your 60%)

`make check TRACK=platform` runs:

| Check | What it does |
|---|---|
| `pipeline_up` | Queries Prometheus for the four named metrics; expects data within 60 s of start. |
| `dashboard_provisioned` | Verifies a dashboard titled "Launch Day" exists via Grafana API and was loaded from a file. |
| `slo_rules_present` | Checks the two recording rules and one alert rule are loaded. |
| `mode_switch_timing` | Toggles `down`; expects `/health` `mode=standin` within 5 s. Toggles `healthy`; expects `mode=live` only after ≥ 3 checks. |
| `alert_fires` | Runs the demo; expects the alert in `firing` state during the outage and `inactive` after. |
| `no_secrets` | Scans repo and images for common secret patterns. |

---

## Panel review (part of your 40%)

| Correctness | Craft | Judgment |
|---|---|---|
| Signals are the *right* signals; breaker timings behave | Everything as code, reproducible, tidy compose/provisioning | Why these SLO targets, what you'd alert on vs. dashboard, what's noise |

---

## Tips

- Nothing else matters until data flows.
- The dashboard must tell the outage story without you narrating it.
- Judges will run the demo themselves. Keep it deterministic.
