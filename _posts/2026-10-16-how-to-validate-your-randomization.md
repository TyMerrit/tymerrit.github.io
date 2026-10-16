---
layout: post
title: "How to validate your randomization (and your A/B platform)"
date: 2026-10-16
excerpt: "Data can pass every schema check and still be unfit for inference. Three data layers, the checks for each, why SRM can lose the race, and why your SRM check is peeking too."
tags: [experimentation, ab-testing, data-quality, srm]
image: /images/posts/how-to-validate-your-randomization.png
thumbnail: /images/posts/how-to-validate-your-randomization-thumb.png
---

A team runs an A/A test, gets p = 0.4, and declares the platform validated. I've seen this more times than I can count, and it proves almost nothing. A single A/A p-value is one draw from a distribution. Even a badly miscalibrated platform hands you p = 0.4 most of the time. You can't check calibration with one draw, and you can't check whether your pipeline is quietly eating events.

This post is about level 1 of the [hierarchy]({{ '/blog/experimentation-hierarchy-of-needs/' | relative_url }}): randomization and data quality. I spent my master's thesis on this problem, and the main thing I took away is that "the data is fine" means three different things, and teams usually check only the first two.

Data can be syntactically valid: right schema, right types, no missing fields. It can be semantically valid: no negative prices, no events from the future. And it can still be useless for inference. Picture purchase events that are perfectly well-formed but get dropped slightly more often in one variant because of a logging bug on one client. Every row passes every check, and the comparison is broken anyway. The property you actually care about is whether the data is *fit for inference*, and no schema validator checks that.

Everything below comes with code: a step-by-step validation pipeline you can point at your own experiment tables, plus a notebook that reproduces every number in this post. It's on GitHub: [ab-data-quality](https://github.com/tymerrit/ab-data-quality).

## Three layers, three sources of truth

Every experiment produces three streams of data: **assignment** (who was put in which variant), **exposure** (who actually saw it), and **outcome** (what they did). Each stream is its own source of truth, and a defect in one can be invisible in the others. Your assignment log can be perfectly balanced while your exposed population is skewed. Your exposures can be complete while half your outcomes haven't arrived yet.

Here's what breaks at each layer and how to catch it:

| Layer | What breaks | Check | If it fails |
|---|---|---|---|
| Assignment | misconfigured splits, hash bugs, users flipping between variants | SRM on the assignment log; units seen in more than one variant | untrustworthy |
| Exposure | events lost in one variant or one segment more than another | exposure/assignment ratio per variant, overall and within critical segments; SRM on exposed units | untrustworthy |
| Outcome | late events (immaturity) or events that never arrive (loss) | maturity against your pipeline's known lag; completeness against an independent source like the orders database | preliminary if still arriving, untrustworthy if lost |
| Identity | one user split across devices, IDs that don't join | IDs-per-user by variant; IDs shared across variants | untrustworthy |

On top of that, pre-period covariate balance catches a split that is balanced in counts but not in composition: say, heavy users leaking into treatment while the same number of light users go to control. SRM can't see that by construction.

One thing worth saying explicitly: **SRM is a symptom, not a cause.** The same defect can show up as different symptoms, and the same symptom can come from different defects. An SRM tells you something is wrong. It doesn't tell you what, and "fixing" it without finding the cause just hides the evidence.

## SRM is a race you can lose

The most common quiet failure is asymmetric exposure loss. Assignment is perfect. But in treatment, some exposure events never get logged, and the analysis runs on logged units, as most dashboards do. If the lost users are different from the rest, the composition shift shows up as a "treatment effect."

The obvious defense is SRM: lose 1% of treatment exposures and the exposed split drifts away from 50/50. So I simulated the race. 1% of treatment exposures go missing, the treatment does nothing, and two tests run side by side as the experiment grows: the SRM check (p < 0.001) and the A/B test itself (p < 0.05). Every significant A/B result is a fake effect caused by the loss.

![1% of treatment exposures lost: who notices first? Probability curves against logged units per arm on a log scale. The A/B test shows a fake effect at around 10k users per arm when the lost users are top 2% spenders, around 100k when they are top 20% spenders, and stays at 5% when loss is random; the SRM check only catches the loss at around 200–300k users per arm.](/images/posts/exposure-loss-race.png)
*1% of treatment exposures lost. Black: probability the SRM check catches it. Dashed: probability the A/B test shows a significant fake effect, for three loss patterns. Revenue per user ~ LogNormal(0, 1); lines are the normal approximation, dots are Monte Carlo.*

