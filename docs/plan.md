# crossrun — Design Document v2.7

> **What this is**: a pinned integration distribution for running and evaluating robot policies in simulation and on hardware.
> **Status**: design phase. The architecture is under validation; code has not started.
> **Companion research**: see [`research/`](research/).

**Version history.** v1.x was scoped as an internal evaluation system. v2.0 reframed it as an open reference design. v2.1 added the execution layer. v2.2 selected XPolicyLab as a policy-side integration candidate. v2.3 corrected over-strong assumptions and added an explicit execution/environment contract. v2.4 chose one default policy runtime boundary and added ALOHA Sim beside LIBERO/Panda. v2.5 defined that boundary as a pinned crossrun-maintained XPolicyLab fork and made Phase 0 π0.5-only. v2.6 aligned policy intake and hardware scope with current upstreams. **v2.7 makes crossrun an integration distribution: upstream-native loops keep episode ownership, wrappers and plugins are preferred over forks, local deltas are governed as temporary patch debt, and crossrun owns the reproducible bundle rather than a mandatory new execution framework.**

---

## 1. Positioning

**In one line:** assemble fast-moving upstream policy, environment and hardware projects into pinned, tested and reproducible bundles without permanently copying their implementations.

The project is not another simulator, benchmark, model zoo, training framework, universal policy protocol or mandatory episode loop. Its short-term value is:

1. exact upstream, checkpoint, container and asset revisions that work together;
2. small wrappers, plugins or fork commits for gaps that block immediate use;
3. explicit compatibility metadata between policies, environments and controllers;
4. one launch and conformance surface across otherwise separate upstream tools;
5. evaluation records that preserve uncertainty, upstream identity and local modifications.

**Deliverables, in priority order:**

1. Reproducible integration bundles that run selected end-to-end paths.
2. Conformance fixtures that make upstream updates and patch drift visible.
3. Generally useful fixes proposed to their natural upstream owners.
4. A design and evidence record that explains any remaining crossrun-only glue.

**Success looks like:** someone can materialize a clean workspace from one bundle manifest, launch a selected policy/environment/hardware path, and see exactly which upstream revisions, overlays, transformations and assumptions produced the result.

---

## 2. Integration strategy

### 2.1 Native execution first

Episode ownership stays with the maintained upstream path that already implements it:

| Path | Default execution owner |
|---|---|
| LeRobot policy + supported simulator | `lerobot-eval` |
| LeRobot policy + supported robot | `lerobot-rollout` and `Robot` |
| RoboDojo benchmark | RoboDojo `EvalEnv` |
| Heterogeneous original-runtime policy | XPolicyLab service, bridged into the selected environment owner |
| Arena/EnvHub task | the Arena or EnvHub environment package plus its supported evaluator |

crossrun adds a new episode loop only after a runnable spike proves that no upstream owner can express the required lifecycle. A local loop is a fallback with an explicit deletion condition, not the architectural default.

### 2.2 Wrapper, plugin, patch, fork

Every integration escalates through the least expensive mechanism that works:

1. **Wrapper:** compose public commands or APIs without modifying upstream code.
2. **Plugin:** use an upstream extension point, preferably out of tree until stable.
3. **Patch:** carry a small temporary source change that can be replayed onto one exact base revision.
4. **Fork:** maintain source-side changes that need their own CI, release artifact or coordinated multi-file evolution.

A fork is not a project achievement. It is accepted maintenance debt. Each local delta records:

```yaml
upstream_repo: required
upstream_base_revision: required
local_revision_or_patch_digest: required
owner: required
rationale: required
affected_bundles: required
regression_fixtures: required
upstream_issue_or_pr: required_or_explained
review_after: required
removal_condition: required
```

No production machine contains an unrecorded manual edit. A clean checkout plus the bundle manifest must reproduce the full source state.

### 2.3 The initial XPolicyLab fork

crossrun maintains a pinned XPolicyLab fork because heterogeneous model dependencies and the existing adapter catalog are immediately useful, while service hardening and Pi adapter gaps currently require source-side changes. The fork:

- regularly syncs a reviewed upstream remote;
- keeps generally useful changes as clean topic commits suitable for upstream PRs;
- isolates crossrun-only bundle metadata from upstream-generic runtime changes;
- never becomes the home for environment loops, simulator code or robot drivers;
- is consumed only at exact upstream-base and fork revisions.

