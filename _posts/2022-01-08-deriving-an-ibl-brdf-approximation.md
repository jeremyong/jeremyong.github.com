---
layout: post
title: Deriving an IBL BRDF approximation
date: 2022-01-08 21:54
categories: graphics
published: false
katex: true
---

Some time ago, I tweeted this observation:

<blockquote class="twitter-tweet" data-dnt="true"><p lang="en" dir="ltr">This is one of those images instantly recognizable to many graphics engineers that has no meaning outside our field <a href="https://t.co/F2ZxVJPyeD">pic.twitter.com/F2ZxVJPyeD</a></p>&mdash; ninepoints (@m_ninepoints) <a href="https://twitter.com/m_ninepoints/status/1478073576745889797?ref_src=twsrc%5Etfw">January 3, 2022</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

This LUT is extremely common in many game engines, and (I believe) owes its origins to the [work presented](https://cdn2.unrealengine.com/Resources/files/2013SiggraphPresentationsNotes-26915738.pdf)
by Brian Karis of Epic Games at the SIGGRAPH 2013 conference. Let's do a brief recap on what this texture means, how it's typically generated, and how it's used before delving into attempting to approximate it.

## Background

An aggressively simple way of describing "rendering" in general is that, for each pixel onscreen, we want to compute the amount of light shooting towards the viewer. This light, in turn, is accumulated from
all the light sources around the pixel. Of those light sources, we have punctual lights, area lights, the sun, etc. One additional light we need to account for is the light from the sky-box (i.e. the
environment light), generally represented as a cubemap. The cubemap envelopes each pixel, so the physically correct thing to do is, per-pixel, sample the cubemap in all directions, accounting for occlusion, and integrating the
radiance from each point of the cubemap. We can improve on this by importance sampling, but of course, both brute force and sampled integration is likely to expensive.

What we'd like to be able to do is answer the question, "how much radiance is emitted from a surface due to the environment light" as quickly as possible. As an equation, this quantity is written as the following integral (ignoring
view-independent diffuse contributions):

$$
L_o(\mathbf{p}, \omega_o) = \int_\Omega L_i(\mathbf{p}, \omega_i) f(\mathbf{p}, \omega_i, \omega_o) (\mathbf{n}\cdot\omega_i) \delta\omega_i
$$

Reading left-to-right, this says: "the outgoing radiance at point $\mathbf{p}$ in the direction $\omega_o$ is equal to the integral over the hemisphere at $\mathbf{p}$
of the incident light in all inbound directions $\omega_i$ times the cosine-weighted BRDF.

The BRDF factor in the integrand is a function of the position we're querying and the inbound and outbound directions. The main idea to accelerate the computation of this integral is to precompute as much as possible
and perform simple lookups at runtime. So, what can we precompute? The chief observation is that, for the most part, $L_i(\mathbf{p}, \omega_i)$
and $f(\mathbf{p}, \omega_i, \omega_o)$ are _uncorrelated_. That is, there is no fundamental principle connecting incident radiance at a point and the fraction of light emanating toward a given direction.
In some cases, this correllation may be higher, and in other cases, this correllation may be lower. As a result, we can write the following approximation:

$$
L_o(\mathbf{p}, \omega_o) \approx \left(\int_\Omega L_i(\mathbf{p}, \omega_i)\delta\omega_i\right) \left(\int_\Omega f(\mathbf{p}, \omega_i, \omega_o) (\mathbf{n}\cdot\omega_i) \delta\omega_i\right)
$$

To understand the source of the error, it's instructive to write the integral as discrete sums and expand out a few terms manually.

