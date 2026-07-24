# crossrun — Design Document v2.3

> **What this is**: an open-source reference design for running and evaluating robot policies in simulation and on hardware.
> **Status**: design phase. The architecture is under validation; code has not started.
> **Companion research**: see [`research/`](research/).

**Version history.** v1.x was scoped as an internal evaluation system. v2.0 reframed it as an open reference design. v2.1 added the execution layer. v2.2 selected XPolicyLab as the default policy-side integration. **v2.3 corrects several over-strong assumptions, adds an explicit Runner-to-Backend contract, and treats protocol coverage, sim/real symmetry, and cross-backend comparability as hypotheses to test rather than settled facts.**

---

## 1. Positioning

**In one line:** define a small execution layer that owns robot-policy episodes while keeping policy serving, environments, statistics, and hardware integration behind explicit boundaries.

The project is not another simulator, benchmark, model zoo, or public protocol specification. Its value is a reference implementation of the seams between those systems, plus experiments that show where the seams hold and where they leak.

Three ecosystem trends make the experiment timely without making it inevitable:

- Isolated client-server policy inference is increasingly common because it separates model dependencies and permits remote deployment. It is not universal; in-process inference remains valid.
- GPU-native physics systems such as Newton and MuJoCo-Warp are maturing. Throughput is promising, while contact behaviour, feature coverage, and migration stability still require measurement.
- Perturbation tests, uncertainty intervals, provenance, and real-to-sim checks are available evaluation practices, but are not consistently applied.

**Deliverables, in priority order:**

1. A design others can inspect and borrow.
2. Reproducible demos that support or falsify the design claims.
3. A working evaluator that carries the first two.

**Success looks like:** someone clones the project, runs a known public baseline, sees exactly which assumptions were made, and can replace either the policy provider or environment backend without editing the episode logic.

---

## 2. What we build and what we adopt

| Layer | Decision | Confidence |
|---|---|---|
| **Model implementations** | Adopt upstream implementations and checkpoints | High; rebuilding models adds no value here |
| **Model packaging** | Adopt containers or isolated environments | High as a pattern, not as a single required tool |
| **Policy-facing interface** | Start with a pinned XPolicyLab adapter | Candidate; coverage and operational semantics must be measured |
| **Execution layer** | Build | Core contribution |
| **Backend interface** | Build a narrow internal contract | Required to make simulator independence real |
| **Simulators and robot drivers** | Adopt | High; expose them through backend adapters |
| **Task suites** | Adopt, beginning with LIBERO | High; comparability depends on preserving task and embodiment definitions |
| **Evaluation statistics** | Build a small implementation around established methods | High; methods are known, metadata discipline is usually missing |

### 2.1 XPolicyLab is a dependency behind an adapter

XPolicyLab currently exposes a model lifecycle that includes observation updates, single and batched action queries, and policy reset. crossrun will pin the exact revision it consumes and record the concrete method and message shapes in a **consumption profile**.

That profile is not advertised as a competing standard. It exists so crossrun can:

- detect upstream API drift;
- validate payloads before they reach a model process;
- map environment observation keys to policy-native inputs;
- tag action semantics, dimensions, frame, units, and horizon;
- define timeout, cancellation, retry, and error behaviour that a method signature alone does not specify.

Phase 0 tests whether this boundary supports latent-token action spaces, reset semantics, batched queries, backpressure, and failure recovery. Until those tests pass, the policy interface is a selected default, not a solved layer.

### 2.2 The novelty claim is deliberately narrow

Many benchmarks already contain evaluation clients and episode loops. The claim here is not that no execution code exists. The narrower claim is:

> We have not found a small, backend-neutral execution layer that exposes one explicit episode lifecycle across simulation and hardware while keeping policy serving and evaluation statistics separate.

The demos must defend that claim. If an existing project already provides the same boundary cleanly, crossrun should integrate or document it rather than compete with it.

---

