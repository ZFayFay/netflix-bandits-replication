# I rebuilt Netflix's billboard bandit. The metric picked the winner.

*Netflix shows one big title at the top of your homepage. They pick it with a bandit. I rebuilt their setup on MovieLens, then tested two things their talk leaves open. Code on [GitHub](https://github.com/ZFayFay/netflix-bandits-replication).*

---

## The setup

The billboard is one slot. Netflix fills it with an ε-greedy bandit. Most of the time it shows the best-scoring title. Some of the time it shows a random one from a candidate pool. The random part does double duty: it explores, and it gives clean training data.

Their talk compares two ways to pick the "best" title:

- **Greedy:** the title you're most likely to play.
- **Incrementality:** the title whose play probability goes up the most *because* it's on the billboard.

Those sound similar. They're not. You were going to watch the new blockbuster anyway. Putting it on the billboard mostly takes credit for a play that would have happened.

## What's real and what's simulated

Real: 890k MovieLens ratings. From them I build a taste vector for each member and each title, plus title popularity.

Simulated: the plays. No public dataset logs billboard impressions. So I wrote a small simulator with numbered assumptions. One of them matters a lot: lesser-known titles gain more from being shown. I built that in on purpose, because it's what Netflix's scatterplot suggests. So I can show the method *recovers* the pattern from logs. I can't claim the pattern is true.

The upside of a simulator: I know the true value of every policy. That's what lets me grade the offline estimators later.

## Result 1: the metric picks the winner

Over 5 seeds:

- **Greedy** wins take rate: 28.1% vs 22.9%.
- **Incrementality** wins incremental plays: 0.158 vs 0.123 per visit. That's +28%.
- Greedy sends 65% of billboards to the 5 most popular titles. Incrementality sends 26%.

Netflix's replay evaluation said greedy looked slightly better offline. Mine says the same, on take rate. But take rate counts plays that would have happened anyway. Change the metric to incremental plays and the answer flips.

This is the same trap as last-touch attribution in marketing. If you measure who was there when the conversion happened, you reward the channel that shows up for people who were converting anyway.

## Result 2: tuning beat the algorithm

The talk names Thompson sampling and UCB as future work. So I ran ε-greedy, LinUCB and Thompson sampling online, on 20,000 visits each.

With textbook defaults, Thompson sampling came out *worst*. That would make a nice contrarian headline. It would also be wrong. Each method has one knob. When I tuned each knob on separate seeds, all three landed within one standard deviation of each other: 769, 808 and 823 lost plays. All three cut regret more than 4× vs random.

So the lesson here isn't "use Thompson". It's this: don't trust a bandit comparison that doesn't say how each method was tuned.

## Result 3: replay needs uniform logs

Replay is simple. Take your logs. Keep the rows where the new policy would have shown the same title. Average the plays.

It works when the logs are uniform, like Netflix's exploration slice. But most real logs come from an ε-greedy system, where the greedy title shows up far more often. Then replay over-counts members where the new policy happens to agree with the old one.

I graded five estimators against the known truth, over 200 logs of 5,000 visits:

- **Replay:** +14% too high. And tight, so it looks precise while being wrong.
- **IPS / SNIPS:** unbiased, but ±20% noise.
- **Direct method:** perfect with a good reward model, −52% with a crude one.
- **Doubly robust:** unbiased with *either* model.

Doubly robust is the safe default when your logs aren't uniform. It didn't cut the noise much here, though. A play is a 0/1 coin flip, and no model removes that.

## What I'd take away

1. **Decide the metric before you compare policies.** Take rate and incremental plays pick different winners.
2. **Tune before you compare algorithms.**
3. **Replay only on uniform logs.** Otherwise use doubly robust, and log your propensities.

---

*Notebook and code: [github.com/ZFayFay/netflix-bandits-replication](https://github.com/ZFayFay/netflix-bandits-replication). The framework belongs to the Netflix team. This is an independent educational rebuild of their [talk](https://aicouncil.com/talks/a-multi-armed-bandit-framework-for-recommendations-at-netflix).*
