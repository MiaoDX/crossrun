# crossrun — Design Document v2.9

> **What this is**: a simulation-first evaluation surface and best-practice incubator for robot policies across multiple physics backends.
> **Status**: design phase. The architecture is under validation; code has not started.
> **Companion research**: see [`research/`](research/).

**Version history.** v1.x was scoped as an internal evaluation system. v2.0 reframed it as an open reference design. v2.1 added the execution layer. v2.2 selected XPolicyLab as a policy-side integration candidate. v2.3 corrected over-strong assumptions and added an explicit execution/environment contract. v2.4 chose one default policy runtime boundary and added ALOHA Sim beside LIBERO/Panda. v2.5 defined that boundary as a pinned crossrun-maintained XPolicyLab fork and made Phase 0 π0.5-only. v2.6 aligned policy intake and hardware scope with current upstreams. v2.7 made crossrun an integration distribution around upstream-native execution, reproducible bundles and governed overlays. v2.8 added best-practice reference integrations. **v2.9 reduces the roadmap to three simulation-first phases, gives crossrun a bounded Gymnasium-compatible sim-eval loop, targets MuJoCo, Isaac and Genesis adapters, and defers all real-robot implementation until the simulation gate passes.**

---

## 1. Positioning

**In one line:** run one policy-evaluation loop against swappable MuJoCo, Isaac and Genesis environments while keeping every backend's semantics and provenance explicit.

The project is not another simulator, benchmark, model zoo, training framework or universal sim/real runtime. It deliberately owns one bounded simulation-evaluation loop. Its short-term value is:

1. exact upstream, checkpoint, container and asset revisions that work together;
2. focused wrappers, plugins, reference integrations or fork commits for gaps and architectural experiments;
3. a Gymnasium-compatible sim interface with explicit backend capabilities;
4. one evaluation, launch and conformance surface across otherwise separate upstream tools;
5. evaluation records that preserve uncertainty, upstream identity and local modifications.

**Deliverables, in priority order:**

1. One reproducible π0.5 + LIBERO + MuJoCo result through the crossrun sim-eval loop.
2. Conformance fixtures that make upstream updates and patch drift visible.
3. MuJoCo, Isaac and Genesis adapters behind the same narrow interface.
4. Best-practice reference implementations evaluated against current upstream baselines.
5. Generally useful fixes proposed to their natural upstream owners after they work in practice.

**Success looks like:** someone can materialize a bundle, evaluate a selected policy through one sim loop, change the backend in configuration, and receive a comparable evidence envelope that still exposes engine-specific differences.

---

## 2. Integration strategy

### 2.1 One bounded sim-eval owner

crossrun owns the portable simulation-evaluation loop: episode reset, policy calls, action-chunk scheduling, termination, recording and evidence emission. The loop consumes one narrow simulation interface and does not import MuJoCo, Isaac or Genesis types.

Maintained upstream simulation paths remain runnable result baselines:

| Native baseline path | Owner |
|---|---|
| LeRobot policy + supported simulator | `lerobot-eval` |
| RoboDojo benchmark | RoboDojo `EvalEnv` |
| Heterogeneous original-runtime policy | XPolicyLab service, bridged into the selected environment owner |
| Arena/EnvHub task | the Arena or EnvHub environment package plus its supported evaluator |

The crossrun path is considered correct only after it agrees with the selected native baseline on fixed observations, actions, seeds and termination semantics within declared tolerances. Upstream lifecycle code is not copied; backend adapters compose public environment APIs.

This ownership is intentionally simulation-only. Hardware rollout, intervention, safety control and manual reset semantics stay outside the current milestone.

### 2.2 Common simulation interface

The common surface follows Gymnasium because LIBERO, gym-aloha, LeRobot and Isaac integrations already converge around its lifecycle. A backend adapter creates an environment that provides:

```python
class SimBackend(Protocol):
    def capabilities(self, config: SimConfig) -> SimCapabilities: ...
    def make(self, config: SimConfig) -> gym.Env | gym.vector.VectorEnv: ...
```

The returned environment uses standard Gymnasium fields:

```text
observation_space and action_space
reset(seed, options) -> observation, info
step(action) -> observation, reward, terminated, truncated, info
render() and close()
```

`SimCapabilities` declares engine and version, devices, headless rendering, vectorisation, deterministic-seed level, control decimation, state snapshot support, privileged success access and native diagnostics. Backend-specific information stays under a namespaced `info` key and in the native result artifact.

The interface standardises lifecycle, not physics or task meaning. A task/embodiment profile separately declares observations, actions, cameras, controllers, reset distribution and success semantics. Switching `backend: mujoco` to `backend: isaac` or `backend: genesis` is allowed only when a conformance fixture exists for that profile; otherwise it is a new environment condition.

### 2.3 Best-practice reference track

A reference integration is first-class work, not an accidental patch. It must state:

```yaml
hypothesis: required
native_baseline: required
reference_architecture: required
expected_improvement: required
comparison_dimensions: required
semantic_equivalence_limits: required
upstream_destination_candidates: required
long_term_options: [upstream, crossrun, fork, delete]
review_after: required
```

For example, RoboDojo currently supplies a valid native baseline using its own managers directly on Isaac Sim and a pinned Isaac Lab fork. crossrun may additionally implement an Arena-based reference path using pinned RoboDojo task intent and assets where licensing permits. That path must not be called upstream RoboDojo or benchmark-equivalent until task, reset, observation, action, controller and success semantics pass declared conformance checks.

The comparison asks whether Arena materially improves composition, standard scene/robot configuration, reuse across embodiments, vectorisation, integration with EnvHub/LeRobot, and upgrade cost. It also measures migration effort, performance regressions, semantic drift and the new dependency surface. “Arena is newer” is not evidence that it is better.

The implementation lands in crossrun first. Once it is runnable and measured:

1. propose small generic fixes to their natural upstream;
2. offer a larger upstream migration only with evidence and a maintainable series of changes;
3. retain the reference integration or a dedicated fork if upstream scope or priorities differ; or
4. delete it if the measured benefit does not justify its maintenance cost.

Upstream acceptance is an outcome, not a prerequisite for experimentation.

### 2.4 Wrapper, plugin, patch, fork

Compatibility fixes escalate through the least expensive mechanism that works. A reference integration may deliberately use a larger mechanism when the architecture itself is the hypothesis:

1. **Wrapper:** compose public commands or APIs without modifying upstream code.
2. **Plugin:** use an upstream extension point, preferably out of tree until stable.
3. **Patch:** carry a small temporary source change that can be replayed onto one exact base revision.
4. **Fork:** maintain source-side changes that need their own CI, release artifact or coordinated multi-file evolution.

A fork or reference integration creates maintenance cost, but it can be the correct incubation vehicle. Each local delta records:

```yaml
source_upstreams: required
upstream_base_revisions: required
local_revision_or_patch_digest: required
owner: required
rationale: required
lifecycle_intent: temporary_hotfix | upstream_candidate | crossrun_reference
affected_bundles: required
regression_fixtures: required
upstream_issue_or_pr: required_for_upstream_candidate
review_after: required
exit_or_reassessment_condition: required
```

No production machine contains an unrecorded manual edit. A clean checkout plus the bundle manifest must reproduce the full source state.

### 2.5 The optional XPolicyLab fork

crossrun maintains a pinned XPolicyLab fork when a selected policy needs heterogeneous original-runtime serving and current service or Pi adapter gaps require source-side changes. It is an enabling dependency, not a phase or prerequisite for the first native π0.5 result. The fork:

- regularly syncs a reviewed upstream remote;
- keeps generally useful changes as clean topic commits suitable for upstream PRs;
- isolates crossrun-only bundle metadata from upstream-generic runtime changes;
- never becomes the home for environment loops, simulator code or robot drivers;
- is consumed only at exact upstream-base and fork revisions.

**Upstream check, 2026-07-28.** XPolicyLab `main` was `5071d8ff557f8f258e50aec5b46a701772bc3295`. Its model lifecycle and WebSocket implementation still lack declared capabilities, bounded payloads, end-to-end deadlines and true transport batching. The WebSocket client implements a batch request as sequential `infer` calls. The Pi adapters require all three ALOHA cameras even though OpenPI permits masked wrist cameras, and the vendored config does not include the public `pi05_aloha` or `pi05_libero` profiles. These are initial fork patches and upstream PR candidates, not permanent crossrun features.

### 2.6 Bundle manifest and service profile

The bundle manifest is the top-level reproducibility unit. It pins every upstream and local delta and references narrower runtime profiles:

```yaml
bundle: pi05-libero-mujoco-v1
crossrun_revision: required

upstreams:
  lerobot: {revision: required}
  openpi: {revision: required}
  libero: {revision: required}

optional_services:
  xpolicylab:
    enabled: false
    upstream_base: required_when_enabled
    fork_revision: required_when_enabled

artifacts:
  checkpoints: [{uri: required, digest: required}]
  containers: [{name: required, digest: required}]

overlays:
  plugins: [{name: required, revision: required}]
  patches: [{name: required, base: required, digest: required}]

fixtures:
  - pi05_libero_smoke
```

ALOHA Sim is added as a second bundle or a later revision after the LIBERO result passes; its dependencies and fixture do not block materializing the first bundle.

The XPolicyLab service profile records only the wire and lifecycle subset consumed by the bridge:

```yaml
profile: crossrun-xpolicylab-v1

upstream_model_lifecycle:
  required:
    - reset
    - update_obs
    - get_action
  optional:
    - update_obs_batch
    - get_action_batch
    - prepare_case
    - on_trial_end

upstream_wire_messages:
  required:
    - hello
    - reset
    - infer
    - trial_end
    - heartbeat

fork_service_extensions:
  - health and version reporting
  - declared capabilities
  - request deadlines
  - bounded queues
  - schema and size validation
  - explicit error categories

bridge_lifecycle: [reset, predict, end_episode]
```

Neither manifest is advertised as an ecosystem standard. They exist to:

- detect upstream API drift;
- make patch replay and service behaviour testable;
- keep environment owners independent from XPolicyLab internals;
- reproduce a complete source and artifact state from a clean workspace.

The profile must not call a loop of single-item requests "batched inference". Transport batching is declared only after one request reaches a model-side batch implementation.

### 2.7 What happens when a model does not fit

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

For a conventional policy, use the owning runtime's normal policy interface. For a stateful planner, world-action model, memory policy or multi-stage agent:

1. map it to an existing upstream lifecycle without losing semantics;
2. extend the natural upstream interface and propose the change there; or
3. carry a versioned bridge extension in crossrun with an explicit reassessment condition and long-term owner.

Failure to fit is evidence about the bridge or upstream interface. It is not a reason to create a second model implementation or hide lifecycle state in arbitrary observation fields.

---

## 3. Ownership

### 3.1 LeRobot

LeRobot is the preferred owner for LeRobot-native policies, processor pipelines, standard Gym/EnvHub evaluation, `Robot` drivers, hardware rollout and LeRobotDataset interchange.

**Upstream check, 2026-07-28.** LeRobot `main` was `95211b98f1cd6b638bda84a8d28f9e41323229dd`. It already contains `lerobot-eval`, original MuJoCo LIBERO, gym-aloha, a native π0.5 implementation and LIBERO checkpoint, EnvHub/Arena integration, async inference, episodic and intervention-aware hardware rollout, and Unitree G1 simulation and real-robot support for 23/29 DoF variants. crossrun reuses these as native baselines and avoids recreating them unless a named reference experiment is testing a materially different architecture.

The first crossrun integration should be an out-of-tree XPolicyLab remote-policy plugin for LeRobot. A checkpoint from another runtime is not assumed equivalent to a LeRobot port of the same model family; original-runtime and converted runs retain separate lineage.

### 3.2 XPolicyLab

XPolicyLab owns heterogeneous original-runtime model adapters, dependency isolation and policy serving. Service hardening, adapter fixes and generally useful lifecycle extensions belong upstream even when the crossrun fork carries them first. crossrun owns only the exact consumption profile, bundle pin and cross-project fixture.

### 3.3 Environment and benchmark owners

- RoboDojo owns its 42 Isaac tasks, `EvalEnv`, result layout and leaderboard protocol.
- Isaac Lab-Arena and LW-BenchHub own Arena composition and adapted Lightwheel-LIBERO tasks.
- OpenPI owns original OpenPI runtime behaviour and checkpoint configuration.
- Genie Sim and AgiBot-owned packages are the natural homes for redistributable G2 simulation assets.

crossrun may wrap these projects, normalize their output, or implement a new composition of their public assets and semantics as a clearly named reference integration. It does not silently copy upstream task implementations, relabel a port as the original benchmark, or use an MIT repository to obscure upstream license restrictions.

### 3.4 crossrun

crossrun owns integration architecture that spans upstream boundaries and needs a runnable home before any one project can reasonably accept it. This includes bundle materialization, cross-project conformance, evidence normalization, and named reference integrations such as the Arena-based RoboDojo experiment.

That ownership is legitimate when the implementation is independently useful, tested against native baselines, and maintained through public upstream boundaries. It does not make the integration an official RoboDojo or Arena path, and it does not prevent later transfer upstream. A rejected upstream proposal may remain here; an unmaintainable or disproven reference path may not.

---

## 4. Design claims to test

Each claim must be visible in code and paired with evidence.

1. **One bundle can materialize a known-good multi-repo stack.** Every source and artifact is pinned and reconstructable.
2. **Native baselines and reference integrations can coexist.** The former preserves upstream behaviour; the latter tests a named architectural improvement.
3. **Checkpoint compatibility is metadata, not a model-name guess.** A policy and environment run only after a profile check.
4. **Local code has an explicit lifecycle.** Every delta is a temporary fix, upstream candidate or crossrun reference with an owner, fixture and reassessment condition.
5. **Models and copied upstream task implementations do not live in this repo.** crossrun may own new integration code, but not unattributed mirrors.
6. **Numbers ship with uncertainty and provenance.** A bare success rate is incomplete.
7. **Perturbation evaluation is part of the baseline.** A standard score alone does not establish robustness.
8. **Cross-environment results are stratified, not pooled.** Report environment-specific outcomes and analyse interactions.
9. **Hardware is an extension boundary, not current scope.** Shared profiles must not block a future real loop, but no real contract is invented before sim evidence exists.
10. **A local episode loop is hypothesis-driven.** It may fill a concrete gap or test a better execution architecture, but must be compared with a native baseline.
11. **Best practice is measured, not declared.** Adoption depends on capability, semantic fidelity, performance, interoperability and maintenance evidence.