## 3. Design claims to test

Each claim must be visible in code and paired with evidence.

1. **Adopt upstream interfaces behind adapters.** An adapter is cheaper to replace than a public specification is to maintain.
2. **Models do not live in this repo.** The repo holds integration metadata and reference containers, not model implementations or weights.
3. **The runner has two explicit boundaries.** `PolicyClient` faces the model; `Backend` faces simulation or hardware.
4. **The default path must be frictionless.** Heavy backends are opt-in.
5. **Numbers ship with uncertainty and provenance.** A bare success rate is an incomplete result.
6. **Perturbation evaluation is part of the baseline.** A standard score alone does not establish robustness.
7. **Cross-backend results are stratified, not pooled.** Report backend-specific outcomes and analyse policy-by-backend interactions.
8. **Something owns the episode.** Models infer and environments evolve; the runner schedules, terminates, and records.
9. **Sim and real share a lifecycle, not identical semantics.** Reset source, success assessment, and clock semantics are first-class seams; observation, control, safety, latency, and failure recovery remain backend capabilities.

---

## 4. Baseline snapshot

The table below is a dated survey snapshot, not a compatibility guarantee. Every implementation phase must pin exact revisions and reproduce one minimal run before relying on an entry.

| Benchmark | Backend | Embodiment | Reported or locally checked models | Checkpoints |
|---|---|---|---|---|
| **LIBERO** | robosuite / MuJoCo | **Panda/Franka** | π0, π0.5, OpenVLA, SmolVLA, Diffusion Policy | public |
| RoboDojo | Isaac Sim | ARX X5 / Piper / Piper X | multiple policies through XPolicyLab | leaderboard |
| RoboLab | Isaac | benchmark-specific | π0.5, π0-FAST, π0, PaliGemma | varies |
| RoboTwin | SAPIEN | Aloha-AgileX and others | several | public |
| SimplerEnv | SAPIEN | Google Robot / WidowX | RT-1, RT-1-X, Octo | public |
| Genie Sim | Isaac | G2 | GO-2, Pi series, GR00T; locally checked in the July 2026 survey | public assets/checkpoints |
| Isaac Lab-Arena | Isaac | multiple | GR00T and openpi-family integrations reported | public |
| SIMPLE | MuJoCo + Isaac | G1, ALOHA, Franka | several VLA/WAM baselines | varies |
| Genesis | native | multiple | no verified VLA evaluation path found in the survey | none selected |

A correlation reported between two benchmark leaderboards is evidence that those particular rankings can agree; it is not proof that the ecosystem is universally comparable. Likewise, failure to reproduce a headline score can come from protocol, software revision, checkpoint, rendering, control, or task-definition drift. Trial metadata must preserve enough information to distinguish them.

---

## 5. Backend strategy

```text
Default path
└── LIBERO / Panda / MuJoCo
    └── public task definitions and checkpoints; minimal dependency surface

Opt-in paths
├── Isaac Lab-Arena / G1
│   └── validate Arena first; integrate GR00T-WBC/SONIC as a separate stack
└── G2 assets derived from a pinned Genie Sim revision

Evaluation candidates
├── a second implementation of a matched task for backend-interaction studies
└── Genesis, behind a measurable porting and rendering gate
```

**Why LIBERO first.** The public evaluation paths and checkpoints target its Panda/Franka embodiment. Calling an ALOHA port “LIBERO” without retraining or validating observation/action compatibility would conflate a task-suite transfer with baseline reproduction.

**Why not make Genesis the default immediately.** A light installation is useful, but the default must have a verified policy-plus-task path. Genesis remains a candidate until a checkpoint, embodiment, rendering path, and task implementation run end to end.

**Why Isaac remains opt-in.** Isaac buys access to embodiments and integrations not available in the default path, but its dependency weight and backend migration churn should be paid only when those capabilities are needed.

---

## 6. Architecture

