# crossrun — Design Document v2.2

> **What this is**: an open-source reference design for running and evaluating robot policies, in simulation and on hardware.
> **Status**: design phase. Architecture settled, code not started.
> **Companion research**: see [`research/`](research/).

**Version history.** v1.x was scoped as an internal evaluation system. v2.0 reframed it as an open reference design. v2.1 added the execution layer. **v2.2 stops inventing a protocol and adopts XPolicyLab's instead** — see §2. Judgements that were overturned along the way are kept in §11 rather than deleted.

---

## 1. Positioning

**In one line**: at this particular moment, redesign how robot policy sim/real execution and evaluation should work, and prove it with reproducible demos.

**Why now matters.** Most teams have either not invested in this yet, or invested years ago and now carry the architecture they started with. 2026 is a good moment to redo it because three things have settled independently: policy serving converged on client–server with per-model isolation; the physics layer is consolidating on Newton / MuJoCo-Warp; and evaluation methodology (perturbation tiers, confidence intervals, real-to-sim correlation) is established practice even if not yet standard practice.

**Deliverables, in priority order:**

1. A design others can read and borrow
2. Demos that make the argument concrete
3. A working evaluator — which carries the first two, and is not the point by itself

**Success looks like**: someone clones it, it runs, and they come away thinking this is how the problem should be approached.

---

## 2. What we build and what we adopt

This is the sharpest decision in the document, and it changed in v2.2.

| Layer | Decision | Why |
|---|---|---|
| **Model packaging** | Adopt — containers, XPolicyLab | Solved. 40+ policies already served with per-model dependency isolation |
| **Policy protocol** | **Adopt XPolicyLab's four operations** | Solved. Inventing a spec here would be the exact mistake this project argues against |
| **Execution layer** | **Build — this is the whole contribution** | Nobody owns it, including XPolicyLab |
| **Simulators / physics** | Adopt | Years of work; reinventing helps no one |
| **Task suites** | Adopt (LIBERO, and later others) | Comparability comes from using what the community uses |

**What "adopt the protocol" means concretely.** Model containers speak XPolicyLab's four operations — observation update, action prediction, policy reset, batched query. We do not publish a competing specification. The thin code we do write on the model side is adaptation, not standardisation: observation-key mapping (real camera keys never match rendered ones), action-space tagging (joint / EEF / SONIC latent token), and adapters for Arena's separate `isaaclab_arena_<policy>` convention.

**Why not build our own protocol.** The reasons that once justified it turned out weak. The concern that XPolicyLab's dictionary could not carry SONIC's 78-dimensional latent token is unverified and probably wrong — its schema constrains *pose* fields, while actions are ordinarily just float arrays. The fact that Arena has its own convention is true but argues for an adapter, not a new spec: you write that adapter either way, and inventing a protocol only adds a second thing to maintain. Container-first packaging is orthogonal to whose protocol you speak.

**When we would reconsider.** Phase 0 measures two things (§12): whether the schema genuinely cannot carry a latent-token action space, and whether batched-query throughput holds under load. **Absent that evidence, we do not build our own protocol.**

**A note on the division of labour.** RoboDojo, XPolicyLab's own companion benchmark, keeps its simulation client and its real-robot evaluation separate from XPolicyLab. Even its authors did not put the execution layer inside the policy server. That gap is where this project lives.

---

## 3. Design claims

Each of these should be visible in the code and defended by a demo.

1. **Adopt protocols, don't invent them.** A new specification needs adopters; an adapter needs none.
2. **Models don't live in this repo.** One container per model, or one bridge for a whole zoo. We hold neither.
3. **Backends need not be compatible with each other** — only with the protocol.
4. **The default path must be frictionless.** Heavy backends are always opt-in.
5. **Numbers ship with uncertainty.** A bare success rate is a bug.
6. **A high score under the standard protocol means little.** Perturbation tiers are not optional.
7. **Across backends, compare rankings — never absolute numbers.**
8. **Something must sit between models and environments.** Models infer, environments simulate; somebody owns the episode.
9. **Name the sim/real differences instead of abstracting them.** There are exactly three: where reset comes from, where success comes from, how time flows.

---

## 4. Baselines known to work

A reference design should start by saying what is definitely runnable. As surveyed in July 2026:

