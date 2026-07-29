# crossrun — Design Document v2.10

> **What this is:** a simulation-first integration distribution with one bounded scalar evaluation loop, explicit policy/runtime contracts, and reproducible evidence across separately identified environment stacks.
> **Status:** design phase. The architecture is under validation; implementation has not started.
> **Companion research:** see [`research/`](research/).

**Version history.** v1.x was scoped as an internal evaluation system. v2.0 reframed it as an open reference design. v2.1 added an execution layer. v2.2–v2.6 explored XPolicyLab and policy-runtime ownership. v2.7 made crossrun an integration distribution around pinned upstreams and governed overlays. v2.8 added hypothesis-driven reference integrations. v2.9 reduced scope to portable simulation evaluation. **v2.10 completes both sides of that boundary: it defines `PolicyEndpoint`, separates policy artifact, endpoint, execution, and environment profiles, makes Phase 0 an exact OpenPI `pi05_libero` lineage, supports only scalar Gymnasium environments initially, records policy randomness explicitly, requires process isolation for simulator runtimes with global lifecycle, and treats MuJoCo, Isaac, and Genesis as different environment-stack identities rather than one symmetric backend enum.**

---

## 1. Positioning

**In one line:** materialize a pinned policy/environment bundle, run one scalar simulation episode loop, and preserve enough identity and behavior evidence to distinguish a faithful integration from a merely runnable one.

crossrun is not another simulator, benchmark, model zoo, training framework, universal policy protocol, or sim/real runtime. It deliberately owns a small integration surface:

1. exact source, checkpoint, processor, asset, container, and configuration revisions that work together;
2. one typed policy endpoint contract for native and remote runtimes;
3. one scalar Gymnasium-compatible environment contract;
4. one action-chunk scheduler and episode driver;
5. compatibility preflight and native-versus-crossrun conformance fixtures;
6. additive evidence records that retain upstream-native diagnostics;
7. governed wrappers, plugins, patches, forks, and reference integrations.

**Deliverables, in priority order:**

1. An exact OpenPI π0.5 + original LIBERO/Panda + MuJoCo result through crossrun, with the OpenPI evaluator beside it as the native oracle.
2. Fixed-observation, fixed-randomness, fixed-action, reset, termination, and evidence fixtures for that path.
3. A second scalar MuJoCo fixture using OpenPI `pi0_aloha_sim` and gym-aloha.
4. Separately identified Isaac/Arena and Genesis environment conditions behind the same scalar episode driver.
5. A dedicated vector driver after scalar semantics are stable.
6. Measured best-practice reference integrations and narrowly scoped upstream proposals.

**Success looks like:** a clean workspace can materialize one bundle, launch the declared worker topology, reproduce the native oracle, run the same artifact lineage through crossrun, and explain every material behavioral difference with evidence rather than inference.

---

## 2. Exact baseline identity

### 2.1 Phase-0 oracle

The Phase-0 baseline is one exact lineage:

```text
policy implementation   Physical Intelligence OpenPI at a pinned revision
policy config           pi05_libero
checkpoint              gs://openpi-assets/checkpoints/pi05_libero, content-digested
processor/transforms    the matching OpenPI LIBERO data and model transforms
native evaluator        OpenPI examples/libero
simulation environment  original LIBERO/Panda on MuJoCo at pinned revisions
```

The native OpenPI evaluator is the oracle for preprocessing, initial-state selection, settling steps, prompt construction, action parameterization, replanning, termination, and success semantics. The crossrun path may wrap public APIs and move process boundaries, but it must use the same artifact lineage when it claims behavioral equivalence.

A LeRobot-native π0.5 implementation, converted checkpoint, or additionally fine-tuned `lerobot/pi05_libero_finetuned` artifact is valuable but is a **separate baseline**. It receives a separate bundle ID, checkpoint lineage, processor identity, and comparison report. “π0.5” is not sufficient identity.

### 2.2 Second scalar fixture

The second fixture is:

