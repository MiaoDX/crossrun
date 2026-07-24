# Red-Team Review of the First Draft Plan

> Conducted July 2026 · an adversarial review of crossrun's early plan, when it was still scoped as an internal evaluation system
> ⚠️ Some premises no longer hold (the compute constraint, the commercialisation assumption) — see [README](README.md)

**The stance of this document**: this is an **adversarial** review. It looks for counterexamples, failure cases, underestimated risks, and arguments that the plan may be wrong as a whole. It does not confirm.

## Executive summary

**The plan's technical judgement is broadly sound, but it places its heaviest bet on the component with the highest ecosystem risk and the highest cost: Isaac.**

Its best parts — a thin orchestration layer, no forks, a policy protocol abstraction, dual-backend cross-validation, SimplerEnv MMRV as an anchor, and LIBERO-Plus/PRO perturbation tiers against memorisation — align well with current academic consensus and rest on strong evidence. Three blind spots remain:

1. **Compute and licensing were never costed.** Isaac Sim's official requirements state plainly that GPUs without RT cores (A100, H100) are unsupported, with a GeForce RTX 4080 (16GB) as the floor. NVIDIA and Lightwheel's 4,096-parallel configuration uses 8× RTX 6000D — roughly $52,000–$64,000 in GPUs alone.
2. **The foundation is a pre-alpha whose public interfaces change without deprecation warnings**, from a vendor with a history of rebuilding its robotics simulation stack.
3. **A deeper doubt about value**: leading labs still treat real-hardware evaluation as the gold standard, with sim eval serving as a cheap first pass.

## Red team: how this could fail

Ordered by probability × impact.

### R1 [high × high] Engineering load buries a small team before the first credible number

Isaac Sim's operational burden is documented repeatedly across the community: installation failures, driver version hell (errors demanding a driver inside a precise range), GPU passthrough failures under Docker and WSL2, containers failing to start against a Nucleus server. These span 2021–2026 and multiple versions. By contrast `pip install mujoco` runs in ten minutes. **Evidence: strong.**

### R2 [medium × high] Isaac ecosystem churn invalidates the foundation

NVIDIA has a clear history here. Isaac Gym Preview 4 is "no longer updated or supported"; IsaacGymEnvs, OmniIsaacGymEnvs, and Orbit were all deprecated in favour of Isaac Lab; in June 2024 the Isaac Gym download link was removed unannounced, and researchers complained on the official forums that a great deal of work depended on it and results were no longer reproducible. Newton (NVIDIA, DeepMind, and Disney, under the Linux Foundation, Apache 2.0, still alpha) has now entered Isaac Lab. Isaac Lab-Arena remains pre-alpha (v0.2.x), with a README stating that public interfaces are under active development and will change without deprecation warnings, and that this is neither Early Access nor GA but a very early community code drop. **Evidence: strong.**

### R3 [medium × high] The numbers convince nobody

(a) The external credibility of simulated numbers is capped by the sim2real gap — RoboArena's paper criticises centralised sim-only leaderboards for systematically understating real-world performance. (b) Internal benchmarks get optimised until they stop carrying signal (Goodhart's law). (c) LIBERO-PRO shows existing VLAs above 90% on standard LIBERO collapsing to 0.0% under perturbation, which the authors attribute to rote memorisation of action sequences and scene layouts. **Evidence: strong.**

### R4 [medium × medium] Compute cost exceeds what the team can absorb

See section B. **Evidence: strong.**

### R5 [medium × medium] A licensing trap detonates at commercialisation

See section C. **Evidence: strong.**

### R6 [low × high] Cross-embodiment evaluation is not methodologically sound

The field broadly accepts that fair cross-embodiment comparison is unsolved when action and observation spaces cannot be cast into a common format — CrossFormer's stated contribution is precisely handling that. **Evidence: medium.**

### R7 [low × medium] Insufficient statistical power leaves numbers unable to discriminate

NVIDIA's own analysis on the RoboLab platform: at an observed 90% success rate over just 70 rollouts, the 95% Clopper–Pearson interval spans 15.4 points (80.5%–95.9%); narrowing to ±2 points (88.0%–91.8%) requires roughly 1,030 rollouts. **Evidence: strong.**

## A. How much is sim eval actually worth?

**Finding 1: sim2real correlation supports ranking, not replacement, and only holds under narrow settings.** SimplerEnv (arXiv:2405.05941) Table I reports Visual Matching averaging Pearson r = 0.924 with MMRV 0.056 on Google Robot tasks, and r = 0.976 on Pick Coke Can. The counterpoint matters: AutoEval states plainly that the sim-real gap affects different policies differently, producing inconsistent rankings between simulation and hardware.

**Finding 2: leading labs treat hardware as the gold standard.** AutoEval opens by describing human-run real evaluation as the gold standard used by most prior work. TRI published *Statistical Thinking for Robot Policy Evaluation* and used real-hardware A/B testing with STEP sequential tests in its Large Behavior Models work.

