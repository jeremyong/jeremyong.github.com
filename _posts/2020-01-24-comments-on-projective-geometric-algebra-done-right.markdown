---
layout: post
title: 'Comments on "Projective Geometric Algebra Done Right"'
published: false
categories:
  - math
---

This is a response to the criticism in a recent post called [Projective Geometric Algebra Done
Right](http://terathon.com/blog/projective-geometric-algebra-done-right/) of work done in the 2019
SIGGRAPH presentation "Geometric Algebra for Graphics Engineers" (I did not personally create the
SIGGRAPH presentation, but I _do_ subscribe to its conclusions). I mean no ill-will to the
author (I even have a couple of his books!), but felt somewhat compelled to write this response
because I believe the view espoused actually prevents a more intuitive grasp of the concepts
therein. Central to the debate is how geometers should represent projective space algebraically.
Namely, what possible representations are there and what are the tradeoffs? I can't get into the
entirety of Geometric Algebra in just a blogpost (that will need to be a longer form writeup), so be
advised that this is written for people that have had some brushes with GA already. All quoted text
in this article is lifted verbatim from the source material as of this writing. So without further
ado, let's dig into it.

> The conventional way in which publications teach you how to transform a point p with a dual
> quaternion is to first turn p into the dual quaternion 1 + εipx + εjpy + εkpz, evaluate the
> product (qr + εqd)(1 + εipx + εjpy + εkpz)(q̃r + εq̃d), and then pick off the coefficients of the
> εi, εj, and εk terms to retrieve the result. This is a hack. What it’s really doing is casting p
> as a translation operator, transforming that operator with the dual quaternion, and then casting
> back to a point. It doesn’t extend to the richer set of geometric objects available in ℝ3,0,1,
> which include vectors, bivectors, points, lines, and planes.

Let's address this comment first. Representing a point $$\mathbf{p}$$ as $$1 + \epsilon(p_x\mathbf{i} +
p_y\mathbf{j} + p_z\mathbf{k})$$ might be characterized as a hack, but only because it's being
represented as a bivector. It isn't being casted "as a translation operator" which implies that it
carries with it a group action or product of some sort. It could be argued (which is the crux of his
post) that there is a more "natural" representation as a vector sum (as opposed to a trivector sum),
so let's see if that holds up.