**Upstream check, 2026-07-28.** XPolicyLab `main` was `5071d8ff557f8f258e50aec5b46a701772bc3295`. Its model lifecycle and WebSocket implementation still lack declared capabilities, bounded payloads, end-to-end deadlines and true transport batching. The WebSocket client implements a batch request as sequential `infer` calls. The Pi adapters require all three ALOHA cameras even though OpenPI permits masked wrist cameras, and the vendored config does not include the public `pi05_aloha` or `pi05_libero` profiles. These are initial fork patches and upstream PR candidates, not permanent crossrun features.

### 2.4 Bundle manifest and service profile

The bundle manifest is the top-level reproducibility unit. It pins every upstream and local delta and references narrower runtime profiles:

```yaml
bundle: pi05-mujoco-v1
crossrun_revision: required

upstreams:
  lerobot: {revision: required}
  openpi: {revision: required}
  libero: {revision: required}
  gym_aloha: {revision: required}

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
  - pi05_aloha_smoke
```

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

### 2.5 What happens when a model does not fit

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
3. carry a versioned bridge extension in crossrun with an explicit removal condition.

Failure to fit is evidence about the bridge or upstream interface. It is not a reason to create a second model implementation or hide lifecycle state in arbitrary observation fields.

---

## 3. Upstream ownership

### 3.1 LeRobot

LeRobot is the preferred owner for LeRobot-native policies, processor pipelines, standard Gym/EnvHub evaluation, `Robot` drivers, hardware rollout and LeRobotDataset interchange.

**Upstream check, 2026-07-28.** LeRobot `main` was `95211b98f1cd6b638bda84a8d28f9e41323229dd`. It already contains `lerobot-eval`, original MuJoCo LIBERO, gym-aloha, a native π0.5 implementation and LIBERO checkpoint, EnvHub/Arena integration, async inference, episodic and intervention-aware hardware rollout, and Unitree G1 simulation and real-robot support for 23/29 DoF variants. crossrun must reuse and validate these paths rather than recreate them.

The first crossrun integration should be an out-of-tree XPolicyLab remote-policy plugin for LeRobot. A checkpoint from another runtime is not assumed equivalent to a LeRobot port of the same model family; original-runtime and converted runs retain separate lineage.

### 3.2 XPolicyLab

XPolicyLab owns heterogeneous original-runtime model adapters, dependency isolation and policy serving. Service hardening, adapter fixes and generally useful lifecycle extensions belong upstream even when the crossrun fork carries them first. crossrun owns only the exact consumption profile, bundle pin and cross-project fixture.

### 3.3 Environment and benchmark owners

- RoboDojo owns its 42 Isaac tasks, `EvalEnv`, result layout and leaderboard protocol.
- Isaac Lab-Arena and LW-BenchHub own Arena composition and adapted Lightwheel-LIBERO tasks.
- OpenPI owns original OpenPI runtime behaviour and checkpoint configuration.
- Genie Sim and AgiBot-owned packages are the natural homes for redistributable G2 simulation assets.

crossrun may wrap any of these projects and normalize their output. It does not copy their task implementations or use an MIT repository to obscure upstream license restrictions.

---

## 4. Design claims to test

Each claim must be visible in code and paired with evidence.

1. **One bundle can materialize a known-good multi-repo stack.** Every source and artifact is pinned and reconstructable.
2. **Native execution owners can be composed without copying their episode loops.** Bridges adapt policies and evidence at published boundaries.
3. **Checkpoint compatibility is metadata, not a model-name guess.** A policy and environment run only after a profile check.
4. **Patch debt is explicit and removable.** Every local delta has an owner, fixture, upstream disposition and removal condition.
5. **Models and upstream task implementations do not live in this repo.** crossrun contains overlays and manifests, not copies.
6. **Numbers ship with uncertainty and provenance.** A bare success rate is incomplete.
7. **Perturbation evaluation is part of the baseline.** A standard score alone does not establish robustness.
8. **Cross-environment results are stratified, not pooled.** Report environment-specific outcomes and analyse interactions.
9. **Sim and real may use different execution owners.** Compatibility and evidence remain comparable without pretending their semantics are identical.
10. **A local episode loop is evidence-driven.** It is added only when upstream-native loops cannot support a required path and carries a deletion gate.

---

## 5. Initial runnable paths

Phase 0 uses two upstream-supported MuJoCo environments with π0.5 policies. They exercise different embodiment semantics while sharing one bundle manifest and evidence schema. They do not need to share an episode loop or policy implementation.

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

