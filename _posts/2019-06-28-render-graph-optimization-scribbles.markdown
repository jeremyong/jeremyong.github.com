---
layout: post
title: 'Render Graph Optimization Scribbles'
categories:
  - rendering
redirect_from:
  - /rendering/2019/06/28/render-graph-optimization-scribbles.html
---

When I implemented my own render graph, I needed to make a number of decisions for proceeding and wanted to
record the various considerations for authoring such a library.
By way of background, a _render graph_ is an acyclic directed graph of nodes, each of which may consume a set of
resources and produce a set of resources.
Edges in the graph denote an execution dependency (the child node should happen after the parent), with the caveat
that the dependency need not be a full pipeline barrier. For example, a parent node may produce results A and B,
but a child node may only depend on A.
Another caveat is that the dependencies may be relaxed to allow some overlapping.
For example, a parent node may produce some result in its fragment stage which is consumed in the fragment stage of a child node.
In such a case, the child node can execute its vertex stage while the parent node is also executing its vertex stage.

In this post, I'm going to focus on the optimization aspects of a render graph as there are many benefits of a render graph in general,
and touching on all of them would result in a much larger post.
Restricting ourselves to the optimization effort, we can then summarize the goals of any render graph processing as the following:

1. Allow overlapping execution as much as possible (maximize chip occupancy)
2. Express dependencies such that flushes are deferred for as long as possible and span the minimum surface area of memory that needs to be flushed

These goals must be accomplished without violating the dependencies encoded in the node edges.
With this, here are some pointers for accomplishing the above without too much fuss.

## We need a good answer to the following question

Suppose you have a graph that looks like the following

    --A---
          \
           C
          /
    --B---

Here, `A` and `B` are dependencies of `C` and they themselves are dependent on other nodes that are not pictured.
The question is, _how should I schedule_ `A` _and_ `B`?
It's important to understand why this is an important question for which the answer isn't immediately obvious.
The first observation one should make is that doing both at the last hour (right before `C` is submitted) is not necessarily ideal,
as `C` will now need to wait for the whichever of `A` and `B` completes the slowest.
Perhaps there was an opportunity for `A` to cleanly slot into available chip space and finish before `B` (or vice versa).
Maybe `A` depends on a number of caches being flushed from a parent node, and scheduling `B` after `A` would unnecessarily
delay `B` from starting earlier.

OK, so there's a decision to be made here, but what's the right way of scheduling `A` and `B`?
The simple heuristic I'll offer is the following:

> A node should be scheduled **later** if it consumes many results that are expensive to produce.
> A node should be scheduled **earlier** if it is expensive to execute and its products are consumed by many dependent nodes.

The two statements above are two sides of the same coin, and while imprecise, still concisely summarize the main idea.
If a node relies on data being produced by many preceding jobs or by jobs that take a long time to run, early submission is wasteful
as execution will be stalled by the parent jobs.
By the same token, if a node is relied on by many downstream nodes, it behooves us to submit it as early as possible so that
by the time those dependent nodes are scheduled, the data is ready to be consumed.

Of course, there is a tension here. What if a node relies on a lot of data from upstream nodes and also produces a lot of
data for downstream nodes?
In such a case, one can view this node as a potential bottleneck and it will be difficult practically to avoid
occupancy bubbles from forming either in front of the node, behind it, or both, depending on when we submit it.

Another consideration that makes things even more difficult is that different nodes have different occupancy characteristics.
We would want to avoid scheduling two transfer-heavy nodes simultaneously if possible as the usage of the
same limited resource prevents maximum chip utilization.
The same is true for co-scheduling nodes that compete for other resources (LDS, registers, etc).

## Implementation strategy

Keeping track of all the above is a pretty daunting ordeal, and a perfect solution is likely to span thousands of lines
of code and possibly be expensive to run per frame if taken to the extreme.
There is a tradeoff between having a "perfect" scheduling algorithm which becomes hard to maintain, but produces
sufficiently good results that required frame times are honored.

Of course, every situation will be different. For your own implementation, you should consider the different hardware
profiles you'll need to ship for, how dynamic the graph workloads will be, and also how much performance is really required
for your engine to do what it needs to do.

