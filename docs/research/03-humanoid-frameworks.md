# Humanoid Loco-Manipulation Evaluation Frameworks: Build or Adopt

> Surveyed July 2026 · prompted by discovering that SIMPLE overlapped heavily with what was then the build plan
> ⚠️ This document's central recommendation — fork SIMPLE as the foundation — was **later overturned** on render-throughput grounds. See [README](README.md)

## TL;DR

- **Do not build from scratch.** SIMPLE (arXiv:2606.08278, USC PSI-Lab, MIT) overlaps almost one-to-one with the build plan at the architectural level: dual simulation with MuJoCo physics and Isaac Sim rendering, G1 whole-body loco-manipulation, native benchmarking of both VLAs and world action models, LeRobot data format, and client-server policy evaluation.
- **But one key assumption needs correcting**: its README roadmap still lists "Integrate SONIC whole-body controller" as an **unchecked TODO**. What the simulation actually uses is **decoupled WBC**; the full GEAR-SONIC universal-token controller lives in the companion Psi0 repository and only for **hardware**.
- **WAM evaluation splits into two unrelated technical paths**: treating a WAM as an ordinary policy and measuring task success in physical simulation (what SIMPLE and Isaac Lab-Arena do, and what makes WAMs and VLAs fairly comparable), versus using a world model itself as the evaluator (WorldEval, RoboWorld, EWMBench, GigaWorld-1/WMBench), which scores generated video quality rather than task success. The two must not be conflated.

## Key findings

**Evidence strength**: strong = directly attested by a primary source; medium = reliable secondary source or inference required; speculative = no direct evidence.

1. **SIMPLE is the closest existing project to the goal (strong).** Per psi-lab.ai/SIMPLE and the paper, it couples MuJoCo's accurate contact dynamics with Isaac Sim's photorealistic rendering, providing a large-scale environment of 60 diverse whole-body tasks, 50 indoor scenes, and over 1,000 object assets. Built on IsaacSim 4.5 and MuJoCo 3.3, supporting franka, aloha dual-arm, dexmate wheeled, and Unitree G1, at roughly 185 stars and 16 forks.
2. **SIMPLE is research-grade, not a product (strong).** Twenty-six commits, no releases, self-described as work in progress, with a heavy dependency stack (IsaacSim + MuJoCo + CuRobo + uv/Nix/Docker).
3. **No single project covers all requirements** — humanoid plus tabletop dual-arm, VLA plus WAM, SONIC, and sim/real consistency.
4. **Dual simulation — one engine for physics, another for rendering — is now a clearly established trend (strong)**, represented by SIMPLE (MuJoCo + Isaac Sim), DISCOVERSE (MuJoCo + 3DGS), RoboGSim (3DGS + Isaac Sim), SplatSim (3DGS + MuJoCo MJX), and the general abstraction layer RoboVerse/MetaSim.
5. **NVIDIA's official stack aligns naturally (strong)**: Arena ships `isaaclab_arena_g1` and integrates GR00T, OpenPi, and DreamZero policies; SONIC's PyTorch checkpoints explicitly support Isaac Lab evaluation.

## A. Humanoid loco-manipulation frameworks

**SIMPLE (USC PSI-Lab) — the primary candidate**
- physical-superintelligence-lab/SIMPLE, MIT, ~185 stars / 16 forks, no releases, docs marked WIP
- Dual-simulation architecture: MuJoCo handles contact physics and high-frequency robot control; Isaac Sim synchronises physics state each step and performs ray-traced rendering (`--render-hz=50`). Data generation can drive both engines with `--sim-mode=mujoco_isaac`
- **Stated limitations (from the paper)**: ① render throughput of roughly **4 fps** ray-traced on a single GPU; ② fully rigid-body assumption, so cloth, rope, and food cannot be simulated
- Policy integration is decoupled client-server: inference runs on a server (the Psi-0 repository) while the SIMPLE client runs the simulation. Data and evaluation environments use LeRobot format
- **SONIC status (verified)**: the README roadmap shows `- [ ] Integrate SONIC whole-body controller` unchecked; simulation uses decoupled WBC. Full GEAR-SONIC lives only in Psi0, whose news entry "[2026-06-13] Released SONIC integration for Psi-0" indicates a hardware workflow
- Its native benchmark covers Psi0, GR00T N1.6, OpenPi π0.5, InternVLA-M1, H-RDT, DreamZero (WAM), EgoVLA, Diffusion Policy, ACT, and Cosmos3 (WAM). Each task has three OOD levels (visual plus distractors / lighting / target pose) with 10 trials per level
- Requirements: Ubuntu 22.04, CUDA 12.x, Python 3.10, RTX-class GPU (4090 16GB+ recommended)

