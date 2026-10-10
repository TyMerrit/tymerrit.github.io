---
layout: post
title: "The experimentation hierarchy of needs"
date: 2026-10-10
excerpt: "Most teams don't need CUPED or anytime-valid inference. Here's what they need first — and when the advanced stuff actually pays off."
tags: [experimentation, ab-testing, methodology]
image: /images/posts/experimentation-hierarchy-of-needs.png
---

Let's be honest: most teams running A/B tests don't need CUPED, ML-based variance reduction or anytime-valid inference. Which is a bit awkward, because that's exactly what I want this blog to be about. Well, almost: one of those three turns out to be most useful for exactly the teams that seem to need it least. We'll get there.

Here's the real problem. Most teams' experiments don't fall apart because they need to cut test duration in half, or anything like that. Nope. They fall apart on the basics. On infrastructure: randomization, SRM checks. On design: the analyst doesn't realize an A/B test simply doesn't work for this particular case. On metrics: we measure something we don't actually care about, and end up without an answer to the question we were asking in the first place. And on decisions: the numbers come in, and nobody agreed beforehand what they'd mean.

None of these fancy methods are wrong. You can absolutely use them. The catch is that using them *correctly* takes a level of control that most teams running A/B tests simply don't have.

And the saddest part: in my own work I've seen the "but this is how Netflix does it" syndrome over and over. Or Google, or Amazon, or whoever's cool this year. Let's do fancy stuff in our A/B tests, because that's what the cool companies do. Except most analysts don't really know what it would actually take to implement the ideas they're so excited about. And to be clear: it works at Netflix because they closed the lower levels years ago. That part rarely makes it into the conference talks.

I once worked with a team running offline A/B tests with a really sophisticated randomization scheme: rerandomization, backed by a whole pile of papers. On paper it looked great. The problem was that they analyzed the results as if it were a plain randomized test, with the standard variance formula. That formula ignores the balancing step, so it overstates the variance. They paid for all that complexity to get more precise tests, then threw the precision away at the analysis stage, and kept wondering why nothing ever came out significant.

Personally, I think of experimentation as a hierarchy of requirements: randomization and data quality, design, metrics, decisions, methods. Roughly speaking, if you can't close one level, the next one won't give you honest answers. The order isn't perfectly strict, and you'll see one exception below, but it's close. And here's the thing: which of the first four levels you're stuck on has nothing to do with how big your company is. It barely depends on how many analysts you have or how much they know. It depends entirely on how mature your methodology is, and how much effort you've put into getting things right. A team running five tests a year can sit higher than one running five thousand.

![The experimentation hierarchy of needs: a five-level pyramid — I · Randomization & Data, II · Design, III · Metrics, IV · Decisions, V · Methods — annotated "most teams are stuck down here" near the bottom, "conference talks start here" at the top, and a crack in the base: "one crack down here and nothing above it holds".](/images/posts/experimentation-hierarchy-of-needs.png){: .post-hero }

## Level 1: Randomization & data quality

The most basic thing you need before you can run a correct A/B test at all is randomization quality. You planned a 60/40 split, and your A/B platform actually gives you 60/40. You don't run into sample ratio mismatch, or if you do, you have checks that catch it. You don't see any other weirdness in assignment either, like everyone on one particular browser version suddenly deciding to land in the treatment group. Until you can guarantee this, nothing else you might bolt onto your A/B tests will work. Randomization is the foundation.

What else goes into this level? Assignment bugs, for one: the same user ending up in both treatment and control within a single experiment because something weird happened. Lost exposure events and lost logs in general, where we don't actually know how the user behaved, or whether they were ever exposed to the experiment at all.

One team ran an A/B test and got a massive SRM: the treatment group was 20% smaller than control. Their first idea? Randomly trim 20% off the control group so the sizes match. When I pushed back, they switched to plan B: run an observational causal inference method on the same data instead.

Both fixes miss the point. SRM means users went missing from treatment for some reason, and that reason almost certainly isn't random. Trimming control doesn't make the groups comparable again; it just hides the evidence. And no observational method can adjust for a selection mechanism you haven't even identified. The only real fix is to find out where those 20% went.

The fix here is boring: checks. For something like SRM, a basic Pearson chi-square test does the job, with a strict threshold like p < 0.001, since you run it on every test. Lost exposures come in two kinds. If both groups lose data at the same rate, you lose power. If one loses more than the other, you get bias, and it usually shows up as SRM. So the SRM check is your main detector here too, and pipeline monitoring covers the rest, the same way any decent DWH platform checks that nothing goes missing. And assignment bugs? Same thing, from the logs: check whether the same person showed up in different variants of the experiment at different points in time.

## Level 2: Design

