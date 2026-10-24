# Runbook: running Kepler-64 compute on GitHub Actions

This repository runs Kepler-64's heavy jobs (training generations, acceptance tests) on GitHub's free runners for
public repositories: up to 20 jobs at once, 4 vCPU each, so about 80 vCPU. Nothing runs by itself. Every campaign is
started by hand.

## 0. Before any campaign
1. Sync the engine from the main repository (on the laptop):
   ```bash
   ~/Desktop/Kepler-64/scripts/compute/sync-compute-repo.sh ~/Desktop/kepler-64-compute
   cd ~/Desktop/kepler-64-compute && git add -A && git commit -m "sync engine $(cut -c1-7 ENGINE_COMMIT)" && git push
   ```
2. That push automatically starts **determinism-check**. Wait for it to go green (about 2–3 minutes). Green means
   GitHub's machines reproduce the laptop's results bit for bit, so the results of any run can be merged with local ones.
   Red means stop: something differs (compiler, data files, a bug), and no campaign should run until it's fixed.
   Every campaign workflow also runs the same check before it plays a single game.

## 1. Start a training run (TDLeaf)
- **Web:** Actions → `tdleaf-round` → **Run workflow**, then fill in:
  - `run`: a new folder name, e.g. `gen001-tdleaf-a`. Never reuse a name, unless you mean to continue that run.
  - `rounds`: how many rounds in total (e.g. `20`).
  - `shards`: `20` (the maximum parallel jobs).
  - `games`: games per round, split across shards (`2000`).
  - `depth`: `4`.
  - `lr`: `0.01`.
  - `seed`: any number; keep a record of it.
  - `gen`: the starting generation, usually `engine/data/generations/gen-000.json`.
- **Command line:**
  ```bash
  gh workflow run tdleaf-round.yml -R r-baruah/kepler-64-compute \
    -f run=gen001-tdleaf-a -f rounds=20 -f shards=20 -f games=2000 -f depth=4 -f lr=0.01 -f seed=1
  ```
- One workflow run is one round: build → determinism check → 20 shards → combine → commit to `runs/<run>/` → start
  the next round. Expected: about 4–6 minutes per round, so a 20-round run takes about 1.5–2 hours, unattended.

## 2. Start a natural-selection run (evolve, SPSA)
```bash
gh workflow run evolve-round.yml -R r-baruah/kepler-64-compute \
  -f run=gen001-evolve-a -f rounds=150 -f shards=20 -f games=400 -f depth=4 -f seed=1
```
- Here `rounds` means iterations and `games` means opening pairs per iteration.
- Each iteration plays twin universes against each other and moves the constants toward the winner.
- Natural selection needs many small steps: plan on 100–200 iterations, about 4–5 minutes each (several hours,
  unattended).

## 3. Acceptance test (SPRT) of a candidate
```bash
gh workflow run sprt.yml -R r-baruah/kepler-64-compute \
  -f name=gen001-tdleaf-a-vs-gen000 -f a=runs/gen001-tdleaf-a/candidate.json \
  -f b=engine/data/generations/gen-000.json -f shards=20 -f pairs=100 -f depth=4 -f seed=1
```
This is 20 × 100 pairs = 4,000 games, in one batch. The result lands in `results/<name>.json`: `H1` = the candidate is
stronger (accept it as the new generation), `H0` = it isn't, `inconclusive` = play more.

## 4. Watch, stop, resume
- **Watch:** the Actions tab, or `gh run list -R r-baruah/kepler-64-compute`. Each round's numbers are committed to
  `runs/<run>/round-NNN.json` as soon as the round finishes.
- **Stop after the current round:** add an empty file `runs/<run>/STOP` (web: Add file → Create new file), or:
  ```bash
  cd ~/Desktop/kepler-64-compute && git pull && touch runs/<run>/STOP && git add runs && git commit -m stop && git push
  ```
- **Stop immediately:** cancel the run in the Actions tab. The last completed round is safe in `runs/<run>/`.
- **Resume:** delete `STOP` and start the same workflow with the **same `run` name**. It continues from the last
  checkpoint.

## 5. Bring results home
Results are only "real" once they're committed in the main repository (its rule: every public number links to a file in
`engine/results/`).
```bash
cd ~/Desktop/kepler-64-compute && git pull
cp runs/<run>/candidate.json  ~/Desktop/Kepler-64/engine/results/<run>-candidate.json
cp runs/<run>/round-*.json    ~/Desktop/Kepler-64/engine/results/<run>/      # trajectory
cp results/<name>.json        ~/Desktop/Kepler-64/engine/results/
```
A candidate becomes a generation (`engine/data/generations/gen-001.json`) only after its SPRT says `H1`. Then add a
finding to `docs/v1/09_FINDINGS.md`.

## 6. Rules
- **Only start a campaign on purpose:** a generation, a ledger row, or an acceptance test. GitHub's terms ask that
  Actions serve the software in the repository, and this workload tests and tunes this engine.
- Don't run more than one 20-shard campaign at a time; they would just queue behind each other.
- Never edit files under `runs/` by hand, except `STOP`. The combine step verifies hashes and will refuse tampered
  shards.