```text
┌─────────────────────────────────────────────────────────────┐
│ Policy providers                                             │
│ individual container or XPolicyLab-backed policy zoo         │
└──────────────────────────┬──────────────────────────────────┘
                           │ PolicyClient
                           │ update / predict / reset / health
┌──────────────────────────▼──────────────────────────────────┐
│ Runner                                                       │
│ episode state machine · deadlines · termination · recording  │
│                                                             │
│ consumes: PolicyClient, Backend, EvaluationConfig            │
└──────────────────────────┬──────────────────────────────────┘
                           │ Backend
                           │ reset / observe / step / stop
┌──────────────────────────▼──────────────────────────────────┐
│ Backend adapters                                             │
│ LIBERO · Arena · Genesis · robot hardware                    │
└─────────────────────────────────────────────────────────────┘

TrialRecord ──► eval-protocol
```

### 6.1 Required contracts

The exact language may change, but the information boundary must not.

```python
class PolicyClient(Protocol):
    def reset(self, context: EpisodeContext) -> None: ...
    def predict(self, observation: Observation, deadline_ns: int) -> ActionChunk: ...
    def health(self) -> HealthStatus: ...

class Backend(Protocol):
    def capabilities(self) -> BackendCapabilities: ...
    def reset(self, spec: ResetSpec) -> Observation: ...
    def step(self, action: Action, deadline_ns: int) -> StepResult: ...
    def stop(self, reason: StopReason) -> None: ...
```

`Action` is not merely a float array. It records at least:

- semantic type: joint, end-effector, velocity, torque, latent token, or another declared type;
- dimensions and chunk horizon;
- frame and units where applicable;
- intended control frequency or timestamps;
- validity limits.

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

The runner may depend on these internal types. It must not import simulator packages; backend adapters own those imports.

### 6.2 Sim/real symmetry and asymmetry

| Capability | Simulation default | Hardware default | Contract consequence |
|---|---|---|---|
| reset | environment reset | manual, fixture, replay guide, or learned reset | `ResetSpec` and reset provenance |
| success | privileged state predicate | classifier, sensors, or human audit | label source, version, confidence, abstention |
| time | stepped or accelerated | wall-clock and deadline constrained | timestamps and deadline semantics |
| observation | deterministic state/render path is possible | noise, calibration drift, loss, asynchronous sensors | capability metadata and missing-data policy |
| control | simulator actuator model | driver and low-level controller | declared units, frames, rate, saturation |
| safety | simulation limits and invalid-state checks | workspace, motor, collision, and emergency-stop checks | safety events are never assumed true |
| failure recovery | reset process | intervention, reboot, or cell repair | explicit termination and intervention records |

Reset, success, and time are the most visible orchestration seams; they are not an exhaustive list of sim/real differences.

### 6.3 XPolicyLab's two roles

| Role | Meaning | crossrun treatment |
|---|---|---|
| policy-side interface source | lifecycle methods and payload conventions | pin and adapt; do not assume stability |
| provider of many policies | a policy zoo with isolated dependencies | optional provider alongside individual containers |

XPolicyLab is not a model. It is also not the execution layer. The adapter must make it possible to replace XPolicyLab without changing the runner state machine.

---

## 7. Repository shape and dependency rules

```text
crossrun/
├── contracts/             # PolicyClient, Backend, TrialRecord, capabilities
├── runner/                # episode state machine; no simulator imports
├── adapters/
│   ├── policy/            # XPolicyLab and individual policy adapters
│   └── backend/           # LIBERO, Arena, hardware, later candidates
├── eval-protocol/         # statistics, reports, comparisons
├── containers/            # reference templates and registry
├── assets/                # source manifests and generation scripts
├── ci/
└── docs/
```

**Hard constraints:**