**Policy-catalog check, 2026-07-28.** XPolicyLab `main` at `5071d8ff557f8f258e50aec5b46a701772bc3295` already contains adapters for OpenVLA-OFT, SmolVLA, GR00T N1.7, DreamZero, AHA-WAM, FastWAM, GigaWorldPolicy, Mem-0, X-WAM and other policy families. Catalog presence proves neither checkpoint compatibility nor a published baseline. Phase 1 consumes representative existing adapters first, runs crossrun conformance and baseline checks, and patches only concrete gaps. Generally useful fixes go upstream; crossrun continues to own manifests, lifecycle compatibility and evaluation evidence.

---

## 6. Architecture and contracts

```text
Bundle manifest
  ├── exact upstream revisions and artifact digests
  ├── overlays: plugins, patches and configuration
  ├── launch recipe and compatibility profile
  └── expected evidence schema
                 │
                 ▼
Materialize + compatibility preflight
                 │
                 ▼
Upstream-native execution owner
  ├── LeRobot eval / rollout / Robot
  ├── RoboDojo EvalEnv
  ├── Arena / EnvHub
  └── other selected upstream entry point
                 │
       optional remote-policy bridge
       to the pinned XPolicyLab fork
                 │
                 ▼
Result adapter ──► evidence envelope ──► reports / datasets
```

The execution owner remains visible in every bundle and result. crossrun does not pretend that distinct upstream loops have identical scheduling, reset, timeout or intervention semantics. It normalises only the evidence needed for reproducibility and comparison, preserving upstream-native output alongside the normalised envelope.

### 6.1 Required integration artifacts