```text
policy implementation   OpenPI
policy config           pi0_aloha_sim
checkpoint              gs://openpi-assets/checkpoints/pi0_aloha_sim
simulation environment  gym-aloha AlohaTransferCube or another pinned supported task
```

This path is selected because it is an upstream-supported ALOHA Sim task/checkpoint pair. It exercises 14-D dual-arm state/action semantics, required `cam_high`, optional wrist cameras, 50 Hz control, and shorter executed prefixes from a longer predicted action chunk. A π0.5 ALOHA zero-shot or task-tuned experiment may be added later under a separate bundle; it is not used as the correctness fixture until validated.

### 2.3 Native baseline and crossrun path

Every bundle distinguishes:

```yaml
native_oracle:
  execution_owner: openpi | lerobot | robodojo | arena | other
  entry_point: required
  artifact_lineage: required

crossrun_path:
  policy_endpoint: required
  simulator_worker: required
  scalar_driver: crossrun
  artifact_lineage: must_match_when_equivalence_is_claimed
```

A result comparison is invalid when the native and crossrun paths silently differ in checkpoint, processor, task revision, action scheduling, initial states, or success predicate.

---

## 3. Runtime topology

### 3.1 Coordinator and workers

The default topology is process-isolated:

```text
crossrun coordinator
  ├── materializes bundle and verifies digests
  ├── starts policy worker
  ├── starts simulator worker
  ├── supervises health, timeouts, and artifacts
  └── assembles the immutable run manifest

policy worker
  └── original model runtime + PolicyEndpoint implementation

simulator worker
  └── crossrun scalar episode driver + SimBackend + environment
```

This is not merely an optimization. It prevents simulator-global state from contaminating policy execution:

- Isaac Lab modules may require application startup before import and select among PhysX/Newton/OvPhysX runtime paths.
- Genesis has process-global initialization and can modify PyTorch default device and dtype.
- Policy runtimes may require incompatible Python, CUDA, JAX, PyTorch, or Transformers environments.

“Switch backend” therefore means materializing and launching another pinned environment topology. It does not mean hot-switching MuJoCo, Isaac, and Genesis inside one long-lived Python interpreter.

### 3.2 Transport

The endpoint may be in-process for unit tests, local IPC, or loopback network transport. Production bundle profiles declare:

```yaml
transport:
  kind: in_process | unix_socket | loopback_tcp | loopback_websocket | private_remote
  authentication: required_when_remote
  encryption: required_when_remote
  max_request_bytes: required
  max_response_bytes: required
  queue_capacity: required
  timeout_behavior: required
```

Transport choice is not policy identity. The same policy artifact may be exposed by several endpoint implementations, each with its own conformance report.

---

## 4. Policy-side contract

### 4.1 Required interface

```python
class PolicyEndpoint(Protocol):
    def capabilities(self) -> PolicyCapabilities: ...

    def reset(self, context: EpisodeContext) -> ResetReceipt: ...

    def predict(self, request: PolicyRequest) -> PolicyResponse: ...

    def close(self) -> None: ...
```

`predict` returns a **complete predicted action chunk**. The crossrun execution profile decides how much of that chunk is applied before replanning. Endpoint implementations must not hide their own unrelated open-loop schedule behind repeated `select_action()` calls.

```python
@dataclass(frozen=True)
class EpisodeContext:
    run_id: str
    trial_id: str
    task_id: str
    policy_seed: int | None
    initial_sequence: int = 0

@dataclass(frozen=True)
class PolicyRequest:
    run_id: str
    trial_id: str
    request_id: str
    sequence_number: int
    observation: Observation
    prompt: str | None
    inference_noise_seed: int | None
    timeout_s: float | None

@dataclass(frozen=True)
class PolicyResponse:
    run_id: str
    trial_id: str
    request_id: str
    sequence_number: int
    action_chunk: ActionChunk
    inference_time_s: float
    runtime_metadata: Mapping[str, object]
```

Responses with the wrong trial, request, or sequence identity are rejected as stale. A late response never advances a terminated episode.