**Finding 3: there is a serious opposing camp.** RoboArena — with Physical Intelligence authors among them — argues for distributed real-hardware evaluation. RobotArena ∞, PhAIL, VLA-REPLICA, and AutoEval all take real hardware or real-to-sim translation as the primary path.

**Finding 4 (an important counterweight): neural simulator and world-model approaches report *higher* correlations.** RoboWorld (arXiv:2607.01060) reports Pearson r = 0.989 and Spearman ρ = 0.970 against the RoboArena leaderboard across eight policies, reproducing the RoboArena benchmark inside its neural simulator for about 100 H100 GPU-hours.

## B. Compute economics

**Hardware floor (strong).** Isaac Sim 5.0's requirements state that GPUs without RT cores — A100 and H100 explicitly — are unsupported. This matters: the cheapest datacentre instances **cannot** run Isaac's rendering-based evaluation. The floor is a GeForce RTX 4080 (16GB); the ideal is an RTX PRO 6000 Blackwell (48GB).

**What 4,096-parallel actually costs.** NVIDIA and Lightwheel use 8× RTX 6000D to run 10 RoboCasa tasks with GR00T N1.5 over 200-step rollouts, claiming up to 13.5× speedup.

**Purchase cost.** RTX 6000 Ada street price runs about $7,159–$7,998. The RTX PRO 6000 Blackwell rose from an $8,565 launch price to $13,250 by June 2026 amid GDDR7 shortages. The RTX 6000D used by Lightwheel is reported at $6,500–$8,000 per card — so eight cards alone come to roughly $52,000–$64,000.

**Cloud hourly rates.** RTX 6000 Ada runs about $0.77–$1.32 per GPU-hour; RTX PRO 6000 Blackwell about $1.29–$2.98; RTX 4090 about $0.41–$0.50. AWS G7e with eight cards is $33.14 per hour.

**MuJoCo comparison.** The Robotics Center of Silicon Valley (April 2026) puts it directly: MuJoCo runs on a $1,500 laptop, while Isaac Sim realistically needs a workstation-class GPU, pushing hardware cost to $3,000–$15,000 per seat. MuJoCo-Warp is reported at 313× faster than MJX for manipulation and 152× for locomotion on an RTX 4090.

## C. Licensing and legal exposure

- **Isaac Sim / Omniverse**: source is Apache 2.0 and internal R&D is free, but the Omniverse Kit SDK it depends on triggers NVIDIA AI Enterprise or Omniverse Enterprise licensing when redistributed as an application to third parties or delivered as a service.
- **Isaac Lab / Isaac Lab-Arena**: both Apache 2.0, but both depend on Isaac Sim and its proprietary components.
- **Isaac GR00T**: N1 and N1.5 are non-commercial; **N1.7 onward is Apache 2.0 and commercially usable**.
- **Genie Sim**: `source/geniesim_*` and `source/data_collection` are MPL 2.0 (file-level copyleft); `source/scene_reconstruction` carries mixed licences.
- **Third-party asset redistribution in RoboCasa, LIBERO and similar**: no confirmation found; a separate audit is advisable.

## D. Isaac ecosystem churn

- **Migration history**: see R2.
- **Newton's arrival**: Newton has entered Isaac Lab, moving the base from PhysX. Linux Journal notes the project remains in alpha, with instability expected and APIs evolving quickly.
- **The counterexample**: MuJoCo is maintained by DeepMind, Apache 2.0, installable with pip, and academically oriented — markedly more stable than Isaac.

## E. Underestimated alternatives

1. **Pure MuJoCo/MJX + ManiSkill3, skipping Isaac entirely — underestimated; recommended as the first-phase main path.** The cases genuinely requiring Isaac are narrow: large-scale parallel humanoid whole-body control, and tasks needing RTX ray-traced visual observation.
2. **LeRobot-only — moderately recommended.** It already wraps LIBERO and Meta-World with one-line `lerobot-eval`, and Arena is available as an optional backend through EnvHub.
3. **Real-eval-first (an AutoEval-style automated hardware rig) — possibly optimal for the credibility goal.** AutoEval combines a fine-tuned VLM success classifier with a reset policy for 24/7 autonomous evaluation, correlating well with human evaluation while saving over 99% of the effort.
4. **World model as evaluator — highest potential, not yet ready as a main path.** RoboWorld (r = 0.989), WorldEval, and GigaWorld-1/WMBench all show strong correlation at manageable cost, but GigaWorld-1 itself notes that what makes a world model reliable remains poorly understood.
5. **Skip building entirely and compete on external leaderboards — possibly the best ROI for a small team.**
6. **Co-develop with a platform vendor.** Arena explicitly invites the community to publish benchmarks on its core, which is higher leverage for building technical influence.

## F. Engineering realism for a small team

