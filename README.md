# crossrun

**One runner. One policy-service boundary. Sim and real.**

A reference design for running and evaluating robot policies across simulation and hardware. The contribution is the execution layer that owns an episode while policy implementations, checkpoints, simulators, and robot drivers remain replaceable.

> **Status: design phase.** The architecture is under validation; code has not started.

## The idea

Models infer. Environments simulate or drive hardware. Something still has to decide when an episode begins, how actions are scheduled, whether the task succeeded, when a rollout must stop, and what evidence is recorded.

crossrun is that execution layer.

It is intentionally conditional. If XPolicyLab plus maintained environment projects provide the same backend-neutral episode lifecycle, compatibility checks and provenance, crossrun should contribute its remaining gaps upstream and shrink or stop rather than become a duplicate platform.

```text
 upstream policy implementations and checkpoints
 OpenPI · OpenVLA · LeRobot · GR00T · WAMs · custom repos
                              │
                              │ model-specific adapter
                              ▼
                   pinned crossrun XPolicyLab fork
                   original runtimes and weights may stay intact
                              │
                              │ crossrun PolicyClient
                              ▼
          ┌──────────────────────────────────────────────┐
          │                   Runner                     │
          │ reset · observe · act · step · stop · record│
          └────────────────────────┬─────────────────────┘
                                   │ Backend
                                   ▼
                  LIBERO · ALOHA Sim · Isaac · hardware
```

**XPolicyLab is the single default policy runtime boundary.** crossrun consumes a pinned build of its own XPolicyLab fork, regularly syncs the upstream remote, and sends generally useful patches upstream. This does not require every model to use XPolicyLab-native weights or training code. Most integrations wrap the upstream runtime and checkpoint, then adapt observations, normalization, action semantics, reset behaviour, and capabilities to the pinned service profile.

**LeRobot has three explicit roles, none of which replaces the runner:**

- a source of LeRobot-native policy implementations and checkpoints, loaded through a generic XPolicyLab bridge where possible;
- a robot-driver backend through its `Robot` interface;
- a dataset and trajectory interchange ecosystem.

Simulation and hardware share an episode lifecycle, but they are not equivalent. Reset source, success assessment, and clock semantics are first-class orchestration seams. Observation quality, control semantics, safety, latency, and failure recovery remain backend capabilities. The available real hardware is Unitree G1 and AgiBot G2; ALOHA remains simulation-only in the initial plan.

## Initial paths

Two upstream-supported MuJoCo paths enter Phase 0:

1. **LIBERO / Panda** with a LIBERO-tuned policy such as `pi05_libero` — benchmark reproduction, statistics, and perturbation evaluation.
2. **ALOHA Sim** with an OpenPI π0.5 ALOHA profile, initially evaluating `pi05_aloha` with `pi05_base` or a validated task-tuned checkpoint — dual-arm actions, higher-frequency control, and a path structurally close to real ALOHA.

They use the same Runner and policy-service boundary, but different checkpoints, policy profiles, observation mappings, action spaces, and normalization statistics.

## Planned demos

1. **One service, two embodiments.** Run LIBERO/Panda and ALOHA Sim with π0.5 policies through the same Runner and pinned XPolicyLab-fork service boundary without moving episode logic into model adapters.
2. **Protocol sensitivity.** Hold a checkpoint fixed while varying episode count, seeds, and perturbation tiers.
3. **Backend interaction.** Report matched tasks separately by backend and measure policy-by-backend interactions.
4. **Perturbation collapse.** Reproduce a high standard score that fails under a controlled perturbation.
5. **Minimal real loop.** Run a safety-bounded static-manipulation lifecycle on the better-matched available G1 or G2 configuration after hardware and checkpoint preflight.

## Design constraints

- Production policy execution goes through one pinned service boundary built from the crossrun XPolicyLab fork.
- The runner depends on internal contracts, not XPolicyLab internals, simulator packages, or robot drivers.
- Policy adapters preserve original checkpoints when practical; weight conversion is optional and explicitly recorded.
- Every policy ships a compatibility profile covering observations, actions, normalization, chunking, timing, statefulness, batching, and reset semantics.
- Evaluation records include policy/runtime/checkpoint digests, backend/version, task revision, perturbation, termination, safety events, and success-label provenance.
- Policy servers default to loopback. Remote serving requires authenticated transport, schema validation, and network isolation.
- Every redistributed asset and model records its exact source revision and license obligations.
- Isaac ports are versioned as distinct backends. Lightwheel-LIBERO is an adapted Isaac Lab-Arena benchmark and is never reported as the original MuJoCo LIBERO baseline.

## Docs

- [`docs/plan.md`](docs/plan.md) — architecture, runtime strategy, contracts, evaluation protocol, phases, risks, and open questions
- [`docs/research/`](docs/research/) — the surveys behind the design, including conclusions that were later overturned

## License

MIT
