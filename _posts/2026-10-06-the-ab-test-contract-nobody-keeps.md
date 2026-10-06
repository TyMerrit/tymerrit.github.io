---
layout: post
title: "The A/B test contract nobody keeps"
date: 2026-10-06
excerpt: "Every textbook A/B test is a contract: pick N in advance, wait, look once. And almost nobody keeps it."
image: /images/posts/ab-test-agreement.png
---

![A mock "A/B Test Agreement" contract: each clause crossed out and annotated with how it actually broke — checked on day 2, MDE set by available traffic, novelty wore off in week 3, a 50.8/49.2 sample ratio mismatch — stamped "SHIPPED" anyway.](/images/posts/ab-test-agreement.png)

Every textbook A/B test is a contract: pick N in advance, wait, look once.

And almost nobody keeps it. The dashboard refreshes hourly. Someone asks "is it significant yet?" on day two. The test gets stopped early because the numbers look good or, what is more common, look bad.

That's usually framed as a discipline problem. I think it's a design problem: the contract assumes a world most experimentation platforms don't live in.

Here is what it quietly assumes, and where each assumption breaks:

1. **One look, at a pre-set time.** Breaks the moment results are monitored. Each look is another chance for noise to cross the threshold. Check daily over two weeks, and the real false positive rate is closer to 20% than 5%.
2. **N derived from a correct MDE and variance.** Both are guesses. The MDE is often reverse-engineered from available traffic; the variance comes from last month's data. Underpowered tests don't just miss effects — the ones they do catch are overestimated.
3. **The effect is stationary.** Novelty inflates early lift, primacy deflates it, weekly cycles shift the mix. A two-week test measures a two-week effect, which then gets reported as the long-term one.
4. **Clean randomization and logging.** Production disagrees: sample ratio mismatch from bots or redirects, users seeing both variants across devices, exposure logged differently per arm. The math holds; the data doesn't.

None of these are failures of discipline. Each one is a gap between what the textbook A/B test assumes and how experiments actually run. The answer isn't stricter rules for the old contract — it's a contract that fits reality.
