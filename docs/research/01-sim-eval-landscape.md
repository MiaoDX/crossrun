# The Robot Sim Eval Open-Source Landscape

> Surveyed July 2026 · the starting point for crossrun's design
> ⚠️ This is a dated survey, not a current compatibility matrix. See [README](README.md). Clear factual and arithmetic corrections are applied in place.

## TL;DR

- **The field has split into four lines of work**: ① classic academic manipulation benchmarks (LIBERO, ManiSkill, RoboCasa) acting as common currency; ② integrated company platforms bundling data, simulation, and evaluation (AgiBot Genie Sim, NVIDIA Isaac Lab-Arena, InternRobotics, RoboTwin); ③ real2sim approaches (SimplerEnv, 3DGS reconstructions like RoboGSim and DISCOVERSE) alongside distributed real-robot evaluation (RoboArena, AutoEval); ④ the emerging use of world models as evaluators (1X World Model, GE-Sim, GigaWorld).
- **Fifteen repos worth watching at the survey date**: huggingface/lerobot, Physical-Intelligence/openpi, isaac-sim/IsaacLab-Arena, NVIDIA/Isaac-GR00T, haosulab/ManiSkill3, AgibotTech/genie_sim, RoboTwin-Platform/RoboTwin, RoboDojo-Benchmark/RoboDojo, simpler-env/SimplerEnv, InternRobotics, Genesis-Embodied-AI/genesis-world, google-deepmind/mujoco, TATP-233/DISCOVERSE, ARISE-Initiative/robosuite and RoboCasa, and humanoid-bench.
- **The core finding**: no single benchmark is a final arbiter of policy quality. Simulated evaluation is sensitive to protocol and can be gamed by memorisation. LIBERO-PRO reports models above 90% under standard LIBERO collapsing to 0.0% under one generalized position setting (OpenVLA 0.96→0.00 and π0 0.94→0.00 on LIBERO-Goal).

## Key findings

1. **Integrated platforms are common company infrastructure.** AgiBot Genie Sim, NVIDIA Isaac Lab-Arena, and Shanghai AI Lab's InternRobotics connect environment generation, data collection, and evaluation, often with LLM/VLM-driven scene or configuration generation.
2. **VLA policy interfaces are converging on decoupled execution.** OpenPI's policy server, InternManip's client-server split, RoboDojo/XPolicyLab, and LeRobot's policy/evaluation abstractions all separate policy dependencies from environment execution, but their lifecycle and failure semantics are not interchangeable by default.
3. **Real2sim credibility has quantitative, mostly self-reported support.** SimplerEnv measures ranking agreement with MMRV and correlation. Genie Sim 3.0 reports strong agreement across selected model configurations, but platform-authored numbers require independent validation.
4. **World models as evaluators moved from concept to benchmark.** 1X, GE-Sim/EWMBench, WorldEval, RoboWorld, and GigaWorld/WMBench investigate whether generated rollouts can rank policies, with maturity and external replication still uneven.
5. **Reproducibility problems are stated openly.** Perturbation suites and reproduction studies show that score, seed, initial states, processors, action scheduling, and simulator settings must be recorded as first-class protocol identity.

## A. Company and institutional platforms

### China