**HumanoidBench (UC Berkeley) — reference only**
Roughly 599–772 stars; v0.2.0 (2 July 2026) added G1. MuJoCo, H1 with Shadow Hands, 27 tasks (12 locomotion, 15 whole-body manipulation). Positioned as an **RL algorithm benchmark**, state-observation oriented, with no native VLA or WAM integration.

**HumanoidMimicGen loco-manipulation benchmark (arXiv:2605.27724) — reference only**
robosuite plus MuJoCo, 9 G1 tasks varying along three axes (base motion extent, interaction complexity, horizon), with binary success conditions.

**LocoMuJoCo, HumanoidVerse, ProtoMotions, AGILE/WBC-AGILE** — all oriented towards locomotion training or motion imitation rather than loco-manipulation evaluation. AGILE's four-stage workflow (interactive debugging GUI → training → unified evaluation → descriptor-driven deployment) has methodological value.

**BEHAVIOR Robot Suite / BEHAVIOR-1K (Stanford) — limited relevance**
The embodiment is a Galaxea R1 (wheeled dual-arm with a 4-DoF torso, not a biped). The WB-VIMA baseline achieves 13× the end-to-end task success of DP3 and 21× that of RGB-DP across five household tasks at 15 randomised rollouts per policy. It underpins the first BEHAVIOR Challenge at NeurIPS 2025.

## B. Dual and hybrid simulation

- **Trend assessment (medium)**: the driver is that no single engine excels at both contact physics and photorealistic rendering. SIMPLE's contribution is engineering this into a producer-consumer pipeline that achieves zero-shot sim-to-real.
- **DISCOVERSE** (arXiv:2507.21981): MuJoCo physics with 3DGS rendering, aimed at real2sim2real.
- **RoboGSim** (arXiv:2411.11839): 3DGS reconstruction plus an Isaac Sim digital twin with a closed-loop VLA evaluator.
- **SplatSim / SplatMesh** (arXiv:2506.04120): 3DGS appearance with differentiable MuJoCo (MJX) physics, validated on ALOHA 2 dual-arm.
- **RoboVerse/MetaSim** (arXiv:2504.18904, ~1.78k stars, Apache 2.0): not a dual-simulation implementation but a **cross-simulator abstraction layer**, explicitly offering hybrid simulation (pairing an advanced physics engine with a strong renderer), cross-simulator, and cross-embodiment capabilities. **Its interface design is worth borrowing directly.**
- **Trade-offs (medium/speculative)**: the strengths are complementary — each engine contributes what it does best, and rendering diversity supports perceptual generalisation. The costs are complex state synchronisation, render throughput as the bottleneck, doubled asset maintenance, and harder debugging.

## C. GEAR-SONIC and GR00T-WholeBodyControl integration

- **GR00T-WholeBodyControl (NVlabs)** is the official stack, comprising ① decoupled WBC (RL lower body plus IK upper body, used by GR00T N1.5/N1.6); ② the GEAR-SONIC series (a 42M-parameter behaviour foundation model trained on 100M+ frames of human motion; the universal-token controller emits 64-dimensional latent motion tokens at 50 Hz; C++/TensorRT deployment; PyTorch checkpoints support Isaac Lab evaluation and continued training); ③ MotionBricks.
- **GR00T N1.7** supports SONIC through the `UNITREE_G1_SONIC` embodiment tag. The complete collect→finetune→deploy workflow is supported only for GEAR-SONIC — the `UNITREE_G1` tag is compatible with decoupled WBC but not with the end-to-end workflow.
- **Two paths (strong)**: SIMPLE's simulation follows decoupled WBC; NVIDIA's official N1.7 plus Isaac Lab follows SONIC universal tokens.
- **Whether Arena already integrates SONIC**: no direct evidence found (**no evidence located**). Arena demonstrably integrates GR00T, OpenPi, and DreamZero policies, and SONIC checkpoints support Isaac Lab evaluation, but whether the two are connected end to end needs measurement.
- **`isaaclab_arena_g1` contents (strong)**: a G1 humanoid embodiment with examples; the official documentation's sample task is loco-manipulation, moving a box to a pallet.

## D. World action model evaluation

**The methodological split (strong)**: WAM evaluation means two entirely different things.