| Benchmark | Backend | Embodiment | Verified models | Checkpoints |
|---|---|---|---|---|
| **LIBERO** | robosuite / **MuJoCo** | Franka | π0, π0.5, OpenVLA, SmolVLA, Diffusion Policy | public |
| RoboDojo | Isaac Sim | ARX X5 / Piper / Piper X | **30 policies** via XPolicyLab (40+ integrated) | leaderboard |
| RoboLab | Isaac | — | π0.5, π0-FAST, π0, PaliGemma | — |
| RoboTwin | SAPIEN | Aloha-AgileX and others | several | public |
| SimplerEnv | SAPIEN | Google Robot / WidowX | RT-1, RT-1-X, Octo | public |
| **Genie Sim** | Isaac | **G2** | GO-2, Pi series, GR00T | public, **locally verified by us** |
| Arena | Isaac | multiple | GR00T N, π0/π0.5, SmolVLA | public |
| SIMPLE | MuJoCo + Isaac | G1, aloha, franka | ~10 including WAMs | — |
| **Genesis** | own | — | **no VLA ecosystem found** | none |

**The ecosystem is healthy.** RoboLab compared per-policy RoboLab-120 success rates against RoboArena Elo and found ranking perfectly preserved (Spearman ρ = 1.00), with scores positively correlated (Pearson r = 0.68). The numbers also discriminate cleanly: under colour perturbation π0.5 scores 96.7%, π0-FAST 93.3%, and π0 just 6.7%; under shadows, 100% / 90% / 0%.

What is fragile is *exact reproduction of a headline number* — a protocol problem, not a capability problem.

---

## 5. Backend strategy

A reference repo is judged on its first experience: clone, install, get a number. That decides the default.

```
Default path — frictionless, public checkpoints exist
└── MuJoCo / LIBERO ─────── pip-installable; π0.5, OpenVLA, SmolVLA all live here

Opt-in heavy backends — each with a reason nothing else covers
├── Isaac Lab-Arena ─────── exists for G1 + SONIC
└── Genie Sim ───────────── exists for G2 (locally verified)

Verification backends
├── MuJoCo sim2sim ──────── checks whether rankings are stable across backends
└── Genesis ─────────────── evaluation gate (§9), not a default
```

**Why not Genesis by default.** It installs lightly, is vendor-neutral across CUDA / Metal / Vulkan, ships its own path-traced renderer, and has real long-term promise — but there is no verified VLA checkpoint ecosystem on it today. The default path has to run something immediately.

**Why mjlab is out.** Its paper is explicit that RGB rendering for vision policies is a non-goal, favouring distillation from full-state privileged policies plus external rendering, and that cross-simulator portability is likewise a non-goal. Good for locomotion RL, wrong for VLA evaluation.

**Isaac's weight is paid only where it buys something.** That is an engineering judgement and a design claim at once: a reference design is worth what people can actually run.

---

## 6. Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  Policy providers — not in this repo                          │
│  All speak XPolicyLab's four operations                       │
│                                                               │
│   one model each                     a whole zoo              │
│  ┌────────┐ ┌────────┐ ┌────────┐  ╔══════════════════════╗  │
│  │ π0.5   │ │OpenVLA │ │ yours  │  ║ XPolicyLab           ║  │
│  │        │ │        │ │ / WAM  │  ║ 40+ policies         ║  │
│  └───┬────┘ └───┬────┘ └───┬────┘  ╚═══════════╤══════════╝  │
└──────┼──────────┼──────────┼───────────────────┼─────────────┘
       └──────────┴─────┬────┴───────────────────┘
                        │  obs update · predict · reset · batched
┌───────────────────────┴──────────────────────────────────────┐
│  ⭐ Execution layer — the one thing we build                   │
│  ┌──────────────────────────┬──────────────────────────────┐ │
│  │ SimRunner                │ RealRunner                   │ │
│  │  reset  : env.reset()    │  reset  : learned / manual   │ │
│  │  success: state predicate│  success: VLM classifier     │ │
│  │  safe   : always true    │  safe   : limits + motors    │ │
│  │  time   : stepped, batch │  time   : wall-clock, single │ │
│  └──────────────────────────┴──────────────────────────────┘ │
│  Upward: symmetric to the policy. Downward: absorbs the gap.  │
│  Zero simulator dependencies — installs on the robot too.     │
└───────────────────────┬──────────────────────────────────────┘
                        │ backend adapters
   ┌──────────┬─────────┼──────────┬───────────┬──────────────┐