- `runner/` imports contracts, not simulator or robot-driver packages.
- Backend-specific types do not leak into `PolicyClient` or `TrialRecord`.
- `eval-protocol/` consumes `TrialRecord` objects, not a lossy `(success, trajectory)` tuple.
- No model weights or model implementations are committed to this repository.
- The XPolicyLab consumption profile is versioned internally but is not promoted as a competing public standard.
- Generated assets are reproducible from source manifests where upstream terms permit it.
- Large binary assets and datasets live outside git with immutable hashes.
- Policy servers default to loopback. Remote mode is explicit and authenticated.

**Minimum `TrialRecord` identity:**

```text
trial_id and run_id
task and task-definition revision
seed and initial-condition identifier
policy adapter, checkpoint digest, container digest
normalisation-statistics digest
backend, backend version, physics configuration
perturbation tier and parameters
action/control configuration
termination and failure reason
safety events and interventions
success label, source, classifier version, confidence, audit state
trajectory or trajectory digest
```

---

## 8. Evaluation protocol

### 8.1 Cross-backend comparison

The same nominal task can differ across physics engines because of contact, friction, rendering, control, and solver behaviour.

**Rules:**

- Never pool results from different backends into one success rate.
- Report backend and version as first-class grouping variables.
- Absolute success rates may appear in the same report when clearly stratified; they must not be interpreted as equal-difficulty measurements.
- Compare ranking agreement, matched failure modes, and policy-by-backend interactions.
- A ranking reversal is evidence of an interaction. It does not by itself prove that either backend contains an artefact.
- Where matched initial conditions are available, preserve pairing and use paired analysis.

A second backend is therefore not a source of a “truer” score. It is a way to measure sensitivity to implementation conditions.

### 8.2 Statistical discipline

- Record the complete trial identity above.
- Report a binomial interval for each policy × task × backend × perturbation condition. Clopper-Pearson is a conservative default; the report may additionally include Wilson or Bayesian intervals if named explicitly.
- Do not infer equality from overlapping confidence intervals.
- Predeclare pairwise comparisons and correct for multiplicity where many policies or tiers are tested.
- Use paired tests or hierarchical models when runs share seeds or initial conditions.
- Separate rollout sampling uncertainty from success-label uncertainty.

**Power reference:** 63 successes in 70 Bernoulli trials gives an exact 95% interval approximately 15 percentage points wide. Achieving a roughly ±2-point interval near 90% success requires on the order of one thousand trials; the exact requirement depends on the interval and stopping rule.

**Protocol comparison wording:** OpenVLA reports 50 rollouts per task per seed in the cited setup. A reproduction default of 10 episodes per task is therefore a fivefold per-task difference, not a comparison of 1,500 total rollouts with 10 total rollouts.

### 8.3 Perturbation tiers

Standard and perturbed results are always reported together. Perturbations must record their generation seed and parameters, and should distinguish at least:

- visual appearance and distractors;
- lighting and rendering;
- target or object pose;
- observation degradation;
- action latency or control disturbance, where safe.

A collapse under one perturbation is evidence about that perturbation family, not a universal statement that the model has no capability.

### 8.4 Hardware success labels

A VLM or learned classifier is a measurement instrument, not ground truth. Hardware reports must include:

- classifier and prompt/configuration revision;
- confidence or score and abstention behaviour;
- a sampled human-audit set;
- confusion estimates on the evaluated task distribution;
- the policy for disagreements and uncertain outcomes.

The confidence interval over rollouts does not account for label error unless the analysis models it explicitly.

---

## 9. Scope and phases

> Make the pattern clear on the smallest reproducible example, then test whether it extends.

**Minimum demonstrable set:** LIBERO on the Panda/Franka embodiment in MuJoCo, one VLA, and one world-action model.

**Extensibility candidates:** G1 through Arena plus a separately validated GR00T-WBC/SONIC stack; G2 through versioned assets derived from Genie Sim.

### Phase 0 — Boundaries before coverage

