# The Robot Sim Eval Open-Source Landscape

> Surveyed July 2026 · the starting point for crossrun's design
> ⚠️ Some judgements have since been corrected — see [README](README.md). Clear factual and arithmetic corrections are applied in place.

## TL;DR

- **The field has split into four lines of work**: ① classic academic manipulation benchmarks (LIBERO, ManiSkill, RoboCasa) acting as common currency; ② integrated company platforms bundling data, simulation, and evaluation (AgiBot Genie Sim, NVIDIA Isaac Lab-Arena, InternRobotics, RoboTwin); ③ real2sim approaches (SimplerEnv, 3DGS reconstructions like RoboGSim and DISCOVERSE) alongside distributed real-robot evaluation (RoboArena, AutoEval); ④ the emerging use of world models as evaluators (1X World Model, GE-Sim, GigaWorld).
- **Fifteen repos worth watching**: huggingface/lerobot, Physical-Intelligence/openpi, isaac-sim/IsaacLab-Arena, NVIDIA/Isaac-GR00T, haosulab/ManiSkill3, AgibotTech/genie_sim, RoboTwin-Platform/RoboTwin, RoboDojo-Benchmark/RoboDojo, simpler-env/SimplerEnv, InternRobotics (InternUtopia / InternManip / InternNav), Genesis-Embodied-AI/Genesis, google-deepmind/mujoco (+ Playground), TATP-233/DISCOVERSE, ARISE-Initiative/robosuite (+ RoboCasa), carlosferrazza/humanoid-bench.
- **The core finding**: no single benchmark can serve as the final arbiter of policy quality. Simulated evaluation is high-variance and easily gamed by memorisation. LIBERO-PRO (arXiv:2510.03827) is explicit — models exceeding 90% accuracy under standard LIBERO evaluation collapse to 0.0% under its generalised setting (OpenVLA 0.96→0.00 and π0 0.94→0.00 on LIBERO-Goal position perturbation).

## Key findings

1. **Integrated platforms are now standard for companies.** AgiBot Genie Sim 3.0 (announced at CES 2026), NVIDIA Isaac Lab-Arena, and Shanghai AI Lab's InternRobotics all connect environment generation, data collection, and standardised evaluation, and all emphasise LLM/VLM-driven generation of scenes and evaluation configs.
2. **VLA policy interfaces are standardising.** openpi's policy server (WebSocket, port 8000), InternManip's client-server split, RoboDojo/XPolicyLab's unified interface, and LeRobot's `lerobot-eval` CLI all follow the same decoupling: integrate a model once, integrate a benchmark once.
3. **Real2sim credibility now has quantitative support.** SimplerEnv measures agreement between simulated and real rankings using MMRV (mean maximum rank violation) and Pearson correlation. Genie Sim 3.0's technical report (arXiv:2601.02078) reports R² = 0.924 with slope ≈ 1.045 across 16 model configurations on a π0.5 baseline — though these figures come from the platform's own authors and warrant caution.
4. **World models as evaluators moved from concept to benchmark.** 1X frames world-model evaluation as the goal of its challenge; AgiBot ships GE-Sim and EWMBench; GigaWorld-1 (arXiv:2607.02642) systematically studies what makes a world model a reliable evaluator, analysing seven video world models, four action-representation schemes, and over 324,000 simulated policy rollouts.
5. **Reproducibility problems are now stated openly.** LIBERO-PRO and LIBERO-Plus show that high scores under the standard protocol largely reflect memorised action sequences and layouts; success rates can swing by more than 30 points across random seeds.

## A. Company and institutional platforms

### China

