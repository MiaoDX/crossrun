# crossrun — Design Document v2.4

> **What this is**: an open-source reference design for running and evaluating robot policies in simulation and on hardware.
> **Status**: design phase. The architecture is under validation; code has not started.
> **Companion research**: see [`research/`](research/).

**Version history.** v1.x was scoped as an internal evaluation system. v2.0 reframed it as an open reference design. v2.1 added the execution layer. v2.2 selected XPolicyLab as a policy-side integration candidate. v2.3 corrected over-strong assumptions and added an explicit Runner-to-Backend contract. **v2.4 chooses one default policy runtime boundary: every production policy is served through a pinned XPolicyLab-compatible service, while original model runtimes and checkpoints may remain intact. It also adds ALOHA Sim beside LIBERO/Panda as a Phase-0 path and defines LeRobot's roles explicitly.**

---

## 1. Positioning

**In one line:** define a small execution layer that owns robot-policy episodes while every policy runs behind one versioned service boundary and every environment sits behind a backend contract.

The project is not another simulator, benchmark, model zoo, training framework, or public policy protocol. Its value is:

1. one episode state machine for simulation and hardware;
2. one policy-service boundary for heterogeneous upstream runtimes;
3. explicit compatibility metadata between a checkpoint and a backend;
4. evaluation records that preserve uncertainty and provenance.

**Deliverables, in priority order:**

1. A design others can inspect and borrow.
2. Reproducible demos that support or falsify the design claims.
3. A working evaluator that carries the first two.

**Success looks like:** someone can select an upstream checkpoint, launch it through the pinned policy-service profile, run a compatible simulation or robot backend, and see exactly which transformations and assumptions produced the result.

---

## 2. Runtime decision

### 2.1 One default policy runtime

All production policy execution goes through a **pinned XPolicyLab-compatible service**.

This is a runtime and serving decision, not a weight-format decision. A policy integration may keep:

- the original repository and inference code;
- the original checkpoint format;
- the original Python/CUDA/dependency environment;
- the original tokenizer, normalisation statistics, action decoder, memory system, or planner.

The model-specific adapter translates between the upstream runtime and the consumed service profile.

```text
 upstream model repository
 + original checkpoint
 + model-specific dependencies
             │
             │ XPolicyLab model adapter
             │ observation/action transforms
             ▼
 pinned XPolicyLab-compatible service
             │
             │ crossrun XPolicyLabPolicyClient
             ▼
           Runner
```

The rule is therefore:

> **Unify executable policy lifecycle and metadata; do not require universal checkpoint conversion.**

### 2.2 Why XPolicyLab is the default

The project begins with many heterogeneous checkpoints from different repositories. They may require incompatible Python, CUDA, Transformers, simulator, C++ or system dependencies. XPolicyLab's model-per-environment and client-server structure is a better match for this intake problem than requiring every model to be reimplemented in one process.

Its current strengths are:

- a large and rapidly growing policy adapter catalog;
- model-side dependency isolation;
- remote or same-machine serving;
- observation/action dictionary conventions;
- single and batched inference lifecycle methods;
- a contribution path focused on wrapping upstream models.

Its weaknesses remain explicit:

- the interface is young and changing;
- operational semantics such as timeout, cancellation, backpressure and failure recovery are not fully specified by method names;
- some stateful policies already need lifecycle extensions beyond the common methods;
- adapter quality and checkpoint reproducibility vary by policy.

Consequently, crossrun pins a reviewed revision and consumes only a documented subset. Phase 0 starts from a reviewed XPolicyLab revision—initially `061093b45bc1b323ed2ce0e50bfa6eb737858a8e`—and upgrades only through the regression gate in §9.

### 2.3 Consumption profile, not a competing standard

crossrun records the exact XPolicyLab methods and payload shapes it consumes in a versioned internal profile.