**AgiBot — Genie Sim / Genie Sim Benchmark** (https://github.com/AgibotTech/genie_sim, MPL 2.0 for relevant source areas at the surveyed revision). Built on Isaac Sim, with reconstructed scenes, USD assets, and generated scenes/tasks/evaluation configurations. Public claims included 200+ tasks, 100,000+ evaluation scenarios, and support for several policy families; these figures are dated and platform-authored.

**AgiBot — GE-Sim / GE-Sim-V2 / Genie-Envisioner.** A world-model simulation and evaluation line paired with EWMBench, which scores generated rollouts along visual, motion, and semantic dimensions.

**Beijing Innovation Center of Human Robotics / X-Humanoid.** RoboMIND and related assets, locomotion, data, and toolchain work span several embodiments. Exact dataset counts and supported paths should be read from the pinned source used by a bundle rather than this survey.

**Shanghai AI Lab / InternRobotics.** InternUtopia, InternManip, and InternNav cover large-scale embodied simulation, manipulation training/evaluation, and navigation. InternManip is especially relevant for its client-server policy deployment and unified environment surface.

**Galbot.** Primarily an SDK and hardware organization; GraspVLA is a notable simulation-trained manipulation result rather than a general open simulation-evaluation framework.

**Galaxea.** Released policy and dataset work with evaluation on public benchmarks such as LIBERO and SimplerEnv; no separate proprietary simulator was identified in the survey.

**ByteDance Seed — GR-3.** Evaluation was predominantly real-hardware and should not be categorized as a simulation benchmark.

**Lightwheel.** Co-developed Isaac Lab-Arena and released LW-BenchHub. Lightwheel-LIBERO tasks are adapted Isaac/Arena environments, not the original MuJoCo LIBERO benchmark. Reported large-scale parallel speedups depend on the exact task, rollout, hardware, and comparison setup.

### International

**NVIDIA — Isaac Lab-Arena / Isaac Lab / Isaac-GR00T.** Arena is a composable policy-evaluation environment layer. Isaac Lab has evolved into a **multi-backend** architecture: PhysX remains the default Isaac Sim path, while Newton-MJWarp, Newton-Kamino, and OvPhysX are selectable paths with different maturity and task coverage. “Moving to Newton” is therefore not equivalent to replacing or removing PhysX.

**Google DeepMind — MuJoCo / MJX / MuJoCo Playground.** MuJoCo is the general-purpose physics engine; MJX and Playground provide accelerated and ready-to-use learning environments. MuJoCo includes a native visualizer and OpenGL onscreen/offscreen renderer; an external renderer is optional, not required.

**Physical Intelligence — OpenPI.** Open-sources π0, π0-FAST, and π0.5 implementations and supported evaluation paths including LIBERO and ALOHA Sim. Its original configs, checkpoints, processors, and evaluators form distinct baseline lineages that should not be silently replaced by ports of the same model family.

**Hugging Face — LeRobot.** Provides policy implementations, processor pipelines, datasets, robot drivers, and Gymnasium-oriented simulation evaluation. LeRobot-native checkpoints or additionally fine-tuned ports are separate artifact lineages from original-runtime checkpoints unless equivalence is explicitly validated.

**Meta — Habitat / PARTNR.** Photorealistic indoor navigation and rearrangement infrastructure, relevant mainly outside the initial manipulation-focused scope.

**1X — world model as evaluator.** The World Model Challenge frames policy ranking inside a learned simulator as an explicit goal, motivated by the expense and instability of repeated real-robot evaluation.

## B. Academic benchmarks, classic and recent

- **LIBERO** — a common VLA evaluation currency. The widely used four 10-task evaluation suites are Spatial, Object, Goal, and Long (`libero_10`), alongside the broader LIBERO-100 split.
- **LIBERO-PRO / LIBERO-Plus** — robustness extensions that vary objects, initial states, language, appearance, and environment factors. Their results support perturbation testing, but a collapse under one perturbation family should not be generalized to all capability.
- **ManiSkill3 / SAPIEN** — GPU-parallel simulation and benchmarking with high-throughput rendering claims that depend on scene and hardware.
- **robosuite / RoboCasa / RoboCasa365** — MuJoCo-based modular manipulation and large kitchen task suites.
- **Genesis World** — a simulation platform with multiple physics solvers, multiple rendering paths, cross-platform compilation, sensors, controllers, and parallel/heterogeneous environments.
- **Others**: Meta-World, RLBench, CALVIN, VIMA-Bench, ARNOLD, The Colosseum, VLABench, GenManip, EmbodiedBench, MimicGen/DexMimicGen, GenSim/RoboGen, and BEHAVIOR-1K/OmniGibson.

## C. Real2sim and sim2real evaluation

- **SimplerEnv** — real-to-sim VLA evaluation for selected robot/task settings, using ranking and correlation metrics rather than assuming equal absolute difficulty.
- **RoboArena** — distributed real-hardware evaluation using pairwise comparisons across institutions.
- **AutoEval** — autonomous real-robot evaluation with reset, execution, and learned success judgment.
- **3DGS/NeRF reconstruction approaches** — RoboGSim, DISCOVERSE, SplatSim, and related systems combine reconstructed appearance with a physical simulator or learned dynamics.

## D. Humanoid, whole-body, and navigation

- **HumanoidBench** — MuJoCo-based humanoid RL benchmark work; task count and supported embodiments changed over time.
- **LocoMuJoCo** — imitation-learning locomotion benchmark across humanoid, quadruped, and biomechanical models.
- **SIMPLE** — couples MuJoCo contact dynamics with Isaac Sim rendering for whole-body tasks; its rendering throughput and dependency stack must be evaluated against actual episode length and parallelism rather than compared directly with rollout count.

## E. Engine and rendering stack comparison

| Stack | Physics | Parallelism / throughput | Rendering | Asset ecosystem | License note |
|---|---|---|---|---|---|
| **MuJoCo / MJX / MJWarp** | MuJoCo-family rigid-body/contact paths | CPU MuJoCo plus accelerated MJX/MJWarp paths | native OpenGL onscreen/offscreen rendering; external renderers optional | MJCF, Menagerie and benchmark-specific assets | MuJoCo Apache-2.0; asset licenses vary |
| **Isaac Sim / Isaac Lab** | PhysX default, with selectable Newton-MJWarp, Newton-Kamino and OvPhysX paths in newer Isaac Lab | large-scale GPU paths, task-dependent | Isaac Sim RTX, Newton renderer, or OvRTX depending on selected stack | extensive USD ecosystem | mixed dependency and asset terms |
| **ManiSkill3 / SAPIEN** | SAPIEN GPU simulation | GPU-parallel, workload-dependent | raster and ray-tracing paths | URDF and native assets | component-specific open-source licenses |
| **Genesis World** | unified multi-solver platform | parallel and heterogeneous environments, workload-dependent | Nyx, Luisa and Pyrender paths | URDF/MJCF/USD/mesh formats | code and optional components must be audited separately |
| **PyBullet / Gazebo / Drake** | mature general robotics engines/frameworks | typically less focused on massive visual-policy parallelism | built-in or ecosystem rendering | URDF/SDF and framework-specific assets | project-specific open-source licenses |

The rows are not one symmetric “physics backend” axis. Isaac Lab is an environment/research framework with selectable physics/rendering backends; Genesis World is a broader simulation platform; MuJoCo is principally a physics engine with native visualization. crossrun records these dimensions separately.

## F. Methodology and infrastructure

- **Protocol design**: fixed seeds and initial states, success predicates, task tiering, processor identity, action scheduling, and episode counts. The OpenVLA LIBERO protocol uses 50 rollouts per task, 500 per 10-task suite, and 2,000 across four suites for one seed; other evaluators may use smaller smoke or reproduction protocols and must name them.
- **Known problems**: seed-driven variation, slow evaluation, memorisation, omitted normalization and preprocessing details, undocumented settling/replanning behavior, simulator gaps, and platform self-reporting.
- **Unified harnesses**: multiple projects use WebSocket/gRPC-like policy serving, Docker/process isolation, sharded episodes, and result manifests. Similar transport does not establish equivalent reset, batching, timeout, retry, or state semantics.
- **VLA interfaces**: OpenPI, LeRobot, XPolicyLab/RoboDojo, InternManip, and GR00T all expose useful abstractions, but cross-project compatibility remains checkpoint-, processor-, action-, timing-, and lifecycle-specific.

## Anchor repositories examined

1. **AgibotTech/GE-Sim-V2** — world-model simulator and evaluator work.
2. **AgibotTech/genie_sim** — integrated data/simulation/evaluation platform on Isaac Sim.
3. **Physical-Intelligence/openpi** — original π policy implementations and supported LIBERO/ALOHA evaluation paths.
4. **RoboTwin-Platform/RoboTwin** — dual-arm manipulation benchmark and data ecosystem.
5. **RoboDojo-Benchmark/RoboDojo** — Isaac-based sim/real benchmark with XPolicyLab policy integrations.

## Trends and gaps

1. **No unified evaluation standard has won.** Several projects cover overlapping but non-identical policy, task, runtime, and reporting surfaces.
2. **Real2sim credibility rests heavily on setting-specific and often self-reported evidence.**
3. **World-model evaluation is rising but still lacks stable external validation across broad policy/task distributions.**
4. **Benchmark saturation makes protocol, perturbation, and provenance increasingly important.**

## Caveats

- Repository activity, versions, stars, task counts, supported models, and branch status are dated July 2026 snapshots and may change quickly.
- Platform-authored correlation, throughput, or sim-real claims should be treated as self-reported until independently reproduced.
- License labels at repository level do not automatically cover every bundled asset, dataset, model weight, or proprietary runtime dependency.
- An adapter catalog entry proves wiring, not checkpoint compatibility, task suitability, or a reproduced result.