If the loss is random, nothing bad happens: the estimate stays unbiased, and SRM eventually catches it at around 200-300k users per arm. If the loss comes from your top 2% of spenders, treatment's average revenue drops by about 6%, purely because its best customers vanished from the log. At 10k users per arm, the A/B test reports that drop as significant in 96% of runs, while SRM catches the loss in 1%.

What decides the race is not sample size. With $m$ units per arm and loss rate $L$, the SRM z-score is about $L\sqrt{m/2}$, and the fake effect's z-score is about $d \cdot L\sqrt{m/2}$, where $d$ is how far the lost users sit from the average, in standard deviations. Both grow like $\sqrt{m}$, so more traffic doesn't help SRM catch up: the ratio is just $d$. The A/B test crosses 1.96 when SRM is only at $1.96/d$, while SRM needs 3.29 to fire. So once $\lvert d \rvert$ is above about 0.6, you're more likely to see a fake effect than an SRM alarm. On heavy-tailed metrics like revenue or engagement, selective loss gets there easily.

Two practical consequences. First, **check the exposure layer directly instead of waiting for SRM.** In the pipeline's own demo, 5% of treatment exposures on iOS go missing: SRM on exposed units comes back at p = 0.007, which passes at the 0.001 threshold, while the exposure-rate check per variant and per platform flags it immediately. Loss concentrated in one segment is much easier to see inside that segment than in the overall split. Second, **where you can, analyze by assignment rather than by logged exposure.** Exposure logging bugs can't bias a comparison that doesn't use the exposure log.

## Your SRM check is peeking too

Most platforms recompute SRM every day. That's a repeated significance test, and everything we know about peeking applies to it.

![Monitoring SRM is repeated testing: false alarm and detection rates per monitoring rule. p < 0.05 daily: 21.8% false alarms, 98% detection. p < 0.001 daily: 0.7%, 73%. p < 0.001 hourly: 2.0%, 77%. Anytime-valid at α = 0.001: 0.0%, 41%. Anytime-valid with calibrated α: 0.6%, 74%. p < 0.001 at the end only: 0.1%, 67%.](/images/posts/srm-monitoring.png)
*14-day test, 10,000 users per day, 20,000 simulated tests per rule. Orange: false alarms when the split is truly 50/50. Blue: detection when it's 49.5/50.5.*

Check SRM daily at α = 0.05, and about one healthy test in five raises a false alarm (21.8%). A strict threshold fixes most of it: 14 daily looks at p < 0.001 can't exceed 1.4% false alarms by a plain union bound, and in practice give 0.7%, while still catching a half-point SRM 73% of the time.

What about anytime-valid inference, the textbook fix for peeking? At the same nominal α = 0.001 it looks weak: 41% detection. But that comparison is unfair. Ville's inequality, which gives these methods their guarantee, is conservative, and the realized false alarm rate is around 0.01%. Calibrate the anytime-valid test to the same realized false alarm rate as daily chi-square, and the gap disappears: 74% detection, and slightly earlier, because it can look every hour. The catch is that a calibrated threshold is only guaranteed for the traffic, horizon and look schedule it was calibrated on.

So the honest conclusion is boring. If you look once a day, a strict fixed threshold is simple and just as good. Anytime-valid methods earn their keep when you monitor continuously and need a guarantee that holds whenever you look. What doesn't work is the default most people start with: α = 0.05, checked every morning.

## Validating the platform itself

Everything above validates individual experiments. The platform needs its own checks, and they come in two kinds.

**Offline checks on assignment.** Push a few hundred thousand synthetic IDs through your hashing function and check three things: buckets come out uniform, assignment is deterministic, and two different experiments assign independently. Each check exists because each catches a bug the others miss. A modulo-bias bug (two hex digits mod 100) gives a 58.5/41.5 split and fails uniformity, but it's perfectly deterministic and independent. A shared-salt bug, where every experiment splits users identically, passes uniformity and determinism and only fails the independence check. Two "independent" experiments that are perfectly correlated are not independent.