```yaml
profile: crossrun-xpolicylab-v1
upstream_revision: 061093b45bc1b323ed2ce0e50bfa6eb737858a8e

required_model_lifecycle:
  - reset
  - update_obs
  - get_action

optional_model_lifecycle:
  - update_obs_batch
  - get_action_batch
  - begin_episode
  - step

crossrun_requirements:
  - health reporting
  - declared capabilities
  - request deadlines
  - bounded queues
  - schema and size validation
  - explicit error categories
```

The profile is not advertised as an ecosystem protocol. It exists to:

- detect upstream API drift;
- make service behaviour testable;
- keep the Runner independent from XPolicyLab internals;
- permit a future runtime replacement without rewriting episode logic.

### 2.4 What happens when a model does not fit

A model must not hide important semantics inside arbitrary dictionary fields merely to appear compatible.

Every policy declares capabilities such as:

```yaml
stateful: true
batching: false
custom_episode_start: true
returns_action_chunks: true
external_decoder: false
supports_cancellation: false
planner_latency_class: long
```

For a conventional policy, the adapter implements the common observation-update, action-query and reset lifecycle.

For a stateful planner, world-action model, memory policy or multi-stage agent, one of two things happens:

1. map its lifecycle to a documented optional extension such as `begin_episode` and `step`; or
2. extend the internal service profile after demonstrating that the existing lifecycle loses required semantics.

Failure to fit is evidence about the runtime boundary, not a reason to put model-specific episode logic into the Runner.

---

## 3. LeRobot's roles

LeRobot is deliberately not selected as the sole policy runtime. It remains a major upstream dependency in three independent roles.

### 3.1 LeRobot-native policy source

LeRobot implements policies through `PreTrainedPolicy`, policy configs and pre/post-processing pipelines. A generic XPolicyLab bridge should load LeRobot-native checkpoints where those conventions are sufficient:

```text
 LeRobot checkpoint
 + PreTrainedPolicy implementation
 + processor pipeline and dataset statistics
                │
                ▼
      XPolicyLabLeRobotModel
                │
                ▼
    XPolicyLab-compatible service
```

The bridge is generic only at the LeRobot boundary. Individual policies may still require their plugin package and dependencies in the service environment.

A checkpoint from another runtime is not assumed to load directly into the LeRobot implementation of the same model family. For example, an OpenPI checkpoint and a LeRobot-packaged π model may represent related parameters but use different implementation, configuration or packaging. The choices are:

- run the original checkpoint through its original runtime adapter; or
- use an official or validated conversion and record the converted artifact's lineage.

### 3.2 LeRobot robot backend

LeRobot's `Robot` interface can be adapted as a hardware backend:

```text
LeRobotRobotBackend
├── get observation
├── send action
├── connection and calibration state
└── hardware-specific diagnostics
```

A LeRobot policy does not require a LeRobot robot backend, and a LeRobot robot backend does not require a LeRobot policy.

### 3.3 Dataset and trajectory ecosystem

LeRobotDataset is a candidate interchange format for demonstrations and exported trajectories. Evaluation identity and provenance remain defined by `TrialRecord`; exporting to LeRobotDataset must not discard those fields.

---

## 4. Design claims to test

Each claim must be visible in code and paired with evidence.

1. **One service boundary can federate heterogeneous policy runtimes.** Original weights and dependencies remain isolated behind adapters.
2. **The runner has two explicit boundaries.** `PolicyClient` faces the service; `Backend` faces simulation or hardware.
3. **Checkpoint compatibility is metadata, not a model-name guess.** A policy and backend run only after a profile check.
4. **Models do not live in this repo.** The repo holds contracts, adapters, manifests and reference containers, not model implementations or weights.
5. **The default path must be reproducible.** Exact upstream revisions, checkpoints, transforms and statistics are pinned.
6. **Numbers ship with uncertainty and provenance.** A bare success rate is incomplete.
7. **Perturbation evaluation is part of the baseline.** A standard score alone does not establish robustness.
8. **Cross-backend results are stratified, not pooled.** Report backend-specific outcomes and analyse interactions.
9. **Sim and real share a lifecycle, not identical semantics.** Observation, control, safety, latency and recovery remain backend capabilities.
10. **Runtime escape hatches are evidence-driven.** A direct provider is not added merely because an adapter is inconvenient.

