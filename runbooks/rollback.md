# Rollback — Vision Moderation

**Goal:** get production back to the last known-good model image, fast. Roll back first, diagnose later.

## 1. When to roll back (any one trigger)
- [ ] `AvailabilityBurnFast` paging (5xx burn ≥ 14.4x budget on 1h + 5m).
- [ ] `LatencyP99High` paging (p99 > 1s ≥ 5 min) and rising.
- [ ] `ModelVersionMismatch` paging (replicas serving an unexpected version).
- [ ] Quality proxy collapsed: moderation approve/block ratio off baseline by > 20% for 10 min.
- [ ] Smoke/canary-verify failed but a bad rollout reached prod anyway.

> If a deploy went out in the last 30 min and any trigger fires → **roll back now.**

## 2. How to roll back
- [ ] Find last known-good SHA: GitHub → Deployments → `production` → last successful.
- [ ] Trigger rollback (reverts to previous image, skips canary):
  ```bash
  gh workflow run rollback.sh -f env=production -f to_sha=<good-sha>
  # or, if run outside Actions:
  ./scripts/rollback.sh production <good-sha>
  ```
- [ ] Confirm the job reaches **full rollout** and all replicas report `<good-sha>`.

## 3. What to verify (must return to green)
- [ ] `AvailabilityBurnFast` / `AvailabilityBurnSlow` resolved.
- [ ] `LatencyP99High` resolved; p99 < 1s on the latency dashboard.
- [ ] `ModelVersionMismatch` cleared — registry == served version on every replica.
- [ ] Error-rate + latency dashboards back to baseline for 10 min.

## 4. Who to notify
- [ ] **#vision-moderation-incidents** (Slack) — post "rolling back prod to `<good-sha>`, reason: …".
- [ ] Page **on-call SRE** (PagerDuty) if not already engaged.
- [ ] Tag **model owner** and **EM** in the incident thread.

## 5. What NOT to do
- [ ] Do **not** roll *forward* / hotfix before root cause is known.
- [ ] Do **not** edit prod manifests by hand — use `rollback.sh`.
- [ ] Do **not** silence the alerts instead of fixing; ack, don't mute.
- [ ] Do **not** skip the canary on the eventual re-deploy.

## 6. When to roll forward
Re-deploy the fixed image **only when all hold:**
- [ ] Root cause identified and fixed with a test that reproduces it.
- [ ] Full pipeline green (lint → test → scan → smoke).
- [ ] Prod stable ≥ 30 min on the good SHA.
- [ ] Deploy during business hours, via normal canary (10% → verify → full).