I don't want to leave this scribble on a "do what works for you" quip, so I'll offer a modest starting point for more general
purpose engines. Here was my approach for my engine which you can either take wholesale or use as a starting point for your
design:

First, reshape your render graph into an n-ary tree that looks something like the following:

    +---------------+---------------+
    |  A    B    C  |    D     E    |
    +-------+---+---+---+---+-------+
    |   F   | G | H | I | J |   K   |
    +---+---+---+---+---+---+---+---+
    | L | M |               | N | O |
    +---+---+               +---+---+

In the above example, each "box" can be thought of as a bucket that contains nodes that can be ordered anywhere among the
nodes in the boxes below it and each other. Here, `A`, `B`, and `C` can happen in any order.
Furthermore, they can happen in any order relative to the nodes in any box below.
Boxes that are adjacent to
each other must happen one after the other. So, `F` needs to happen before `G`, and `G` needs to happen before `F` (but
any of `A`, `B`, or `C` could happen before, after, or in between `F` and `G`).
Each vertical line (`|`) denotes a memory dependency that must be inserted in the final submission.
An alternative way of viewing this tree is to realize that any node can be scheduled as early as the `|` to the left and as
late as the `|` to the right. As a result, this data structure effectively encodes the state space of all permissible submission orders that we can select from.

There's another important invariant of this tree to discuss. Let us define the cardinality of each
box as the number of constituent nodes (i.e. the cardinality of the upper-right box is 2 in this example).
Note that we can enforce that the cardinality of a box at a lower depth (higher up pictorially) must be greater than or equal to the cardinality of any of its children.
To see why this is so, consider if we had a tree that looked like the following:

    +-----+
    |  A  |
    +-----+
    | B C |
    +-----+

This tree indicates that `A` can go anywhere between `B` and `C`, but also that `B` and `C` can be reordered.
Thus, this tree is equivalent to the following:

    +-------+
    | A B C |
    +-------+

Q.E.D.

After we have reshaped the tree as above (or possibly built it in this fashion from the get-go),
we can proceed by assigning each job a weight to eventually arrive at a total ordering.
Let us define the weight, loosely, as the cost to execute the job (more on this later). For this simple implementation, we will assume that jobs
occupy the chip uniformly. If we didn't want to make this assumption, we could adapt this algorithm to use parametric weights,
but I will avoid doing this here for simplicity's sake. From here, we can collapse the box-tree into a single flat array recursively as follows:

1. Starting from a box $$X$$, look at its descendents $$X_d$$
2. If the descendents are leaves, collapse $$X$$ and $$X_d$$ into an ordered list ordered by the execution weight of each item descending.
   Then proceed to the next box (sibling)
3. If the descendents aren't leaves, look at each descendent individually and recusively start from step 1 of this algorithm. Once they have been
   flattened, continue as before.

At the end of this procedure, we'll have collapsed all our boxes into a single depth of submission-ordered nodes.
When doing this procedure of course, we need to remember to inject memory barriers corresponding to the box separators.

Following this heuristic, we have a rough way of ensuring that prior to each barrier, the worst offenders in terms of execution time have
had the most time to complete their tasks. In addition, by bucketing things the way we did, items that are dependencies for many downstream
nodes get bubbled to the front of the queue naturally (they would end up down and to the left in the original reshaped tree).

We still need to take care of the _execution weight_ which we assumed existed arbitrarily. One option is to allow the user to take a stab
at guessing this, possibly with benchmarked values. A more flexible approach is to use actual numbers from a previous frame to decide
the ordering. This has the benefit of making the submission list adaptible to changing scene rendering conditions.

## Conclusion

We have as stated, a few high level aspirations for maximizing chip occupancy by scheduling tasks so that they can overlap using
permissive barriers between them (as necessary). In addition, we looked at a one option (of many) to accomplishing our goals.
I should mention at this point, that many (most?) engines will not necessarily benefit from this abstraction depending on the
anatomy of a frame. If there aren't many opportunities to reorder work or many independently describable nodes, trying to come
up with this overly generalized approach is likely not worth it. For engines that have deep pipelines and highly variable workloads,
however, the abstraction is well worth the investment.
