---
layout: post
title: Vulkan Synchronization Primer - Part I
katex: true
categories:
  - vulkan
  - graphics
  - rendering
redirect_from:
  - /c++/graphics/gpu/vulkan/2018/11/22/vulkan-synchonization-primer-part-i.html
  - /c++/graphics/gpu/vulkan/2018/11/22/vulkan-synchronization-primer.html
---

The intent of this post is to provide a mental model for understanding the various synchronization
nouns, verbs, and adjectives Vulkan offers. In particular, after reading this series, hopefully, you'll
have a good understanding of what problems exist, when you should use which sychronization feature, and
what is likely to perform better or worse. There are no real prerequisites to reading this, except that
you've at least encountered a few barriers in tutorial or sample code. I would guess that many readers
might have also tried to read the standard on synchronization (with varying degrees of success).

Part I of this series will primarily cover memory barriers (`VkMemoryBarrier`,
`VkBufferMemoryBarrier` and `VkImageMemoryBarrier`) as well as some of the fundamentals that will be
important later on. This is a bird's eye view approach and not intended to ultimately replace careful
reading of the documentation and standard. I definitely won't be able to summarize everything the
standard offers on this topic, but hopefully a good chunk of it should be immediately more accessible
to you.

(Part II is [out!](/c++/graphics/gpu/vulkan/2018/11/23/vulkan-synchonization-primer-part-ii.html))

## The "Factory"

Suppose you own a factory that makes a complicated item, the flurble. There are many types of flurbles,
each type with different constituent components and unique assembly instructions. Furthermore, market
demand for a specific type of flurbles can fluctuate greatly, so you often have to change up the type
of flurbles being generated from the factory. To make matters worse, the flurble factory is located in
a remote region, and instructions between you and the factory are delivered via email. The workers at
the flurble factory are precise workers, and will follow your instructions to the letter.

As the principal flurble designer, you need to deliver your directions precisely. Otherwise, you risk
requiring a lot of back-and-forth. Furthermore, if you don't deliver an _efficient_ set of instructions,
you risk the factory churning out fewer flurbles than you might like. Let's look at a hypothetical
example to see what I mean. Suppose you design a flurble that has three parts, $$F_a$$, $$F_b$$, and $$F_c$$.
To combine them, $$F_a$$ plugs into $$F_b$$, and the combination $$F_{ab}$$ plugins into $$F_c$$ to make the
finished product, $$F_{abc}$$.

A very poor way to create this product might look like the following.

$$F_a \rightarrow F_b \rightarrow F_{ab} \rightarrow F_c \rightarrow F_{abc}$$

In my made up notation, the equation above prescribes a purely sequential set of instructions, and combined
subscripts like $$F_{ab}$$ imply a combination effect between two or more dependent objects. Creating
$${F_b}$$ doesn't happen until $$F_a$$ is finished for example. $$F_c$$ is made until just before it's
needed at the very end. It should be pretty clear that we can do a lot better than this. For example,
we could have different people at the factory create the separate flurble parts $$F_a$$ through $$F_c$$
in _parallel_, and do the assembly afterwards. To annotate this, we might describe this variant like so:

$$\begin{bmatrix} F_a \\ F_b \\ F_c \end{bmatrix} \rightarrow F_{ab} \rightarrow F_{abc} $$

This seems better in the sense that $$F_a$$, $$F_b$$, and $$F_c$$ can all be built in parallel, but
if you stop and stare for a bit, you should be able to find problems with this description as well.

The issue is, of course, that we don't want to wait for $$F_c$$ to finish construction before
starting the assembly of $$F_{ab}$$. We like that the creation of $$F_c$$ starts in parallel with the
other components, but unfortunately, our description is still overconstrained, and we've introduced
a potential _pipeline stall_ in our program.

## From Flurble-land to GPU-land

I won't carry the analogy any further, because hopefully, the simple example above should clarify
the types of issues you might encounter. First, I'll summarize the GPU's execution and memory model
a bit:

- Every command we issue to the GPU may read or write memory in some way, shape, or form
- Every command is submitted in a `command buffer` which determines a dispatch order between all
  the commands inside the buffer
- Each command buffer is submitted to a particular queue, and different GPUs may or may not have
  multiple queues
- Different queues might only support a subset of the available commands (e.g. a dedicated transfer
  queue supports image transfers and buffer transfers)
- The GPU operates in a well-defined pipeline, and any given command may or may not have work for
  each stage of the pipeline

And things you need to worry about:

- Just because a command was submitted early in a buffer doesn't mean it has finished all of its
  instructions before a command submitted later
- Just because a command buffer was submitted to a queue earlier than other command buffers on the
  same queue, doesn't mean that all of its commands (or indeed, any of them) are guarantted to be
  finished before a later command buffer starts