### 4.2 Capability model

A single `stateful: true|false` flag is insufficient. Capabilities declare independent state scopes:

```yaml
state:
  episode_memory: false
  inference_rng: true
  endpoint_action_cache: false
  planner_state: false
  reset_required: true
  reset_scope: [declared_cache]
  concurrency_scope: one_session_per_worker

batching:
  model_native: true
  endpoint_native: true
  transport_native: false
  cross_session: false

cancellation:
  queued_request: true
  running_inference: false
  state_rollback: false
```

OpenPI π policies are not treated as stateless merely because they lack task memory: inference noise and runtime RNG progression affect output. LeRobot policies may also hold action queues. These states must be reflected in the endpoint profile and tests.

### 4.3 Randomness

The run manifest records independent randomness sources:

```yaml
seeds:
  environment_seed: required
  initial_state_id: required_when_available
  policy_seed: required_when_policy_is_stochastic
  inference_noise_seed: required_when_policy_is_stochastic
  perturbation_seed: required_when_perturbed
  scheduler_seed: required_when_nondeterministic
```

For conformance, the preferred rule is:

```text
inference_noise_seed = PRF(run_seed, trial_id, sequence_number)
```

When a runtime accepts explicit noise, native and crossrun action comparisons use the same noise tensor. When it does not, the limitation is declared and the fixture controls process startup, request order, seed, and concurrency rather than claiming strict action equality.

### 4.4 Retry and idempotency

State-changing requests are never retried merely because the transport disconnected. An endpoint profile declares:

```yaml
idempotency:
  key: [trial_id, sequence_number]
  duplicate_behavior: return_cached_response | reject

retry:
  reset: disabled_unless_idempotent
  predict: disabled_unless_idempotent
```

A timeout does not imply server-side cancellation. If running inference cannot be cancelled or rolled back, crossrun rejects the late response and restarts the isolated policy worker before continuing with another trial.

---

## 5. Simulation-side contract

### 5.1 Scalar interface first

Phase 0 supports exactly one scalar Gymnasium environment:

```python
class SimBackend(Protocol):
    def capabilities(self, config: SimConfig) -> SimCapabilities: ...
    def make(self, config: SimConfig) -> gym.Env: ...
```

The environment provides:

```text
observation_space and action_space
reset(seed, options) -> observation, info
step(action) -> observation, reward, terminated, truncated, info
render() and close()
```

The scalar driver owns episode reset, policy reset, observation capture, action-chunk execution, environment stepping, termination, truncation, success extraction, and evidence emission.

### 5.2 Vector environments are a separate driver

`gym.Env | gym.vector.VectorEnv` is not one behavioral contract. Vector environments introduce batched spaces, slot identity, partial reset, per-slot termination, terminal-observation placement, and multiple autoreset modes.

A later `VectorEpisodeDriver` must declare at least:

```yaml
vector:
  autoreset_mode: disabled
  explicit_reset_mask: true
  stable_slot_identity: true
  terminal_observation_source: required
  per_slot_policy_state: required
  seed_expansion: recorded
```

The first vector implementation uses disabled autoreset and explicit reset masks. It may add `NEXT_STEP` or `SAME_STEP` only after fixtures prove that terminal observations, success labels, episode lengths, and trial IDs remain correct.

### 5.3 Environment capabilities

```yaml
capabilities:
  scalar: true
  vector: false
  deterministic_seed_level: exact | distribution_only | unsupported
  state_snapshot: false
  headless_rendering: true
  privileged_success: true
  control_decimation: 1
  native_diagnostics: true
```

Unsupported configurations fail during preflight rather than during rollout.

---

## 6. Environment identity and backend taxonomy

MuJoCo, Isaac, and Genesis are not symmetric values of one `backend` enum. Evidence separates the environment package from the selected physics and rendering implementations:

```yaml
environment:
  environment_id: required
  environment_framework: required
  task_package: required
  task_revision: required
  embodiment: required

simulation_stack:
  physics_backend: required
  renderer_backend: required
  framework_revision: required
  physics_revision: required
  renderer_revision: required
  asset_manifest_digest: required
  controller_id: required_when_applicable
```