- **Isaac's operational burden**: installation, driver, and container failures recur across versions. MuJoCo is ten minutes with pip.
- **Time to onboard a new robot**: no public figures found. Qualitative evidence strongly suggests days to weeks per robot — mechanical URDF→USD import is documented at 10–20 minutes, but actuator modelling (ImplicitActuator versus DCMotor, stiffness and damping tuned repeatedly without stabilising, per IsaacLab Discussion #3789), sim2real latency and impedance matching, and known ghost/duplicate collision bugs in the new URDF importer are all sinks. **No first-hand timings; this is a medium-confidence inference.**

## G. Deeper methodological problems

- **Statistical power**: see R7. RoboLab acknowledges that at N = 10, per-task CIs near p = 0.5 span roughly ±30%.
- **Multiple comparisons**: TRI's *Statistical Thinking* recommends corrected pairwise tests, compact letter displays, and Bayesian beta-posterior violin plots, and warns explicitly that two heavily overlapping confidence intervals can still be separated by a direct hypothesis test — **overlapping CIs do not establish "no significant difference."**
- **Goodhart and overfitting**: LIBERO-PRO (>90% → 0.0%), LIBERO-Plus (7 dimensions across 21 sub-dimensions), and vla-eval (reproduction traps across 14 benchmarks and 657 leaderboard entries) are the required reading.
- **Reproducibility infrastructure**: SureSim proposes correcting large-scale simulated bias with a small number of paired real trials via prediction-powered inference, yielding proper confidence intervals.
- **Cross-embodiment**: CrossFormer's contribution is precisely handling observation and action spaces that cannot be cast to a common format; RL-ViGen reports that no algorithm handles cross-embodiment generalisation.

## H. The non-technical side of credibility

No robotics-specific literature was found, but the requirements can be inferred: verifiable correlation with hardware; statistical rigour; perturbation tiers against overfitting; and reproducibility. For a small team, **competing on public hardware leaderboards and contributing benchmarks to an adopted community framework** offers better ROI than building an internal simulated leaderboard.

## I. Multi-embodiment and WAM evaluation

- **ALOHA tabletop versus G1 whole-body**: the action spaces differ enormously and fair cross-embodiment comparison is unsolved. **This is closer to needing two evaluation protocols than one.**
- **World action models**: where a model rolls out a world model or emits latent actions, the obs→action-chunk paradigm partly suffices, but the metrics need to change to specialised benchmarks such as EWMBench or WMBench.
- **Classical versus learned methods on one benchmark**: motion planning typically needs privileged state while VLAs consume only images, so the comparison is between pipelines rather than between policies given equal information. This must be stated explicitly in any report.

## J. Timing

- **A standard is forming**: LeRobot EnvHub has integrated Isaac Lab-Arena, and NVIDIA is aggregating NIST, GR00T Industrial, DexBench, and RoboLab onto the Arena core. **Arena plus LeRobot EnvHub looks likely to become the de facto orchestration layer.**
- **Now versus six months from now**: Isaac Lab-Arena is pre-alpha and Newton is alpha, so deep coupling to Isaac carries high opportunity cost, while the MuJoCo and LeRobot parts are stable today.

## Recommended changes

### Must fix
1. **Demote Isaac/Arena from day-one foundation to a second-phase, on-demand backend.**
2. **Produce a budget and a licence audit.**
3. **Spell out how external credibility will be earned, and include real-hardware evaluation.**
4. **Make the statistics explicit**: Clopper–Pearson intervals and corrected pairwise tests; never infer "no difference" from overlapping CIs.
5. **Leave room in the policy protocol for models that are not obs→action** (latent actions, world-model rollouts).

### Should fix
6. Prefer LeRobot as the orchestration layer.
7. Give ALOHA and G1 separate evaluation protocols.
8. Flag cross-embodiment and privileged-state fairness explicitly in the report template.
9. Adopt SureSim-style paired real2sim correction.
10. If Isaac is adopted, pin versions, containerise, and budget for migration.

## Evidence strength

| Conclusion | Strength |
|---|---|
| Sim eval ranks but does not replace hardware | strong |
| SimplerEnv Visual Matching r ≈ 0.924 is a reasonable anchor | strong |
| LIBERO-class benchmarks suffer serious memorisation overfitting | strong |
| Isaac Sim requires RT-core GPUs; 4,096-parallel needs 8× RTX 6000D | strong |
| Isaac has repeatedly churned; Arena is pre-alpha, Newton alpha | strong |
| Omniverse Kit redistribution triggers enterprise licensing; GR00T ≤N1.6 non-commercial | strong |
| At 70 rollouts, a 90% success rate carries a 15.4-point CI | strong |
| MuJoCo/MJX TCO is an order of magnitude below Isaac | strong |
| World-model evaluators correlate well but are immature | medium |
| Fair cross-embodiment comparison is unsolved | medium |
| Onboarding a robot to Isaac takes days to weeks | medium (no first-hand timings) |
| Third-party asset redistribution limits | no adequate evidence |

## In one line

The plan's **thinking is right** — thin layer, reuse, anti-overfitting, real2sim anchoring — but its **sequencing and risk pricing need revisiting**: it makes the most expensive, least stable, hardest-to-operate component the foundation, and stakes credibility on simulated numbers whose ceiling is set by the sim2real gap.
