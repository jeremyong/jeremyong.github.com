---
layout: post
title: Much Ado About Sampling
date: 2022-05-17 22:26
categories:
- Math
- Monte Carlo
katex: true
---

Periodically, I think it's important to revisit the fundamentals. Even if you've seen a concept a million times, sometimes, seeing it
for the million-and-oneth time helps concepts "click" more, especially so if you see a new formulation, or a twist on the original
explanation. Recently, I found myself wading (again) through piles of books and literature on the various sampling techniques that have
been developed over the last few years (ReSTIR, GRIS) and thought it would be a fun exercise for me to do a summary post on the topic
of Monte Carlo integration and sampling techniques in rendering. I'm going to start from the beginning(-ish), but assume you've seen
a fair bit of this already (aka the early parts of this post should be review). For that haven't seen any of these concepts yet, check
out the recommended reading and references sections all the way at the bottom. Also, note that this is one of the blog posts written
somewhat selfishly, intended for myself to refer back to as needed. It might be useful for you also, but it might not also!

## Monte Carlo Integration

Ok, let's recap the problem statement. We have the following fancy equation we want to solve:

$$
L(x\rightarrow \Theta) = L_e(x\rightarrow \Theta) + \int_A f_r(x, \Psi \rightarrow \Theta) L(y\rightarrow -\Psi)V(x, y)\frac{\cos(N_x, \Psi)\cos(N_y, -\Psi)}{r_{xy}^2}dA_y
$$

This is the area formulation of the rendering equation, with the LHS being the radiance emitted from the area at $x$ in the direction $\Theta$.
On the RHS, the equation accounts for $L_e(x\rightarrow\Theta)$ (the radiance emitted from $x$ towards $\Theta$), and the irradiance at $x$ (which occupies some finite area $A$).
The factors in the integral include the BRDF (fraction of light that flows from $\Psi$ to $\Theta$), the light incoming from the direction $-\Psi$ from $y$, a binary visibility term $V(x, y)$,
and a few geometric factors to account for the differential solid angle.