---

## 5. Initial runnable paths

Phase 0 uses two upstream-supported MuJoCo paths. They exercise different policy and embodiment semantics while sharing the same Runner and policy-service boundary.

| Path | Representative upstream config | Observation/action shape | Primary purpose |
|---|---|---|---|
| **LIBERO / Panda** | OpenPI `pi05_libero` or another public LIBERO checkpoint | single-arm state, third-person + wrist images, 7-D relative EEF/gripper action | benchmark reproduction, statistics, perturbations |
| **ALOHA Sim** | OpenPI `pi0_aloha_sim` | dual-arm state, ALOHA cameras, 14-D joint/gripper action | embodiment variation, 50 Hz control, sim-to-real-shaped lifecycle |

The two paths do **not** share a checkpoint or policy profile. They share:

- the XPolicyLab-compatible service lifecycle;
- the crossrun `PolicyClient`;
- the Runner state machine;
- evaluation and provenance infrastructure.

They differ in:

- model/checkpoint;
- observation keys and cameras;
- state and action dimensions;
- action semantics;
- normalisation statistics;
- control frequency and chunk execution.

This distinction is essential: “same runtime boundary” does not mean “same checkpoint works on every robot.”

### 5.1 Baseline snapshot

The table below is a dated survey snapshot, not a compatibility guarantee.

| Ecosystem | Relevant path | crossrun treatment |
|---|---|---|
| OpenPI | LIBERO, ALOHA Sim, ALOHA real, DROID | wrap original runtime/checkpoint through XPolicyLab adapters |
| OpenVLA | LIBERO and other embodiment-specific evaluation paths | wrap original runtime; do not infer ALOHA support from model family alone |
| OpenVLA-OFT | LIBERO plus ALOHA training/evaluation flows | use original runtime adapter unless a validated native package is selected |
| LeRobot | native policies, LIBERO env, ALOHA env, robot drivers | generic policy bridge, hardware backend, dataset interchange |
| XPolicyLab | policy zoo and adapter lifecycle | pinned default service runtime |
| GR00T / WBC / SONIC | embodiment-specific policy and controller stacks | separate policy/runtime, decoder and backend/controller validation |

---

## 6. Architecture and contracts

```text
┌─────────────────────────────────────────────────────────────┐
│ Upstream policy implementations and checkpoints              │
│ OpenPI · OpenVLA · LeRobot · GR00T · WAMs · custom repos    │
└──────────────────────────┬──────────────────────────────────┘
                           │ model adapter
┌──────────────────────────▼──────────────────────────────────┐
│ Pinned XPolicyLab-compatible service                        │
│ isolated dependencies · lifecycle · capability declaration  │
└──────────────────────────┬──────────────────────────────────┘
                           │ XPolicyLabPolicyClient
┌──────────────────────────▼──────────────────────────────────┐
│ Runner                                                       │
│ episode state · deadlines · chunk scheduling · termination  │
│ success · safety · interventions · recording                │
│ consumes: PolicyClient, Backend, EvaluationConfig            │
└──────────────────────────┬──────────────────────────────────┘
                           │ Backend
┌──────────────────────────▼──────────────────────────────────┐
│ Backend adapters                                             │
│ LIBERO · ALOHA Sim · Arena · LeRobot Robot · hardware       │
└─────────────────────────────────────────────────────────────┘

TrialRecord ──► eval-protocol ──► reports / datasets
```

### 6.1 Required contracts