Examples:

```yaml
# Original LIBERO
environment_framework: libero
physics_backend: mujoco
renderer_backend: mujoco_opengl

# Arena condition
environment_framework: isaaclab_arena
task_package: lw_benchhub
physics_backend: physx | newton_mjwarp | newton_kamino | ovphysx
renderer_backend: isaacsim_rtx_renderer | newton_renderer | ovrtx_renderer

# Genesis condition
environment_framework: crossrun_genesis_fixture
physics_backend: genesis_rigid
renderer_backend: pyrender | luisa | nyx
```

The common interface standardizes lifecycle only. Task ports remain separate environment conditions unless reset distribution, observations, actions, controller, assets, timing, and success semantics pass a declared equivalence fixture.

---

## 7. Profiles and bundle manifest

### 7.1 Separate profiles

A runnable bundle references four independent profiles.

#### Policy artifact profile

```yaml
policy_artifact:
  policy_id: openpi/pi05_libero
  model_family: pi05
  provider_runtime: openpi
  provider_revision: required
  checkpoint_uri: gs://openpi-assets/checkpoints/pi05_libero
  checkpoint_digest: required
  processor_revision: required
  normalisation_digest: required
  observation_schema_digest: required
  action_schema_digest: required
```

#### Endpoint profile

```yaml
endpoint:
  implementation: crossrun_openpi_endpoint
  implementation_revision: required
  process_environment_digest: required
  transport: loopback_websocket
  state_capabilities: required
  batching_capabilities: required
  cancellation_capabilities: required
  idempotency: required
```

#### Execution profile

```yaml
execution:
  predicted_chunk_horizon_steps: required
  executed_prefix_steps: required
  replan_interval_steps: required
  control_period_ns: required
  scheduling: open_loop_prefix | receding_horizon | temporal_ensemble
  interpolation: none | declared
  clipping: none | declared
  stale_action_policy: stop | hold_last | declared
```

#### Environment profile

```yaml
environment_profile:
  environment_id: required
  observation_space_digest: required
  action_space_digest: required
  reset_distribution_revision: required
  success_semantics_revision: required
  simulation_stack: required
```

### 7.2 Bundle

```yaml
bundle: openpi-pi05-libero-mujoco-v1
crossrun_revision: required

source_locks:
  openpi: {revision: required}
  libero: {revision: required}
  mujoco: {revision: required}

profiles:
  policy_artifact: profiles/policy/openpi-pi05-libero.yaml
  endpoint: profiles/endpoint/openpi-local-v1.yaml
  execution: profiles/execution/pi05-libero-v1.yaml
  environment: profiles/environment/libero-mujoco-v1.yaml

runtime_topology:
  coordinator: crossrun
  policy_worker: isolated
  simulator_worker: isolated

artifacts:
  checkpoints: [{uri: required, digest: required}]
  containers: [{name: required, digest: required}]
  assets: [{manifest: required, digest: required}]

overlays:
  plugins: []
  patches: []

fixtures:
  - materialization
  - fixed_observation_fixed_noise
  - fixed_action_trace
  - reset_and_termination
  - evidence_schema
```

A bundle never tracks a moving branch. A clean checkout plus the manifest reproduces the full source and artifact state without manual edits.

---

## 8. Action execution and timing

The model-output horizon and the environment execution horizon are different quantities. Every result records:

```text
model_output_horizon_steps
executed_prefix_steps
replan_interval_steps
control_period_ns
actual_action_apply_timestamps
interpolation and clipping
controller or decoder identity
chunk index of each applied action
```

A policy may predict 50 steps while the driver executes 10 before replanning. A LIBERO policy may predict 10 while the native oracle replans after a shorter prefix. Neither case is represented by a single ambiguous `chunk_horizon` field.

Cross-process requests use relative time budgets. Absolute monotonic timestamps from different machines are not compared without a declared synchronization method and uncertainty. Step evidence records:

```text
observation acquired
request queued
request sent
inference started and finished, when available
response received
scheduled action time
actual action applied
observation age
queue delay
network round-trip time
deadline miss
```

---

## 9. Optional XPolicyLab bridge

XPolicyLab remains the preferred upstream owner for heterogeneous original-runtime adapters and policy serving. crossrun pins an optional fork only when an active bundle requires it.

The consumed profile includes upstream lifecycle and wire messages, plus required service extensions:

```yaml
profile: crossrun-xpolicylab-v1

upstream_lifecycle:
  required: [reset, update_obs, get_action]
  optional: [update_obs_batch, get_action_batch, prepare_case, on_trial_end]

service_extensions:
  - health, readiness, and version reporting
  - loaded policy and checkpoint attestation
  - declared state, batching, cancellation, and concurrency capabilities
  - per-trial session isolation or one-session-per-worker enforcement
  - strict sequence ordering
  - idempotency keys and duplicate handling
  - relative request deadlines
  - bounded payloads and queues
  - schema validation and explicit error categories
  - no automatic infer retry unless idempotency is guaranteed
  - true transport batching before any batching claim
```

The upstream state checked in July 2026 used one model instance and a global model lock, accepted unbounded WebSocket frames, created tasks without an application queue bound, and implemented client “batching” as sequential single-item inference. Those behaviors are not silently rebranded as crossrun capabilities.

The first remote-policy integration, **when required by a selected bundle**, should use an out-of-tree plugin at the natural environment boundary. It is not the first crossrun integration and does not block Phase 0.

---

## 10. Evidence model

### 10.1 Immutable run identity

```text
run_id and trial_id
crossrun revision and complete resolved-config digest
bundle ID and manifest digest
all upstream, fork, plugin, patch, and adapter revisions
policy artifact, endpoint, execution, and environment profile digests
checkpoint and conversion lineage
processor and normalisation digests
container or process-environment digests
environment framework, task package, physics backend, renderer, controller, and asset digests
separate environment, initial-state, policy, inference-noise, perturbation, and scheduler seeds
native execution owner and oracle artifact
```

### 10.2 Per-step evidence

```text
episode_step and policy sequence_number
policy request_id
observation digest and selected raw observation artifact
observation simulation and monotonic timestamps
requested action chunk
applied action
chunk index
controller, decoder, interpolation, and clipping transforms
reward
terminated and truncated
success observation or label input
final observation
failure reason
policy queue, inference, and transport timing
environment step timing
safety and intervention events, when present
namespaced native info and native artifact digest
```

Requested and applied actions are distinct fields. Terminal observations are preserved even when an upstream vector wrapper would otherwise replace them with an autoreset observation.

Result adapters are additive: they preserve upstream output and attach a versioned evidence envelope rather than reducing everything to `(success, trajectory)`.

---

## 11. Conformance

### 11.1 Policy endpoint acceptance

An endpoint is accepted only when it:

- reports artifact and implementation attestation;
- validates observation and action schemas;
- proves declared reset scope;
- reproduces fixed-observation outputs under controlled randomness within tolerance;
- rejects stale trial/request/sequence identities;
- demonstrates timeout and late-response behavior;
- demonstrates duplicate/idempotency behavior;
- does not claim native batching when it loops over scalar requests;
- preserves model-native metadata and timing.

### 11.2 Scalar environment acceptance

A scalar adapter is accepted only when it:

- passes reset, step, reward, termination, truncation, render, and close fixtures;
- reports capabilities and rejects unsupported configuration before launch;
- reproduces a fixed action trace against its native environment within declared tolerances;
- preserves final observations, success inputs, diagnostics, and failure reasons;
- records reset seed behavior, control timing, physics, renderer, and task identity;
- completes one end-to-end policy rollout through the unchanged scalar driver.

### 11.3 Native-versus-crossrun comparison

The Phase-0 gate compares in increasing scope:

1. materialized files and artifact digests;
2. processor output on fixed raw observations;
3. policy action chunks on fixed processed observations and controlled noise;
4. environment transitions on a fixed action trace;
5. reset, settling, termination, and success semantics;
6. complete fixed-seed episodes;
7. aggregate success-rate evidence.

A mismatch is classified, not averaged away.

---

## 12. Evaluation protocol

### 12.1 Environment stratification

- Never report an unstratified pooled success rate across environment conditions as the primary result.
- Report environment framework, task revision, physics backend, renderer, assets, and controller as first-class grouping variables.
- Absolute success rates may appear together when clearly stratified; they are not equal-difficulty measurements by default.
- Compare ranking agreement, matched failure modes, and policy-by-environment interactions.
- Preserve pairing when matched initial states or perturbations exist.
- A ranking reversal is evidence of an interaction, not proof that one engine is wrong.

### 12.2 Statistical discipline

- Report task-level counts and intervals, not only a suite aggregate.
- Use a named binomial interval for each policy × task × environment × perturbation condition.
- Clopper–Pearson is a conservative default; alternatives are permitted when named.
- Do not infer equality from overlapping confidence intervals.
- Predeclare primary comparisons and correct multiplicity where appropriate.
- Use paired tests or hierarchical models for shared initial states or clustered/vectorized execution.
- Separate rollout-sampling uncertainty from success-label uncertainty.
- Record fixed-n or sequential stopping rules before execution.

**Power reference:** 63 successes in 70 Bernoulli trials gives an exact 95% interval about 15 percentage points wide. A roughly ±2-point interval near 90% success requires on the order of one thousand independent trials; the exact requirement depends on interval, clustering, and stopping rule.

### 12.3 Perturbations

Standard and perturbed results are reported together. Perturbations record generation seed and parameters and distinguish visual appearance, lighting/rendering, object pose, observation degradation, and safe action latency/control disturbance. A collapse under one family is evidence about that family, not a universal statement of incapability.

---

## 13. Reference integrations and overlay governance

A best-practice reference integration must declare:

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

Compatibility work escalates through the smallest mechanism that works:

1. wrapper;
2. plugin;
3. replayable patch;
4. maintained fork;
5. named crossrun reference integration when the cross-project architecture itself is the hypothesis.

Every local delta records its upstream base, digest, owner, rationale, affected bundles, regression fixtures, lifecycle intent, review date, upstream issue/PR where applicable, and exit condition. No production workspace contains an unrecorded manual edit.

RoboDojo's native environment managers remain the oracle for RoboDojo. An Arena-based composition is a separately named reference condition until task intent, assets, reset, observations, actions, controller, timing, and success semantics pass the declared fixture.

---

## 14. Repository shape

```text
crossrun/
├── upstreams.lock.yaml
├── bundles/
├── coordinator/
├── policy/
│   ├── interface/
│   └── endpoints/
├── eval/
│   ├── scalar/
│   └── vector/                 # later, separate semantics
├── sim/
│   ├── interface/
│   └── backends/
├── profiles/
│   ├── policy-artifact/
│   ├── endpoint/
│   ├── execution/
│   └── environment/
├── integrations/
├── plugins/
├── patches/
├── launch/
├── results/
├── conformance/
├── containers/
├── assets/
└── docs/
```

**Hard constraints:**

- production bundles pin exact source and artifact revisions;
- policy and simulator worker topology is explicit;
- scalar and vector episode drivers are separate;
- model-specific transforms stay with the policy runtime or policy plugin;
- environment-specific transforms stay with the environment integration;
- `eval/scalar` imports interfaces, not simulator packages;
- native artifacts and diagnostics are preserved;
- model weights and copied upstream model/task implementations do not live in this repository;
- conversions record source artifact, converter revision, and validation result;
- large binaries live outside git with immutable hashes;
- future hardware namespaces do not imply a designed or supported real-robot loop.

---

## 15. Scope and phases

### Phase 0 — exact scalar baseline