- Command buffers submitted on different queues similarly do not provide any guarantees

Why does the world operate this way? Well, this is where having an understanding of the CPU's memory
model can help. On the CPU, we also need to be worried about read and write hazards. If a thread
writes data to a particular memory address, how can we be sure that a thread (that may be running on
a different core) can read that same memory address in a well-defined way? The answer is that we need
some form of synchronization primitive. If you're familiar already with `std::memory_order`, you'll
know that most processors provide different types of memory barriers. The memory barriers on the CPU
impose restrictions of varying levels of strictness. These barriers, when executed
essentially say something akin to "wait until all writes executed prior to this point are visible"
before continuing (in reality, there are different types of barriers that impose different levels of
consistency).

On the GPU though, things are a bit more complicated than the CPU. The CPU is also deeply pipelined,
but in general, the programmer doesn't think about the different pipeline stages. The entire thing is
more or less treated as a black box. Also, full memory/execution barriers on the GPU are _very_
expensive. The GPU's pipelines are deep and comparatively expensive to run compared to the CPU's
pipeline. For example, just rasterization of triangles alone is a boatload of instructions and occupies
its own stage in the GPU's pipeline. This is another way of saying that the GPU is optimized for
throughput; at least, relative to the CPU. The final difference we'll consider (there are more, but
arguably less important differences) is that the GPU has many different _types_ of memory writes.
For example, a fragment shader might write to a framebuffer. Alternatively, a command might transition
an image buffer from one memory layout to another (for more optimized sampling, or swap chain
presentation). Maybe a vertex shader writes out transform feedback data to a storage buffer. Thus,
when we issue a barrier that says "make _all_ writes prior to this point visible," this could be a
very expensive barrier indeed, since all the various buffers now need to perform all necessary
operations (some of which are fairly expensive indeed) before continuing.

The way we have to approach things then, is a much more _explicit_ barrier. We need a `barrier` that says:
given a pipeline stage and all the types of memory we're about to care about, make sure we're
finished with those writes before proceeding to access this type of memory at these specific
stages. A bit of a mouthful? Let's look at an example:

```c++
vk::MemoryBarrier barrier{
    vk::AccessFlagBits::eTransferWrite,      // Src access mask
    vk::AccessFlagBits::eVertexAttributeRead // Dst access mask
};

command_buffer.pipelineBarrier(
    vk::PipelineStageFlagBits::eTransfer,   // Src pipeline stage mask
    vk::PipelineStageFlagBits::eVertexInput // Dst pipeline stage mask
    1,                                      // Memory barrier count
    &barrier                                // Memory barriers
);
```

We can read this code in two parts. First, the memory barrier defines what memory must be
visible (here, transferred memory from something like a `vkCmdBufferCopy`) to where (subsequent
commands that rely on reading vertex attribute memory). The second part, the `vkCmdPipelineBarrier`,
informs the driver that the memory barrier kicks in when we reach the vertex input stage of the
pipeline, and the relevant memory written by the transfer stage must have been published at
this point in time. This barrier applies to _all_ commands submitted prior to the same
command buffer, and _all_ commands submitted in a different buffer earlier on the same queue.
Remember also that each command may or may not invoke each pipeline stage. In this example,
if commands submitted before the barrier did not have a `transfer` stage, they would not
factor into the execution of the barrier. Similarly, commands submitted after the barrier
that do not have the `VertexInput` stage enabled will execute as though this barrier didn't
exist. The memory and pipeline barrier together then, can be thought of as a specification of
"filters" that define dependencies between a subset of commands submitted after to a subset
of commands submiitted before.

We should now be able to simply _read_ arbitrary barriers and understand what information
they encode (regardless of whether or not they make sense). For example:

```c++
vk::MemoryBarrier barrier{
    vk::AccessFlagBits::eUniformWrite |
        vk::AccessFlagBits::eDepthStencilAttachmentWrite, // Src access mask
    vk::AccessFlagBits::eIndexRead                        // Dst access mask
};

command_buffer.pipelineBarrier(
    vk::PipelineStageFlagBits::eEarlyFragmentTests, // Src pipeline stage mask
    vk::PipelineStageFlagBits::eGeometryShader |
        vk::PipelineStageFlagBits::eTransfer        // Dst pipeline stage mask
    1,                                              // Memory barrier count
    &barrier                                        // Memory barriers
);
```

This "nonsense" barrier basically reads like "before trying to read any index buffers from
either the transfer or geometry shading stage (or later), make sure that writes to any uniforms and
depth-stencil attachments from the early fragment tests stage (or earlier) have completed."