```python
class PolicyClient(Protocol):
    def capabilities(self) -> PolicyCapabilities: ...
    def begin_episode(self, context: EpisodeContext) -> None: ...
    def predict(self, observation: Observation, deadline_ns: int) -> ActionChunk: ...
    def end_episode(self, reason: StopReason) -> None: ...
    def health(self) -> HealthStatus: ...

class Backend(Protocol):
    def capabilities(self) -> BackendCapabilities: ...
    def reset(self, spec: ResetSpec) -> Observation: ...
    def step(self, action: Action, deadline_ns: int) -> StepResult: ...
    def stop(self, reason: StopReason) -> None: ...
```

`XPolicyLabPolicyClient` maps these calls to the pinned service profile. The Runner never imports XPolicyLab model classes.

### 6.2 Policy profile

Every runnable checkpoint has a machine-readable profile:

```yaml
policy_id: openpi/pi0_aloha_sim
model_family: pi0
provider_runtime: openpi
service_adapter: xpolicylab/openpi

checkpoint:
  uri: gs://openpi-assets/checkpoints/pi0_aloha_sim
  digest: required
  source_revision: required

observation:
  state:
    dimension: 14
    semantics: dual_arm_joint_position
  cameras:
    required: [cam_high]
    optional: [cam_left_wrist, cam_right_wrist]

action:
  dimension: 14
  semantics: absolute_joint_position
  units: declared
  chunk_horizon: 10

execution:
  control_hz: 50
  stateful: true
  batching: false

normalisation:
  source: checkpoint
  digest: required
```

The exact values above are illustrative until pinned by the implementation. The schema requirement is not.

### 6.3 Backend capabilities

Each backend declares matching properties:

```yaml
backend_id: aloha_sim/gym_aloha
embodiment: aloha
domain: simulation
observation:
  state_dimension: 14
  cameras: [cam_high]
action:
  dimension: 14
  semantics: absolute_joint_position
control_hz: 50
supports:
  reset: automatic
  privileged_success: true
  parallel_instances: false
```

### 6.4 Compatibility preflight

Before an episode starts, crossrun checks at least:

- required observation keys and shapes;
- camera count, naming and image properties;
- action dimension, semantic type, frame and units;
- absolute versus relative control;
- normalisation statistics and their digest;
- control frequency and action chunk timing;
- required decoder or low-level controller;
- stateful reset semantics;
- batching and concurrency;
- timeout and cancellation capability.

A mismatch is rejected or resolved by an explicitly named adapter. It is never silently inferred from a shared model-family name.

### 6.5 Action and step records

`Action` records at least:

- semantic type: joint, end-effector, velocity, torque, latent token or another declared type;
- dimensions and chunk horizon;
- frame and units where applicable;
- intended control frequency or timestamps;
- validity limits;
- decoder identity when actions are latent.

`StepResult` records at least:

```text
observation
backend timestamp and wall-clock timestamp
terminated / truncated
success observation or success-label input
failure reason
safety events
intervention state
backend-specific diagnostics
```

---

## 7. Repository shape and dependency rules

```text
crossrun/
├── contracts/                 # PolicyClient, Backend, profiles, TrialRecord
├── runner/                    # episode state machine; no upstream imports
├── adapters/
│   ├── policy/
│   │   └── xpolicylab/       # the production policy boundary
│   └── backend/               # LIBERO, ALOHA Sim, hardware, later backends
├── service-profiles/          # pinned XPolicyLab consumption profiles
├── policy-manifests/          # checkpoint/profile metadata, no weights
├── eval-protocol/             # statistics, reports, comparisons
├── containers/                # service and backend reference templates
├── assets/                    # source manifests and generation scripts
├── ci/
└── docs/
```

**Hard constraints:**