---

## 5. Initial runnable paths

Phase 0 gets one complete result first: π0.5 on original MuJoCo LIBERO through the crossrun sim loop. ALOHA Sim follows as the second MuJoCo fixture to exercise different embodiment semantics without blocking the first result.

| Path | Representative upstream config | Observation/action shape | Primary purpose |
|---|---|---|---|
| **LIBERO / Panda** | OpenPI `pi05_libero` or another public LIBERO checkpoint | single-arm state, third-person + wrist images, 7-D relative EEF/gripper action | benchmark reproduction, statistics, perturbations |
| **ALOHA Sim** | OpenPI `pi05_aloha` with `pi05_base`, or a validated π0.5 task-tuned checkpoint | 14-D dual-arm state, ALOHA cameras, 14-D absolute joint/gripper action | embodiment variation, 50 Hz control, sim-to-real-shaped lifecycle |

The two paths do **not** share a checkpoint or policy profile. They share:

- pinned upstream and overlay revisions;
- bundle materialization and launch conventions;
- compatibility preflight;
- evaluation and provenance infrastructure.

They differ in:

- model/checkpoint;
- observation keys and cameras;
- state and action dimensions;
- action semantics;
- normalisation statistics;
- control frequency and chunk execution.

This distinction is essential: “same bundle” does not mean “same runtime or checkpoint works on every robot.”

The raw policy inputs are intentionally different. OpenPI's LIBERO transform consumes an 8-D state, one third-person image and one wrist image, then returns a 7-D relative end-effector/gripper action. Its ALOHA transform consumes a 14-D joint/gripper state, requires `cam_high`, optionally consumes left and right wrist images, and returns a 14-D absolute joint/gripper action. Both are padded and mapped to the model's internal 32-D state/action width and three image slots only after the embodiment-specific transform.

`pi05_aloha` with `pi05_base` is an upstream-supported zero-shot ALOHA candidate, not a published ALOHA Sim task-tuned baseline. Phase 0 must select a task/checkpoint pair through a smoke test before treating success rate as meaningful. If zero-shot performance is not usable, the path requires a public or locally trained π0.5 ALOHA checkpoint; it does not fall back to π0 merely to reuse `pi0_aloha_sim`.

### 5.1 Isaac ports are separate environments

An Isaac robot asset or a similarly named task does not reproduce an original MuJoCo benchmark. A port changes physics, rendering, assets, reset distributions, cameras, controllers and often success predicates. crossrun therefore gives every port its own environment and task revision and never pools its result with the original benchmark.

**Upstream check, 2026-07-28.** Isaac Lab-Arena `main` at `99b557d0d7473d7fc81c0d64308e744899f68ad8` provides Franka and G1 embodiments and links to Lightwheel's LW-BenchHub. LW-BenchHub `main` at `b2bcb2d00edef691f9fcc49039cbf0bcc7464605` advertises 130 **adapted** Lightwheel-LIBERO tasks on Isaac Lab-Arena, including datasets for G1 and other robots. This is a useful second-environment candidate, but it is not the original LIBERO environment or a drop-in replacement for the `pi05_libero` baseline.

RoboDojo `main` at `e0703b03bb1af6075400e9d60dc17a792793960c` runs custom RoboDojo environment managers, task YAML and success logic directly on Isaac Sim 5.1 and a pinned Isaac Lab 2.3 fork. Although its README acknowledges Isaac Lab-Arena as an upstream project, the checked execution path neither imports `isaaclab_arena` nor includes it as a submodule; RoboDojo is not an Arena benchmark package at this revision. It includes a Franka robot and 42 simulated tasks, but no LIBERO benchmark implementation and no ALOHA robot or environment. XPolicyLab supplies policy-side adapters and a client-server boundary; it does not turn those RoboDojo tasks into LIBERO or ALOHA. No current XPolicyLab/RoboDojo path supports treating ALOHA Sim as an Isaac environment. Such a path would require an ALOHA articulation/USD, cameras, controller and action mapping, task/reset definitions, success predicates, and a separately validated checkpoint profile.

### 5.2 Baseline snapshot

The table below is a dated survey snapshot, not a compatibility guarantee.