- [ ] Define `PolicyEndpoint`, `PolicyArtifactProfile`, `EndpointProfile`, `ExecutionProfile`, scalar `SimBackend`, and evidence schemas.
- [ ] Implement fake policy and fake scalar environment fixtures for reset, chunk scheduling, timeout, late response, termination, truncation, and evidence.
- [ ] Materialize pinned OpenPI, LIBERO, MuJoCo, checkpoint, processor, and container artifacts without manual edits.
- [ ] Reproduce OpenPI `pi05_libero` through the native OpenPI evaluator.
- [ ] Implement the original-runtime OpenPI endpoint and original LIBERO scalar adapter.
- [ ] Compare fixed processors, controlled-noise action chunks, fixed action traces, reset, termination, and fixed-seed episodes.
- [ ] Produce a small success-rate report with complete evidence identity.
- [ ] Add the upstream-supported `pi0_aloha_sim` + gym-aloha scalar fixture after LIBERO passes.

**Exit:** one bundle command reproduces the exact OpenPI π0.5 + LIBERO lineage through crossrun, with the native oracle and conformance report beside it.

### Phase 1 — environment and vector extensions

- [ ] Add one Isaac/Arena environment condition with explicit framework, physics, renderer, task package, assets, and controller identity.
- [ ] Add one Genesis environment condition in its own isolated simulator worker.
- [ ] Select one narrow task/embodiment fixture implementable on at least two stacks without claiming benchmark identity.
- [ ] Run the unchanged scalar driver on each condition and publish semantic-drift evidence.
- [ ] Add `VectorEpisodeDriver` with disabled autoreset, stable slot identity, explicit reset masks, terminal-observation preservation, and per-slot policy state.
- [ ] Complete the RoboDojo-native versus Arena-reference comparison.
- [ ] Enable and harden XPolicyLab only for an active bundle that needs it.

**Exit:** scalar adapters pass on the selected MuJoCo-, Isaac-, and Genesis-based conditions; the vector driver passes its independent contract fixtures.

### Phase 2 — release and next-scope gate

- [ ] Add CI smoke matrices with explicit GPU and Isaac availability handling.
- [ ] Verify clean materialization, patch replay, container digests, and artifact fetch policies.
- [ ] Stabilize evidence schemas, reports, endpoint timeout/retry behavior, and native diagnostics.
- [ ] Publish supported combinations and explicit non-equivalences.
- [ ] Submit generally useful changes upstream and reassess retained local deltas.
- [ ] Decide whether a separately designed hardware milestone is justified.

**Exit:** a versioned sim-eval release is reproducible for the supported matrix, with no hidden real-robot implementation or unsupported equivalence claim.

---

## 16. Risks, security, and licensing

| Risk | Level | Response |
|---|---|---|
| Baseline lineage drift | High | exact artifact profiles; native oracle; reject same-family substitutions |
| Policy randomness changes results | High | separate seeds; controlled noise fixtures; state capability declarations |
| Scalar/vector semantic confusion | High | distinct drivers and acceptance tests |
| Simulator global state contaminates policy | High | isolated workers and pinned process environments |
| Environment-stack identity is flattened | High | separate framework, physics, renderer, task, asset, and controller fields |
| Adapter is syntactically but not behaviorally compatible | High | processor, action, transition, reset, and end-to-end fixtures |
| Retry duplicates state-changing inference | High | idempotency keys; no retry without guarantee; worker restart after non-cancellable timeout |
| Multi-upstream release skew | High | bundle-level locks and regression gates |
| Patch/fork burden becomes permanent | High | owner, fixture, review date, upstream disposition, and exit condition |
| “Best practice” is branding rather than evidence | High | native baseline and measured fidelity/performance/maintenance comparison |
| Result normalization loses detail | High | additive evidence and preserved native artifacts |
| Asset/license ambiguity | High | per-source, per-file, per-dataset, and per-weight manifests |

### 16.1 Security baseline