**AgiBot — Genie Sim / Genie Sim Benchmark** (https://github.com/AgibotTech/genie_sim, ~1.1k stars, MPL 2.0). Built on Isaac Sim, with 3DGS scene reconstruction converted to USD and LLM-driven generation of scenes, tasks, and evaluation configs. Genie Sim 3.0 launched at CES 2026 covering 200+ tasks and 100,000+ evaluation scenarios, with 10,000+ hours of synthetic data released, supporting GO-2, the Pi series, and GR00T. The benchmark divides into five capability suites: instruction following, spatial understanding, manipulation skill, disturbance adaptation, and train-deploy generalisation.

**AgiBot — GE-Sim / GE-Sim-V2 / Genie-Envisioner.** Genie Envisioner is a world-foundation-model platform; GE-Sim is its world simulator, with V2 built on Cosmos2. It ships alongside **EWMBench**, which scores video-based world models on scene, motion, and semantic quality.

**Beijing Innovation Center of Human Robotics (X-Humanoid).** With Peking University, released **RoboMIND** — 5 scene categories, 96 object classes, 6 manipulation types, 479 tasks, drawing on 31,005 Franka trajectories, 9,686 from the Tien Kung humanoid, 8,030 from AgileX dual-arm, and 6,911 from UR-5e. Also open-sourced the Tien Kung-Lab locomotion framework, ArtVIP articulated assets, and a training toolchain. Its Huisi Kaiwu platform targets one brain across many robots and many skills.

**Shanghai AI Lab / InternRobotics.** InternUtopia (formerly GRUtopia, ~1.2k stars, MIT) is a city-scale embodied simulation platform on Isaac Sim. InternManip (MIT) is an all-in-one training and evaluation suite unifying data formats, model interfaces, and evaluation protocols through a client-server deployment. InternNav builds on PyTorch, Habitat, and Isaac Sim, supporting VLN-CE and point/image/trajectory goal navigation.

**Galbot.** Primarily SDK and hardware. Its notable evaluation work is **GraspVLA** (with Peking University's EPIC Lab, CoRL 2025), trained entirely in simulation on SynGrasp-1B — a billion frames of synthetic grasping with domain randomisation.

**Galaxea.** Released GalaxeaVLA (the G0/G0.5 dual-system VLA) and the Galaxea Open-World Dataset (500+ hours of real mobile manipulation in LeRobot v2.1 format). Evaluation relies on LIBERO and SimplerEnv; no proprietary simulator.

**ByteDance Seed — GR-3** (arXiv:2507.15493). GR-3's evaluation is **predominantly real-hardware** (the ByteMini dual-arm mobile robot, 99 objects, 861 trials), not a simulation-benchmark effort.

**Lightwheel.** Co-developed Isaac Lab-Arena with NVIDIA and open-sourced **LW-BenchHub** (268 tasks: 130 Lightwheel-LIBERO plus 138 Lightwheel-RoboCasa). Its benchmarking study reports up to ~13.5× speedup running GR00T N1.5 across 1,024 / 2,048 / 4,096 parallel environments on 8× RTX 6000D, compared with a conventional Robosuite/MuJoCo stack.

### International

**NVIDIA — Isaac Lab-Arena / Isaac Lab / Isaac-GR00T.** Isaac Lab-Arena (with Lightwheel, released pre-alpha) is a composable GPU-accelerated policy evaluation framework integrated with HuggingFace's LeRobot Environment Hub. Per NVIDIA's technical blog, evaluating 10 RoboCasa tasks with Isaac GR00T N1.5 across 4,096 environment variants per task on 8× RTX 6000D took 0.76 hours — 40× faster than the sequential 34.9 hours. Isaac-GR00T supports `UNITREE_G1` as of N1.7 and ships a PolicyServer inference interface.

**Google DeepMind — MuJoCo / MJX / MuJoCo Playground.** MuJoCo (Apache 2.0) is the general-purpose physics engine; Playground builds on MJX with quadruped, humanoid, dexterous-hand, and manipulator environments, emphasising zero-shot sim-to-real from both state and pixel inputs.

**Physical Intelligence — openpi** (~11.8k stars). Open-sources π0, π0-FAST, and π0.5 with LIBERO and ALOHA sim evaluation. Evaluation runs through a policy server (`serve_policy.py`, port 8000) plus a Dockerised LIBERO workflow. Its configs include `roboarena_config`, connecting to RoboArena.

**HuggingFace — LeRobot** (~25.2k stars, v0.6.0). Wraps third-party simulators (LIBERO, Meta-World, RoboTwin 2.0, RoboCasa365) behind Gymnasium interfaces and evaluates them through a single `lerobot-eval` CLI. v0.6.0 added six simulation benchmarks, world-model policies, a reward-models API, and LIBERO-plus with roughly 10,000 perturbation variants.

**Meta — Habitat / PARTNR.** Photorealistic indoor navigation and rearrangement, used as a navigation backend by InternNav among others.

**1X — world model as evaluator** (https://github.com/1x-technologies/1xgpt). The World Model Challenge runs in three stages — compression, sampling, evaluation — with "given N policies, rank them inside the world model" as the ultimate goal. 1X's blog frames this as a fix for irreproducible real-robot evaluation: identical model weights can degrade sharply within days from subtle changes in background or ambient lighting.

## B. Academic benchmarks, classic and recent

- **LIBERO** — the common currency of VLA evaluation. Four suites (Spatial, Object, Goal, Long), 10 tasks and 500 demonstrations each.
- **LIBERO-PRO / LIBERO-Plus** — robustness extensions. LIBERO-PRO (arXiv:2510.03827) perturbs objects, initial states, instructions, and environments; OpenVLA drops from 0.96 to 0.00 and π0 from 0.94 to 0.00 on LIBERO-Goal position shifts, with π0.5 at 0.38 the only non-zero. LIBERO-Plus grades difficulty L1–L5 across roughly 56K robustness scenarios for ten VLAs.
- **ManiSkill3** (~3.1k stars, RSS 2025) — GPU-parallel simulation and benchmarking on SAPIEN, reaching 2000+ FPS in complex ReplicaCAD scenes. SimplerEnv's GPU version is folded into it.
- **robosuite / RoboCasa / RoboCasa365** — robosuite is the modular MuJoCo manipulation framework; RoboCasa is a large kitchen benchmark (120 scenes, 25 atomic plus 75 LLM-generated composite tasks, 100k+ trajectories). RoboCasa365 extends this to 365 kitchen tasks across 2,500 procedurally generated kitchens.
- **Genesis** (~30k stars, Apache 2.0) — unified multi-physics solvers (rigid, MPM, SPH, FEM, PBD, stable fluid) with ray-traced rendering. Now backed by Genesis AI and renamed Genesis World 1.0.
- **Others**: Meta-World, RLBench, CALVIN, VIMA-Bench, ARNOLD, The Colosseum, VLABench, GenManip, EmbodiedBench, MimicGen/DexMimicGen, GenSim/RoboGen, BEHAVIOR-1K/OmniGibson.

## C. Real2sim and sim2real evaluation

- **SimplerEnv** (~950 stars, CoRL 2024). Real-to-sim evaluation of VLAs under two settings — Visual Matching and Variant Aggregation — for Google Robot and WidowX+Bridge, quantifying ranking agreement with MMRV and Pearson correlation.
- **RoboArena** (arXiv:2506.18123, CoRL 2025). Distributed **real-hardware** evaluation via crowdsourced double-blind pairwise comparison: decentralised evaluation of 7 generalist policies across 7 universities, 612 pairwise comparisons, claimed to rank generalist policies more accurately than centralised evaluation.
- **AutoEval** (Berkeley). Autonomous real-robot evaluation with automatic scene reset, execution, and success judgement.
- **3DGS/NeRF reconstruction approaches**: RoboGSim (Megvii, real2sim2real), DISCOVERSE (~476 stars, IROS 2025, Tsinghua AIR, 3DGS + MuJoCo at 650 FPS RGB-D), SplatSim, real2sim-eval.

## D. Humanoid, whole-body, and navigation

- **HumanoidBench** (RSS 2024) — MuJoCo, Unitree H1 with Shadow Hands, 101 DoF, 27–31 tasks, MJX domain randomisation and multimodal observation.
- **LocoMuJoCo** — imitation-learning locomotion benchmark spanning 12 humanoids, 4 quadrupeds, and 4 biomechanical human models, with 22,000+ motion-capture samples.
- **SIMPLE** — couples MuJoCo contact dynamics with Isaac Sim photorealistic rendering across 60 whole-body tasks, 50 indoor scenes, and 1,000+ assets.

## E. Engine and rendering stack comparison

| Engine | Physics | Parallelism / throughput | Rendering | Asset ecosystem | Licence |
|---|---|---|---|---|---|
| **MuJoCo / MJX / MJWarp** | high contact fidelity | MJX GPU parallel | moderate; needs external renderer | MJCF, Menagerie | Apache 2.0 |
| **Isaac Sim + PhysX / Isaac Lab** | PhysX, high fidelity | large-scale GPU parallel | RTX ray tracing, best in class | most complete USD ecosystem | mixed |
| **ManiSkill3 / SAPIEN** | GPU parallel | 2000+ FPS including rendering | raster + ray tracing | URDF and native | MIT / Apache |
| **Genesis / Genesis World** | unified multi-solver | claimed 43M FPS (Franka) | Nyx ray tracing | URDF/MJCF/USD/GLB | Apache 2.0 |
| **PyBullet / Gazebo / Drake** | mature, CPU-oriented | weak parallelism | basic | URDF/SDF | open source |
| **Newton (NVIDIA/DeepMind/Disney)** | next-gen differentiable GPU physics | GPU | — | — | open source |

## F. Methodology and infrastructure

- **Protocol design**: fixed seeds and initial states, success predicates, task tiering, episode counts (the OpenVLA LIBERO protocol uses 50 rollouts per task, 500 per 10-task suite and 2,000 across four suites for one seed; RoboTwin commonly uses 100 rollouts per task), and multi-seed averaging with confidence intervals.
- **Known problems**: seed-driven swings above 30 points; slow evaluation; memorisation overfitting; undocumented protocol details (seeds, normalisation statistics, physics-settling steps routinely omitted from papers); the sim2real gap; and correlation figures published by the platforms themselves.
- **Unified harnesses**: vla-eval (arXiv:2603.13966) uses WebSocket + msgpack with Docker isolation, supports 14 simulation benchmarks and 6 model servers, shards episodes for up to 47× speedup, and archives the Docker image tag, seed, and episode count alongside results.
- **VLA interfaces**: openpi's policy server, InternManip and RoboDojo's client-server designs, LeRobot's unified policy abstraction, and GR00T's PolicyServer — all converging on gRPC or WebSocket remote inference.

## Anchor repositories examined

1. **AgibotTech/GE-Sim-V2** — Genie Envisioner World Simulator 2.0, a Cosmos2-based world-model simulator paired with EWMBench.
2. **AgibotTech/genie_sim** — AgiBot's integrated data-simulation-evaluation platform on Isaac Sim with 3DGS and LLM scene generation, 200+ tasks, five capability suites.
3. **Physical-Intelligence/openpi** — π0/π0.5 with LIBERO and ALOHA sim evaluation behind a policy server.
4. **RoboTwin-Platform/RoboTwin** — dual-arm manipulation benchmark. RoboTwin 2.0 covers 50+ tasks, 731 objects across 147 categories, 100k demonstrations, and heavy domain randomisation. CVPR 2025 Highlight, with a leaderboard and an IsaacLab-Arena branch. ~2.4k stars, MIT.
5. **RoboDojo-Benchmark/RoboDojo** — unified sim-and-real benchmark, 42 simulated plus 18 real tasks across 3 embodiments, running on Isaac Sim across five capability dimensions, paired with RoboDojo-RealEval for remote hardware and XPolicyLab for policies.

## Trends and gaps

1. **No unified evaluation standard yet** — several parties are competing to become one; none has won.
2. **Real2sim credibility rests largely on self-reporting**, with little independent large-scale verification.
3. **World-model evaluation is rising**, with high reported correlations (RoboWorld r = 0.989, PolaRiS r = 0.98) but uncertain maturity.
4. **Benchmark saturation** — high scores under standard protocols carry steadily less information.

## Caveats

- Star counts and update times are approximate, current as of July 2026.
- **Self-reporting bias**: Genie Sim's "<10% sim-real discrepancy, R² = 0.924" and the correlation claims from RoboGSim and DISCOVERSE all come from their own authors, without independent reproduction.
- **GR-3 categorisation**: ByteDance's GR-3 evaluates primarily on hardware and is not a simulation-benchmark project.
- **Fast-moving area**: world-model evaluators, real-to-sim translation, and GPU-parallel evaluation are all iterating quickly; these conclusions may date rapidly.