Note that this function is recursive in nature because the function for radiance $L$ appears in both the LHS and the RHS (in other words, the rendering equation is a Fredholm equation
of [the second kind](https://en.wikipedia.org/wiki/Fredholm_integral_equation#Equation_of_the_second_kind)).

As graphics engineers, we need a strategy to solve this integral numerically, and there's a wide spectrum of techniques to do so with varying degrees of speed and accuracy.
For now though, let's remind ourselves why we bother with Monte Carlo at all.

If I gave you an equation like $f(x) = x^2$ and asked you to integrate it numerically over a domain of say $[0, 1]$, what would you do? Chances are, you'd probably
partition the domain and use Riemann integration. Maybe to converge more accurately, you'd sum trapezoidal areas instead of rectangles, or leverage Gaussian quadrature.
These approaches all work for this toy example, but requires that _we know how to partition our domain_. If I asked you to change the sample rate to 100 samples, or 1000,
then each slice of the domain shrinks accordingly to 1/100 or 1/1000 in this case.

How would you partition the rendering equation above though? Because of its recursive nature, the rendering equation has a domain which is effectively _infinite_ in dimensionality.
We can pick a countable number of bounces, and for each bounce count, identify an infinite number of light paths that have that length.
Evaluating high dimensional integrals using the "obvious" techniques is tricky precisely because each dimension needs to be partitioned, and for our rendering equation, it feels like
we'd need an infinite number of samples to even get started.

And so, we get to the first character in our story: _Monte Carlo integration_. The main idea of Monte Carlo integration is that given some integral $\int_\Omega f(\omega)d\omega$,
we approximate it by drawing samples randomly from the distribution (instead of picking samples uniformly as in our toy $x^2$ example). Supposing we draw samples according to a some random distribution $p(\omega)$:

$$
\int_\Omega f(\omega)d\omega \approx \frac{1}{N}\sum_{i=1}^N \frac{f(\omega_i)}{p(\omega_i)}
$$

The RHS is referred to as the Monte Carlo _estimator_.
This approximation follows by taking the expected value of the RHS and showing that it equals the LHS.

$$
\begin{aligned}
E\left[\frac{1}{N}\sum_{i=1}^N \frac{f(\omega_i)}{p(\omega_i)} \right]  &= \frac{1}{N} \sum_{i=1}^N E\left[\frac{f(\omega_i)}{p(\omega_i)}\right] \\
&= \frac{1}{N} \sum_{i=1}^N \int_\Omega \frac{f(\omega)}{p(\omega)}p(\omega)d\omega \\
&= \frac{N}{N} \int_\Omega f(\omega)d\omega \\
&= \int_\Omega f(\omega)d\omega
\end{aligned}
$$

To follow the above, it's enough to bear in mind the definition of the expected value function (used to go from line 2 to 3) and its linearity properties.

This is a useful formulation because we can vary the number of samples $N$ arbitrarily, without needing to think about how to partition the domain. In other words,
if we want more accuracy, we can increase $N$ to be arbitrarily large, and it's ok if the dimensionality of $f$ is infinite.

## Importance Sampling

The next problem is that of convergence. As seen above, Monte Carlo is generalized to allow us to draw samples from _any_ probability distribution.
If you solve for the variance of the estimator, differentiate with respect to 0, and try to minimize the variance, you'd find that a perfect estimator
draws samples according to a probability distribution function (PDF) that matches the "shape" of the integral we're trying to approximate.

This is easier said than done though. In the context of lighting, the samples we draw from are typically some form of query between points $A$ and $B$.
Say we're trying to light a surface from some known position. We probably know enough about the surface to evaluate the BRDF, so that portion of the
integral's kernel is known and we could imagine sampling proportionally to the BRDF. Similarly, we know, for any incident direction to that surface,
how to evaluate the geometric term. This leaves visibility and the incident radiance, both of which provide some challenges.

What we want to do, is draw rays (selecting the originating direction of radiance) incident on a surface according to the distribution of the entire product:

$$
f_r(x, \Psi \rightarrow \Theta) L(y\rightarrow -\Psi)V(x, y)\frac{\cos(N_x, \Psi)\cos(N_y, -\Psi)}{r_{xy}^2}
$$

But it's not easy to sample in a way that accounts for variations in how light is distributed in the scene $L(y\rightarrow -\Psi)$ or visibility $V(x, y)$
between our surface $x$ and sample $y$.

## Resampled Importance Sampling (RIS)

RIS from Talbot's 2005 thesis gives us a simple strategy to sample from multiple distributions. Usually, we might know _something_ about the scene,
for example, where the major occluders are (which might help us sample $V(x, y)$ more accurately) or where the major light sources are (which might help
us sample $L(y\rightarrow -\Psi)$ more accurately).

The idea is to draw $M$ samples from one distribution, then, from those samples, draw samples according to a different distribution. A commonly
used example is to sample incident directions $y\rightarrow -\Psi$ from known light sources, and subsequently filter the candidate samples by sampling
the BRDF. The exact algorithm (modified slightly for clarity) is summarized according to the following steps:

1. Generate a sequence of $M$ i.i.d. (independent and identically distributed) random samples $(X)_{i=1}^M$ according to a source PDF $p(X_i)$
2. Assign to each sample a weight $w(X_i) = \frac{q(X_i)}{p(X_i)}$
3. Normalize each weight by the sum of all weights to assign each sample its respective probability mass
4. Draw $N$ samples $(Y)_{i=1}^N$ randomly according to their probability masses
5. Evaluate the following estimator:

$$
\langle I\rangle = \frac{1}{N} \sum_{i=1}^N \frac{f(Y_i)}{q(Y_i)} \cdot \frac{1}{M}\sum_{j=1}^M \frac{q(X_i)}{p(X_i)}
$$

In many papers, it's assumed that $N$ is simply 1, but it's important to mention that there may be reasons to draw more "final" samples from the candidate
samples. Also, because our final samples are unbiased, we can technically chain this strategy multiple times, potentially drawing candidate samples from
3 or more PDFs if we so choose.

## Multiple Importance Sampling (MIS)

In the algorithm above, the candidate samples $(X)\_{i=1}^M$ were drawn from the same probability distribution. However, we may want to draw
from multiple distributions instead. Multiple Importance Sampling (MIS) gives us a general strategy for combining samples drawn from different
distributions via appropriate weighting, and if we opt to filter samples drawn from multiple distributions, this is referred to as _resampled MIS_.

Given $N$ sampling strategies (distinct PDFs), we can weight the outcome of each sample with some $m_i, 1\leq i \leq N$ such that the weights
are a partition of unity (the sum of the weights is 1). Veach's thesis proposes several heuristics for weighing the strategies, the most straightforward
one being the "balance heuristic" (MIS weights are in proportion to relative sample probability). Another heuristic commonly employed is the
"power heuristic" which weights samples according to their relative probability masses raised to some power (most commonly 2). Using the power
heuristic emphasizes strategies that produce greater signals (e.g. particularly illuminating samples) while depressing the weights of other samples.

To combine MIS and RIS, we need to change the estimator. As presented, the RIS estimator computes the resampling weights using the average weights drawn
from the first distribution. Instead of dividing by $M$, the idea is to replace $\frac{1}{M}$ with our MIS weight $m_i$, computed by whatever heuristic
we deem appropriate (e.g. "balance" or "power").

A natural question to ask is, why would we want to use RIS, or RIS in combination with MIS, instead of just MIS? After all, MIS seems more straightforward
to apply. The answer is that MIS requires that it is somehow cheap to sample from a given PDF we want to use. Consider the scene visibility. This
gives us a function that tells us if two points in the scene are occluded or not. Surely our integral variance drops if we can sample lights according
to the visibility between the light and our sample location. However, this PDF isn't something we can sample easily. RIS allows us to filter by something
expensive (like visibility) _after_ we've determined an initial set of candidates to filter.

## ReSTIR

OK, so we now have a number of tools at our disposal. We have some target distribution that's intractably hard to sample from (combining the BRDF,
visibility, light distribution in the scene), but now we can sample individual components of that distribution in several ways. We can draw first from
one distribution, then filter those samples with another. We can also draw from multiple distributions at once, and then combine them. We can do both.
The only tricky thing so far is the bookkeeping needed to track weights so that quantities remain unbiased, but assuming we've done that, RIS and
MIS both come with nice proofs (which we won't go into here) demonstrating that they converge more rapidly than sampling like a chaos monkey (recall
that Monte Carlo converges at rate $O(\sqrt{n})$, meaning that to double our accuracy, we need quadruple the number of samples).

The problem we'd run into if we just leveraged RIS and MIS above is performance. Namely, to pull $N$ samples from $M$ candidates, we would need M candidates
to be resident in memory for each sample we wish to perform in parallel. The [ReSTIR paper](https://cs.dartmouth.edu/wjarosz/publications/bitterli20spatiotemporal.html)
shows how RIS can be combined with yet another general technique from statistics known as reservoir sampling to pull $N$ samples from $M$ candidates
in $O(N)$ memory (as opposed to $O(N + M)$). If we choose to filter candidates down to a single sample, the memory used is constant.

The basic idea is to process the stream of $M$ candidates one at a time, and place them into a reservoir containing $N$ samples.
As we examine new candidates, we accept the current sample if a random unit uniform variable
(commonly referred to as $\xi$) is less than the ratio of the current weight and the sum of the weights encountered thus far. That is, look at a sample,
compute its weight according to the the RIS or RIS+MIS scheme described above, update the total weight, generate a new $\xi$ (random number between 0 and 1),
accept the sample if $\xi < w_i / \sum w_j$.

One really nice property about reservoirs are that they are "memoryless" with respect to all the individual samples considered prior to the currently tracked sample (samples
for reservoirs where $N > 1$). Since reservoirs track the current total weight, count of samples seen, and the current sample weight, we can easily
combine reservoirs by:

- Initializing a new reservoir
- Update the new reservoir with each reservoir we want to merge by _considering each reservoir as a new sample in a stream_
- Summing input reservoir sample counts to determine the merged reservoir sample count
- Compute the merged reservoir total weight as the average weight of all the reservoirs divided by the PDF evaluated for the ultimately selected sample (this computation is just like the RIS estimator)

Intuitively, we can generate streams of samples and cheaply filter them to combine multiple probability distributions and strategies. We can then treat
each of these reservoirs as effectively samples (coalescing streams of samples into samples), that we then treat as yet another stream to combine again.
The ReSTIR paper applies this technique with a few additional tricks to handle visibility and share data across samples in the spatial and temporal domains.

As described in the paper, the general algorithm is to:

- Initialize reservoirs for each pixel and perform samples using RIS
- For each reservoir, do a visibility check and 0 out the reservoir if the selected sample is shadowed
- Combine each pixel's reservoir with its reservoir computed from the last frame (establish pixel correspondance via motion vectors)
- Combine each pixel's reservoir with reservoirs from its neighbors (applied in multiple rounds)
- Shade the pixel
- Save this frame's reservoirs for the next frame

The original paper uses the scene lights to generate initial samples, and this is extended in the [ReSTIR GI paper](https://d1qx31qr3h6wln.cloudfront.net/publications/ReSTIR%20GI.pdf)
to instead use one indirect bounce to gather initial samples, effectively using the same strategy to accumulate GI instead of DI.

## Addressing Bias

Now, an important detail being glossed over so far is that we've sort of combined reservoirs with reckless abandon, spatially and temporally. A key requirement
of the RIS proof of unbiased convergence was that the samples chosen were i.i.d. (identically and independentally chosen) according to a given PDF. When combining
samples across reservoirs, however, this requirement is _not_ met because the PDFs associated with each pixel is actually different. For example, two adjacent
pixels may have an entirely different set of lights to consider and sample from. They also may reference different surface materials with BRDFs that behave
very differently.

Luckily, the ReSTIR paper has a section dedicated to understanding this bias and eliminating it ($\text{\sect} 4$). To account for bias, the paper authors
check for occlusion/visibility at the for the candidate shared-sample at the merge point. The art of transferring domains (spatial temporal, or any other general
domain) is more fully explored and addressed in the [GRIS paper](https://d1qx31qr3h6wln.cloudfront.net/publications/sig22_GRIS.pdf). GRIS (Generalized RIS)
augments RIS with the ability to draw samples from distributions in different domains, and does that by incorporating a "shift mapping" to fix the
resampling weights.

The resampling weight as defined by GRIS is given as:

$$
w_i = m_i\left(T_i(X_i)\right)\hat{p}\left(T_i(X_i)\right)W_i\cdot \left|\frac{\partial T_i}{\partial X_i}\right|
$$

with $m_i$ being our MIS weight (or $1/M$ for sampling from a single PDF), $\hat{p}$ being the target PDF, $T_i(X_i)$ being the transform from the domain of $X_i$
to $Y$, and $W_i$ being 1 over the PDF used to sample $X_i$ (or its contribution weight from a prior RIS pass if its PDF isn't tractable). The question is then,
how do we define shift mappings between pixels?

As it turns out, this is yet another rabbit hole to delve into, and there is an existing corpus of work that transforms neighboring paths to resuse computation
and improve path tracer performance. The GRIS paper outlines this in $\text{\sect} 7$, going into detail specifically into how to determine resampling weights
for shift mappings between paths, which they demonstrate in the ReSTIR-PT path tracer implementation.

## Recommended Reading

_Note: book links to Amazon.com are *not* affiliate links so click away._

The fundamentals of Monte Carlo integration, importance sampling, and MIS are covered quite well in the book [Advanced Global Illumination](https://www.amazon.com/Advanced-Global-Illumination-Philip-Dutre/dp/1568813074)
and [PBRT](https://www.pbrt.org/).

[Veach's thesis](https://cseweb.ucsd.edu/~viscomp/classes/cse168/sp21/readings/veach.pdf) also has an accessible treatment of MIS, being the original source of the idea.

[Talbot's thesis](https://scholarsarchive.byu.edu/cgi/viewcontent.cgi?article=1662&context=etd#:~:text=Global%20illumination%20is%20an%20application,consisting%20of%20reflectors%20and%20absorbers.) outlines the RIS approach
for resampling from a single strategy, and generalizes the idea to incorporate MIS also.

The [Spatiotemporal reservoir resampling for real-time ray tracing with dynamic direct lighting](https://cs.dartmouth.edu/wjarosz/publications/bitterli20spatiotemporal.html)
combines RIS+MIS with reservoir sampling and demonstrates the reservoir merging capability to perform resampling across space and time.

The [ReSTIR GI: Path Resampling for Real-Time Path Tracing](https://research.nvidia.com/publication/2021-06_restir-gi-path-resampling-real-time-path-tracing) paper applies
ReSTIR to compute GI efficiently.

[World-space spatiotemporal reservoir reuse for ray-traced global illumination](https://gpuopen.com/download/publications/SA2021_WorldSpace_ReSTIR.pdf) is another application of
ReSTIR for GI, caching irradiance into cells of a hash grid in a GPU friendly way.

[Generalized Resampled Importance Sampling: Foundations of ReSTIR](https://research.nvidia.com/publication/2022-07_generalized-resampled-importance-sampling-foundations-restir)
revisits the original ReSTIR paper with additional theory regarding unbiased resampling weight computation and applies it to a path tracer using shift mappings.

[SIGGRAPH 2021: Global Illumination Based on Surfels](https://www.ea.com/seed/news/siggraph21-global-illumination-surfels) from EA SEED also uses ReSTIR
to sample lights as one of many extensions to their previous presentation of surfel-based GI in the PICA PICA demo.