- no pickle or executable deserialization across a network boundary;
- schema and size validation for every request and response;
- loopback binding by default;
- authenticated and encrypted private remote transport;
- bounded queues, rate limits, and relative request deadlines;
- least-privilege worker processes or containers;
- health, readiness, and loaded-artifact attestation;
- stale-response rejection by trial/request/sequence identity;
- isolated worker restart after a non-cancellable timeout;
- no automatic state-changing retry without idempotency.

### 16.2 Licensing baseline

Every external source, checkpoint, dataset, and asset records:

```text
source URL and exact revision
file/package license
model-weight or dataset license, when distinct
redistribution and derivative permissions
required notices and attribution
use restrictions
local modifications and generated-file provenance
conversion lineage
```

Open-source and non-commercial intent do not remove these obligations.

---

## 17. Explicitly rejected for the initial scope

| Option | Why not now |
|---|---|
| One universal sim-and-real loop | hardware safety, reset, intervention, and timing require a later design |
| `gym.Env | VectorEnv` as one driver contract | hides terminal observation, reset, slot, and batching semantics |
| Treat model-family name as artifact identity | ignores checkpoint, implementation, processors, and normalization |
| Use a LeRobot-finetuned checkpoint as the OpenPI oracle | different lineage and evaluation protocol |
| Use an unvalidated π0.5 ALOHA candidate as the second correctness fixture | conflates embodiment coverage with checkpoint suitability |
| Hot-switch simulator stacks in one Python process | unsafe with Isaac application and Genesis global initialization |
| Treat `backend: isaac` as sufficient identity | omits environment framework, physics, renderer, task package, assets, and controller |
| Retry inference after disconnect by default | may duplicate state-changing stochastic work |
| Call sequential scalar requests “batching” | misrepresents transport and model execution |
| Convert every checkpoint to one weight format | expensive, lossy, and unnecessary for execution integration |
| Vendor upstream repositories or apply manual edits | obscures ownership, upgrades, provenance, and licenses |
| Put model-specific transforms in the generic episode loop | duplicates policy ownership and couples every model to crossrun |
| Pool different environment conditions into one primary success rate | conflates condition changes with policy performance |

---

## 18. Open questions

1. Which exact OpenPI endpoint shape best preserves original runtime behavior while accepting controlled inference noise?
2. What tolerances should processor, action, and environment-transition fixtures use for the Phase-0 path?
3. Which native LIBERO settings must be lifted verbatim into the execution profile: settling steps, image transforms, replanning interval, and max horizon?
4. Which narrow task/embodiment fixture can be implemented on MuJoCo-, Isaac-, and Genesis-based stacks without implying benchmark equivalence?
5. Which `SimCapabilities` fields are necessary for deterministic reset, rendering, control timing, and future vector preflight?
6. Which active path truly requires XPolicyLab rather than a native endpoint?
7. Which XPolicyLab models can support per-trial session isolation, and which require one worker per trial?
8. Which assets and weights may be redistributed, and which must be fetched by the user?
9. What quantitative gate should justify a separate hardware milestone?

---

## Links

| | |
|---|---|
| OpenPI | https://github.com/Physical-Intelligence/openpi |
| LeRobot | https://github.com/huggingface/lerobot |
| XPolicyLab | https://github.com/XPolicyLab/XPolicyLab |
| Gymnasium | https://github.com/Farama-Foundation/Gymnasium |
| LIBERO | https://github.com/Lifelong-Robot-Learning/LIBERO |
| gym-aloha | https://github.com/huggingface/gym-aloha |
| Isaac Lab | https://github.com/isaac-sim/IsaacLab |
| Isaac Lab-Arena | https://github.com/isaac-sim/IsaacLab-Arena |
| LW-BenchHub | https://github.com/LightwheelAI/LW-BenchHub |
| Newton Physics | https://github.com/newton-physics/newton |
| RoboDojo | https://github.com/RoboDojo-Benchmark/RoboDojo |
| Genesis World | https://github.com/Genesis-Embodied-AI/genesis-world |
| SIMPLE | https://github.com/physical-superintelligence-lab/SIMPLE |
| AutoEval | https://auto-eval.github.io/ |