**A/A replays on historical data.** Instead of one live A/A test, take historical data, draw a few hundred random splits, and push each through your actual analysis code. The share of significant results should sit within binomial noise around α, and the p-values should be uniform.

![500 A/A replays on the same data: p-value histograms. Unit-level analysis is flat, with p < 0.05 in 5% of replays; session-level analysis piles up near zero, with p < 0.05 in 42% of replays.](/images/posts/aa-replay-pvalues.png)
*500 A/A replays on the same session-level data. Left: aggregate to the user first. Right: treat every session as independent.*

Same data, two analyses. Aggregating to the randomization unit gives a 5% false positive rate. Treating sessions as independent gives 42%. A single live A/A test would most likely have passed either one.

## From signals to decisions

Checks only matter if they change what happens to the result. Three statuses are more useful than a binary pass/fail:

- **Trustworthy:** all confirmatory checks pass. Publish.
- **Preliminary:** the data is still maturing, but there's no sign of permanent damage. Wait, and don't ship on it yet.
- **Untrustworthy:** SRM, asymmetric exposure or completeness, broken identity, covariate imbalance. Block the result until the cause is found.

The line between preliminary and untrustworthy matters more than it looks. "The data isn't complete yet" and "the data will never be complete" call for opposite responses. A pipeline that treats them the same either blocks everything or blocks nothing.

One more distinction: confirmatory checks versus diagnostic ones. SRM on the main population is confirmatory, so it changes the status. SRM by browser, country or app version is diagnostic: you use it to locate a problem you already know about. If every segment-level SRM could block a result, you'd have built an automated p-hacking machine in reverse. With enough segments, something always fails.

## Validating the validator

A data-quality pipeline is itself a decision procedure, so it has error rates. I ran the full pipeline on 200 synthetic experiments per scenario, 40,000 users each:

| Scenario | Expected status | Share with expected status |
|---|---|---|
| Healthy | trustworthy | 99% |
| Split 51/49 instead of 50/50 | untrustworthy | 79% |
| 5% of treatment iOS exposures lost | untrustworthy | 86.5% |
| Slow pipeline, data pulled 6 hours after the window | preliminary | 99% |
| 1% of units in both variants | untrustworthy | 100% |
| 5% of treatment outcomes lost for good | untrustworthy | 100% |
| 10% of treatment users get an extra device id | untrustworthy | 100% |
| Heavy users leak into treatment, counts stay 50/50 | untrustworthy | 100% |

About 1% of healthy experiments get blocked. That's the price of running around fifteen confirmatory checks per experiment, even at strict thresholds. The misses are power limits, not blind spots: a one-point SRM at 40,000 users is a z-score of about 4, and the pipeline catches what the data can show.

## The checklist

If you automate nothing else, automate this:

1. Validate the platform once and after every change: hash uniformity, determinism, independence, and an A/A replay through your real analysis code.
2. SRM on every test at p < 0.001, or an anytime-valid version if you look continuously.
3. Exposure/assignment ratio per variant, overall and within the segments where logging is known to break.
4. A count of units that appear in more than one variant.
5. Outcome maturity against your pipeline's known lag, and completeness against an independent source.
6. Pre-period covariate balance.

All of it is implemented in [ab-data-quality](https://github.com/tymerrit/ab-data-quality), in that order, with a [notebook](https://github.com/tymerrit/ab-data-quality/blob/main/walkthrough.ipynb) that reproduces every figure here.

None of this is glamorous. It's also the reason the fancy methods further up the pyramid can be trusted at all.

---

### References

- Kohavi, R., Tang, D., & Xu, Y. (2020). *Trustworthy Online Controlled Experiments: A Practical Guide to A/B Testing*. Cambridge University Press.
- Fabijan, A., et al. (2019). Diagnosing Sample Ratio Mismatch in Online Controlled Experiments: A Taxonomy and Rules of Thumb for Practitioners. *KDD*.
- Lindon, M., & Malek, A. (2022). Anytime-Valid Inference for Multinomial Count Data. *NeurIPS*.
- Johari, R., Pekelis, L., & Walsh, D. (2017). Always Valid Inference: Continuous Monitoring of A/B Tests. *Operations Research*.