We should include a few caveats. First, masking memory access bits for a stage that the
stage doesn't actually use doesn't make a lot of sense. For example, defining a memory
barrier that waits for all shader writes from late fragment tests stage is odd because
no shader invocation happens in that pipeline stage whatsover. Furthermore, it doesn't make
sense to place a barrier between two stages within a render pass where the source stage 
happens _after_ the destination stage in the pipeline. Third, it doesn't make sense 
to mask access bits that correspond to reads (e.g. `eShaderRead`) in the source access mask. Reading from
memory from different stages without a write issued in between is well-defined and requires
no barrier. Last, the scopes defined by a pipeline barrier refer to commands submitted
_prior_ to the barrier and commands submitted after. Thus, in the above example, if you
were to submit the pipeline barrier after a draw command that uses the geometry shader,
the barrier won't apply to it (and you may be violating a memory hazard if that shader
accessed a uniform).

## Multiple Queue Submission

The definitions above applied to submissions that occurred on the same queue. As we
mentioned earlier though, GPUs have multiple queues. Most generally, they will have
some number (0 or more) graphics, compute, and transfer queues. Submitting to multiple
queues unlocks parallel submission on one hand, but on the other hand, there is now an
entirely separate class of possible memory hazards we need to watch out for. Going back
to the Flurble factory, imagine if there were multiple people emailing the factory
workers with interdependent instructions. Of course, we'd like to not have to deal
with this complexity at all, but in general, applications can be hugely bandwidth-bound,
and this abstraction simply offers too much performance to leave off the table.

There are two main options for synchronizing work between queues. First, there is
the broad-phase synchronization known as the `VkSemaphore`. The way it works is much
like the standard OS-provided semaphore. Semaphores to be signaled are submitted
to one queue at submission time, and semaphores are submitted to a different queue
to be waited on (also at submission time). I call this "broad-phase" for pretty obvious
reasons; it's heavy-weight and blocks all execution on the second queue until all
operations of the first queue finish. Sometimes, this is simply exactly what you want.
For example, finishing all rendering on a graphics queue before attempting to send
the framebuffer to the presentation queue.

Other times, we need a finer grained form of synchronization. The most common
examples of this are a transfer job to submit buffers or images to the GPU that
get consumed later by the graphics queue. Alternatively, we might have interdependencies
between the graphics queue and the compute queue or vice-versa. Luckily, encoding
this information is not so difficult as we have two memory barrier types for dealing
with these cases specifically: the `VkBufferMemoryBarrier` and `VkImageMemoryBarrier`.
Both of these structures contain member fields to encode the source and destination
queue families. These barriers can also be used usefully on a single queue since they
let you encode a barrier on a sub-region of either the buffer or image, or an image
format transition in addition to every thing else in the case of an image barrier.
When used to describe a queue transfer however, these barriers need to be submitted
to _both queues_ with the source and destination queue reversed. Depending on the
queue they are submitted to, the barriers define a release or consume dependency
between the queues. Another difference is that when these barriers are used to describe
a queue ownership transfer, when a _release_ is defined, the `dstStageMask` is ignored -
after all, the commands submitted afterwards in the _same queue_ do not care about
the barrier. Similarly, the `srcStageMask` is ignored in the consume operation on the
other side for an analogous reason.

## Render Passes and Subpasses, Fences, and Events

Render passes operate similarly to the other barriers but are much
more convenient to use in the context of draw commands and optimizations for tiled
renderers. Fences and events will have to wait for later as well, but in all, I expect
any subsequent discussion to be a bit easier once the contents in this post are firmly
grasped (these topics are covered in [part ii](/c++/graphics/gpu/vulkan/2018/11/23/vulkan-synchonization-primer-part-ii.html)).

## Conclusion

And that's it for this part of the "primer." Hopefully, this much is enough that you can reason about
when and why dependencies occur in Vulkan (or any other modern graphics API), and how
to encode them. To actually use them effectively in the wild, remember not to encode
redundant dependencies. For example, if $$C$$ depends on both $$A$$ and $$B$$, but
$$B$$ depends on $$A$$, you can encode this using two dependencies $$A \rightarrow B$$
and $$B \rightarrow C$$. The dependency $$A \rightarrow C$$ is redundant. Also, trying
to get a perfect representation of all your resources in the application is often
counterproductive. It's better to think of synchronization as necessary for making
your application correct, but not in and of themselves free. After all, it takes
some amount of computational work to evaluate the dependency graph for the driver as
well. How granular your dependencies should be is definitely outside the scope of this
article, but experimentation is encouraged, and personally, I would opt for less granularity
upfront, with additional changes once profiling has identified a bubble. As a final
point, if you've tried reading the spec before and perhaps got discouraged or disinterested,
you may want to try giving it another go :). As always, feedback is welcome and you
can reach me via email or twitter at the links below.
