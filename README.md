# HDFC Bank EFS Hackathlon 2026 — Official Participant Starter Kits

This is the **official public distribution repository** for Hackathlon 2026.

Participants do **not** need a GitHub account to download these files.

## 1. Download only your assigned track

| Track | Problem statement | Starter kit |
|---|---|---|
| Mobile — Live Drops | [Read brief](./public-briefs/01-mobile-live-drops.md) | [Download mobile.tar.gz](./kits/mobile.tar.gz) |
| Backend — Sell-out Safe | [Read brief](./public-briefs/02-backend-sell-out-safe.md) | [Download backend.tar.gz](./kits/backend.tar.gz) |
| Platform / SRE — Survive the Spike | [Read brief](./public-briefs/03-platform-survive-the-spike.md) | [Download platform.tar.gz](./kits/platform.tar.gz) |
| SDET / Test Engineering — Prove It Never Oversells | [Read brief](./public-briefs/04-sdet-prove-it-never-oversells.md) | [Download sdet.tar.gz](./kits/sdet.tar.gz) |
| AI / Data Science — Ask the Drop | [Read brief](./public-briefs/05-ai-ask-the-drop.md) | [Download ai.tar.gz](./kits/ai.tar.gz) |

Read the [Overview & Rules](./public-briefs/00-overview-and-rules.md) before starting.

## 2. AI tool rule — authoritative

AI chatbots such as ChatGPT may be used **only for assistance**, such as explanation, debugging guidance, or review.

The following are restricted:
- automated AI agents or autonomous systems;
- IDE-integrated AI chatbots/agents that generate or modify substantial portions of the solution;
- tools or workflows that generate substantial or complete working code/solutions on the participant's behalf.

Every participant must be able to explain and modify the submitted implementation during the live defense.

If any file inside a kit contains older or conflicting wording, **this repository README and the Overview & Rules are authoritative**.

## 3. Work locally

Extract the archive for your assigned track and follow its README.

Typical commands are:

```bash
make up
make check
```

Use the exact track-specific commands documented inside your kit.

Do not upload the starter kit or your solution to a public repository.

## 4. Submission

No participant GitHub account is required.

Before the freeze:
1. Complete `SUBMISSION.md`.
2. Run the required track checks.
3. Make a final local Git commit:
   ```bash
   git add .
   git commit -m "FINAL SUBMISSION"
   git rev-parse HEAD > FINAL_COMMIT.txt
   ```
4. Create a Git bundle:
   ```bash
   git bundle create <ParticipantId>-<track>.bundle --all
   ```
5. Create `<ParticipantId>-<track>.zip` containing your completed solution, `SUBMISSION.md`, and `FINAL_COMMIT.txt`.
6. Upload both the `.zip` and `.bundle` to the official submission channel announced by the organizers before the cutoff.

The organizer receipt timestamp is authoritative.

## 5. Integrity checks

SHA-256 checksums for all published participant artifacts are in [SHA256SUMS.txt](./SHA256SUMS.txt).

On Linux/macOS:
```bash
sha256sum -c SHA256SUMS.txt
```

On Windows PowerShell, for a downloaded kit:
```powershell
Get-FileHash .\ai.tar.gz -Algorithm SHA256
```

Compare the result with the corresponding entry in `SHA256SUMS.txt`.

## 6. Sealed constraint

A track-specific sealed constraint will be released by organizers at the announced event time. It is intentionally not included here before release.

---

**Official repository:** https://github.com/HDFC-Bank-EFS-Hackathlon-2026/hackathlon-2026-starter-kits
