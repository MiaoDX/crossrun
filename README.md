# crossrun

**One portable sim-eval loop. Explicit runtime boundaries. Reproducible evidence.**

A simulation-first integration distribution and best-practice incubator for evaluating robot policies across MuJoCo-, Isaac-, and Genesis-based environments. crossrun pins known-good upstream revisions, materializes complete bundles, checks policy/environment compatibility, runs a bounded scalar episode loop, and records enough evidence to reproduce a result.

> **Status: design phase.** The architecture is under validation; implementation has not started.

## The idea

A working robot-policy experiment depends on an exact combination of policy code, checkpoint, processors, environment, assets, controller, renderer, launch topology, and evaluation settings. Waiting for every upstream to converge blocks use; copying those projects creates permanent maintenance work.

crossrun is the controlled middle ground: preserve upstream-native execution as the oracle, build a small portable evaluation path against the same artifacts, and use conformance evidence before deciding whether a change should move upstream, remain here, or be deleted.

```text
 pinned bundle
 policy artifact · endpoint · environment · execution · evidence
                              │
                    crossrun coordinator
                    materialize · launch · supervise
                              │
              ┌───────────────┴────────────────┐
              ▼                                ▼
       policy worker                    simulator worker
 native runtime or service       scalar episode loop + environment
              │                                │
              └──────── PolicyEndpoint ────────┘
                                               │
                                  Gymnasium scalar lifecycle
                                               │
                         ┌─────────────────────┼─────────────────────┐
                         ▼                     ▼                     ▼
                LIBERO + MuJoCo      Arena/Isaac environment   Genesis environment
```

**Get one exact result first.** Phase 0 uses Physical Intelligence's original OpenPI `pi05_libero` configuration, original checkpoint, original LIBERO/Panda environment, and OpenPI evaluator as the native oracle. The crossrun path must use the same policy artifact and task definitions; a LeRobot port or additionally fine-tuned checkpoint is a separate lineage, not an interchangeable baseline.

**Define both sides of the loop.** `PolicyEndpoint` owns policy reset and complete action-chunk prediction. `SimBackend` creates one scalar Gymnasium environment. Policy artifact metadata, endpoint/runtime capabilities, and action-execution scheduling are separate profiles.

**Keep process boundaries explicit.** The policy and simulator run in separate workers by default. This isolates model dependencies from Isaac application startup and Genesis global initialization, and it makes “switching backend” a pinned launch-topology change rather than an unsafe hot switch inside one Python process.

**Keep environment switching honest.** The common surface standardizes reset/step/termination lifecycle only. Environment framework, physics backend, renderer, task package, assets, controller, reset distribution, and success semantics remain first-class identity fields. An Isaac or Genesis port is never reported as the original MuJoCo benchmark.

## Initial paths

1. **Primary baseline:** OpenPI `pi05_libero` + original LIBERO/Panda on MuJoCo.
2. **Second scalar fixture:** OpenPI `pi0_aloha_sim` + gym-aloha. It exercises 14-D dual-arm state/action semantics, optional wrist cameras, a 50 Hz control loop, and a predicted chunk that is executed in shorter prefixes.
3. **Later backend fixture:** one narrowly defined task/embodiment profile implemented on at least two environment stacks with explicit semantic limits.

XPolicyLab remains an optional endpoint implementation for policies that need heterogeneous original-runtime serving. It is not required for the first result and is enabled only after its session, retry, deadline, queue, payload, and attestation semantics satisfy the selected bundle.

## Planned demos

1. **Native equivalence.** Reproduce the OpenPI π0.5 + LIBERO result, then compare fixed observations, controlled inference randomness, action chunks, reset behavior, and termination semantics through crossrun.
2. **Environment change.** Run the unchanged scalar episode driver against separately identified MuJoCo-, Isaac-, and Genesis-based environment conditions.
3. **Best-practice comparison.** Run a RoboDojo-native baseline beside an Arena-based reference integration and measure semantic fidelity, composition, performance, interoperability, migration effort, and maintenance cost.

## Design constraints

- Phase 0 supports one scalar `gym.Env`; `VectorEnv` uses a separate driver and is not hidden behind a union return type.
- The simulator worker owns environment lifecycle; the policy worker owns model runtime state. Cross-process messages carry stable trial, request, and sequence identities.
- Policy randomness is explicit. Environment, initial-state, policy, inference-noise, perturbation, and scheduler seeds are recorded separately.
- Action profiles distinguish model-output horizon, executed prefix, replan interval, control period, interpolation, clipping, and stale-action behavior.
- Native baselines and crossrun paths use the same artifact lineage when behavioral equivalence is claimed.
- Backend adapters preserve engine-native diagnostics and terminal observations instead of reducing results to a lossy success tuple.
- Environment identity separates framework, physics backend, renderer, task package, asset revision, and controller.
- Local wrappers, plugins, patches, forks, and reference integrations have an owner, pinned base, regression fixture, review date, and exit condition.
- Policy servers bind to loopback by default. Remote mode requires authenticated transport, bounded payloads and queues, schema validation, and network isolation.
- Every redistributed source, asset, dataset, and model records its exact revision and applicable license obligations.

## Roadmap

1. **Phase 0 — exact scalar baseline:** OpenPI π0.5 + original LIBERO + MuJoCo through a typed `PolicyEndpoint`, scalar `SimBackend`, and evidence envelope.
2. **Phase 1 — environment and vector extensions:** add separately identified Isaac/Arena and Genesis conditions; add a dedicated vector driver only after scalar semantics are stable.
3. **Phase 2 — release and scope gate:** stabilize the supported matrix, optional XPolicyLab bridge, CI, artifacts, and reports; then decide whether a separate hardware milestone is justified.

## Docs

- [`docs/plan.md`](docs/plan.md) — runtime topology, contracts, bundle profiles, conformance, evaluation protocol, phases, risks, and open questions
- [`docs/research/`](docs/research/) — dated surveys behind the design, including conclusions later superseded

## License

MIT