- `runner/` imports contracts, not XPolicyLab, simulator or robot-driver packages.
- Production policies are reached through `XPolicyLabPolicyClient` and a pinned service profile.
- Model-specific transforms stay in the policy-service environment, not in the Runner.
- Backend-specific types do not leak into `PolicyClient` or `TrialRecord`.
- `eval-protocol/` consumes `TrialRecord` objects, not a lossy `(success, trajectory)` tuple.
- No model weights or upstream model implementations are committed to this repository.
- Checkpoint conversion is optional; every conversion records source artifact, converter revision and validation result.
- Large binary assets and datasets live outside git with immutable hashes.
- Policy servers default to loopback. Remote mode is explicit and authenticated.

### 7.1 Minimum TrialRecord identity

```text
trial_id and run_id
task and task-definition revision
seed and initial-condition identifier
policy profile and service-profile revision
upstream runtime revision
checkpoint digest and conversion lineage
container digest
normalisation-statistics digest
backend, backend version and physics configuration
perturbation tier and parameters
action/control configuration
termination and failure reason
safety events and interventions
success label, source, classifier version, confidence and audit state
trajectory or trajectory digest
```

---

## 8. Evaluation protocol

### 8.1 Cross-backend comparison

The same nominal task can differ across physics engines because of contact, friction, rendering, control and solver behaviour.

- Never pool results from different backends into one success rate.
- Report backend and version as first-class grouping variables.
- Absolute success rates may appear together when clearly stratified; do not interpret them as equal-difficulty measurements.
- Compare ranking agreement, matched failure modes and policy-by-backend interactions.
- A ranking reversal is evidence of an interaction, not proof that either backend is wrong.
- Preserve pairing when matched seeds or initial conditions exist.

### 8.2 Statistical discipline

- Record the complete trial identity above.
- Report a binomial interval for each policy × task × backend × perturbation condition.
- Clopper–Pearson is a conservative default; alternatives may be added if named explicitly.
- Do not infer equality from overlapping confidence intervals.
- Predeclare comparisons and correct for multiplicity where appropriate.
- Use paired tests or hierarchical models for shared seeds or initial conditions.
- Separate rollout-sampling uncertainty from success-label uncertainty.

**Power reference:** 63 successes in 70 Bernoulli trials gives an exact 95% interval approximately 15 percentage points wide. Achieving a roughly ±2-point interval near 90% success requires on the order of one thousand trials; the exact requirement depends on interval and stopping rule.

### 8.3 Perturbation tiers

Standard and perturbed results are reported together. Perturbations record generation seed and parameters and should distinguish:

- visual appearance and distractors;
- lighting and rendering;
- target or object pose;
- observation degradation;
- action latency or control disturbance, where safe.

A collapse under one perturbation is evidence about that perturbation family, not a universal statement that the model has no capability.

### 8.4 Hardware success labels

A learned classifier is a measurement instrument, not ground truth. Hardware reports include:

- classifier and prompt/configuration revision;
- score and abstention behaviour;
- a sampled human-audit set;
- confusion estimates on the evaluated task distribution;
- the policy for disagreements and uncertain outcomes.

---

## 9. Scope and phases

### Phase 0 — one service, two MuJoCo paths

- [ ] Pin the reviewed XPolicyLab revision and write `crossrun-xpolicylab-v1`.
- [ ] Implement `PolicyClient`, `Backend`, `PolicyProfile`, `BackendCapabilities`, `StepResult` and `TrialRecord`.
- [ ] Implement `XPolicyLabPolicyClient` without importing model implementations into crossrun.
- [ ] Bring up **LIBERO/Panda + `pi05_libero`** through the pinned service.
- [ ] Bring up **ALOHA Sim + `pi0_aloha_sim`** through the same service and Runner.
- [ ] Demonstrate that only profiles and adapters differ; episode logic remains unchanged.
- [ ] Implement compatibility preflight and reject intentional mismatch fixtures.
- [ ] Implement provenance, trajectory digests and confidence intervals.
- [ ] Add schema validation, deadlines, bounded queues and loopback-only defaults.

