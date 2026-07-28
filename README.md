# crossrun

**One sim-eval surface. Multiple physics backends. Reproducible results.**

A simulation-first integration distribution and best-practice incubator for evaluating robot policies across MuJoCo, Isaac and Genesis. crossrun owns a small evaluation harness, pins known-good upstream revisions, checks policy/environment compatibility, and records enough evidence to reproduce a run.

> **Status: design phase.** The architecture is under validation; code has not started.

## The idea

The robotics stack is moving quickly, but a working experiment still depends on an exact combination of policy code, checkpoint, environment, robot driver, processors, controller and launch configuration. Waiting for every upstream to converge blocks use; copying those projects creates permanent maintenance work.

crossrun is the controlled middle ground: preserve upstream execution paths as reproducible baselines, build better reference paths where current upstream architecture is limiting, and use evidence before deciding whether a change should move upstream, remain here, or be deleted.

```text
 policies and checkpoints
 LeRobot · XPolicyLab · OpenPI · other runtimes
                         │
                         ▼
              crossrun sim eval
 bundle · policy endpoint · episode loop · evidence
                         │
                         ▼
     Gymnasium-compatible simulation interface
                         │
             ┌───────────┼───────────┐
             ▼           ▼           ▼
          MuJoCo    Isaac / Arena   Genesis
```

**Get one result first.** The first vertical slice is π0.5 + original LIBERO/Panda on MuJoCo. It proves bundle materialization, the policy endpoint, the sim interface, the evaluation loop and the evidence record end to end. ALOHA Sim becomes the second contract fixture after that path works.

**Keep backend switching honest.** The shared surface follows Gymnasium reset/step/termination semantics and adds explicit capabilities, task identity and provenance. It makes backend adapters easy to exchange; it does not claim that a MuJoCo task, an Isaac port and a Genesis port are physically or semantically identical.

**Incubate before upstreaming.** RoboDojo's native Isaac Sim/Isaac Lab path remains a baseline. An Arena-based version can first live here behind the common sim interface, where it can be compared before deciding whether to upstream it, retain it, fork it or delete it.

Real-robot support is deferred until sim eval works across backends. The policy endpoint and evidence schema stay extensible for the available Unitree G1 and AgiBot G2, but the current milestone contains no real driver, controller or hardware rollout.

## Initial paths

The first MuJoCo bundle is intentionally narrow:

1. **Primary:** LIBERO / Panda with `pi05_libero` — the first end-to-end result.
2. **Second fixture:** ALOHA Sim with a validated π0.5 ALOHA profile — proves that the interface handles different observation and action semantics.

Use the native LeRobot/OpenPI path as a result baseline. Add the XPolicyLab remote-policy bridge only when the selected model path needs original-runtime serving; it is no longer a phase by itself.

## Planned demos

1. **First result.** Reproduce π0.5 + LIBERO through the crossrun sim interface from one pinned bundle.
2. **Backend switch.** Run the unchanged evaluation loop against MuJoCo, Isaac and Genesis, including one declared task/embodiment profile on at least two engines.
3. **Best-practice comparison.** Run a RoboDojo-native baseline beside an Arena-based reference integration and measure whether Arena improves composition, reuse and maintenance enough to justify migration.

## Design constraints

- The crossrun episode loop is simulation-only and consumes a Gymnasium-compatible environment; it is not a universal sim/real runner.
- Backend adapters expose capabilities and preserve engine-native diagnostics instead of hiding semantic differences.
- For compatibility fixes, prefer wrapper, then plugin, then temporary patch, then fork; reference experiments choose the smallest shape that can test their architectural hypothesis.
- Fork commits stay small, independently testable and suitable for upstream submission where generally useful.
- Best-practice claims require an explicit comparison against the upstream-native baseline; newer architecture is not assumed better by name alone.
- Policy integrations preserve original checkpoints when practical; conversion is optional and explicitly recorded.
- Every runnable bundle declares policy/environment compatibility across observations, actions, normalization, timing, statefulness and controllers.
- Evaluation records include upstream revisions, local deltas, checkpoint digests, environment identity, task revision, termination, safety events and success-label provenance.
- Policy servers default to loopback. Remote serving requires authenticated transport, schema validation and network isolation.
- Every redistributed asset and model records its exact source revision and license obligations.
- Isaac ports are distinct environments. Lightwheel-LIBERO is never reported as the original MuJoCo LIBERO baseline.
- Genesis is the targeted physics backend; AgiBot Genie Sim belongs to deferred G2 work and is not part of the current milestone.

## Roadmap

1. **Phase 0 — first sim result:** π0.5 + LIBERO + MuJoCo through the crossrun eval loop.
2. **Phase 1 — backend switching:** add Isaac/Arena and Genesis without changing the loop.
3. **Phase 2 — release and decide:** stabilize the sim matrix, then decide whether to start a separate hardware milestone.

## Docs

- [`docs/plan.md`](docs/plan.md) — bundle architecture, overlay governance, compatibility, evaluation protocol, phases, risks, and open questions
- [`docs/research/`](docs/research/) — the surveys behind the design, including conclusions that were later overturned

## License

MIT