> [Dual quaternions don't] extend to the richer set of geometric objects available in ℝ3,0,1, which include
> vectors, bivectors, points, lines, and planes.

I mostly agree with this. The dual quaternion is a subalgebra of the full geometric algebra after
all. It does support more than just points, but let's continue...

> In their SIGGRAPH 2019 course “Geometric Algebra for Computer Graphics”, the presenters point out
> that general QxQ̃ transformations can be achieved by “turning the algebra on its head” and working
> with the so-called dual construction. In this formulation, homogeneous vectors represent planes,
> and homogeneous trivectors represent points, which is the opposite of what makes intuitive sense
> with respect to the dimensionality of those geometric objects. (Lines are still represented by
> homogeneous bivectors, but the parts that correspond to the direction and moment are reversed.)
> This way of doing things is a giant kludge that misses an important part of the big picture, and I
> find it particularly disagreeable due to the problems that it causes. I’ll get back to those
> below.

Here I'm going to pause and say that calling this a "giant kludge that misses an important part of
the big picture" strikes me as pretty unnecessarily flowery language to say the least. We still
haven't gotten to what the author deems "unintuitive" so let's skip to that step where the key
points are summarized and address them in turn. _edit: it appears that since authoring this post,
the language in the original post has been edited_

> Their model requires that points be represented by trivectors and planes be represented by
> vectors. This is backwards. In 4D projective space, points should be one-dimensional, and planes
> should be three-dimensional. The projection into 3D space removes a dimension and should naturally
> make points zero-dimensional and planes two-dimensional, but that’s not what they have.
> Furthermore, swapping points and planes has serious implications for nonorthogonal change-of-basis
> transformations, which happen in computer graphics all the time, but they completely ignore this
> issue.

My interpretation of this objection is that because points are "zero-dimensional" entities in 3D, in 4D
(where we tack on a dimension and embed $$\mathbb{R}_3$$), the point should be "one-dimensional."
This strikes me as an odd requirement to say the least. Points are "zero-dimensional"
regardless of what dimension you refer to them in! Similarly lines span a single dimension, planes
two dimensions, etc. What I think the author is trying to say is that to map from a geometric entity
in $$\mathbb{R}_4$$ to $$\mathbb{R}_3$$, we "should" use a geometric entity that spans one
additional dimension than what we are trying to model. This is where I think my opinion (and those
of other practitioners in this field) will start to diverge from the author's viewpoint. Here, the
claim is made that, in essence, a ray-plane intersection (vector onto a plane) is _the correct_
choice for representing points, in contrast to a 3-plane intersection. I haven't yet said why I
think the latter is more _natural_ in this context, but surely it should be clear that such a claim
is made with no justification beyond "feeling."

The notion of "dimension" is important to clear up here. Points, from the perspective of physics
perhaps, might be considered as "zero-dimensional" in that they occupy an infinitesimal amount of
space. Mathematically however, this is not the notion of dimensionality that is consistent with the
rules of Algebra. The dimensionality of an entity depends on the field or space we operate in. For
example, if we take the Cartesian plane, a "point" is two-dimensional. It requires two coordinates
to specify its location. If we define planes as our basis in 3D projective geometry, points are
three-dimensional in the sense that three planes are required to specify one. The other ambiguity is
that we can consider the dimension _spanned_ by the basis elements that comprise of an entity. The
point is that because there are multiple notions of dimensionality at play, I don't believe it
behooves us to prioritize one notion in the choice of our representation over any other.

The second point here is that points and planes are "swapped" somehow which also, is not true.
Planes are still planes. Points are still points. What is simply being _defined_ is the manner in
which they are embedded in the space in relation to the metric which is $$\mathbb{R}_{(3, 0, 1)}$$.
It is this metric that will ultimately dictate what embedding makes the most sense. If what is meant
is that the coordinate basis needs to change... well that's a given. When you transfer from one
algebra to the next, we generally don't retrofit one algebra to ensure that the basis aligns
perfectly with the other. What ends up happening is we inject the coordinates with the correct
signs, at the right alignment (in our constructor or what have you). After all, we've been doing
this with quaternions for decades, moving from a standard Cartesian basis to the Quaternionic one.
Non-orthogonal changes of basis aren't really relevant in this discussion. The statement only makes
sense if you enter the discussion with the notion that planes are somehow built from points, as
opposed to the opposite notion that points are built by intersecting planes.

> Because points and planes are backwards, the geometric meaning of the wedge and antiwedge products
> are also necessarily backwards. Their model requires that ∧ corresponds to a dimension-decreasing
> meet operation and ∨ corresponds to a dimension-increasing join operation, but in the Grassmann
> algebra used by the rest of the world, the meanings are the other way around.

This is factually incorrect and can be disproven trivially. The exterior product of $$\mathbf{e_0}$$
and $$\mathbf{e_0}$$ is exactly zero regardless of what metric you work in. But aside from that,
this statement amounts to saying that the "rest of the world" leverages exclusively an
inner-product-null-space view (IPNS) as opposed to an outer-product-null-space view (OPNS) which is
also factually incorrect. In some contexts, OPNS is more natural, and certainly no less valid. In
terms of grade-increasing, in PGA, the wedge product still increases the grades of the elements.
What the author means to say here is that the grade reflects the complement of the entity
represented, as opposed to the entity's dimensionality as he would like. This is the same objection
as the point above. How geometric entities behave in this space should be the same as any other. The
_mapping_ to $$\mathbb{R}_3$$ which demands a surjection is the point of confusion.

The next point made in the author's summary is that normalization is backwards (in the same sense
that everything else is backwards) because Cartesian points, lines, and planes are all dualized in
this representation. Instead of addressing it directly though, I really need to talk about why the
dualized representations were used in the first place. Note that this is in no way my invention.
Thinking in terms of dual spaces and dual maps has been a large topic in mathematics for a long
time, and much of my understanding is due to great mathematicians such as Poincare, Klein, as well
as countless others that have built on top of their work and written great material about it. The
first thing to realize is that, in $$\mathbf{P}(\mathbb{R}^*_{(3, 0, 1)})$$, vectors ... are still
vectors. Bivectors are still bivectors, trivectors are still trivectors. The geometric
interpretation of all of those entities _do not change_. The fact that there is a correspondence
between points in $$\mathbb{R}_3$$ to trivectors in
$$\mathbf{P}(\mathbb{R}^*_{(3, 0, 1)})$$, is something that takes effect once we are ready to map the
results of our computation back to $$\mathbb{R}_3$$. Note that at this stage, we don't even care
about the metric of the latter space. In the former space however, normalization occurs exactly as
you'd expect. The norm of a motor, for example, is just $$\mathbf{m}\mathbf{\tilde{m}}$$. It simply
_doesn't matter_ what the original meaning of the coordinates were prior to our embedding.