- [ ] Pin the XPolicyLab revision and write the consumed lifecycle/profile explicitly.
- [ ] Define `PolicyClient`, `Backend`, `BackendCapabilities`, `StepResult`, and `TrialRecord`.
- [ ] Implement SimRunner + LIBERO/Panda + one official public checkpoint end to end.
- [ ] Implement seed management, provenance, trajectory digests, and confidence intervals.
- [ ] Measure latent-token payload support, reset behaviour, batching, throughput, timeouts, cancellation, and server failure.
- [ ] Add schema-validation and loopback-only security defaults.
- [ ] Verify the exact Isaac Lab and Arena revisions before planning an integration.

### Phase 1 — Claims and minimal hardware symmetry

- [ ] Demo 1: protocol and sample-size sensitivity.
- [ ] Demo 3: controlled perturbation collapse.
- [ ] Add a second conventional policy and one world-action model.
- [ ] Implement a RealRunner skeleton with manual reset guidance and a minimal hardware loop.
- [ ] Record success-label provenance and intervention events from the first hardware run.
- [ ] Evaluate a second backend only after a matched task and control definition exists.

### Second-backend gate

| Criterion | Pass | Failure means |
|---|---|---|
| porting cost | one task and embodiment in five engineering days or less | investigate whether the backend contract leaks |
| rendering/control viability | enough throughput and fidelity for the planned trial count | retain as a watch item or narrow the demo |
| contract reuse | runner state machine and evaluation schema remain unchanged | revise contracts before adding more backends |
| task matching | task, embodiment, observations, and action semantics are documented | do not interpret differences as backend effects |

Passing the gate makes the backend eligible for the interaction demo. Failure is a design result, not a reason to hide the attempt.

### Phase 2 — G1 and whole-body control

- [ ] Validate Arena's G1 path independently.
- [ ] Validate the GR00T-WBC/SONIC encoder, decoder, hands, cameras, and controller independently.
- [ ] Build an adapter between the two only after both paths run.
- [ ] Record the 78-dimensional action interpretation and decoder revision rather than assuming generic float-array compatibility is sufficient.
- [ ] Keep the low-level controller swappable so policy and controller failures can be separated.

### Phase 3 — G2 assets

- [ ] Pin the exact Genie Sim source revision and inventory file-level licenses.
- [ ] Produce versioned assets and generation/extraction scripts where redistribution is allowed.
- [ ] Keep the default path runnable without initialising the upstream repository.
- [ ] Use a known-good checkpoint only after its weight license and redistribution terms are recorded.

### Phase 4 — Outward

- [ ] Publish the design and negative results, not only successful demos.
- [ ] Complete hardware reset, success, safety, and recovery paths.
- [ ] Consider EnvHub registration once contracts have survived two backends.
- [ ] Contribute integrations upstream where ownership and maintenance are clearer there.

---

## 10. Risks, security, and licensing

| Risk | Level | Response |
|---|---|---|
| **Policy interface drift** | High | pin revisions; contract tests; keep the adapter replaceable |
| **Runner/backend boundary leaks** | High | second-backend gate; prohibit simulator types in contracts |
| **Isaac/Newton migration churn** | High | pin exact revisions; treat backend changes as measured upgrades |
| **G1 dependency chain** | Med-high | validate Arena and WBC/SONIC separately before integration |
| **Success-label error on hardware** | High | audit labels and report the measurement process |
| **Cross-backend overinterpretation** | Med-high | stratified reports and interaction analysis |
| **Demos remain toy examples** | Med-high | real checkpoints, perturbations, sufficient trials, failure records |
| **Asset/license ambiguity** | High | per-file and per-weight manifests; no blanket ecosystem claim |

### Security baseline

A server not exposed to the public internet can still be unsafe on a shared or compromised network. The baseline is:

- no pickle or executable deserialisation across the boundary;
- schema and size validation for every request and response;
- loopback binding by default;
- authenticated and encrypted remote transport, normally through a private network or tunnel;
- network allow-lists and explicit remote-mode configuration;
- request deadlines, rate limits, and bounded queues;
- least-privilege containers or processes;
- health checks and a fail-safe stop path when the policy server is unavailable.