### Phase 1 — policy intake

- [ ] Add an OpenVLA or OpenVLA-OFT checkpoint through its original runtime.
- [ ] Build `XPolicyLabLeRobotModel` for LeRobot-native `PreTrainedPolicy` checkpoints.
- [ ] Validate one LeRobot-native checkpoint against its published baseline.
- [ ] Add one world-action or memory/planning policy with declared stateful capabilities.
- [ ] Test a policy that does not support batching.
- [ ] Test server crash, timeout, malformed payload, stale observation and cancellation behaviour.
- [ ] Publish an adapter checklist and conformance report for every policy.

### Phase 2 — claims and minimal hardware loop

- [ ] Protocol/sample-size sensitivity demo.
- [ ] Controlled perturbation-collapse demo.
- [ ] Implement a RealRunner skeleton with manual reset guidance.
- [ ] Use LeRobot's `Robot` abstraction for one hardware backend where practical.
- [ ] Record success-label provenance, safety events and interventions from the first hardware run.
- [ ] Test an ALOHA real path only with a checkpoint/profile matching the actual hardware configuration.

### Phase 3 — second backend and whole-body control

- [ ] Add a second implementation of a matched task only after task and control semantics are documented.
- [ ] Validate Arena's G1 path independently.
- [ ] Validate GR00T-WBC/SONIC encoder, decoder, hands, cameras and controller independently.
- [ ] Connect the stacks only after both run separately.
- [ ] Keep the low-level controller swappable so policy and controller failures can be separated.

### Phase 4 — G2 and outward work

- [ ] Pin Genie Sim source revisions and inventory file-level licenses.
- [ ] Produce versioned G2 assets and scripts where redistribution is allowed.
- [ ] Keep the default paths runnable without the upstream asset repository.
- [ ] Publish negative results and rejected integrations, not only successful demos.
- [ ] Consider EnvHub and upstream contributions after contracts survive multiple policies and backends.

### 9.1 XPolicyLab upgrade gate

Never track XPolicyLab `main` implicitly. An upgrade is a separate change with:

1. an old-to-new consumption-profile diff;
2. adapter static checks;
3. debug closed-loop checks;
4. end-to-end regression on:
   - π0.5 + LIBERO/Panda;
   - π0 + ALOHA Sim;
   - one LeRobot-native policy through the generic bridge;
   - one stateful or world-action policy;
   - one non-batched policy;
5. performance measurements for latency, throughput and memory;
6. result comparison against the previous pinned service revision.

An upstream addition is not automatically enabled merely because it exists in the XPolicyLab catalog.

---

## 10. Risks, security and licensing

| Risk | Level | Response |
|---|---|---|
| **XPolicyLab interface churn** | High | pin exact revisions; maintain consumption profile; regression-gated upgrades |
| **Adapter quality varies by model** | High | conformance tests, known-good checkpoint run and provenance manifest |
| **Runtime boundary cannot express a complex policy** | High | capability extensions; do not leak policy logic into Runner |
| **Checkpoint/model-family confusion** | High | immutable policy profiles and compatibility preflight |
| **LeRobot bridge falsely appears universal** | Med-high | define supported `PreTrainedPolicy` subset; require policy-specific validation |
| **Runner/backend boundary leaks** | High | second-backend gate; prohibit upstream types in contracts |
| **Success-label error on hardware** | High | audit labels and report measurement process |
| **Cross-backend overinterpretation** | Med-high | stratified reports and interaction analysis |
| **Asset/license ambiguity** | High | per-file and per-weight manifests; no blanket ecosystem claim |

### 10.1 Security baseline

- no pickle or executable deserialisation across the network boundary;
- schema and size validation for every request and response;
- loopback binding by default;
- authenticated and encrypted remote transport through a private network or tunnel;
- network allow-lists and explicit remote-mode configuration;
- request deadlines, rate limits and bounded queues;
- least-privilege containers or processes;
- health checks and a fail-safe stop path when the policy service is unavailable.

