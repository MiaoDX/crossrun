# crossrun

**One runner. Explicit contracts. Sim and real.**

A reference design for running and evaluating robot policies across simulation and hardware. The interesting part is not another benchmark: it is the execution layer that owns an episode while keeping policy serving and environment integration replaceable.

> **Status: design phase.** The architecture is under validation; code has not started.

## The idea

Models infer. Environments simulate or drive hardware. Something still has to decide when an episode begins, how actions are scheduled, whether the task succeeded, when a rollout must stop, and what evidence is recorded.

crossrun is that execution layer.

```text
 policy providers                         environment backends
 one model each        XPolicyLab zoo     MuJoCo · Isaac · hardware
       │                      │                      │
       └──────────┬───────────┘                      │
                  │ pinned policy adapter            │ backend adapter
                  ▼                                  ▼
          ┌──────────────────────────────────────────────┐
          │                  Runner                      │
          │ reset · observe · act · step · stop · record│
          └──────────────────────────────────────────────┘
                  │                                  │
          policy-facing contract             backend contract
```

**XPolicyLab is the default policy-facing integration candidate, not a proven universal standard.** crossrun will pin the version it consumes and keep it behind an adapter. Phase 0 validates action-space coverage, reset semantics, batching, throughput, timeouts, and failure behaviour before treating that boundary as stable.

Simulation and hardware share an episode lifecycle, but they are not equivalent. Reset source, success assessment, and clock semantics are three first-class orchestration seams. Observation quality, control semantics, safety, latency, and failure recovery remain explicit backend capabilities rather than assumed similarities.

## Why now

- **Isolated policy serving is increasingly common.** Client-server inference is useful for dependency isolation and remote deployment, although in-process inference remains valid.
- **GPU-native physics is maturing.** Newton and MuJoCo-Warp are promising, but solver behaviour, feature coverage, and migration stability still require measurement.
- **Evaluation practice is improving.** Perturbation tests, uncertainty intervals, provenance, and real-to-sim checks exist, but are not consistently applied.

The contribution is assembling these pieces with narrow contracts and making the assumptions measurable.

## Planned demos

1. **Protocol sensitivity.** Hold the checkpoint fixed while varying episode count, seeds, and perturbation tiers. At 63 successes in 70 Bernoulli trials, the exact 95% interval is roughly 15 percentage points wide.
2. **Backend interaction.** Run the same policies and task definitions on two backends, report results separately, and measure whether policy ordering or failure modes change. A ranking reversal is evidence of an interaction, not proof that one backend is wrong.
3. **Perturbation collapse.** Reproduce a model that scores above 90% under the standard setup and fails under a controlled position perturbation.

## Scope

Small on purpose: **LIBERO on its Panda/Franka embodiment in MuJoCo, one VLA, and one world-action model.** This preserves compatibility with public LIBERO checkpoints and evaluation paths.

Extensibility is tested afterwards, not assumed: **Unitree G1 through Isaac Lab-Arena with a separately validated GR00T-WBC/SONIC integration**, and **AgiBot G2 through versioned Genie Sim assets**.

## Design constraints

- The runner depends on internal contracts, not simulator packages.
- Policy adapters and backend adapters are separate boundaries.
- Evaluation records include task, seed, policy/checkpoint digest, backend/version, perturbation tier, termination reason, safety events, and success-label provenance.
- Policy servers default to loopback. Remote serving requires authenticated transport, schema validation, and network isolation.
- Every redistributed asset and model records its exact source revision and license obligations.

## Docs

- [`docs/plan.md`](docs/plan.md) — architecture, contracts, evaluation protocol, phases, risks, and open questions
- [`docs/research/`](docs/research/) — the surveys behind the design, including conclusions that were later overturned

## License

MIT
