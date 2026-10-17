---
layout: post
title: "How to validate your randomization (and your A/B platform)"
date: 2026-10-16
excerpt: "Data can pass every schema check and still be unfit for inference. The checks for each data stream, why SRM can lose the race, and why your SRM check is peeking too."
tags: [experimentation, ab-testing, data-quality, srm]
image: /images/posts/how-to-validate-your-randomization.png
thumbnail: /images/posts/how-to-validate-your-randomization-thumb.png
---

A team runs an A/A test, gets p = 0.4, and declares the platform validated. I've seen this more times than I can count, and it proves almost nothing. A single A/A p-value is one draw from a distribution. Even a badly miscalibrated platform returns p > 0.05 in most runs, so one p-value can't tell a calibrated platform from a broken one. Nor can it tell you whether your pipeline is quietly eating events.

This post is about level 1 of the [hierarchy]({{ '/blog/experimentation-hierarchy-of-needs/' | relative_url }}): randomization and data quality. I spent my master's thesis on this problem, and the main thing I took away is that "the data is fine" can mean several different things, and teams usually check only the easy ones.

Data can be syntactically valid: right schema, right types, no missing fields. It can be semantically valid: no negative prices, no events from the future. And it can still be useless for inference. Picture purchase events that are perfectly well-formed but get dropped slightly more often in one variant because of a logging bug on one client. Every row passes every check, and the comparison is broken anyway. The property you actually care about is whether the data is *fit for inference*, and no schema validator checks that.

Everything below comes with code: a step-by-step validation pipeline you can point at your own experiment tables, plus a notebook that reproduces every number in this post. It's on GitHub: [ab-data-quality](https://github.com/tymerrit/ab-data-quality).

![SRM is a race you can lose: as users per arm grow on a log scale, the A/B test reports a "−6% revenue!" fake effect long before the SRM check notices anything — the SRM curve stays near zero, "all good", until around 100k users per arm.](/images/posts/how-to-validate-your-randomization.png){: .post-hero }

## Three data streams and the key that joins them

Every experiment produces three streams of data: **assignment** (who was put in which variant), **exposure** (who actually saw it), and **outcome** (what they did). Each stream is its own source of truth, and a defect in one can be invisible in the others. Your assignment log can be perfectly balanced while your exposed population is skewed. Your exposures can be complete while half your outcomes haven't arrived yet. And all three are joined by an identity key, which breaks in its own ways.

Here's what breaks in each stream, plus the key, and how to catch it:

| Stream | What breaks | Check | If it fails |
|---|---|---|---|
| Assignment | misconfigured splits, hash bugs, users flipping between variants | SRM on the assignment log; units seen in more than one variant | untrustworthy |
| Exposure | events lost in one variant or one segment more than another | exposure/assignment ratio per variant, overall and within critical segments; SRM on exposed units | untrustworthy |
| Outcome | late events (immaturity) or events that never arrive (loss) | maturity against your pipeline's known lag; completeness against an independent source like the orders database | preliminary if still arriving, untrustworthy if lost |
| Identity (the join key) | one user split across devices, IDs that don't join | IDs-per-user by variant; IDs shared across variants | untrustworthy |

One caveat on the exposure check: it assumes the treatment doesn't change who reaches the exposure point. If it does (a new entry point, a faster page that more users reach), exposure rates should differ, and the check would block a healthy test. In that case log exposure at a point both arms pass through in the same way (a counterfactual trigger), and run the check there.

On top of that, pre-period covariate balance catches a split that is balanced in counts but not in composition: say, heavy users leaking into treatment while the same number of light users go to control. SRM can't see that by construction.

One thing worth saying explicitly: **SRM is a symptom, not a cause.** The same defect can show up as different symptoms, and the same symptom can come from different defects. An SRM tells you something is wrong. It doesn't tell you what, and "fixing" it without finding the cause just hides the evidence.

## SRM is a race you can lose

The most common quiet failure is asymmetric exposure loss. Assignment is perfect. But in treatment, some exposure events never get logged, and the analysis runs on logged units, as most dashboards do. If the lost users are different from the rest, the composition shift shows up as a "treatment effect."

The obvious defense is SRM: lose 1% of treatment exposures and the exposed split drifts away from 50/50. So I simulated the race. 1% of treatment exposures go missing, the treatment does nothing, and two tests run side by side as the experiment grows: the SRM check (p < 0.001) and the A/B test itself (p < 0.05). Every significant A/B result is a fake effect caused by the loss.

