# I rebuilt Netflix's billboard bandit. The metric picked the winner.

## Greedy vs incremental, three bandit algorithms, and an offline test that lies on the wrong logs

*Part 2 of **Netflix Tech Blog, rebuilt**. [Part 1: artwork personalization](https://medium.com/@faycal.zouine.usa/i-rebuilt-netflixs-artwork-personalization-system-on-movielens-here-s-what-actually-matters-d0540b7c43eb) · [Part 3: what recommendations are worth](https://medium.com/@faycal.zouine.usa/your-dashboard-says-recommendations-multiply-viewing-by-7-5-the-truth-is-3-1-a7b68e8334c3)*

*5-minute read. In a hurry? Read the bold lines.*

Open Netflix and the top of the screen is one big title. The billboard. One slot, millions of members, and someone has to decide what goes there.

Netflix fills it with a bandit (an algorithm that mostly shows its best guess and sometimes tries something random to keep learning). Their [talk](https://aicouncil.com/talks/a-multi-armed-bandit-framework-for-recommendations-at-netflix) compares two ways to define "best":

- **Greedy:** the title you're most likely to play.
- **Incrementality:** the title whose chance of being played goes up the most *because* it's on the billboard.

Sounds like the same thing. It isn't.

You were going to watch the new blockbuster anyway. Putting it on the billboard just takes credit for it.

> Being there when it happened is not the same as making it happen.

I own an offer bandit at work, so this one is close to home. I rebuilt Netflix's setup on MovieLens and pushed on two things their talk leaves open.

## What's real, what's simulated

Real: 890k MovieLens ratings, turned into a taste profile for every member and title, plus title popularity.

Simulated: the plays. No public dataset logs billboard impressions. So I wrote a small simulator with every assumption numbered. One matters a lot: lesser-known titles gain more from being shown. I built that in on purpose, because Netflix's data suggests it. **So I can show the method finds that pattern. I can't claim the pattern is true.**

The upside of a simulator: I know the true answer for every policy. That's what lets me grade the methods later.

## Result 1: the metric picks the winner

Over 5 runs:

- Greedy wins **take rate** (share of billboards that get played): 28.1% vs 22.9%.
- Incrementality wins **extra plays** (plays that only happened because of the billboard): 0.158 vs 0.123 per visit. That's +28%.
- Greedy sends 65% of billboards to the 5 most popular titles. Incrementality sends 26%.

![Greedy vs incrementality](https://raw.githubusercontent.com/ZFayFay/netflix-bandits-replication/main/figures/fig1_exploit_vs_incr.png)

Netflix's offline test said greedy looked slightly better. Mine says the same, on take rate. Switch the metric to extra plays and the answer flips.

Marketers have seen this movie. It's called last-touch attribution: credit goes to whatever was there at the end, even if the sale was already happening.

**Pick the metric before you compare policies. Take rate and extra plays crown different winners.**

## Result 2: tuning beat the algorithm

The talk lists Thompson sampling and UCB as future work. So I ran three bandits head to head for 20,000 visits: ε-greedy, LinUCB and Thompson sampling.

With textbook settings, Thompson sampling came out *worst*. Great contrarian headline. Also wrong.

Each method has one dial that controls how much it explores. When I tuned each dial fairly, on separate runs, all three finished within one standard deviation of each other: 769, 808 and 823 lost plays. All three beat random by more than 4×.

![Online regret](https://raw.githubusercontent.com/ZFayFay/netflix-bandits-replication/main/figures/fig3_online_regret.png)

**The lesson isn't "use Thompson." It's: never trust a bandit comparison that doesn't say how each one was tuned.**

## Result 3: Netflix's offline test needs random logs

Netflix grades new policies offline with replay. It's simple. Take the logs. Keep the rows where the new policy would have picked the same title. Average the plays.

That's fair when the logs are random, like Netflix's exploration slice. But most real logs come from a bandit that mostly shows its favorite. Then replay over-counts the members where the new policy happens to agree with the old one.

I graded five methods against the true answer, over 200 sets of logs:

- **Replay:** 14% too high. And very consistent, so it looks precise while being wrong.
- **IPS / SNIPS** (weight each result by 1 ÷ the odds it was shown): right on average, but ±20% noise.
- **Direct method** (just ask a model): perfect with a good model, 52% too low with a crude one.
- **Doubly robust** (direct method plus an IPS correction): right on average with *either* model.

![Offline estimators](https://raw.githubusercontent.com/ZFayFay/netflix-bandits-replication/main/figures/fig4_ope.png)

Doubly robust didn't cut the noise much here, though. A play is a yes-or-no coin flip, and no model removes that.

**Replay only on random logs. Otherwise use doubly robust, and always log the odds.**

## What to remember

1. **Decide what "best" means first.** Take rate and extra plays pick different winners.
2. **Tune before you compare.**
3. **Know which offline test fits your logs.**

---

*Notebook and code: [github.com/ZFayFay/netflix-bandits-replication](https://github.com/ZFayFay/netflix-bandits-replication). The framework belongs to the Netflix team. This is an independent rebuild of their [talk](https://aicouncil.com/talks/a-multi-armed-bandit-framework-for-recommendations-at-netflix).*
