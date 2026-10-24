# kepler-64-compute

Compute runs for [Kepler-64](https://github.com/r-baruah/kepler-64). This repository holds a copy of the engine
(`engine/`, synced from the main repository; `ENGINE_COMMIT` names the exact commit) and GitHub Actions workflows
that split training and testing into independent work units.

- `tdleaf-round`: one self-play training round (TDLeaf) split across up to 20 jobs, reduced into one update, committed
  under `runs/<name>/`; it starts the next round itself until the requested number of rounds is done.
  Create `runs/<name>/STOP` to stop after the current round.
- `evolve-round`: one iteration of cosmological natural selection (SPSA with cooling): twin universes with perturbed
  constants play each other across up to 20 jobs, and the winner pulls the constants its way. It chains itself like
  `tdleaf-round` and stops on `runs/<name>/STOP`.
- `sprt`: a batch sequential-probability test between two generations, committed under `results/`.

Everything here is depth-limited and seeded, so every result can be reproduced from the files in this repository.
The method and the rules for what counts as a result are in the main repository under `docs/v1/`.

**How to run things:** see [RUNBOOK.md](RUNBOOK.md).
- `determinism-check`: reruns fixed reference units and compares their result hashes with the laptop's. It runs on every engine sync and inside every campaign.