1. **Evaluating a WAM as a policy (task success)** — competing directly with VLAs. SIMPLE already does this, putting DreamZero and Cosmos3 in the same success-rate table as VLAs. **This is the main line**, and it is naturally fair across VLAs, WAMs, and classical methods. Where a model emits latent actions rather than joint angles, the interface answer is **latent action → decoder / WBC → joint commands**, with the evaluation layer caring only about final task success.
2. **Using a world model as the evaluator** — scoring generated video quality rather than real task success:
   - **EWMBench** (AgibotTech, arXiv:2505.09694): open source, three metric dimensions (visual scene consistency, motion correctness, semantic alignment), underpinning the WM track of the AgiBot World Challenge
   - **WorldEval / Policy2Vec** (arXiv:2505.19017): turns a video generation model into a world simulator, injecting policy latent embeddings and measuring ranking correlation with real success rates
   - **RoboWorld** (arXiv:2607.01060): autoregressive video world model with task-progress-aware VLM scoring, reporting Pearson r = 0.989 and Spearman ρ = 0.970 against real evaluation
   - **GigaWorld-1 / WMBench** (arXiv:2607.02642): built from real teleoperation data paired with matching policy rollouts
   - **MotionWAM** (arXiv:2606.09215, Mondo Robotics and HKUST): not an evaluation framework but a WAM in its own right (real-time humanoid loco-manipulation on G1), usable as a **subject** of evaluation

## E. Whole-body VLAs and their evaluation protocols

- **WholeBodyVLA** (arXiv:2512.11047, ICLR 2026): unified latent VLA validated on the AgiBot X2 humanoid, 21.3% above baseline
- **TrajBooster** (arXiv:2509.11839): cross-embodiment trajectory-centric learning from as little as 10 minutes of teleoperation
- **WOLF-VLA** (arXiv:2606.25591): a VLA benchmark for whole-body humanoid locomotion, promising dataset, checkpoints, and a simulated evaluation suite (**release status unverified**)
- **GR00T N1.5/N1.7, Being-H0.7, Humanoid-VLA, LDA-1B, EgoScale**: each uses its own protocol. **No consensus task set and no shared public leaderboard exist** (strong)

## F. Cross-embodiment unified evaluation

- **Isaac Lab-Arena (NVIDIA and Lightwheel, Apache 2.0)**: composable LEGO-style task assembly, GPU-parallel rollout, RTX rendering, LeRobot-compatible datasets, native support for GR00T N1x, π0, SmolVLA, ACT, and Diffusion Policy, and G1, GR1, and Galileo humanoids. **Currently the only open framework that serves both humanoid loco-manipulation and tabletop manipulation with industrial-grade maintenance.**
- **RoboVerse/MetaSim**: a cross-embodiment unified interface, a candidate abstraction for one config across many backends.
- **Genie Sim 3.0 (AgibotTech, MPL 2.0)**: on Isaac Sim. Per the CES 2026 announcement, its benchmark spans more than 200 tasks across over 100,000 simulated scenarios, with more than 10,000 hours of synthetic data. **For heavy use of AgiBot G2, Genie Sim is the closest official evaluation stack.**

## G. The Chinese ecosystem

- **Unitree (strong)**: `unitree_rl_gym` (BSD-3, Go2/H1/H1_2/G1, Isaac Gym training with MuJoCo sim2sim and hardware deployment) and `unitree_rl_mjlab` (mjlab-based, MuJoCo backend). Both are **locomotion RL scaffolding rather than loco-manipulation evaluation benchmarks**.
- **AgiBot (strong)**: Genie Sim 3.0 plus EWMBench plus the AgiBot World dataset form a full stack from data through simulated evaluation to hardware. The AgiBot World Challenge 2026 uses Genie Sim Benchmark and EWMBench as its evaluation base, with the offline final on the G2 humanoid.
- **X-Humanoid (strong)**: RoboMIND (arXiv:2412.13877, 107k demonstration trajectories across 479 tasks and 96 object classes on four embodiments), the Huisi Kaiwu platform, the Tien Kung-Lab locomotion framework, ArtVIP high-fidelity assets (included in NVIDIA Isaac Sim), and Labimus (a chemistry-lab humanoid simulation evaluation platform). It leads standards work including humanoid intelligence grading.
- **OpenLoong (Humanoid Robot Shanghai, OpenAtom Foundation)**: the Qinglong full-size open humanoid, an MPC plus WBC control framework deployed in MuJoCo, and the Gewu simulation platform.
- **Shanghai AI Lab InternRobotics, Galbot, Galaxea, Fourier, Kepler, Leju**: **no substantial open humanoid evaluation frameworks located** (no evidence found; existence not excluded).