```yaml
source_lock:
  repository: https://github.com/XPolicyLab/XPolicyLab
  upstream_revision: required
  fork_revision: required_when_forked

overlay:
  kind: plugin | patch | fork
  source: required
  digest: required
  owner: required
  upstream_issue_or_pr: required_or_not_applicable
  review_after: required
  removal_condition: required

launch_recipe:
  execution_owner: lerobot | robodojo | arena | other
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

`SourceLock` makes upstream identity reproducible. `Overlay` makes every local delta accountable and removable. `LaunchRecipe` delegates episode ownership to a named upstream entry point. `CompatibilityProfile` rejects mismatched observations, actions and timing before launch. `ResultAdapter` adds crossrun evidence without discarding the source runtime's richer output.

A local execution loop may be introduced only after a bundle demonstrates that no upstream entry point can satisfy a required path. That loop must name the measured gap, have conformance fixtures, and carry a deletion condition tied to an upstream capability or accepted contribution.

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
- Every local delta is an installable plugin, replayable patch or named fork commit; manual source edits are invalid.
- Episode ownership stays with the named upstream entry point unless a measured, documented gap justifies a temporary local loop.
- Model-specific transforms stay with the policy runtime or a policy plugin; environment-specific transforms stay with the environment integration.
- Result adapters preserve upstream-native records and add a versioned evidence envelope; they do not reduce output to a lossy `(success, trajectory)` tuple.
- A clean checkout can materialize a bundle without relying on an unrecorded developer workspace.
- No model weights or upstream model implementations are committed to this repository.
- Checkpoint conversion is optional; every conversion records source artifact, converter revision and validation result.
- Large binary assets and datasets live outside git with immutable hashes.
- Policy servers default to loopback. Remote mode is explicit and authenticated.

### 7.1 Minimum evidence identity

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

### 8.4 Hardware success labels

A learned classifier is a measurement instrument, not ground truth. Hardware reports include:

- classifier and prompt/configuration revision;
- score and abstention behaviour;
- a sampled human-audit set;
- confusion estimates on the evaluated task distribution;
- the policy for disagreements and uncertain outcomes.

---

## 9. Scope and phases

### Phase 0 — first reproducible bundle

- [ ] Define `upstreams.lock.yaml`, the bundle schema, overlay metadata and the evidence envelope.
- [ ] Materialize a clean pinned LeRobot/OpenPI environment without manual source edits.
- [ ] Run **original LIBERO/Panda + `pi05_libero`** through its native LeRobot/OpenPI-supported entry point and reproduce a known-good baseline.
- [ ] Smoke-test **gym-aloha ALOHA Sim + `pi05_aloha`/`pi05_base`** through its native entry point and select a meaningful π0.5 task/checkpoint pair before making a success-rate claim.
- [ ] Keep these as two original MuJoCo paths. Isaac ports are explicitly outside Phase 0 and are never presented as the original benchmarks.
- [ ] Implement compatibility preflight and intentional mismatch fixtures for both paths.
- [ ] Preserve native output and add provenance, trajectory digests and confidence intervals through result adapters.
- [ ] Prove that both paths rematerialize from a clean checkout at their pinned revisions.

### Phase 1 — XPolicyLab fork and remote-policy bridge

- [ ] Create the crossrun XPolicyLab fork, pin upstream-base and fork revisions, and define `crossrun-xpolicylab-v1`.
- [ ] Harden capability discovery, health checks, deadlines, bounded payloads and queues, true batching semantics, and supervised recovery where current upstream behaviour is insufficient.
- [ ] Implement an out-of-tree LeRobot remote-policy plugin so a native LeRobot rollout can consume the XPolicyLab service without moving episode ownership into crossrun.
- [ ] Fix the π adapter's unconditional ALOHA-camera requirements and add public, fixture-backed `pi05_libero` and `pi05_aloha` profiles where the upstream maintainers accept them.
- [ ] Inventory representative existing XPolicyLab adapters and classify each as wiring-only, reproducible baseline, or bundle-conformant.
- [ ] Consume XPolicyLab's existing OpenVLA, SmolVLA, GR00T, world-action and memory-policy work before adding any overlapping integration.
- [ ] Add crossrun code only for bundle lifecycle, interoperability or evidence gaps; submit generally useful adapter/runtime changes upstream.
- [ ] Test crash, timeout, malformed payload, stale observation, stateful reset, cancellation and non-batched policy behaviour.

XPolicyLab's expanding model catalog and crossrun Phase 1 are complementary only under this ownership rule: XPolicyLab owns model adapters and service behaviour; crossrun owns pinned consumption, the LeRobot bridge, compatibility fixtures and evidence. Duplicating a model adapter in crossrun is a conflict and must be removed or proposed upstream.

### Phase 2 — available G1 and G2 hardware

- [ ] Inventory the exact available Unitree G1 and AgiBot G2 configurations: cameras, hands, controllers, emergency-stop path, control frequency and checkpoint compatibility.
- [ ] Validate LeRobot's existing G1 sim and real support against the available 23- or 29-DoF configuration before writing an alternative driver.
- [ ] For G2, first inventory Genie Sim, vendor SDK and public driver/plugin boundaries; add a narrow plugin only for the missing boundary needed by the selected run.
- [ ] Select one minimal static-manipulation path on the better-supported available robot, with manual reset and a named human safety owner.
- [ ] Use the upstream hardware rollout/intervention loop where it meets the path; add only the launch, supervision and evidence overlay required for reproducibility.
- [ ] Record success-label provenance, safety events and interventions from the first hardware run.
- [ ] Keep the first claim to lifecycle, safety and repeatability; do not infer sim-to-real capability from a wiring run.

No other robot is a Phase-2 target unless hardware availability changes and this plan is revised.

### Phase 3 — Isaac integrations and cross-environment evidence

- [ ] Integrate RoboDojo through its published `EvalEnv`, task YAML and manager boundaries; do not describe that path as Isaac Lab-Arena.
- [ ] Evaluate Lightwheel-LIBERO on Isaac Lab-Arena as an explicitly adapted environment; document task, reset, camera, controller and success-predicate differences before selecting a matched study.
- [ ] Keep original LIBERO, Lightwheel-LIBERO and RoboDojo result groups separate while using the common evidence envelope.
- [ ] Validate Arena's G1 path independently from the original MuJoCo and physical G1 paths.
- [ ] Add GR00T-WBC/SONIC only after its encoder, decoder, hands, cameras and low-level controller are independently validated for the selected embodiment.
- [ ] Publish one policy-by-environment interaction study without claiming that adapted tasks are benchmark-identical.

### Phase 4 — stabilize, upstream and reduce patch debt

- [ ] Replay every patch against current pinned upstream candidates and delete patches made obsolete by upstream changes.
- [ ] Submit or track generally useful changes in LeRobot, XPolicyLab, RoboDojo, Arena or their actual owner repository.
- [ ] Turn accepted upstream changes into bundle revision updates and removal of the corresponding overlay.
- [ ] Pin Genie Sim revisions and inventory file-level licenses only if G2 remains the selected hardware path.
- [ ] Publish negative results, rejected integrations, upstream disposition and remaining patch ownership.
- [ ] Demonstrate that the useful surface of crossrun is still bundles, conformance and evidence rather than a duplicate simulator, policy catalog or episode framework.

### 9.1 Phase 0 continuation gate

crossrun is an integration distribution, not a platform that must exist forever. After Phase 0, continue as a standalone repository only while it makes selected multi-upstream paths materially faster and more reproducible than assembling them ad hoc:

- a clean checkout materializes both Phase-0 paths at exact revisions without manual source edits;
- versioned compatibility profiles and provenance catch real policy/environment mismatches that upstream launch scripts do not;
- native upstream execution output is preserved while a common evidence envelope enables reproducible comparison;
- every overlay replays cleanly, has a named owner and fixture, and has an upstream disposition and removal condition;
- total patch debt remains small enough to review during every upstream sync.

If upstreams provide a coherent replacement, crossrun removes the affected plugin or patch and narrows to the remaining bundles, conformance fixtures and documentation. A local episode loop must be deleted when its named upstream gap closes. If most work becomes permanent source-level adaptation of upstream internals, the thin-distribution premise has failed and the affected integration should move to a maintained fork or upstream repository.

### 9.2 XPolicyLab upgrade gate

Never track XPolicyLab `main` implicitly. An upstream sync into the crossrun fork is a separate bundle change with:

1. an old-to-new consumption-profile diff;
2. adapter static checks;
3. patch replay and clean bundle materialization;
4. end-to-end bundle regression on:
   - π0.5 + LIBERO/Panda;
   - π0.5 + ALOHA Sim;
   - one LeRobot-native rollout through the remote-policy plugin;
   - one stateful or world-action policy;
   - one non-batched policy;
5. performance measurements for latency, throughput and memory;
6. native-output and evidence-envelope comparison against the previous pinned service revision;
7. updated upstream PR/issue disposition and removal dates for every retained patch.

During bootstrap, the gate runs every fixture that exists at the current phase and records later-phase fixtures as unavailable, not passed. Each fixture becomes mandatory as soon as its phase lands and can never be silently removed from the gate. A security-only sync may use this scoped gate, but still requires a profile diff, the current fixtures and an explicit rollback revision.

An upstream addition is not automatically enabled merely because it exists in the XPolicyLab catalog. Fork patches are rebased deliberately, and conflicts are resolved as product changes rather than hidden merge maintenance.

---

## 10. Risks, security and licensing

| Risk | Level | Response |
|---|---|---|
| **Multi-upstream release skew** | High | pin a bundle as one unit; upgrade through bundle fixtures rather than component availability |
| **Patch drift or permanent patch burden** | High | replay every patch in CI; upstream general changes; enforce owner, review date and removal condition |
| **Fork becomes the default response** | High | require wrapper/plugin/patch evidence before escalation; measure fork delta at every sync |
| **XPolicyLab interface churn** | High | pin exact revisions; maintain consumption profile; regression-gated upgrades |
| **Adapter quality varies by model** | High | conformance tests, known-good checkpoint run and provenance manifest |
| **Remote-policy bridge cannot express a complex policy** | High | preserve native execution where possible; version a narrow extension and propose it upstream |
| **Checkpoint/model-family confusion** | High | immutable policy profiles and compatibility preflight |
| **LeRobot bridge falsely appears universal** | Med-high | declare supported lifecycle/capabilities; require policy-specific validation |
| **Execution-owner semantics differ** | High | record owner and lifecycle; compare only explicitly aligned evidence fields |
| **Result normalization loses upstream detail** | High | preserve native output; make result adapters additive and versioned |
| **Success-label error on hardware** | High | audit labels and report measurement process |
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
| Build a universal crossrun episode framework first | duplicates maintained upstream loops before a concrete gap is measured |
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

These are measurements to make, not conclusions already reached.

1. Which fork extensions should be proposed upstream immediately, and which remain crossrun-specific?
2. Is the selected XPolicyLab fork revision reliable under concurrent load, true transport batching, timeout and supervised restart?
3. Which stateful policies require lifecycle semantics beyond reset, predict and end-episode?
4. Which XPolicyLab policy lifecycles can the LeRobot remote-policy plugin cover without policy-specific crossrun code?
5. Can converted and original-runtime versions of the same model be shown behaviourally equivalent on fixed observations?
6. Which fields are sufficient for a safe policy/environment compatibility preflight?
7. How should action chunks be resampled when service and environment frequencies differ?
8. Does `pi05_aloha` with `pi05_base` produce a meaningful ALOHA Sim smoke test, and if not, which π0.5 task-tuned checkpoint should Phase 0 use?
9. Which exact G1 or G2 hardware configuration has the best-matched public controller, observation profile and checkpoint for the first minimal real run?
10. Which XPolicyLab adapters have a publicly reproducible known-good result rather than only a wiring check?
11. Can a second environment reproduce a matched task closely enough for an interpretable interaction study?
12. Which upstream assets and weights can legally be redistributed, and which must be fetched by the user?
13. Which Lightwheel-LIBERO tasks are close enough to original LIBERO tasks for a controlled environment-interaction study, after all semantic differences are recorded?

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
