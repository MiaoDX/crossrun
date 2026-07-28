# crossrun

**Pinned upstreams. Small overlays. Reproducible sim and real runs.**

A short-cycle integration distribution for running and evaluating robot policies across simulation and hardware. crossrun pins known-good upstream revisions, carries the smallest necessary plugins or patches, launches isolated runtimes, checks compatibility, and records enough evidence to reproduce a run.

> **Status: design phase.** The architecture is under validation; code has not started.

## The idea

The robotics stack is moving quickly, but a working experiment still depends on an exact combination of policy code, checkpoint, environment, robot driver, processors, controller and launch configuration. Waiting for every upstream to converge blocks use; copying those projects creates permanent maintenance work.

crossrun is the controlled middle ground: use upstream execution paths wherever they work, maintain temporary overlays where they do not, and continuously send generally useful fixes back upstream.

```text
 upstream repos and artifacts
 LeRobot · XPolicyLab · RoboDojo · Arena · OpenPI · Genie Sim
                         │
                         ▼
             crossrun integration bundle
 lockfile · plugins · patch queue · launch · conformance
                         │
                         ▼
 upstream-native execution owners
 lerobot-eval/rollout · RoboDojo EvalEnv · Arena/EnvHub
                         │
       optional XPolicyLab remote-policy service
                         │
                         ▼
       normalized provenance and comparison reports
```

**Use the native owner first.** LeRobot owns its policies, environments, robot drivers and rollout loops. RoboDojo owns its benchmark lifecycle. XPolicyLab owns heterogeneous model adapters and remote policy serving. crossrun does not replace those owners merely to make the diagram uniform.

**Maintain overlays deliberately.** The initial distribution uses a pinned crossrun XPolicyLab fork because its policy catalog and dependency isolation are valuable while several service and Pi adapter gaps still require source changes. LeRobot integrations should begin as out-of-tree plugins. RoboDojo and Arena should be wrapped at their published boundaries and forked only when a proven blocker cannot be fixed externally.

Every local delta has an upstream base revision, owner, regression fixture, upstream issue or PR, and review date. Production bundles never track a moving `main` branch.

Simulation and hardware may use different upstream execution owners. Reset source, success assessment, clock semantics, observation quality, control, safety, latency and recovery therefore remain explicit compatibility and evidence fields. The available real hardware is Unitree G1 and AgiBot G2; ALOHA remains simulation-only in the initial plan.

## Initial paths

Two upstream-supported MuJoCo paths enter the first bundle:

1. **LIBERO / Panda** with a LIBERO-tuned policy such as `pi05_libero` — benchmark reproduction, statistics, and perturbation evaluation.
2. **ALOHA Sim** with an OpenPI π0.5 ALOHA profile, initially evaluating `pi05_aloha` with `pi05_base` or a validated task-tuned checkpoint — dual-arm actions and a higher-frequency control path.

Prefer LeRobot's existing `lerobot-eval`, LIBERO and gym-aloha paths for the first working runs. In parallel, validate an XPolicyLab-to-LeRobot bridge so original or non-LeRobot policy runtimes can reach the same environments without moving model code into crossrun.

## Planned demos

1. **Known-good bundle.** Reproduce π0.5 + LIBERO and smoke-test π0.5 + ALOHA Sim from one pinned manifest.
2. **Remote-policy bridge.** Run an XPolicyLab-served policy through an upstream environment owner without a second environment loop.
3. **Upgrade proof.** Change one upstream revision and show that patch replay plus conformance tests expose the compatibility delta.
4. **Minimal real loop.** Reuse LeRobot's G1 support or a G2 plugin for a safety-bounded hardware run.
5. **Environment interaction.** Keep original MuJoCo and adapted Isaac results separate while recording their semantic differences.

## Design constraints

- Upstream-native execution loops remain the episode owners unless a measured gap proves that none can support the path.
- Prefer wrapper, then plugin, then temporary patch, then fork; every escalation requires evidence.
- Fork commits stay small, independently testable and suitable for upstream submission where generally useful.
- Policy integrations preserve original checkpoints when practical; conversion is optional and explicitly recorded.
- Every runnable bundle declares policy/environment compatibility across observations, actions, normalization, timing, statefulness and controllers.
- Evaluation records include upstream revisions, local deltas, checkpoint digests, environment identity, task revision, termination, safety events and success-label provenance.
- Policy servers default to loopback. Remote serving requires authenticated transport, schema validation and network isolation.
- Every redistributed asset and model records its exact source revision and license obligations.
- Isaac ports are distinct environments. Lightwheel-LIBERO is never reported as the original MuJoCo LIBERO baseline.

## Docs

- [`docs/plan.md`](docs/plan.md) — bundle architecture, overlay governance, compatibility, evaluation protocol, phases, risks, and open questions
- [`docs/research/`](docs/research/) — the surveys behind the design, including conclusions that were later overturned

## License

MIT