| Ecosystem | Relevant path | crossrun treatment |
|---|---|---|
| OpenPI | LIBERO, ALOHA Sim, ALOHA real, DROID | use a native supported path first; use XPolicyLab when original-runtime serving is required |
| OpenVLA | LIBERO and other embodiment-specific evaluation paths | wrap original runtime; do not infer ALOHA support from model family alone |
| OpenVLA-OFT | LIBERO plus ALOHA training/evaluation flows | use original runtime adapter unless a validated native package is selected |
| LeRobot | native policies, LIBERO env, ALOHA env, robot drivers | native execution owner, remote-policy plugin, hardware and dataset interchange |
| XPolicyLab | policy zoo and adapter lifecycle | pinned optional service for heterogeneous original runtimes |
| GR00T / WBC / SONIC | embodiment-specific policy and controller stacks | separate policy/runtime, decoder and environment/controller validation |

**Policy-catalog check, 2026-07-28.** XPolicyLab `main` at `5071d8ff557f8f258e50aec5b46a701772bc3295` already contains adapters for OpenVLA-OFT, SmolVLA, GR00T N1.7, DreamZero, AHA-WAM, FastWAM, GigaWorldPolicy, Mem-0, X-WAM and other policy families. Catalog presence proves neither checkpoint compatibility nor a published baseline. A selected bundle consumes an existing adapter first, runs crossrun conformance and baseline checks, and patches only a concrete blocking gap. Model-catalog expansion is not a roadmap phase.

---

## 6. Architecture and contracts

```text
Bundle manifest
  ├── exact upstream revisions and artifact digests
  ├── policy/checkpoint profile
  ├── task/embodiment profile
  └── backend configuration
                  │
                  ▼
Materialize + compatibility preflight
                  │
       ┌──────────┴──────────┐
       ▼                     ▼
Policy endpoint        crossrun sim-eval loop
native or optional     reset · predict · schedule
XPolicyLab service     step · terminate · record
       └──────────┬──────────┘
                  │ Gymnasium-compatible SimBackend
        ┌─────────┼─────────┐
        ▼         ▼         ▼
     MuJoCo   Isaac/Arena  Genesis
        └─────────┬─────────┘
                  ▼
native artifact + evidence envelope ──► reports
```

The crossrun loop owns simulation episodes. The policy runtime and physics engine remain replaceable and visible in every result. Native upstream evaluation remains a separately launched baseline, not another hidden implementation of this loop. crossrun does not pretend that backend ports have identical reset, control or success semantics; it preserves engine-native output alongside the evidence envelope.

### 6.1 Required integration artifacts

```yaml
source_lock:
  repository: required
  upstream_revision: required
  fork_revision: required_when_forked

overlay:
  kind: plugin | patch | fork | reference_integration
  source: required
  digest: required
  owner: required
  lifecycle_intent: temporary_hotfix | upstream_candidate | crossrun_reference
  upstream_issue_or_pr: required_for_upstream_candidate
  review_after: required
  exit_or_reassessment_condition: required

reference_integration:
  hypothesis: required_when_applicable
  native_baseline_bundle: required_when_applicable
  semantic_conformance_fixture: required_when_applicable
  comparison_report: required_when_applicable

launch_recipe:
  path_class: native_baseline | crossrun_sim_eval
  execution_owner: lerobot | robodojo | crossrun | other
  entry_point: required
  dependency_lock_digest: required
  arguments_and_environment: recorded

compatibility_profile:
  policy: required
  environment: required
  explicit_adapters: []

result_adapter:
  upstream_output: preserved
  evidence_schema_version: required
```

`SourceLock` makes upstream identity reproducible. `Overlay` gives every local delta an accountable lifecycle. `ReferenceIntegration` binds an architectural hypothesis to its native baseline and comparison evidence. `LaunchRecipe` distinguishes the crossrun sim loop from an upstream baseline. `CompatibilityProfile` rejects mismatched policies, tasks and backends before launch. `ResultAdapter` adds crossrun evidence without discarding the source runtime's richer output.

A local execution loop may fill a measured upstream gap or test a named reference architecture. It must have conformance fixtures and a native comparison path. Its reassessment gate may lead to upstreaming, continued crossrun ownership, a dedicated fork or deletion.

### 6.2 Policy profile

Every runnable checkpoint has a machine-readable profile:

```yaml
policy_id: openpi/pi05_aloha_zero_shot_candidate
model_family: pi05
provider_runtime: openpi
service_adapter: crossrun-xpolicylab/Pi_05

checkpoint:
  uri: gs://openpi-assets/checkpoints/pi05_base
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
  predicted_chunk_horizon: 50

execution:
  control_hz: 50
  open_loop_steps: 10
  stateful: false
  batching: false

normalisation:
  source: checkpoint
  digest: required
```

The profile above is a candidate grounded in the current OpenPI defaults. The task, prompt, checkpoint suitability, digests and open-loop execution length remain unvalidated until the Phase-0 smoke test. The schema requirement is not provisional.

### 6.3 Environment capabilities

Each environment declares matching properties:

```yaml
environment_id: aloha_sim/gym_aloha
backend: mujoco
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

### 6.5 Evidence records

`Action` records at least:

- semantic type: joint, end-effector, velocity, torque, latent token or another declared type;
- dimensions and chunk horizon;
- frame and units where applicable;
- intended control frequency or timestamps;
- validity limits;
- decoder identity when actions are latent.

The normalised step evidence records at least:

```text
observation
environment timestamp and wall-clock timestamp
terminated / truncated
success observation or success-label input
failure reason
safety events
intervention state
environment-specific diagnostics
```

---

## 7. Repository shape and dependency rules

```text
crossrun/
├── upstreams.lock.yaml        # exact upstream, fork and artifact identities
├── bundles/                   # runnable integration manifests
├── eval/                      # bounded simulation episode loop
├── sim/
│   ├── interface/             # Gymnasium contract and capabilities
│   └── backends/              # MuJoCo, Isaac/Arena and Genesis adapters
├── integrations/              # crossrun-owned best-practice reference paths
├── plugins/                   # installable extensions at upstream boundaries
├── patches/                   # small, reviewable and replayable deltas
├── profiles/                  # policy, environment and service capabilities
├── launch/                    # native entry-point recipes and supervision
├── results/                   # adapters, evidence schemas and reports
├── containers/                # reproducible environment templates
├── conformance/               # materialization, mismatch and regression fixtures
├── assets/                    # source manifests and generation scripts
└── docs/
```

**Hard constraints:**

- Every production bundle pins exact source and artifact revisions; none tracks a moving branch.
- Every local delta is an installable integration or plugin, replayable patch, or named fork commit; manual source edits are invalid.
- Every reference integration names its native baseline, hypothesis, semantic limits, comparison dimensions and reassessment date.
- Episode ownership stays explicit. Portable sim bundles use `eval/`; native-baseline bundles use their upstream owner.
- `eval/` depends on the common sim interface, never directly on a physics engine or simulator package.
- Backend adapters return Gymnasium-compatible environments and preserve native diagnostics under namespaced evidence.
- Model-specific transforms stay with the policy runtime or a policy plugin; environment-specific transforms stay with the environment integration.
- Result adapters preserve upstream-native records and add a versioned evidence envelope; they do not reduce output to a lossy `(success, trajectory)` tuple.
- A clean checkout can materialize a bundle without relying on an unrecorded developer workspace.
- No model weights or upstream model implementations are committed to this repository.
- Checkpoint conversion is optional; every conversion records source artifact, converter revision and validation result.
- Large binary assets and datasets live outside git with immutable hashes.
- Policy servers default to loopback. Remote mode is explicit and authenticated.

### 7.1 Deferred hardware boundary

The policy endpoint, observation/action profiles and evidence envelope avoid simulator-only assumptions so a future real-robot loop can reuse them. The repository may reserve configuration namespaces for Unitree G1 and AgiBot G2, but the current milestone does not define a `RealBackend`, robot driver, controller adapter or hardware episode loop. Those contracts are designed only after the simulation gate passes.

### 7.2 Minimum evidence identity

```text
trial_id and run_id
task and task-definition revision
seed and initial-condition identifier
policy, environment and service-profile revisions
crossrun revision and complete run-config digest
all upstream-base, fork and overlay revisions
plugin, patch and result-adapter revisions
upstream model runtime revision and dependency-lock digest
checkpoint digest and conversion lineage
container or process-environment digest
normalisation-statistics digest
environment, execution owner, versions and physics configuration
perturbation tier and parameters
action/control configuration
termination and failure reason
safety events and interventions
success label, source, classifier version, confidence and audit state
trajectory or trajectory digest
```

The immutable run manifest contains the full command, service launch parameters, resolved profiles and configuration files. The evidence envelope references its digest so local source mounts and non-container development runs remain attributable.

---

## 8. Evaluation protocol

### 8.1 Cross-environment comparison

The same nominal task can differ across physics engines because of contact, friction, rendering, control and solver behaviour.

- Never pool results from different environments into one success rate.
- Report environment and version as first-class grouping variables.
- Absolute success rates may appear together when clearly stratified; do not interpret them as equal-difficulty measurements.
- Compare ranking agreement, matched failure modes and policy-by-environment interactions.
- A ranking reversal is evidence of an interaction, not proof that either environment is wrong.
- Preserve pairing when matched seeds or initial conditions exist.

### 8.2 Statistical discipline

- Record the complete trial identity above.
- Report a binomial interval for each policy × task × environment × perturbation condition.
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

### 8.4 Sim adapter acceptance

A backend adapter is accepted only when it:

- passes reset, step, termination, truncation, render and close contract fixtures;
- reports its capabilities and rejects unsupported configuration before launch;
- reproduces a fixed action trace against its native environment within declared tolerances;
- preserves engine-native diagnostics and failure reasons;
- records seed behaviour, control timing and vectorisation semantics;
- has one end-to-end policy rollout through the unchanged crossrun eval loop.

---

## 9. Scope and phases

### Phase 0 — first sim result

- [ ] Define the bundle, policy endpoint, Gymnasium-compatible `SimBackend`, MuJoCo adapter and evidence envelope.
- [ ] Materialize a clean pinned LeRobot/OpenPI/LIBERO environment without manual source edits.
- [ ] Reproduce **original LIBERO/Panda + `pi05_libero`** through the native evaluator as the baseline.
- [ ] Run the same policy/checkpoint/task through the crossrun sim-eval loop and compare fixed-seed traces and termination semantics.
- [ ] Add compatibility preflight, trajectory digests, failure categories and a small success-rate report.
- [ ] Only after the LIBERO slice works, add ALOHA Sim as a second MuJoCo contract fixture with its own π0.5 profile; do not block the first result on ALOHA checkpoint quality.
- [ ] Add XPolicyLab serving only if a selected policy cannot use a native endpoint; do not expand the policy catalog in this phase.

**Exit:** one command or bundle produces a reproducible π0.5 + LIBERO result through crossrun, with the native baseline beside it.

### Phase 1 — backend switching

- [ ] Add an Isaac adapter, preferring Isaac Lab-Arena composition for the crossrun reference path while retaining RoboDojo `EvalEnv` as a native baseline.
- [ ] Add a Genesis adapter. Genesis is the physics engine; AgiBot Genie Sim remains outside the current milestone.
- [ ] Select one narrow manipulation task/embodiment fixture that can be implemented on at least two engines with declared semantic limits.
- [ ] Switch MuJoCo, Isaac and Genesis through bundle configuration without changing the policy endpoint or eval loop.
- [ ] Run adapter contract tests and at least one end-to-end policy rollout on every backend.
- [ ] Preserve separate result groups by engine and publish performance, determinism, vectorisation and semantic-drift evidence.
- [ ] Complete the RoboDojo-native versus Arena-reference comparison on the narrow task slice.
- [ ] Harden the optional XPolicyLab bridge only for failures observed in selected sim bundles.

**Exit:** all three adapters pass the common contract, and the unchanged eval loop has produced evidence on MuJoCo, Isaac and Genesis.

### Phase 2 — release and next-scope decision

- [ ] Add a CI smoke matrix for MuJoCo, Isaac and Genesis with explicit GPU/Isaac availability handling.
- [ ] Pin containers and assets, replay patches, and verify clean bundle materialization.
- [ ] Stabilize result schemas, comparison reports, timeout handling and backend-native diagnostics.
- [ ] Submit generally useful changes upstream after the reference paths are measured; retain valuable cross-project integrations when no upstream owns them.
- [ ] Publish supported task/backend combinations and explicit non-equivalences.
- [ ] Review the available Unitree G1 and AgiBot G2 only after the simulation completion gate; create a separate hardware milestone if approved.

**Exit:** a versioned sim-eval release is reproducible across the three backend adapters, with no real-robot implementation hidden in scope.

### 9.1 Simulation completion and hardware gate

Real-robot work is discussed only after:

- the π0.5 + LIBERO MuJoCo baseline is reproducible from a clean bundle;
- the same eval loop runs unchanged against MuJoCo, Isaac and Genesis adapters;
- every adapter passes lifecycle, capability and native-trace conformance fixtures;
- task ports are labelled as distinct conditions unless semantic equivalence is demonstrated;
- policy/environment mismatches fail during preflight rather than during a rollout;
- evidence records preserve engine versions, native diagnostics, timing and failure reasons;
- the remaining patch and integration burden is reviewed and owned.

Passing this gate authorizes a new hardware design discussion; it does not automatically select G1, G2 or a real-robot interface.

### 9.2 Optional XPolicyLab upgrade gate

Never track XPolicyLab `main` implicitly. If an active sim bundle uses the fork, an upstream sync requires:

1. a consumption-profile diff and patch replay;
2. clean materialization of every active bundle that uses it;
3. end-to-end regression of those sim fixtures through the unchanged eval loop;
4. latency, memory, timeout and failure-recovery comparison;
5. an explicit rollback revision and updated upstream disposition for retained patches.

An adapter is not added merely because it exists in the XPolicyLab catalog. It enters the matrix only when a selected sim-eval path requires it.

---

## 10. Risks, security and licensing

| Risk | Level | Response |
|---|---|---|
| **Multi-upstream release skew** | High | pin a bundle as one unit; upgrade through bundle fixtures rather than component availability |
| **Patch drift or permanent patch burden** | High | replay every patch in CI; upstream general changes; enforce owner, review date and reassessment condition |
| **Fork becomes the default response** | High | require wrapper/plugin/patch evidence before escalation; measure fork delta at every sync |
| **“Best practice” is assumed from branding** | High | require a native baseline and measured capability, fidelity, performance and maintenance comparison |
| **Reference integration silently becomes a second benchmark** | High | name it separately; publish semantic limits; never inherit upstream leaderboard identity without conformance and owner agreement |
| **Crossrun-owned code diverges from upstream internals** | High | integrate at public assets/APIs; pin revisions; move fragile source adaptations to a fork or redesign the boundary |
| **XPolicyLab interface churn** | High | pin exact revisions; maintain consumption profile; regression-gated upgrades |
| **Adapter quality varies by model** | High | conformance tests, known-good checkpoint run and provenance manifest |
| **Remote-policy bridge cannot express a complex policy** | High | preserve native execution where possible; version a narrow extension and propose it upstream |
| **Checkpoint/model-family confusion** | High | immutable policy profiles and compatibility preflight |
| **LeRobot bridge falsely appears universal** | Med-high | declare supported lifecycle/capabilities; require policy-specific validation |
| **Common interface hides backend differences** | High | standardise lifecycle only; require capability profiles, namespaced diagnostics and separate result groups |
| **Adapter conforms syntactically but not behaviourally** | High | compare fixed action traces and native baselines within declared tolerances |
| **Execution-owner semantics differ** | High | record owner and lifecycle; compare only explicitly aligned evidence fields |
| **Result normalization loses upstream detail** | High | preserve native output; make result adapters additive and versioned |
| **Cross-environment overinterpretation** | Med-high | stratified reports and interaction analysis |
| **Asset/license ambiguity** | High | per-file and per-weight manifests; no blanket ecosystem claim |

### 10.1 Security baseline

- no pickle or executable deserialisation across the network boundary;
- schema and size validation for every request and response;
- loopback binding by default;
- authenticated and encrypted remote transport through a private network or tunnel;
- network allow-lists and explicit remote-mode configuration;
- request deadlines, rate limits and bounded queues;
- least-privilege containers or processes;
- health checks and a fail-safe stop path when the policy service is unavailable;
- reject late responses by episode/request identity and restart the isolated service after a non-cancellable timeout.

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
| Build one universal sim-and-real episode framework | hides hardware safety and reset semantics; own only the bounded sim loop now |
| Treat current upstream architecture as the ceiling | prevents cross-project ideas from being implemented and evaluated |
| Propose a large upstream migration before it runs in crossrun | shifts design risk to maintainers and makes rejection likely without evidence |
| Convert every checkpoint to one universal weight format | expensive, lossy and unnecessary for execution unification |
| Require every model to become LeRobot-native | forces reimplementation or conversion before evaluation value is proven |
| Treat XPolicyLab as a permanent public standard | it is one pinned optional service behind a versioned plugin |
| Track XPolicyLab `main` automatically | rapid changes would invalidate reproducibility |
| Vendor upstream repositories into crossrun | obscures ownership, upgrades and licenses; pin external revisions instead |
| Apply untracked manual edits in a materialized workspace | cannot be reproduced, reviewed, replayed or removed |
| Put model-specific observation/action transforms in a generic crossrun layer | duplicates policy ownership and makes each model a crossrun change |
| Infer compatibility from model name or embodiment label | ignores checkpoint, normalisation, action and timing differences |
| Run LIBERO tasks on ALOHA and call it the original LIBERO baseline | embodiment transfer is a different configuration requiring validation |
| Put model code and weights in this repo | creates dependency and licensing conflicts |
| Report one pooled number across environments | conflates condition changes with policy performance |

---

## 12. Open questions

These are measurements that can block the three current phases.

1. What is the smallest task/embodiment fixture that can exercise MuJoCo, Isaac and Genesis without pretending the ports are benchmark-identical?
2. Which `SimCapabilities` fields are required for deterministic reset, vectorisation, rendering and control-timing preflight?
3. What tolerance should fixed-action native-versus-crossrun traces use for each engine?
4. How should action chunks be resampled when policy and environment frequencies differ?
5. Does `pi05_aloha` with `pi05_base` produce a useful second MuJoCo fixture, and if not, which π0.5 task-tuned checkpoint should replace it?
6. Does the Arena-based RoboDojo reference path improve composition and maintenance after accounting for migration effort, performance and semantic drift?
7. Which Genesis public APIs provide the cleanest Gymnasium adapter boundary, and which capabilities require explicit unsupported declarations?
8. Which active sim path actually requires XPolicyLab rather than a native policy endpoint?
9. Which selected assets and weights may be redistributed, and which must be fetched by the user?

---

## Links

| | |
|---|---|
| XPolicyLab | https://github.com/XPolicyLab/XPolicyLab |
| OpenPI | https://github.com/Physical-Intelligence/openpi |
| OpenVLA | https://github.com/openvla/openvla |
| OpenVLA-OFT | https://github.com/moojink/openvla-oft |
| LeRobot | https://github.com/huggingface/lerobot |
| Gymnasium | https://github.com/Farama-Foundation/Gymnasium |
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
