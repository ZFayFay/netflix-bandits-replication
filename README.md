# Rebuilding Netflix's billboard bandit on MovieLens

An independent, educational rebuild of Netflix's [multi-armed bandit framework for the homepage billboard](https://aicouncil.com/talks/a-multi-armed-bandit-framework-for-recommendations-at-netflix), on public data. It then goes two steps further than the talk.

The billboard shows one title per member. Netflix's talk covers:
- ε-greedy exploration, where the random slice doubles as unbiased training data
- two exploit policies: **greedy** (highest play probability) vs **incrementality** (biggest increase in play probability from being shown)
- **replay** for offline evaluation on uniform-exploration logs

## What's in the notebook

`bandits_replication.ipynb` is self-contained. It downloads MovieLens ratings from HuggingFace. It runs on CPU in about 1 minute.

1. **Taste vectors** from ~890k real MovieLens ratings (truncated SVD).
2. **Billboard simulator**: a pool of 20 titles across the popularity range. Organic plays and billboard plays come from numbered assumptions.
3. **Netflix's experiment**: greedy vs incrementality, scored with the true values and with replay.
4. **Beyond the talk, part 1**: ε-greedy, LinUCB and Thompson sampling online. Each method's knob is tuned on separate seeds first.
5. **Beyond the talk, part 2**: off-policy evaluation on non-uniform (ε-greedy) logs. Replay, IPS, SNIPS, the direct method and doubly robust are graded against the known truth, over 200 logs.

## Results (5 seeds, ±1 sd)

| policy | take rate | incremental plays / visit | billboards to 5 most popular |
|---|---|---|---|
| greedy | **0.281** | 0.123 | 65% |
| incrementality | 0.229 | **0.158** (+28%) | 26% |
| random | 0.107 | 0.072 | 25% |

- **The metric picks the winner.** Replay on take rate picks greedy, which matches Netflix's offline result. On incremental plays, incrementality wins by 28%.
- **Online bandits.** With textbook defaults, Thompson sampling was the worst learner. After fair tuning, ε-greedy (823), LinUCB (808) and Thompson (769) are within one sd of each other. Tuning mattered more than the algorithm.
- **Off-policy evaluation** (true value 0.232):

| estimator | bias on ε-greedy logs | note |
|---|---|---|
| Replay | **+0.033 (+14%)** | unbiased only on uniform logs |
| IPS / SNIPS | ~0 | noisy: ±20% at 5,000 visits |
| DM, good model | ~0 | |
| DM, crude model | **−0.120 (−52%)** | only as good as its model |
| DR, either model | ~0 | the "doubly robust" promise |

## What's real vs simulated

**Real:** MovieLens ratings, the taste vectors, title popularity.

**Simulated:** plays (Assumptions 1–4 in the notebook). Assumption 2 *builds in* "lesser-known titles gain more from exposure". So the scatterplot shows we can *recover* that pattern from logs. It does not show the pattern is true. Effect sizes are not real-world numbers.

## Run it

Open the notebook on Kaggle, Colab or locally, turn internet on and run all.

## Credit

The framework belongs to the Netflix team (Jaya Kawale et al.). This repo is an unaffiliated educational replication.
