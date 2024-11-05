---
title: "Flipping Coins in 100,000 Universes Wouldn't Be as Close as the Polls in Wisconsin"
author: "Connor Boyle"
---

I just read Nate Silver's [blog post](https://www.natesilver.net/p/theres-more-herding-in-swing-state), where he writes
that pollsters are systematically altering their data to roughly match the average of existing polls. Rather than
releasing their findings as-is, they're worried they'll look uniquely wrong, and so they're settling for blending in
with the crowd. Nate Silver infers this from the numbers that the pollsters themselves report; the margins in several
swing states are too consistently close to be plausible, even if the election truly is a dead tie among decided voters.

Other than mentioning the binomial distribution and some probabilities, Silver doesn't "show his work" with much detail.
Since probability math can be really easy to get wrong (at least for me!), I thought I'd take a stab at trying the brute
force option of simulating polls in a hypothetical dead tie, i.e. exactly 50% of decided voters plan to vote for each of
the major candidates, Harris & Trump (I also happen to be a computer programmer by hobby and profession, so maybe I'm
just a hammer looking for a nail).

This little project was made possible thanks to Nate Silver's blog, Silver Bulletin, collecting and distributing poll
results. Here are the data files containing the poll results that I used 
for [Wisconsin](https://static.dwcdn.net/data/PMbPp.csv), [Pennsylvania](https://static.dwcdn.net/data/uyZgi.csv),
and [New Hampshire](https://static.dwcdn.net/data/nLq7K.csv)

## Simulating Wisconsin polls

If the exact same number of likely or registered voters (depending on which poll) plan to vote for Harris as Trump, we
can easily simulate the act of surveying them by flipping a coin. Even more easily, we can run the random number
generator on my computer and checking whether the output floating point number is greater than `0.5`; if it is, that's a
Trump voter. Otherwise, that's a Harris voter.

After we simulate our polls, lets extract our statistic of interest: the **mean absolute margin**. For example, if I have three polls with margins:

$$\text{Trump} \space \text{+5%}$$

$$\text{Harris} \space \text{+2%}$$

$$\text{Harris} \space \text{+3%}$$

then their absolute margins are:

$$\text{0.05}$$

$$\text{0.02}$$

$$\text{0.03}$$

and the mean absolute margin for this universe of polls is:

$$ \frac{0.05 + 0.02 + 0.03}{3} \approx 0.03333333333 $$


Here's what the actual polls in real world Wisconsin done by real pollsters look like:

<img alt="Wisconsin observed polling margins" src="/images/poll_margins/wisconsin_observed_margins.png">

The average poll of Wisconsin has one candidate beating the other (some of them Trump beating Harris, some of them vice
versa) by about 2% (or 0.203, as shown in the graph). While the trendline is not terribly strong, we do find that the
absolute margin of a poll goes down as sample size goes up, as we'd expect if the race were truly tied.

And here's a simulation of what those polls *could* look like in an alternate universe where pollsters perfectly
randomly sample the same number of people in a perfectly matched race between Donald Trump and Kamala Harris:

<img alt="Wisconsin simulated polling margins" src="/images/poll_margins/wisconsin_simulated_margins.png">

The output of this simulation certainly differs from our observed results--our mean absolute margin is a full percentage
point higher than our observed one. That doesn't prove anything on its own, though; maybe this simulation of the
Wisconsin polls just happened to result in a high mean absolute margin by chance.

## Simulating a multiverse of polls

What happens if we run that simulation many, many times, keeping track of the resulting mean absolute margin for each
simulation? Let's look at the histogram we get when we do that:

<img alt="Multiverse of simulated Wisconsin polling margins" src="/images/poll_margins/wisconsin_mam_multiverse.png">

Whoa! Our *observed* mean absolute margin of polls (the dashed red line to the left) is *way* lower than any of the MAMs
in the multiverse where Harris and Trump are neck-and-neck. In fact, the lowest MAM out of 100,000 simulated universes
is 0.02092 or 2.092%, still 0.06 percentage points higher than our observed MAM. Does this mean something is wrong?
Well, I can't think of any way these polls could get consistently closer margins than our simulations while still
remaining scientifically valid. It's hard to get a low variance estimate of a mean without increasing your sample size;
that's why the sample size $$n$$ is so important in scientific papers.

Recall also that we generously assumed the candidates had exactly even shares of decided voters. The more imbalanced the
share of voters between the candidates, the higher we would expect the mean absolute margin to be. If you're not
convinced, look at this graph of simulations with varied shares for the candidates:

<img alt="Universes of simulated Wisconsin polling margins with varied Harris shares" src="/images/poll_margins/wisconsin_harris_shares.png">

So, assuming the presidential race in Wisconsin isn't *exactly* tied, the poll margins would look even more suspiciously
close to zero than they already do!

## How is this happening?

To be clear, no individual poll--even one with a very close margin--is by itself indicative of foul play by the pollster
who created it. Rather, the aggregation of poll results for each of multiple swing states indicate systemic problems. I
know almost nothing about political polls, but I recently read a great book about systemic problems in modern science
called [Science Fictions](https://www.sciencefictions.org/p/book) and it seems like there's a *lot* of ways to
manipulate your data--even without intending to or realizing that you are doing it.

It's totally plausible to me that pollsters are just focusing a lot more scrutiny on any result that shows a strong
swing toward one candidate or another; maybe they're more likely to throw out outliers or keep collecting more data if
they start to see "too" wide of a margin.

## Why does this Matter?

After election results (either from exit polls or counting the votes) are announced, it may turn out that one or more of
the swing states goes to either Trump or Harris by a very wide margin. Some people might look back at these polls and
conclude that they constitute evidence of interference, cheating, voter suppression, fraud, or the like. After all, the
pollsters nearly *all* agreed that these swing states were *right* on the margin. However, the high level of agreement
between pollsters is not evidence that we know what the results will be, but rather that we can't trust these polls, and
therefore should be very *un*certain about the outcome of this race.

