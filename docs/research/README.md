# Research archive

The current design in [`../plan.md`](../plan.md) grew out of the three surveys below. They are snapshots of a design process, not current compatibility guarantees. Historical judgements remain intact; clear factual or arithmetic errors are corrected in place and the current decision is recorded here.

| # | Document | When | Question it answered |
|---|---|---|---|
| 01 | [Ecosystem landscape](01-sim-eval-landscape.md) | start | Who is working on sim eval, and along which lines? |
| 02 | [Red-team review](02-plan-red-team.md) | after the first draft plan | How could this plan fail? What is being underestimated? |
| 03 | [Humanoid frameworks survey](03-humanoid-frameworks.md) | after finding SIMPLE | Build it, or adopt something existing? |

---

## Known corrections

### 01 — Ecosystem landscape

- **Isaac Lab is now multi-backend; it is not simply replacing PhysX with Newton.** At the checked July 2026 state, PhysX remains the default Isaac Sim path, while Newton-MJWarp, Newton-Kamino and OvPhysX are selectable backends with different maturity and task coverage. “`main` was frozen” describes a repository transition, not removal of PhysX.
- **The original LIBERO episode counts were wrong.** The OpenVLA protocol uses 50 rollouts per task, 500 per 10-task suite and 2,000 across four suites for one seed, not 500 per task and 2,000 per suite.
- **MuJoCo does not require an external renderer.** It has a native OpenGL visualizer with onscreen and offscreen rendering. External renderers remain optional for higher-end photorealism or specialized sensor pipelines.

### 02 — Red-team review

- **The cost and compute section no longer determines the roadmap.** It assumed a compute-constrained team and recommended deferring Isaac on that basis; the team has a 4090 cluster, so that premise does not hold.
- **Commercial product licensing is not the immediate scope driver**, because the project is open source with no planned commercialisation. Redistribution, attribution, model-weight, dataset, asset, and bundled-component obligations still apply; the license audit remains required.
- Its analysis of ecosystem churn, statistical power, and Goodhart effects remains useful, but every dated platform claim must be rechecked before use in a bundle.

### 03 — Humanoid frameworks survey

- **SIMPLE does not ship GEAR-SONIC in its simulation path.** The roadmap item was unchecked at the survey revision, and the implemented path used decoupled WBC.
- **The recommendation to fork SIMPLE as the foundation was overturned.** Roughly 4 fps of ray-traced rendering can make large evaluations expensive, but fps and rollout count are not directly comparable. Wall time must be estimated from episode steps, achieved parallelism, rendering cadence, and trial count. SIMPLE remains a task-design reference rather than a current dependency.

---

## Superseded narratives

A handful of GitHub issues originally led to a “VLA reproducibility crisis” framing. That framing was rejected because issue trackers systematically over-sample failures. The more defensible conclusion is narrower: models and leaderboards exist, while exact headline-number reproduction is fragile when checkpoints, transforms, seeds, reset distributions, action scheduling, and simulator details are underspecified.

The current project therefore does not infer compatibility or credibility from model-family names, catalog entries, or a single score. It requires pinned artifact lineage, a native oracle, typed runtime profiles, and conformance evidence.