The reason this bears repeating is that when we are in $$\mathbf{P}(\mathbb{R}^*_{(3, 0, 1)})$$, the
geometric product, exterior product, contractions, inner products, etc _all work as they did before_
with any other metric signature, in any dimension. For example, the geometric product between two
vectors can produce a reflection via the sandwich product. The next detail we need to look at is the
metric signature $$(3, 0, 1)$$. The $$1$$ in the third component here signifies that there exists a
single vector basis element that squares to 0 (conventionally, $$\mathbf{e_0}^2 = 0$$). With this,
the stage is set to determine how we should proceed.

The first thing to consider is the inner product, which is what we use to measure angles. Suppose we
represent planes as trivectors as proposed in the article such that a homogeneous plane looks like
$$n_x\mathbf{e_{230}} + n_y\mathbf{e_{310}} + n_z\mathbf{e_{120}} + d\mathbf{e_{321}}$$ (note that I am
using $$e_0$$ as the element that spans the ideal axis instead of $$e_4$$ since this is an
established convention that fixes the ideal axis at zero regardless of the dimensionality of the
space of interest). The dot product of two planes with this convention produces something that
depends only on the homogeneous coordinates. Already we've just lost an operation for ostensibly no
reason. In contrast, if we take a plane in the dual sense expressed as its normal vector
$$n_x\mathbf{e_1} + n_y\mathbf{e_2} + n_z\mathbf{e_3} + d\mathbf{e_0}$$, the dot product corresponds
_exactly_ to the angle between the two planes. This wasn't an accident either. The ideal component
squaring to zero as a result of the degenerate metric has an interesting side effect of "squashing
distances" in the ideal plane. This is why, despite the author's claims, there isn't a strict
symmetry between the two operations.

Now, it's important to realize that the reason why the author needed to introduce a new "geometric
antiproduct" was honestly to sidestep this issue and move the zeroes of the Cayley table to the
right place to work. You'll notice that in the cheatsheet he provides
[here](http://terathon.com/pga_lengyel.pdf), there is no mention of the inner product. This is
because we'd need to invent some sort of "inner antiproduct" or somesuch contraption to make things
work as you'd expect. By the same token, we lose the ability to express projections and rejections
of elements as well (all of which require the inner product measure to function correctly). At a
certain point, it feels like we'll end up fighting the framework _more_ in the name of something
that is "more intuitive." Unfortunately, what results is frankly something far less intuitive. It's
unclear in the "new" framework "done right" what, if any, use there is for the inner product, or
geometric product in this context. Such products have meaning regardless of the metric signature,
and if there is a hack here, it's the claim that the issue is with the geometric product itself (and
replacing it with a _bunch_ of new concepts, the anti-reverse, the anti-product, etc). In the space
containing our embedding, the geometric product works, just as it always did. In contrast, the the
"anti-product" (which frankly, conveys little intuition on its own) simply attempts to sidestep the
issue by redefining the metric to be $$(1, 0, 3)$$ for a _single operation_.

OK that was inner product-measures (which includes projections/rejections). What about reflections?
In "normal" GA, a sandwich of an entity (regardless of if its a point, plane, etc) with a vector
will reflect that entity about the vector. Ergo, $$\mathbf{pXp}$$ will reflect $$\mathbf{X}$$ about
$$\mathbf{p}$$ (here the plane $$\mathbf{p}$$ is given in the dual sense in terms of its normal
vector). The equivalent formulation given with the author's cheatsheet involves two geometric
anti-products and an anti-reverse.

