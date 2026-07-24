# crossrun

**One protocol. Any robot, any policy, any backend. Sim and real.**

A reference design for running and evaluating robot policies — rethought from scratch, at a moment when the community finally has the pieces to do it properly.

> **Status: design phase.** The architecture is written down; the code is not there yet.
> See [`docs/plan.md`](docs/plan.md) for the full design, and [`docs/research/`](docs/research/) for the survey work behind it.

---

## Why now

Robot policy evaluation is in a strange state. There are dozens of benchmarks, several excellent simulators, a handful of serious policy-serving frameworks — and almost no agreement on how the pieces should fit together. Most teams either haven't invested yet, or invested years ago and now carry the architecture they started with.

2026 happens to be a good moment to redo this:

- **Policy serving has converged on a pattern.** Client–server inference over a socket, one process per model, is now how everyone does it.
- **The physics layer is consolidating.** Isaac Lab is moving to Newton, whose primary backend is MuJoCo-Warp. Accuracy and throughput are no longer opposed.
- **Evaluation methodology has caught up.** Perturbation tiers, confidence intervals, and real-to-sim correlation checks are established practice — just not yet standard practice.

None of this is our invention. The contribution here is putting it together cleanly, and being explicit about what the pieces are.

## What this is

A thin, opinionated middle layer — plus the demos that argue for it.

```
model containers          π0.5 · OpenVLA · XPolicyLab · your own
        │                 (obs update / predict / reset / batched)
        ▼
   protocol              strong schema, obs key mapping,
        │                action-space tagging
        ▼
   ⭐ runner              SimRunner          RealRunner
        │                 reset:   env       reset:   learned/manual
        │                 success: predicate success: VLM classifier
        │                 safe:    always    safe:    limits + motors
        │                 time:    stepped   time:    wall-clock
        ▼
   backends              MuJoCo · Isaac Lab-Arena · Genie Sim · real hardware
```

The **runner** is the part worth stealing. It is what makes "the same policy, the same protocol, in simulation and on hardware" a fact rather than a slogan.

## Design theses

Each of these should show up in the code and be defended by a demo:

1. **Protocol before implementation.** Models and simulators are both replaceable.
2. **Models do not live in this repo.** One container per model; we hold the contract.
3. **Backends need not be compatible with each other** — only with the protocol.
4. **The default path must be frictionless.** Heavy backends are opt-in, always.
5. **Numbers ship with uncertainty.** A bare success rate is a bug.
6. **A high score under the standard protocol means little.** Perturbation tiers are not optional.
7. **Across backends, compare rankings — never absolute numbers.**
8. **Something must sit between models and environments.** Models infer, environments simulate; somebody has to own the episode.
9. **Name the sim/real differences, don't abstract them away.** There are exactly three: where reset comes from, where success comes from, how time flows.

## Planned demos

The architecture is an argument, and arguments need evidence. All three run on LIBERO + MuJoCo with public checkpoints:

1. **How much does the protocol move the number?** Same checkpoint, varying episode counts, seeds, and perturbation tiers. Official protocols use 3 seeds × 500 rollouts; community reproductions often use 10 episodes. At 70 rollouts, a 95% CI is ~15 points wide.
2. **Rankings disagree across backends.** Same policies, different order on MuJoCo vs Isaac. If this *doesn't* happen, that is also worth knowing.
3. **90% → 0%.** A model that scores above 90% under the standard protocol, collapsing under position perturbation.

## Scope

Deliberately small at first: **LIBERO / ALOHA on MuJoCo, one VLA and one world-action model.** That is enough to demonstrate every thesis above.

Extensibility gets proven afterwards, not assumed: **Unitree G1** with whole-body control on Isaac Lab-Arena, and **AgiBot G2** via Genie Sim assets.

## Documentation

| | |
|---|---|
| [`docs/plan.md`](docs/plan.md) | Full design: architecture, evaluation protocol, phases, risks, rejected alternatives |
| [`docs/research/`](docs/research/) | The surveys this design came out of — ecosystem landscape, an adversarial review of an earlier draft, and a build-vs-adopt study of humanoid frameworks |

Design docs are currently written in Chinese; translations are welcome.

## License

MIT