The next place a lot of teams fail is design. That covers how long the test runs and what minimum effect it can detect, both of which you calculate *before* launch, not after. But the question I see skipped most often comes even earlier: can you randomize here at all, and at what level?

Take network effects. On a job search platform, a marketplace or a social network, users interact with each other, so treating one user changes outcomes for others. Randomize by user and your estimate is biased. Sometimes the fix is to randomize at a higher level: clusters, regions, or time slots with switchback designs. Sometimes even that isn't possible, and you need observational methods: diff-in-diff, synthetic control, CausalImpact.

This one isn't from my own work. It happened at a large social network, which I won't name. They launched stickers, big animated emojis, and decided a classic user-level A/B test was fine. Then a user in treatment, who had stickers, sent one to a friend in control, who didn't. Control was no longer control: it was being exposed to the feature through its friends. And the treatment effect depended on whether your friends happened to land in treatment too. A textbook network effect.

Or take offline retail. You're testing a new fridge that costs $100k per store. You'll get a handful of stores, not thousands, so power is a problem from day one. And usually it's not a coin flip that picks the stores. It's someone in operations. That's selection, not randomization, and no amount of t-testing fixes it. If you can get operations to hand you a shortlist and let a coin pick from it, this is exactly where rerandomization or matched pairs earn their keep, as long as the analysis knows about it. Remember the team from the intro.

None of this is fixable after launch, so all you can really do is pin things down beforehand: the duration, the MDE, the unit of randomization. And write it all down in a design doc. The first question here shouldn't be "how long will the test run?" It should be "how can we run it here at all, and if we can, what unit of randomization do we actually have?"

## Level 3: Metrics

Next problem area: metrics. There are two issues here. First, how do you choose metrics that answer the question you actually care about? Second, how do you compute them? The first one is much harder than the second. To run tests that actually get you where you're supposed to go, you need a good target metric (what Kohavi et al. (2020) call the OEC, the overall evaluation criterion). You need a set of guardrail metrics you must not break, no matter what. And you need a pool of secondary metrics, so the test doesn't just tell you whether to ship. It also teaches you something you can use to come up with the next feature that moves user metrics even further.

The uncomfortable truth is that this problem is mostly organizational. An analyst on their own can rarely figure out what the business really can't afford to break in a given case, or what we should actually be aiming for. So what matters here is a working relationship between analytics and product teams, where everyone is working toward what the business actually needs. Not toward pretty math, and not toward tests that come out significant.

Now, sometimes you pick the right target and still compute it wrong. This will sound obvious to experimentation specialists, but here goes. When you randomize by user and compute results per session, per order or per click, those observations aren't independent. Treat them as independent and you seriously inflate your type I error. You could call it pseudoreplication, really. The fix is the delta method, linearization plus a t-test, or a bootstrap clustered on the randomization unit. It's a couple of lines of code. It's not an advanced technique, and plenty of people know about it. But it's what gets you an honest standard error.

The remedy for both problems is the same as for design: write it down before launch. The target metric, the guardrails, the secondary metrics: all of it goes into the design doc before the test starts, not after you've seen which numbers turned green. And for every ratio metric, the variance gets computed at the randomization unit. No exceptions.

## Level 4: Decisions

The next level is whether a result actually turns into the right action. It looks like an org problem, but it has statistical roots.

The saddest version I've seen was back when I was an intern analyst. A team ran a test with three variants. The split was supposed to be a third each. In reality it came out more like 30/30/40. And here's the ironic part: they made a real product decision based on that experiment, completely seriously. I told them the results couldn't be trusted. Naturally, nobody listens to the intern.

That one's extreme. The subtler failures look like this.

The first one: no decision rule. Target metric up 1%, one guardrail down 0.5%. Ship or not? If the answer gets negotiated from scratch every time, the test is just decoration.

The second: pretending all errors cost the same. A false positive on checkout and a false positive on a button color are not the same thing. α = 0.05 for everything is a default, not a decision.

The third is the winner's curse. Once a test hits significance, the measured effect is biased upward, and the lower the power, the worse it gets. Add up every uplift you reported over a year, and the total will be way bigger than what actually showed up in the business. Nobody's lying. That's just selection. Holdouts make the gap visible; shrinkage corrects for it.

So: write the ship criteria down before the test starts (yes, in the same design doc), and every now and then compare the sum of your claimed wins with what long-running holdouts show.

## Level 5: Methods

And now we get to the part I actually want this blog to be about. It sits at the top for a reason: without a decision rule, a better method just gives you a more precise number to argue about. This is the stuff that keeps big tech researchers busy. Advanced methods that solve fairly narrow problems. Or not that narrow, but a lot less common. Heavy variance reduction, so you can run more tests on the same traffic or run the same tests much faster. Methods that systematically deal with peeking, or even turn it into an advantage for your platform. For example, using anytime-valid inference to monitor guardrail metrics, so if something is getting worse, you don't have to wait for the whole test to finish.