![1% of treatment exposures lost: who notices first? Probability curves against assigned units per arm on a log scale. The A/B test shows a fake effect at around 10k users per arm when the lost users are top 2% spenders, around 100k when they are top 20% spenders, and stays at 5% when loss is random; the SRM check only catches the loss at around 200–300k users per arm.](/images/posts/exposure-loss-race.png)
*1% of treatment exposures lost. The x-axis counts assigned users per arm, before the loss. Black: probability the SRM check catches it. Dashed: probability the A/B test shows a significant fake effect, for three loss patterns. Revenue per user ~ LogNormal(0, 1); lines are the normal approximation, dots are Monte Carlo.*

If the loss is random, nothing bad happens: the estimate stays unbiased, and SRM eventually catches it at around 200-300k users per arm. If the loss comes from your top 2% of spenders, treatment's average revenue drops by about 6%, purely because its best customers vanished from the log. At 10k users per arm, the A/B test reports that drop as significant in 96% of runs, while SRM catches the loss in under 1% (the normal approximation gives about 0.5%; the simulation saw 2 alarms in 200 runs).

What decides the race is not sample size. With $m$ assigned units per arm and loss rate $L$, the SRM z-score is about $L\sqrt{m/2}$, and the fake effect's z-score is about $d \cdot L\sqrt{m/2}$, where $d$ is how far the lost users sit from the average, in standard deviations. Both grow like $\sqrt{m}$, so more traffic doesn't help SRM catch up: the ratio is just $d$. The A/B test crosses 1.96 when SRM is only at $1.96/d$, while SRM needs 3.29 to fire. So once $\lvert d \rvert$ is above about 0.6, you're more likely to see a fake effect than an SRM alarm. On heavy-tailed metrics like revenue or engagement, selective loss gets there easily.

Two practical consequences. First, **check the exposure layer directly instead of waiting for SRM.** In the pipeline's own demo, 5% of treatment exposures on iOS go missing: SRM on exposed units comes back at p = 0.007, which passes at the 0.001 threshold, while the exposure-rate check per variant and per platform flags it immediately. Loss concentrated in one segment is much easier to see inside that segment than in the overall split. Second, **where you can, analyze by assignment rather than by logged exposure.** Exposure logging bugs can't bias a comparison that doesn't use the exposure log. That's an intent-to-treat estimate, so the effect gets diluted by users who were assigned but never exposed, and it only helps if outcome logging doesn't suffer from the same bug.

## Your SRM check is peeking too

Most platforms recompute SRM every day. That's a repeated significance test, and everything we know about peeking applies to it.

![Monitoring SRM is repeated testing: false alarm and detection rates per monitoring rule. p < 0.05 daily: 22.1% false alarms, 99% detection. p < 0.001 daily: 0.7%, 84%. p < 0.001 hourly: 2.0%, 87%. Anytime-valid at α = 0.001: 0.0%, 56%. Anytime-valid with calibrated α: 0.8%, 84%. p < 0.001 at the end only: 0.1%, 80%.](/images/posts/srm-monitoring.png)
*14-day test with ~12,200 users per day, sized so that a single check at the end has the textbook 80% power against a 49.5/50.5 split. 20,000 simulated tests per rule. Orange: false alarms when the split is truly 50/50. Blue: detection when it's 49.5/50.5.*

The rightmost pair is the reference: one chi-square check at the end of the test, 0.1% false alarms and 80% power. Check SRM daily at α = 0.05 instead, and about one healthy test in five raises a false alarm (22%). A strict threshold fixes most of it: 14 daily looks at p < 0.001 can't exceed 1.4% false alarms by a plain union bound, and give 0.74% in practice.

Don't read the 84% detection of the daily rule as a power gain, though. It's bought with seven times the false alarms of the single check. Hold false alarms equal instead: a single end-of-test check at α ≈ 0.007 has about 92% power. What daily looks buy you is speed, not power: they catch the SRM around day 8 instead of day 14, while there's still time to fix the bug.

