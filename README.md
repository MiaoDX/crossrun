# crossrun

**Pinned upstreams. Proven reference paths. Reproducible sim and real runs.**

A short-cycle integration distribution and best-practice incubator for running and evaluating robot policies across simulation and hardware. crossrun pins known-good upstream revisions, carries focused integrations or patches, launches isolated runtimes, checks compatibility, and records enough evidence to reproduce a run.

> **Status: design phase.** The architecture is under validation; code has not started.

## The idea

The robotics stack is moving quickly, but a working experiment still depends on an exact combination of policy code, checkpoint, environment, robot driver, processors, controller and launch configuration. Waiting for every upstream to converge blocks use; copying those projects creates permanent maintenance work.

crossrun is the controlled middle ground: preserve upstream execution paths as reproducible baselines, build better reference paths where current upstream architecture is limiting, and use evidence before deciding whether a change should move upstream, remain here, or be deleted.

```text
 upstream repos and artifacts
 LeRobot · XPolicyLab · RoboDojo · Arena · OpenPI · Genie Sim
                         │
                         ▼
             crossrun integration bundle
 lockfile · reference integrations · patches · launch · conformance
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

**Keep the native owner as the baseline.** LeRobot owns its policies, environments, robot drivers and rollout loops. RoboDojo owns its benchmark lifecycle. XPolicyLab owns heterogeneous model adapters and remote policy serving. Their current architecture is the comparison point, not an automatic ceiling on what crossrun may implement.

**Incubate before upstreaming.** The initial distribution uses a pinned crossrun XPolicyLab fork because its policy catalog and dependency isolation are valuable while several service and Pi adapter gaps still require source changes. LeRobot integrations can begin as out-of-tree plugins. A larger idea, such as expressing RoboDojo-style tasks through Isaac Lab-Arena, may first live as a crossrun reference integration so it can run, be measured and mature without depending on prior upstream acceptance.

Every local delta has an upstream base revision, owner, regression fixture, lifecycle intent, upstream issue or PR where appropriate, and review date. Production bundles never track a moving `main` branch. A local implementation may remain when it demonstrates a coherent cross-project architecture that no single upstream owns, but it must remain pinned, tested and cheaper to maintain than the value it provides.

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
6. **Best-practice comparison.** Run a RoboDojo-native baseline beside an Arena-based reference integration and measure whether Arena improves composition, reuse and maintenance enough to justify migration.

## Design constraints

- Upstream-native execution loops remain reproducible baselines; reference integrations may use a different owner when they test a named architectural hypothesis.
- For compatibility fixes, prefer wrapper, then plugin, then temporary patch, then fork; reference experiments choose the smallest shape that can test their architectural hypothesis.
- Fork commits stay small, independently testable and suitable for upstream submission where generally useful.
- Best-practice claims require an explicit comparison against the upstream-native baseline; newer architecture is not assumed better by name alone.
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