$$
\begin{aligned}
L_o(\mathbf{p}, \omega_o) &\approx \left(\int_\Omega L_i(\mathbf{p}, \omega_i)\delta\omega_i\right) \left(\int_\Omega f(\mathbf{p}, \omega_i, \omega_o) (\mathbf{n}\cdot\omega_i) \delta\omega_i\right)\\
&\approx \frac{1}{N}\left(\sum_{j=0}^{N-1} L_i(\mathbf{p}, \omega^j) \right) \frac{1}{N}\left(\sum_{k = 0}^{N-1} f(\mathbf{p}, \omega^k, \omega_o) (\mathbf{n}\cdot\omega^k)\right)\\
&= \frac{1}{N^2}\times [ L_i(\mathbf{p}, \omega^0)f(\mathbf{p}, \omega^0)(\mathbf{n}\cdot\omega^0) +  L_i(\mathbf{p}, \omega^1)f(\mathbf{p}, \omega^1)(\mathbf{n}\cdot\omega^1) + \dots\\
&\color{red}\qquad\qquad\quad + L_i(\mathbf{p}, \omega^0)f(\mathbf{p}, \omega^1)(\mathbf{n}\cdot\omega^1) + L_i(\mathbf{p}, \omega^0)f(\mathbf{p}, \omega^2)(\mathbf{n}\cdot\omega^2) + \dots \\
&\color{red}\qquad\qquad\quad + L_i(\mathbf{p}, \omega^1)f(\mathbf{p}, \omega^0)(\mathbf{n}\cdot\omega^0) + L_i(\mathbf{p}, \omega^1)f(\mathbf{p}, \omega^2)(\mathbf{n}\cdot\omega^2) + \dots \\
&\color{red}\qquad\qquad\quad + L_i(\mathbf{p}, \omega^2)f(\mathbf{p}, \omega^0)(\mathbf{n}\cdot\omega^0) + L_i(\mathbf{p}, \omega^2)f(\mathbf{p}, \omega^1)(\mathbf{n}\cdot\omega^1) + \dots \\
&\color{red}\qquad\qquad\quad + \dots ] \\
\end{aligned}
$$

Here, we index the various incident directions of our hemisphere from $0$ to $N$ and denote the incident direction as $\omega^i$ with a superscript to disambiguate the subscript as designating
ingoing and outgoing direction. Note also that when substituting the Riemann sum approximations, we could have the summations accumulate a different number of samples, but we simply use $N-1$ for both
sample counts. Note also that for brevity, all of the BRDF terms $f$ also have a dependence on the outgoing view direction but are omitted.
The terms in red are the _cross-multiplication_ terms that would not show up if we didn't use the split-sum approximation. The approximation is good if these red terms somehow cancel out.

So how can we understand that error? In the hemisphere, pick any two incident directions, $\omega^i$ and $\omega^j$ where $i\neq j$. There are two corresponding cross-terms associated with this pair in the expansion:

$$
L_i(\mathbf{p}, w^i)f(\mathbf{p}, \omega^j)(\mathbf{n}\cdot \omega^j) + L_i(\mathbf{p}, w^j)f(\mathbf{p}, \omega^i)(\mathbf{n}\cdot \omega^i)
$$

Substituting the expression for $f(\mathbf{p}, \omega)$:

$$
\begin{aligned}
&L_i(\mathbf{p}, w^i)f(\mathbf{p}, \omega^j)(\mathbf{n}\cdot \omega^j) + L_i(\mathbf{p}, w^j)f(\mathbf{p}, \omega^i)(\mathbf{n}\cdot \omega^i) \\
&\qquad = L_i(\mathbf{p}, w^i)\frac{D(\omega^j, \omega_o)F(\omega^j, \omega_o)G(\omega^j, \omega_o)}{4 (\mathbf{n}\cdot \omega^j)\mathbf{n\cdot \omega_o}}(\mathbf{n}\cdot \omega^j) + L_i(\mathbf{p}, w^j)\frac{D(\omega^i, \omega_o)F(\omega^i, \omega_o)G(\omega^i, \omega_o)}{4(\mathbf{n}\cdot\omega^j)\mathbf{n}\cdot \omega_o}(\mathbf{n}\cdot \omega^i) \\
&\qquad = \frac{1}{4(\mathbf{n}\cdot \omega_o)}\left(L_i(\mathbf{p}, w^i)D(\omega^j, \omega_o)F(\omega^j, \omega_o)G(\omega^j, \omega_o) + L_i(\mathbf{p}, w^j)D(\omega^i, \omega_o)F(\omega^i, \omega_o)G(\omega^i, \omega_o)\right) \\
\end{aligned}
$$

The first factor with $L_i(\mathbf{p}, \omega_i)$ in the integrand can be precomputed for each skybox and retrieved at runtime with a texture sample against what is referred to as a "prefiltered environment map."
The main detail to remember here is that the mip-map level we query from our prefiltered map is dependent on surface roughness (intuitively, the rougher the surface, the more dispersion in the lobe we need to sample).
When generating the prefiltered map, we are actually making a big assumption here, which is that surface roughness alone can produce a meaningful result for this query and we are ignoring the view direction.
In other words, the prefiltered map is computed by integrating over an isotropic lobe with a fixed symmetrical orientation in the hemisphere.
This approximation is then the least accurate when sampling the map at glancing angles.

The right factor is parameterized by two primary parameters: BRDF roughness and $\mathbf{n}\cdot\omega_i$ (we assume no metalness is present in the skybox).