What about anytime-valid inference, the textbook fix for peeking? I used the Dirichlet-multinomial mixture likelihood ratio, with the prior concentrated around the design (concentration 10,000, a prior spread of about half a point, matched to the deviation we're hunting). That choice matters: misjudge the spread by an order of magnitude and power drops sharply, so the numbers below describe this tuning, not the method in general. At the same nominal α = 0.001 it looks weak: 56% detection, well below the 80% reference. But that comparison is unfair. Ville's inequality, which gives these methods their guarantee, is conservative, and the realized false alarm rate is around 0.03%. Calibrate the anytime-valid test to the same realized false alarm rate as daily chi-square (0.76% versus 0.74%, a difference within simulation noise), and the gap disappears: 84% detection, and about a day earlier, because it can look every hour. The catch is that a calibrated threshold is only guaranteed for the traffic, horizon and look schedule it was calibrated on. It really does move: the same calibration at 100,000 users a day gives an equivalent nominal α of 0.012, against 0.019 at 12,200 a day.

So the honest conclusion is boring. If you look once a day, a strict fixed threshold is simple and just as good. Anytime-valid methods earn their keep when you monitor continuously and need a guarantee that holds whenever you look. What doesn't work is the default most people start with: α = 0.05, checked every morning.

## Validating the platform itself

Everything above validates individual experiments. The assignment machinery underneath needs its own checks, run once and again after every change to it.

Push a few hundred thousand synthetic IDs through your hashing function and check three things: buckets come out uniform, assignment is deterministic, and two different experiments assign independently. Each check exists because each catches a bug the others miss. A modulo-bias bug (two hex digits mod 100) gives a 58.5/41.5 split and fails uniformity, but it's perfectly deterministic and independent. A shared-salt bug, where every experiment splits users identically, passes uniformity and determinism and only fails the independence check. Two "independent" experiments that are perfectly correlated are not independent.

Then run a live A/A test through the real platform, end to end. It won't tell you much about statistical calibration (that's one draw, as we saw at the start), but it exercises the parts that offline tests can't: real clients, real logging, real joins. Run every check from this post on it. Whether your analysis code itself is calibrated is a different question, and it belongs to level 3 of the pyramid, metrics. The repo already has the tool for it, an A/A replay of the analysis code on historical data, as a bridge to that post.

## From signals to decisions

Checks only matter if they change what happens to the result. Three statuses are more useful than a binary pass/fail:

- **Trustworthy:** all confirmatory checks pass. Publish.
- **Preliminary:** the data is still maturing, but there's no sign of permanent damage. Wait, and don't ship on it yet.
- **Untrustworthy:** SRM, asymmetric exposure or completeness, broken identity, covariate imbalance. Block the result until the cause is found.

The line between preliminary and untrustworthy matters more than it looks. "The data isn't complete yet" and "the data will never be complete" call for opposite responses. A pipeline that treats them the same either blocks everything or blocks nothing.

One more distinction: confirmatory checks versus diagnostic ones. Confirmatory checks change the status: SRM on the main population, and exposure rate by variant within a short list of segments you declare in advance because logging is known to break there (in the pipeline's demo, just the platform column, with a Bonferroni correction across its levels). That's why the iOS loss below is expected to block the result. Everything else is diagnostic: SRM by browser, country or app version across every segment you have, used to locate a problem you already know about, never to block. If every segment-level check could block a result, you'd have built an automated p-hacking machine in reverse. With enough segments, something always fails.

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

About 1% of healthy experiments get blocked (2 of 200). The pipeline runs 17 confirmatory checks. Eleven are counts against fixed thresholds (duplicates, orphans, crossover units and so on) that don't fire on clean data. Six are statistical tests: SRM on assigned and on exposed units, exposure rate by variant overall and within the platform segment, IDs per user, and covariate balance over two pre-period covariates. Each is held at α = 0.001 overall, with a Bonferroni correction inside the multi-part ones. By a union bound that's at most 0.6% false blocks, and 2 of 200 is consistent with it. The misses are power limits, not blind spots: a one-point SRM at 40,000 users is a z-score of about 4, and the pipeline catches what the data can show.

## The checklist

If you automate nothing else, automate this:

1. Validate the assignment machinery once and after every change: hash uniformity, determinism, independence, plus a live A/A test through the real platform.
2. SRM on every test at p < 0.001, or an anytime-valid version if you look continuously.
3. Exposure/assignment ratio per variant, overall and within a short, pre-declared list of segments where logging is known to break.
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