The LeRobot async-server vulnerability is a useful precedent for the class of risk, not evidence that merely changing the bind address solves it.

### Licensing baseline

Open-source and non-commercial intent do not remove license obligations. Every external artifact or weight must record:

```text
source URL and exact revision
file or package license
model-weight license, if distinct
redistribution and derivative permissions
required notices and attribution
use restrictions
local modifications and generated-file provenance
```

No document should claim that an entire multi-purpose upstream repository has one license unless that has been checked file by file.

---

## 11. Explicitly rejected for the initial scope

| Option | Why not now |
|---|---|
| Build a simulator or benchmark from scratch | years of work and no immediate comparability |
| Fork fast-moving upstreams | maintenance and comparison drift; prefer extensions and pinned adapters |
| Publish a competing policy protocol | start with an adapter and collect evidence before standardising anything |
| Treat XPolicyLab as a model | it is a policy-serving framework and policy zoo |
| Treat XPolicyLab as permanently mandatory | its coverage and operational semantics are not yet proven for every target |
| Use Genie Sim as a default runtime dependency | too broad for the minimal path; consume only explicitly licensed, versioned pieces |
| Use SIMPLE as the foundation | useful task designs, but not selected as the default execution stack |
| Use Genesis as the default before an end-to-end baseline exists | installation weight alone does not prove ecosystem readiness |
| Put model code and weights in this repo | creates dependency and licensing conflicts |
| Hide low-level controller choice | prevents attribution of whole-body failures |
| Report one pooled number across backends | conflates condition changes with policy performance |
| Frame all reproduction variance as a crisis | causes are heterogeneous and need trial-level evidence |

---

## 12. Open questions

These are measurements to make, not conclusions already reached.

1. Which XPolicyLab revision and exact method set should the first consumption profile pin?
2. Can the selected payload and adapter represent SONIC's latent-token action semantics, including decoder identity and timing, without an unsafe convention?
3. What throughput, queueing, timeout, and cancellation behaviour appears under batched load?
4. Which Arena revision and Isaac backend does the G1 path actually require?
5. Does Arena integrate any part of SONIC natively, or is the connection entirely ours?
6. Which G1 hands, cameras, and low-level controller match the chosen checkpoint?
7. Can a second backend reproduce a matched task closely enough for an interpretable interaction study?
8. How accurate and calibrated is the hardware success classifier on each task?
9. What is the actual engineering time to add a robot after physical validation and safety review?
10. Which upstream assets and weights can legally be redistributed, and which must be fetched by the user?

---

## Links

| | |
|---|---|
| XPolicyLab | https://github.com/XPolicyLab/XPolicyLab |
| openpi | https://github.com/Physical-Intelligence/openpi |
| OpenVLA | https://github.com/openvla/openvla |
| LIBERO | https://github.com/Lifelong-Robot-Learning/LIBERO |
| LeRobot | https://github.com/huggingface/lerobot |
| Isaac Lab-Arena | https://github.com/isaac-sim/IsaacLab-Arena |
| Newton Physics | https://github.com/newton-physics/newton |
| Isaac-GR00T | https://github.com/NVIDIA/Isaac-GR00T |
| GR00T Whole-Body Control / SONIC | https://github.com/NVlabs/GR00T-WholeBodyControl |
| RoboDojo | https://github.com/RoboDojo-Benchmark/RoboDojo |
| Genie Sim | https://github.com/AgibotTech/genie_sim |
| SIMPLE | https://github.com/physical-superintelligence-lab/SIMPLE |
| Genesis | https://github.com/Genesis-Embodied-AI/genesis-world |
| AutoEval | https://auto-eval.github.io/ |
| SimplerEnv | https://github.com/simpler-env/SimplerEnv |