The only point at which things start to appear "normal" again, frankly, is when we finally see the
motor operations (and translations and rotations), and only because the multiplication tables have
been specifically crafted for this particular set of operations. Everything that it is built on is
largely inconvenienced or without sound mathematical justification. Even the multiplication table of
the "geometric antiproduct" is somewhat bizarre. For example, $$\mathbf{e_2}^2 = 1^2 =
\mathbf{e_{31}}^2 = \dots = 0$$, but the pseudoscalar is exactly unity. Why does this not qualify as
a "hack" or a "kludge?" Just looking at the table alone, it's clearly a particularly awkward
relabeling of all the original basis elements to "hide" the duality of the space.

The multiplication table is even weirder when we consider how it might generalize to other
dimensions. Even if you only ever intend on working in, say, 3D, seeing how a concept generalizes is
a useful way to detect a "smell" as it were in the formalism. Incidentally, this is how you might
determine that somehow, the inner product is far more fundamental than the cross-product (the latter
only works in exactly 3 dimensions, and lacks associativity). Let's start by looking at a simple
rotor in $$\mathbf{P}(\mathbb{R}^*_{(2, 0, 1)})$$ which is given as $$\cos\frac{\alpha}{2} +
\sin\frac{\alpha}{2}\mathbf{p}$$ where $$\mathbf{p}$$ is a dual-Euclidean point. Expressed in the
"alternative view," the rotor would look like $$\cos\frac{\alpha}{2}\mathbf{e_{123}} +
\sin\frac{\alpha}{2}\mathbf{p'}$$ where $$\mathbf{p'}$$ is a vector that maps the point we wish to
rotate about. The immediately evident tradeoff is the presence of the mysterious
$$\mathbf{e_{123}}$$ factor that's needed to make the so-called "antiproduct" work. Why is this
strange? Well, we know that rotations are an immediate consequence of the exponential map which
gives us the rotation action at a tangent frame. An exponential has the form $$1 + x +
\frac{x^2}{2}-\dots$$ so we could easily derive the standard rotor definition in a closed form. In
contrast, the new form of a rotor bears little resemblance to this ubiquitous power expansion,
mainly because everything has been shuffled around.

Note also that the rotor and motor elements in the standard formulation occupy the even-subalgebra
(generally written as $$C\ell^+_n$$). The even sub-algebra is known to generate rotations in any
dimension and are generated by the bivectors. The author makes a separate claim in a twitter thread
that due to symmetry, his "odd subalgebra" works just as well. This only makes sense if you're
willing to accept that with his product, 1 times 1 is, well, 0. The scalar quantity present in the
even-subalgebra is an encoding of "measure" with respect to the metric as-defined. Trying to frame
things in terms of the "odd-subalgebra" (which has scant mention in any literature) necessarily
pushes the responsibility to some other element which in the author's case, is the coefficient of
the pseudoscalar. That is... odd to say the least. Again, we must ask, why is this more intuitive?
Is this not "backwards" relative to the formalism as stated? What's stranger still about his new
algebra, is that the parity of the subalgebra changes with dimension! Because the identity element
is now associated with the pseudoscalar, the motor subalgebra is odd in 2D, then even, then odd
again.

Summarizing, it strikes me that the main motivations for going down the path of the "geometric
antiproduct" was a belief that points, lines, and planes in 1-up space _should be represented a
certain way_, and proceeding based on that belief. To the contrary, a more sound approach would have
been to recognize that there are at least two mappings to get between projective space and Euclidean
space, and seeing how the entities behave under various group/algebraic operations and symmetries
would have dictated which representation makes the most sense. To do otherwise is putting the cart
before the horse, so to speak. The algebra must serve the geometry, not the other way round. The
fact that when doing projective geometry, the algebraic manipulation relies on representing a plane
as its normal, or that the point should be represented as a trivector is _OK_. Once the conversion
is done, we may trust that all identities and relations proved function properly for vectors,
bivectors, and trivectors, and we can faithfully compute with them until the time to read off the
result based on our interpretation is done. Instead, the author chose to force an interpretation of
entities a particular way (the dual approach was deemed "backwards") and so he had to invert the
reversion operator, invert the geometric product's Cayley table, and invert the subalgebra itself.
In the process, we also needed to abandon a wealth of established conventions, theorems, and
literature. Is the new upside-down world of special-cases and mathematical artifacts more intuitive?
Personally, I don't think it's worth the trouble.
