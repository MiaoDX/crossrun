# crossrun

**One protocol. Any robot, any policy, any backend. Sim and real.**

A reference design for running and evaluating robot policies. The interesting part is not another benchmark — it is the layer that lets the same policy run, unchanged, in simulation and on hardware.

> **Status: design phase.** The architecture is written down. The code is not there yet.

## The idea

Robot policy evaluation has no shortage of benchmarks, several good simulators, and by now a settled pattern for serving models. What is missing is the piece in the middle: the thing that owns an episode.

Models infer. Environments simulate. Somebody still has to decide when an episode begins, whether it succeeded, and what happens between two steps. In simulation all of that is free — `env.reset()`, a success predicate, stepped time. On hardware none of it is.

crossrun is that middle piece.

```
 policy providers     one model each         a whole zoo
                      π0.5 · OpenVLA         ╔══════════════════╗
                      · your own             ║ XPolicyLab (40+) ║
                             │               ╚═════════╤════════╝
                             └────────┬────────────────┘
                                      │  obs update · predict
                                      │  · reset · batched
                                      ▼
 ⭐ runner             SimRunner               RealRunner
                      reset:   env            reset:   learned / manual
                      success: predicate      success: VLM classifier
                      safe:    always         safe:    limits + motors
                      time:    stepped        time:    wall-clock
                                      │
                                      ▼
 backends             MuJoCo · Isaac Lab-Arena · Genie Sim · real hardware
```

**We adopt XPolicyLab's protocol instead of inventing one.** It already serves 40+ policies with per-model dependency isolation. Reinventing that would be exactly the mistake this project argues against. What nobody has built — including XPolicyLab itself, whose companion benchmark keeps the simulation client separate — is the execution layer.

Sim and real differ in precisely three ways: where reset comes from, where success comes from, and how time flows. crossrun names them rather than abstracting them away. That is the whole trick.

## Why now

- **Policy serving has converged.** Client–server inference over a socket, one process per model, is now how everyone does it.
- **The physics layer is consolidating.** Isaac Lab is moving to Newton, whose primary backend is MuJoCo-Warp. Accuracy and throughput are no longer opposed.
- **Evaluation methodology has caught up.** Perturbation tiers, confidence intervals, and real-to-sim correlation checks are established practice — just not yet standard practice.

None of this is our invention. The contribution is assembling it cleanly and being explicit about the seams.

## Planned demos

An architecture is an argument, and arguments need evidence. All three run on LIBERO + MuJoCo with public checkpoints:

1. **How much does the protocol move the number?** Same checkpoint, varying episode counts, seeds, and perturbation tiers. At 70 rollouts a 95% CI is roughly 15 points wide; official protocols use 3 seeds × 500 rollouts, community reproductions often use 10 episodes.
2. **Rankings disagree across backends.** Same policies, different order on MuJoCo versus Isaac. If this *doesn't* happen, that is worth knowing too.
3. **90% → 0%.** A model scoring above 90% under the standard protocol, collapsing under position perturbation.

## Scope

Small on purpose: **LIBERO / ALOHA on MuJoCo, one VLA and one world-action model.** Enough to demonstrate every design claim.

Extensibility gets proven afterwards, not assumed: **Unitree G1** with whole-body control on Isaac Lab-Arena, and **AgiBot G2** through Genie Sim assets.

## Docs

- [`docs/plan.md`](docs/plan.md) — full design: architecture, evaluation protocol, phases, risks, and the alternatives we rejected
- [`docs/research/`](docs/research/) — the surveys behind it, including the parts that turned out wrong

## License

MIT