### 10.2 Licensing baseline

Every external artifact or weight records:

```text
source URL and exact revision
file or package license
model-weight license, if distinct
redistribution and derivative permissions
required notices and attribution
use restrictions
local modifications and generated-file provenance
conversion lineage
```

Open-source and non-commercial intent do not remove these obligations.

---

## 11. Explicitly rejected for the initial scope

| Option | Why not now |
|---|---|
| Support several first-class policy runtimes directly in Runner | multiplies lifecycle and failure semantics; use one service boundary |
| Convert every checkpoint to one universal weight format | expensive, lossy and unnecessary for execution unification |
| Require every model to become LeRobot-native | forces reimplementation or conversion before evaluation value is proven |
| Treat XPolicyLab as a permanent public standard | it is a pinned dependency behind crossrun contracts |
| Track XPolicyLab `main` automatically | rapid changes would invalidate reproducibility |
| Put model-specific observation/action transforms in Runner | destroys separation and makes every model a Runner change |
| Infer compatibility from model name or embodiment label | ignores checkpoint, normalisation, action and timing differences |
| Run LIBERO tasks on ALOHA and call it the original LIBERO baseline | embodiment transfer is a different configuration requiring validation |
| Put model code and weights in this repo | creates dependency and licensing conflicts |
| Report one pooled number across backends | conflates condition changes with policy performance |

---

## 12. Open questions

These are measurements to make, not conclusions already reached.

1. Which exact XPolicyLab methods and transport messages should `crossrun-xpolicylab-v1` pin?
2. Is the selected XPolicyLab revision reliable under concurrent batched load, timeout and cancellation?
3. Which stateful policies require `begin_episode`/`step` rather than the common update/query lifecycle?
4. How much of the LeRobot policy catalog can one generic bridge cover without policy-specific code?
5. Can converted and original-runtime versions of the same model be shown behaviourally equivalent on fixed observations?
6. Which fields are sufficient for a safe policy/backend compatibility preflight?
7. How should action chunks be resampled when service and backend frequencies differ?
8. Which ALOHA Sim task/checkpoint pair is the best stable Phase-0 baseline?
9. Which real ALOHA hardware configuration matches available public checkpoints closely enough for a meaningful run?
10. Which XPolicyLab adapters have a publicly reproducible known-good result rather than only a wiring check?
11. Can a second backend reproduce a matched task closely enough for an interpretable interaction study?
12. Which upstream assets and weights can legally be redistributed, and which must be fetched by the user?

---

## Links

| | |
|---|---|
| XPolicyLab | https://github.com/XPolicyLab/XPolicyLab |
| OpenPI | https://github.com/Physical-Intelligence/openpi |
| OpenVLA | https://github.com/openvla/openvla |
| OpenVLA-OFT | https://github.com/moojink/openvla-oft |
| LeRobot | https://github.com/huggingface/lerobot |
| LIBERO | https://github.com/Lifelong-Robot-Learning/LIBERO |
| gym-aloha | https://github.com/huggingface/gym-aloha |
| Isaac Lab-Arena | https://github.com/isaac-sim/IsaacLab-Arena |
| Newton Physics | https://github.com/newton-physics/newton |
| Isaac-GR00T | https://github.com/NVIDIA/Isaac-GR00T |
| GR00T Whole-Body Control / SONIC | https://github.com/NVlabs/GR00T-WholeBodyControl |
| RoboDojo | https://github.com/RoboDojo-Benchmark/RoboDojo |
| Genie Sim | https://github.com/AgibotTech/genie_sim |
| SIMPLE | https://github.com/physical-superintelligence-lab/SIMPLE |
| Genesis | https://github.com/Genesis-Embodied-AI/genesis-world |
| AutoEval | https://auto-eval.github.io/ |
