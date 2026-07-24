# Research archive

The design in [`../plan.md`](../plan.md) did not appear from nowhere. These are the three surveys behind it, in the order they were produced.

**How to read these.** They are **snapshots of a design process**, not final conclusions. The project was rescoped partway through — from an internal evaluation system to an open reference design — so some early judgements have since been overturned. **Known corrections are listed below**; the documents themselves are left intact so the reasoning stays legible.

| # | Document | When | Question it answered |
|---|---|---|---|
| 01 | [Ecosystem landscape](01-sim-eval-landscape.md) | start | Who is working on sim eval, and along which lines? |
| 02 | [Red-team review](02-plan-red-team.md) | after the first draft plan | How could this plan fail? What is being underestimated? |
| 03 | [Humanoid frameworks survey](03-humanoid-frameworks.md) | after finding SIMPLE | Build it, or adopt something existing? |

---

## Known corrections

### 01 — Ecosystem landscape

- **The picture of the Isaac ecosystem is too static.** Later checking found that Isaac Lab has frozen its `main` branch and is migrating its physics layer from PhysX to Newton, whose primary backend is MuJoCo-Warp. That shift became clear only after this document was written.

### 02 — Red-team review

- **The cost and compute section no longer applies.** It assumed a compute-constrained team and recommended deferring Isaac on that basis; the team in fact has a 4090 cluster, so the constraint does not hold.
- **The commercial-licensing items in the "must fix" list no longer apply** — the project is open source with no commercialisation.
- Its **analysis of Isaac ecosystem churn, statistical power, and Goodhart effects remains valid**, and has since been reinforced by further evidence.

### 03 — Humanoid frameworks survey

- **The document itself corrects one significant error**: SIMPLE does not ship GEAR-SONIC. The item is still an unchecked TODO in its README, and its simulation path uses decoupled WBC.
- **Its central recommendation — fork SIMPLE as the foundation — was later overturned.** The reason is render throughput: roughly 4 fps of ray tracing on a single GPU, two to three orders of magnitude short of the target statistical scale (about 1,030 rollouts for a ±2-point confidence interval). SIMPLE is now a **task-design reference** (MIT-licensed, legitimately borrowable) rather than a dependency.

---

## One narrative that was thrown out

Midway through, a handful of GitHub issues led to a "VLA reproducibility crisis" framing. **That framing was rejected.** The sourcing was biased: GitHub issues are a complaint channel, and nobody opens one to report that things worked.

Checking the other direction gave a more accurate picture: **models run, leaderboards get published, comparisons hold.** What is fragile is reproducing a specific headline number exactly — a protocol problem, not a capability problem. The supporting evidence is in `../plan.md` §4.

The lesson is worth recording on its own: **searching deliberately for negative evidence systematically overstates how bad things are.**