## Trends

1. **Dual simulation has moved from papers to standard engineering practice (medium)**
2. **Humanoid evaluation standardisation remains an open space (medium)** — SIMPLE, Isaac Lab-Arena, and Genie Sim are all competing for it; none has won
3. **WAM evaluation methodology has bifurcated (strong)**
4. **NVIDIA is consolidating the humanoid evaluation-and-control stack around Isaac Lab-Arena and GR00T-WBC/SONIC (strong)**

## Comparison table

| Project | Organisation | Backend | Embodiment | Tasks | VLA | WAM | SONIC | Usable | Licence |
|---|---|---|---|---|---|---|---|---|---|
| **SIMPLE** | USC PSI-Lab | MuJoCo + Isaac Sim | G1/franka/aloha/dexmate | 60 | ✅ | ✅ | ❌ decoupled WBC only | research-grade | MIT |
| **Isaac Lab-Arena** | NVIDIA + Lightwheel | Isaac Sim | G1/GR1/Galileo + tabletop | composable | ✅ | ✅ | ⚠️ unconfirmed | industrial | Apache 2.0 |
| **Genie Sim 3.0** | AgiBot | Isaac Sim | G2/humanoid + tabletop | 200+ | ✅ | partial | ❌ | industrial | MPL 2.0 |
| **HumanoidBench** | UC Berkeley | MuJoCo | H1/G1 + Shadow Hands | 27 | ❌ | ❌ | ❌ | mature (RL) | — |
| **GR00T-WBC** | NVlabs | Isaac Lab | G1 | controller | ✅ | — | ✅ native | official | NV Open Model |
| **RoboVerse/MetaSim** | PKU/UCB | multi-backend abstraction | cross-embodiment | many | ✅ | ✅ | ❌ | active | Apache 2.0 |
| **EWMBench** | AgiBot | (WM evaluator) | — | — | — | ✅ generation | — | open | — |
| **unitree_rl_gym** | Unitree | IsaacGym + MuJoCo | Go2/H1/G1 | locomotion | ❌ | ❌ | ❌ | mature | BSD-3 |

## Recommendations as written at the time (partly overturned)

### Must fix
1. **Correct the assumption that SIMPLE ships SONIC.** Committing to GEAR-SONIC means either wiring SONIC into SIMPLE's simulation loop yourself, or going through the official Isaac Lab-Arena plus GR00T-WBC path.
2. **Split "WAM evaluation" into two explicit sub-tracks.**

### Should fix
3. **Narrow the build scope to the integration layer, the task and judgement criteria, and the mechanisms that make numbers credible**:
   - **Do not build**: the dual-simulation physics-rendering pipeline, the G1 loco-manipulation task set with OOD tiers, or the LeRobot data pipeline
   - **Do build**: a unified policy-server abstraction; consistent success criteria and metric definitions across stacks; onboarding for proprietary robots
4. ~~**Hybrid approach**: SIMPLE for G1 loco-manipulation (forked and modified) plus Isaac Lab-Arena for tabletop~~ ← **this recommendation was later overturned**
5. **Borrow RoboVerse/MetaSim's config abstraction**

### Thresholds for revisiting
- If SIMPLE's render throughput (~4 fps) makes evaluation cycles unacceptable → fall back to Isaac Lab-Arena alone ← **this is exactly the trigger that fired**
- If SIMPLE checks off its SONIC integration TODO → its weight rises substantially
- If NVIDIA announces native SONIC integration in Arena → all G1 evaluation converges on Arena
- If the team cannot maintain both stacks → keep Arena only

## Caveats

- **SIMPLE's practical runnability is unverified (medium).** No public reports of installation or execution failures were found — which does not mean there are none; its heavy dependency stack is a notoriously fragile combination.
- **The "500 Hz MuJoCo" figure is not stated verbatim in SIMPLE's documentation (medium)**: the project page says only that MuJoCo handles high-frequency control, with Isaac rendering at `--render-hz=50`.
- **No direct evidence that Isaac Lab-Arena integrates SONIC end to end.**
- **Whether WOLF-VLA's evaluation suite has been released: unverified.**
- **No evidence found for open evaluation work from second-tier Chinese vendors.**
- arXiv identifiers 2606.xxxxx and 2607.xxxxx correspond to June–July 2026, all very recent preprints; some conclusions are self-reported and unreproduced by third parties.