So the real question is when all this complicated methodology actually pays off. The threshold is simple: your experimentation platform's throughput is the bottleneck. When you have more hypotheses than you can test. When effects are small relative to metric noise. And that's exactly why these problems usually don't matter for startups and small, low-traffic companies, which, let's be honest, is where most analysts work. Low traffic sounds like exactly the case for variance reduction, but the effects worth chasing there are usually big, and the pre-period history that CUPED-style methods feed on is often missing anyway.

Why shouldn't teams without solid experimentation infrastructure use these methods? Because all of them amplify errors from the lower levels. Variance reduction on a test with SRM gives you a precise wrong answer. Covariates that are affected by the treatment introduce bias. An ML model trained on the same experiment's outcomes without cross-fitting absorbs part of the effect and breaks the standard errors. So the control these methods require is very concrete: frozen covariates, cross-fitting, stopping rules enforced by the platform rather than by "analytics culture."

The one exception worth calling out is sequential testing. Almost everyone peeks, especially when they're checking that nothing broke mid-test. But with five equally spaced peeks, a nominal 5% type I error rate turns into roughly 14% (Armitage et al., 1969). Check the dashboard daily over a two-week test, and you're past 20%. So something like anytime-valid inference is actually *more* useful for teams without iron discipline: it makes peeking safe instead of trying to ban it.

Group sequential designs get you part of the way too. With an alpha-spending function you don't even have to fix the number of looks in advance, only the maximum sample size, as long as you don't time the looks by watching the data. With O'Brien–Fleming boundaries that costs you just a few percent of sample size. Anytime-valid inference drops the horizon altogether, and that's where the real price is: even tuned to your planned horizon, a normal-mixture mSPRT needs z ≈ 3 instead of 1.96. Intervals end up about 1.5× wider, and you can only pick up fairly large effects. That's another reason I think this fits small companies well: a young product still has big levers to pull, so the true effects worth testing tend to be large. Careful with the evidence, though: "our significant tests showed huge effects" is exactly the winner's curse from level 4.

One setup I think works well: the primary metric runs on a fixed horizon, and a sequential method sits on top as a guardrail layer for early harm detection. The asymmetry is the point. If a guardrail is degrading, every extra day costs you users or money, and stopping early for harm is cheap even when you're wrong: the worst case is a delayed launch. Stopping early for a *win* is a different story. Early stops select for lucky fluctuations, so the effect you report is inflated, and you pay for the sequential method's wider intervals on the one metric where you need power most. Keep the primary metric on a fixed horizon, let the guardrails watch continuously, and you get both: an honest estimate of the win and a fast exit from a loss. Without that split, teams end up either peeking at the primary metric anyway or sitting on a broken test for two weeks because "it's not done yet." One caveat: "cheap" is per alarm. With ten independent guardrails at α = 0.05 each, about 40% of perfectly harmless tests will trip at least one, so the guardrail α has to account for how many guardrails you watch.

What to do here: measure your own numbers first. If you want variance reduction, compute the correlation between your covariate and the outcome. Figure out your typical MDE. Look at how often people actually peek. Then apply the known methods in a targeted way, to the problems you really have.

## Where to start

Find the lowest level that's broken and fix that first. In practice:

1. Run a chi-square SRM check (p < 0.001) on your last 20 tests, plus one live A/A test through your platform. If either fails, you're on level 1. Stop there.
2. Look at your last five tests. Did each one have a design doc with a unit of randomization, a duration and an MDE? Did anyone ask whether randomization even made sense? If not, level 2.
3. Replay a few hundred A/A splits on historical data through your analysis pipeline. The share of significant results at α = 0.05 should stay within binomial noise (roughly 2.5–7.5% for 300 replays), and the p-values should look uniform. If they don't, check which unit your variance is computed at. That's level 3.
4. Look at your last five ship decisions. Were the ship criteria written down before launch, or negotiated after the numbers came in? If the latter, level 4.
5. Only then ask whether throughput is your bottleneck. If it is, congrats: the fancy stuff just became the cheapest way to run more experiments on the same traffic.

Advanced methods aren't a maturity badge. They sit at the top because they only pay off once everything below holds, and only if throughput is your actual bottleneck. Most teams hit four other problems first.

---

*This post is the entry point to the blog. Posts that go deeper into each level are linked from the corresponding section as they're published.*

### References

- Armitage, P., McPherson, C. K., & Rowe, B. C. (1969). Repeated significance tests on accumulating data. *Journal of the Royal Statistical Society, Series A*, 132(2), 235–244.
- Kohavi, R., Tang, D., & Xu, Y. (2020). *Trustworthy Online Controlled Experiments: A Practical Guide to A/B Testing*. Cambridge University Press.