┌──▼────┐ ┌───▼────┐ ┌──▼─────┐ ┌──▼─────┐ ┌───▼────────────┐
│MuJoCo │ │ Arena  │ │GenieSim│ │Genesis │ │ real hardware   │
│LIBERO │ │G1+SONIC│ │  G2    │ │ (gate) │ │ LeRobot Robot   │
│default│ │ opt-in │ │ opt-in │ │        │ │ + SONIC C++     │
└───────┘ └────────┘ └────────┘ └────────┘ └─────────────────┘
```

**Layering rules:**

> 1. **Model implementation ≠ policy serving ≠ execution ≠ environment.**
> 2. **The same policy server serves a sim client and a real client without modification.** This is the entire source of sim/real symmetry.
> 3. **Backends need not be compatible with each other**, only with the protocol.
> 4. **The execution layer has zero simulator dependencies** — the only reason it can install on both an evaluation node and a robot.

### 6.1 Why the execution layer must exist

A model container only infers. An environment only simulates. Neither owns "how an episode runs" — and that is where every sim/real difference lives.

| Capability | SimRunner | RealRunner |
|---|---|---|
| `reset()` | `env.reset()` | learned reset policy, or human plus transparent overlay |
| `is_success()` | environment state predicate | fine-tuned VLM classifier |
| `is_safe()` | always true | workspace limits, motor status, auto-reboot |
| time semantics | stepped, batchable in parallel | wall-clock, single stream |

> **AutoEval's three modules are not separate work — they are RealRunner's implementation.** Reported figures: over 99% of human effort saved, three interventions per 24 hours of autonomous evaluation, and 1–3 hours to stand up a new task cell.

### 6.2 XPolicyLab's two roles

Easy to misread, so stated plainly:

| Role | Meaning | Position |
|---|---|---|
| **Protocol source** | Its four operations are the contract every model container speaks | We adopt it; we do not publish a competitor |
| **Provider of many policies** | It ships 40+ integrated policies and plugs in wholesale | Alongside individual containers, but structurally a different kind of thing |

**It is not a model.** An individual container provides one policy; XPolicyLab provides a zoo. Both sit below the execution layer and speak the same protocol.

---

## 7. Repo and dependencies

**One repo. No forks.**

Arena supports out-of-tree extension — the docs have dedicated chapters on custom embodiments, tasks, and assets; the repository is laid out as core plus satellite packages (`isaaclab_arena_g1`, `isaaclab_arena_gr00t`, `isaaclab_arena_openpi`); assets register through `AssetRegistry`. Precedent: robot_lab states outright that it develops in an isolated environment outside the core Isaac Lab repo, and TienKung-Lab requires cloning separately from Isaac Lab.

```
crossrun/
├── runner/               # ⭐ the contribution
│   ├── sim/              #    backend adapters
│   └── real/             #    AutoEval trio: reset / success / safe
├── adapters/             # obs-key mapping, action-space tagging,
│                         # Arena's own policy convention
├── eval-protocol/        # seeds, episodes, statistics, CIs
│                         # (independent of both simulator and runner)
├── containers/           # reference container template + registry.yaml
├── assets/               # URDF sources and generation scripts only
├── ci/
└── docs/
```

**Hard constraints:**

- `runner/` must not import any simulator package
- `eval-protocol/` must not import any simulator or runner package; it consumes `(success, trajectory)` and nothing else
- No model weights or model implementations in this repo — containers only
- **No competing protocol specification.** Adaptation code is fine; a published spec is not
- Robot assets are generated from URDF, never hand-written as MJCF or USD
- Large files (USD, 3DGS, datasets) go to the HuggingFace Hub, not git
- Policy servers never bind to a public address

**Upstream consumption:**

| Upstream | How | Cost |
|---|---|---|
| XPolicyLab | protocol source + one containerised provider | 0 |
| Isaac Lab-Arena | pinned tag, satellite package | 0 (Newton churn to track) |
| LeRobot | pip dependency, plus `Robot` for hardware I/O | 0 |
| Isaac-GR00T / GR00T-WBC | pinned; SONIC encoder/decoder | low |
| Genie Sim | **pinned submodule, asset extraction only** | 0 |
| SIMPLE | **read its task designs (MIT); not a dependency** | 0 |
| RoboDojo | cloud leaderboard | 0 |

---

## 8. Evaluation protocol

**Capability dimensions**: generalisation, memory, precision, long-horizon, open-vocabulary instruction, plus disturbance robustness. Genie Sim and RoboDojo converged independently on similar five-way splits, which suggests the field broadly agrees on what to measure.

### 8.1 Cross-backend comparability (hard rule)

The same task is not equally hard on two physics engines — contact models, friction, and solver tolerances all move the success rate.

> **Compare rankings within a backend. Never compare absolute numbers across backends.**

- Within a backend: policy ordering is valid, absolute success rates are valid
- Across backends: absolute numbers are **invalid** and must not share a table
- The one valid cross-backend comparison is whether the *ordering* agrees. Disagreement is a finding, not noise — it says at least one backend has an artefact
- Backend and version are first-class fields in the report template, not footnotes

This also gives MuJoCo sim2sim a precise job: **not to produce a truer number, but to test whether rankings hold.**

### 8.2 Statistical discipline

- Record seeds, episode counts, **container image digests**, normalisation statistics, and physics-settling steps
- **Report Clopper–Pearson confidence intervals, never a bare success rate**
- **Do not infer "no significant difference" from overlapping CIs** — use corrected pairwise tests
- Reference papers often use very few trials (SONIC reports 10–20 per task; SIMPLE 10 per tier). Those numbers are not comparable with large-sample results

**Power reference**: at an observed 90% success rate over 70 rollouts, the 95% CI spans about 15.4 points; narrowing to ±2 points takes roughly 1,030 rollouts.
**Protocol gap in the wild**: OpenVLA's official protocol is 3 seeds × 500 rollouts (50 per task); community reproductions commonly use `n_episodes=10` — a fivefold difference.

### 8.3 Guarding against memorisation

LIBERO-PRO showed that models scoring above 90% under the standard protocol can collapse to 0.0% under perturbation — OpenVLA and π0 fall from 0.96 and 0.94 to 0.00 on LIBERO-Goal position shifts, with π0.5 at 0.38 the only non-zero result.

→ **Perturbation tiers are mandatory, and the standard score and perturbed score are always reported together.** SIMPLE's three-tier OOD design (visual + distractors / lighting / target pose) is MIT-licensed and directly borrowable.

### 8.4 Real-hardware symmetry

**Implemented in the execution layer (§6.1), not as a separate module.** RealRunner supplies what simulation gives away free; the evaluation protocol treats both runners identically, consuming only `(success, trajectory)`.

Cheap trick worth stealing: RoboDojo-RealEval restores consistent initial conditions by replaying a target layout image as a transparent overlay on the live observation stream. Lower cost than a learned reset, and a fine first implementation for RealRunner.

---

## 9. Scope and phases

> **A reference design does not chase coverage.** Make the pattern clear on the smallest example, then show it extends.

**Minimum demonstrable set**: LIBERO / ALOHA on MuJoCo, one VLA and one world-action model. Enough to exercise all nine design claims.
**Extensibility proof, afterwards**: G1 + SONIC on Arena, G2 through Genie Sim.

### Phase 0 — Foundations

- [ ] Pin XPolicyLab's protocol version; **write down the four operations as we consume them**
- [ ] Define the Runner interface (alongside the protocol, not after it)
- [ ] SimRunner + LIBERO + official π0.5 checkpoint, end to end
- [ ] `eval-protocol`: seed management, image-digest archiving, Clopper–Pearson CIs
- [ ] **Measure the two things that would justify our own protocol** (see §12): latent-token action spaces, and batched-query throughput
- [ ] **Check which Isaac Lab branch Arena follows** — `main` is frozen, Newton work is on `develop`

### Phase 1 — Demos and architecture validation

- [ ] Demo 1 (protocol sensitivity) and Demo 3 (90% → 0%)
- [ ] Second and third models via containers
- [ ] One world-action model, to show paradigm independence
- [ ] **RealRunner skeleton and a minimal hardware loop** — manual reset with transparent overlay first, AutoEval trio later.
      The point is to prove early that one policy server serves both sim and real; that claim cannot wait until the end
- [ ] **Genesis evaluation gate** (below)
- [ ] Demo 2 (cross-backend ranking), once a second backend exists

**Genesis evaluation gate** (~1 week, measurable criteria)

| Criterion | Pass | Failure means |
|---|---|---|
| Porting cost | one task + embodiment in ≤5 days | >5 days means the abstraction has leaked — itself a valuable finding |
| Render throughput | measured on a 4090; verify the claimed 1080p in ≤4 ms | far below claim removes its main advantage |
| Abstraction sealing | `runner` and `eval-protocol` reused **unmodified** | needing changes means the design is wrong and must be fixed first |

All three pass → Genesis becomes a lightweight backend candidate and the second backend for Demo 2. Any failure → archived as a watch item.

### Phase 2 — Extensibility: G1 + SONIC

- [ ] SONIC adapter: 78-dimensional action space (64 latent motion tokens + 7 per hand), decoder on the environment side
- [ ] Arena + GR00T N1.7 with `UNITREE_G1_SONIC`
- [ ] Swappable low-level controller, so failures can be attributed to the VLA or to the whole-body controller
- [ ] MuJoCo sim2sim ranking check

### Phase 3 — Extensibility: G2

- [ ] **Genie Sim as a pinned submodule**, used only to extract G2 assets
- [ ] Extraction script produces `assets/`, published to the HuggingFace Hub; **no runtime dependency on Genie Sim**
- [ ] Acceptance test: **someone who never initialises the submodule can still run the default path**
- [ ] Containerise Genie Sim's bundled checkpoint as a known-good baseline for pipeline self-checks

### Phase 4 — Outward

- [ ] Documentation and design write-up — a primary deliverable, not an afterthought
- [ ] RealRunner completes the AutoEval trio
- [ ] Register with LeRobot EnvHub
- [ ] Consider contributing SONIC integration upstream to SIMPLE, where it is an open TODO

---

## 10. Risks

| Risk | Level | Response |
|---|---|---|
| **Newton migration** — Isaac Lab has frozen `main`; physics is moving from PhysX to MuJoCo-Warp, and Arena inherits the churn | High | Confirm branch in Phase 0; pin strictly; isolate behind the runner |
| **G1 has the longest dependency chain** (Arena + GR00T + WBC + SONIC C++/ZMQ) | Med-high | Deferred to Phase 2; validate the SONIC stack in MuJoCo first |
| **Arena is pre-alpha; public interfaces change without deprecation warnings** | Med-high | Pin strictly; isolate; upgrades as separate PRs with regression results |
| **XPolicyLab's protocol is young** (paper published mid-2026) | Med | Cheaper than owning a spec. The protocol is the easiest layer to swap — replace the parsing behind the runner and the runner is untouched |
| **Drift in conventions across backends** | Med | §8.1 rule; `eval-protocol` as the single point of truth |
| **Demos end up as toys** | Med-high | See below |

> **The known failure mode of "no legacy, therefore cleaner":** many reference architectures are clean because they have not yet hit reality. The projects carrying baggage usually acquired it by collision.
> **The only defence is that the demos are real.** Real models, real statistics, real perturbations. "A pretty architecture on a toy task" ends up like the dozens of abandoned frameworks before it; "we measured 90% → 0% and you can reproduce it" does not.

**Security**: policy servers must never bind to a public address. Instructive precedent: LeRobot's async inference gRPC PolicyServer carried an unauthenticated pickle-deserialisation RCE (CVE-2026-25874, CVSS 9.8). The same class of risk applies to every policy server and container port.

**Licensing** (no commercialisation, so this stays short): Isaac Lab, Arena, Genesis, and mjlab are Apache 2.0; Genie Sim's code is MPL 2.0; SIMPLE is MIT, so its task designs are legitimately borrowable; GR00T is commercially licensed from N1.7, with SONIC weights under the NVIDIA Open Model License. Nothing obstructs open-source use.

---

## 11. Explicitly rejected (please don't relitigate)

| Option | Why not |
|---|---|
| Build a simulator or benchmark from scratch | Years of work, and a fresh benchmark is comparable with nothing |
| Fork Arena, Genie Sim, or SIMPLE | Fast-moving upstreams turn forks into merge archaeology; private divergence destroys comparability |
| **Publish our own policy protocol** | The justifications were weak: the schema concern is unverified and probably wrong, and Arena's separate convention argues for an adapter rather than a spec. **Adopt XPolicyLab's** |
| **Treat XPolicyLab as a peer of individual models** | It is a policy-serving framework, not a model. Flattening it loses the layer where sim/real symmetry lives |
| Genie Sim as a runtime dependency | A reference design cannot require a ten-purpose repo just to run. Submodule for assets only |
| SIMPLE as a foundation | ~4 fps ray-traced rendering, two to three orders of magnitude short of the target statistical scale; SONIC integration is still an open TODO in its README. **Borrow the task designs instead** |
| Genesis as the default backend | No verified VLA checkpoint ecosystem; the default path must run something immediately |
| mjlab | Explicitly does not provide RGB rendering for vision policies |
| LeIsaac | SO101-centric and aimed at data production, not evaluation |
| LeRobot async inference as the policy server | No dependency isolation, no first-class policy reset, and optimised for "never stop" — the opposite of episodic evaluation |
| Two policy servers | Doubles integration for no gain |
| Model code inside this repo | Dependency conflicts cap it around ten models |
| Decoupled upper body, avoiding SONIC | SONIC's paper is explicit that such action spaces struggle with coordinated hand-and-foot loco-manipulation |
| Pure lower-body locomotion evaluation | We evaluate task completion, not gait quality |
| **A "reproducibility crisis" narrative** | Overstated and badly sourced — GitHub issues are a complaint channel. The ecosystem is healthy; reframed as protocol sensitivity |

---

## 12. Open questions

Inferences, not conclusions. Each needs measurement.

1. **Can XPolicyLab's schema carry a 78-dimensional latent-token action space?** Its dictionary constrains pose fields; actions are ordinarily plain float arrays, so this probably works. **If it does not, that is the trigger for our own protocol.**
2. **Batched-query throughput under load.** The interface specifies it; no measurements published. **The second possible trigger.**
3. **Which Isaac Lab branch does Arena follow?** Directly affects pinning and migration exposure.
4. **G1 variant, hand configuration, camera setup.** SONIC needs a legged G1; its action space allocates 7 dimensions per hand, pointing at 7-DoF hands such as Dex3-1.
5. **How `isaaclab_arena_g1` connects to SONIC.** No evidence found that Arena integrates SONIC natively.
6. **Genesis Nyx render throughput.** The 1080p-in-4 ms figure is a vendor claim; the gate measures it.
7. **Whether Demo 2 is viable at all.** Cross-backend ranking disagreement may simply not occur. **If rankings agree closely, that is also a publishable result** — the demo's framing changes, not its value.
8. **Time to onboard a new robot.** No public figures; qualitative evidence points to days or weeks. **This one is bounded by physical validation, where AI agents do not help.**

---

## Links

| | |
|---|---|
| XPolicyLab (protocol source) | https://github.com/XPolicyLab/XPolicyLab |
| openpi (π0/π0.5 + LIBERO workflow) | https://github.com/Physical-Intelligence/openpi |
| LeRobot | https://github.com/huggingface/lerobot |
| LIBERO | https://github.com/Lifelong-Robot-Learning/LIBERO |
| Isaac Lab-Arena | https://github.com/isaac-sim/IsaacLab-Arena |
| Newton Physics | https://github.com/newton-physics/newton |
| Isaac-GR00T · GR00T-WBC | https://github.com/NVIDIA/Isaac-GR00T · https://github.com/NVlabs/GR00T-WholeBodyControl |
| RoboDojo | https://github.com/RoboDojo-Benchmark/RoboDojo |
| Genie Sim (G2 assets) | https://github.com/AgibotTech/genie_sim |
| SIMPLE (task-design reference, MIT) | https://github.com/physical-superintelligence-lab/SIMPLE |
| Genesis World (evaluation gate) | https://github.com/Genesis-Embodied-AI/genesis-world |
| AutoEval | https://auto-eval.github.io/ |
| SimplerEnv | https://github.com/simpler-env/SimplerEnv |
