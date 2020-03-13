---
layout: post
title: Vulkan Synchronization Primer - Part II
categories:
  - vulkan
  - graphics
  - rendering
redirect_from:
  - /c++/graphics/gpu/vulkan/2018/11/23/vulkan-synchonization-primer-part-ii.html
---

This is part II (and possibly the final part) on this series titled the *Vulkan Synchronization Primer*. For the first
part, click [here](/c++/graphics/gpu/vulkan/2018/11/22/vulkan-synchronization-primer.html).

In the last part, we introduced the Flurble Factory and the difficulties encountered when trying to compute fine-grained
dependencies asynchronously. We noted that memory barriers are configured across two orthogonal dimensions: pipeline
masks, and access masks. In other words, we could specify that *prior* to certain types of memory access from certain
stages, we require that certain types of memory writes from certain stages were made visible or flushed.

This part expands on the last part and also introduces a few new concepts into the mix, occasionally referring again
to the Flurble Factory (if you skipped to this part, you can get the gist from the first couple paragraphs of my
last post linked above).

## Memory vs Execution Barrier

I glossed over this point a bit in the last part, and before continuing, I really need to address it, as the distinction
between an execution barrier and a memory barrier is an important concept. An execution barrier specifies that
instructions submitted prior to the execution barrier occur before instructions submitted after. To newer programmers,
it might be a surprise that such a concept is even important. To illustrate things, take a look at this snippet of
C++ code:

```c++
int reordering_happens(int num) {
    int numx10 = 10 * num;

    if (num < 0)
    {
        return num;
    }
    return numx10;
}
```

Here, we have a simple function which appears to initialize `numx10` to an integer `10 * num`, check if `num` is negative, and
return either `num` or `numx10` depending on the result. Here's the assembly generated (from godbolt.org gcc 8.2 with -std=c++17 and -O2):

```asm
reordering_happens(int):
        mov     eax, edi
        test    edi, edi
        js      .L1
        lea     eax, [rdi+rdi*4]
        add     eax, eax
.L1:
        ret
```

If you don't grok assembly, the easy way to read this snippet is:

1. First, move the integer argument in register `edi` to the return register `eax`
2. Next, test if `edi` is zero (`test` does a bitwise and between the two operands) and set some CPU flags to encode the result
3. Jump to `.L1` if negative (by checking the sign flag `SF`), effectively just returning the original argument if true
4. If the argument in `edi` (aka `num`) wasn't negative, do a load of `(num * 4 + num)` into `eax` and then add the result to itself (assembly shorthand for multiplying by ten)

In this case, the variable `numx10` isn't used at all (it's been elided), but the point is that the compiler was free to
move the operation of multiplying `num` by 10 behind the branch. In general, the compiler has a good amount of freedom
to move loads and stores of variables around to improve performance. If however, we were to change the type of `numx10` from
an `int` to a `std::atomic<int>`, the compiler would not be able to make this optimization, and the generated code becomes the
following (same flags, -std=c++17 and -O2):

```asm
reordering_happens(int):
        lea     eax, [rdi+rdi*4]
        add     eax, eax
        mov     DWORD PTR [rsp-4], eax
        mfence
        test    edi, edi
        js      .L1
        mov     edi, DWORD PTR [rsp-4]
.L1:
        mov     eax, edi
        ret
```

Here, we see that the usage of an atomic variable (which imposes a memory dependency for stores before and loads behind),
the compiler can no longer move the multiply-by-10 behind the branch. Note that if you try this in the compiler explorer yourself,
you won't see the `mfence` instruction unless you pass the atomic by reference to the function. Even if you use a local
atomic though, you should still see the multiply occur before the branch.

The reason this is relevant is that just as the compiler can reorder code with restrictions on the CPU, the GPU dispatcher
also has a good amount of flexibility. If I submit a draw command and a compute command in the same command buffer, there
is nothing there that informs the driver one must happen before the other or vice versa. If I needed to enforce an ordering
between them, I would need an execution barrier. To sequence the compute command after the draw command, for example, we
could invoke `vkCmdPipelineBarrier` with the source stage mask set to `VK_PIPELINE_STAGE_ALL_GRAPHICS_BIT` and the
destination stage mask set to `VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT` and no memory barriers at all.

## Wait, why do we need memory barriers again?

If we can sequence commands one after another in the execution pipeline, why do we need these complicated memory barriers
at all? The answer is that memory writes are not "instantaneous" in the sense that they don't immediately become visible
everywhere. In general, memory is hierarchical in nature, and to keep things fast, caches of the various memory types are
not flushed every time a write happens. On the CPU, for example, if one thread writes to a variable on a core, it can't
(without additional barriers/fences) immediately be available on a different core (possibly spanning millimeters of distance
in silicon!). Memory barriers are thus a necessary abstraction to ensure reads are consistent *globally* in the context
of a CPU, and in the context of the GPU, we specify which stages of the pipeline need to provide visibility where.

## Wait, why do we need execution barriers again?

:) So tying it all together, let's remind ourselves once and for all, why we need *both* execution and memory barriers.
The only insight you need here really is that *when* the memory barrier happens is just as important as what the contents
of the barrier are. The barrier is, after all, just another command that gets passed down the queue. A barrier submitted
too late won't be very useful at all, since everything that may have tripped a memory hazard has already been dispatched
into the void. If the memory barrier is submitted *without* an execution barrier, there would be a chance that a command
submitted earlier slips past the memory barrier, or vice versa. Thus, we need *both* the memory and the execution barrier
to be submitted together for everything to operate coherently. Fortunately, this is reflected exactly in the API, to wit
memory barriers are passed as optional arguments to the pipeline barrier (which acts as the execution barrier).

Note that the API allows us to submit a pipeline (i.e. execution) barrier *without* any memory barriers supplied. There
is at least one common use case where this is useful. If I submit a command that simply reads a resource that must later
be modified, this is known as a Write-After-Read (aka WAR) hazard. For example, suppose I have a compute job that downsamples
the framebuffer to draw a bloom effect later. I also submit a render pass containing UI that will write into the same
framebuffer, blending on top of it. This is an example of the WAR pattern because the downsampling doesn't modify the
framebuffer directly (it will likely output to other render targets), but the UI pass does. Here, no memory barrier is
needed between the compute job and the UI pass (indeed, there is no memory to enforce visibility on), but we do need
to ensure that the UI pass does not start until the bloom job is done. This can be done by executing the pipeline barrier
with 0 memory barriers attached. It should not be too difficult to come up with other examples where such a technique
is useful.

## Render Passes and Subpasses

"Wait," you might interject, "what about render passes?" Yup, getting to that. Let's consider a simple sequence of draws
that draw 3 objects on the screen. Each of these objects are transparent and happen to overlap one another. Thus, we need
to employ a back-to-front drawing algorithm (i.e. Painter's algorithm) with blending-on in order to draw the desired result.
At this point, hopefully, your spidey-senses are tingly and you should wonder, "how are these draw calls synchronized?"
That is to say, if you just submit them one after another, how are you guaranteed that the first draw will complete before
the second, etc? This is an excellent question. Based on everything we've learned so far, the only way we know how to do
this is to inject a memory barrier between each draw call, ensuring that we wait for color attachment writes from the blend
pipeline stage to complete each time.

Obviously, this would be a huge pain to do for every draw call, and this is where a render pass comes in. All draw calls
are issued inside render passes which provide a set of implicit synchronization guarantees. If you heard other reasons
for render passes, my guess is that they are related somehow to these guarantees, but providing synchronization between
draw calls over a shared set of render targets is the *primary reason render passes exist*. Render passes give you the
following:

1. A way to describe load and store operations at the start and end of each subpass (which normally would require an image
   memory barrier)
2. A way of describing barriers between subpass dependencies
3. An implicit guarantee for draw calls within a subpass to be executed possibly concurrently, but not in a way that
   violates the pipeline order

Unpacking this a bit, at the start of every subpass and render pass, there is an image barrier (much like the ones you
saw in part I) that ensures that all relevant memory that has been modified prior to the start of the pass has been
made visible. Once entering the pass, draws are able to be invoked and scheduled by GPU without needing overly gratuitous
barriers because the sequencing is guaranteed not to violate pipeline order. That is to say, if we start fragment shading
for object $$A$$, we can start vertex processing for object $$B$$, but we can't start fragment shading for $$B$$ until
the fragment shading for $$A$$ has finished (at least, within the same render pass). That way, we know that results
within a render pass (or subpass) will be correct, assuming we set up our draws correctly. Finally, the pass executes
the store command on any of its render targets as necessary, discarding temporary buffers/memory, and readying the
framebuffers up for the next render pass.

If you squint slightly at the definition of a [`VkSubpassDependency`](https://www.khronos.org/registry/vulkan/specs/1.1-extensions/man/html/VkSubpassDependency.html), you should see that
it mirrors the memory barrier very closely. In fact subpass dependencies and memory barriers are one and the same,
except subpass dependencies provide a nicer syntactic sugar for most common operations you might encounter in applications
that use multiple render targets (e.g. deferred shading). 

The only remaining point on render passes worth touching on are the dependency flag bits which can be supplied to a
`VkSubpassDependency`. See [here](https://www.khronos.org/registry/vulkan/specs/1.1-extensions/man/html/VkDependencyFlagBits.html)
for the official scoop. The `VK_DEPENDENCY_BY_REGION_BIT` flag is the primary one I want to touch on (although you are
encouraged to read about the others), as this one comes up the most frequently. Essentially, when I am issuing draw
calls, we generally don't want to violate pipeline order specifically in the case when the later draw would affect the
results of the earlier draw (for example in blending). Astute readers would be correct in thinking that a full barrier
per pipeline stage of the *entire* framebuffer is a bit heavy-handed. If I draw a teapot in the top left corner of the
screen after all, surely that shouldn't interfere with the teapot draw in the bottom right corner. Well, if you thought
this on your own, you should give yourself a pat on the back for paying attention. Indeed, this sort of optimization is
highly relevant in modern GPU architectures that do tiled-rendering. Without getting too far into the weeds, there is a class
of hardware that rasterizes triangles in two passes. The first pass bins triangles into small screen-space tile bins.
The second pass executes the full graphics pipeline on a tile-by-tile basis for each of the triangles binned to a given
tile. Without the `VK_DEPENDENCY_BY_REGION_BIT` set, intermediate render-targets must be fully synchronized and flushed
for each draw, regardless of the tiles they affect. Thankfully, with the bit set, we can inform the driver that our
interaction for a given framebuffer dependency is localized entirely based on where the fragment is being drawn. This
allows (as you might imagine), all sorts of optimizations to come about, including reducing memory requirements for
transient framebuffer storage. Of course, don't set this flag if your usage of the framebuffer dependency is not, in fact,
localized. For example, if you sample the framebuffer inside your fragment shader to do screen space reflections, you
could be accessing arbitrary pixels of your framebuffer regardless of where you are rendering.

## GPU-to-CPU synchronization

Moving forward from here, you should be comfortable reasoning about the various dependencies we can describe to the GPU
that exist between all the various commands that can execute on the GPU. We can describe buffer dependencies, image
dependencies, and framebuffer dependencies (which really are just a special case of memory). We also learned about the
various implicit guarantees and simpler (but similar) API we get with render passes to save a ton of boilerplate.
From this point on then, the remaining concepts (semaphores, fences, and events) should hopefully be straightforward to explain.

The first observation to make to dive into the concepts is that pipeline barriers and memory barriers are submitted
asynchronously (just like any other command we submit to the GPU). This means, for example, that if you record a series
of commands to a buffer, submit the buffer, and then submit a pipeline barrier as well, you aren't actually free to
`free` (heh) that buffer just yet. After all, just because you submitted the pipeline barrier, there's no guarantee
that the barrier itself has made its way through the GPU's command queue. The pipeline barrier is *only* useful for
sequencing actions that are submitted on the same queue, asynchronously to the GPU.

The easiest (and most heavy-handed) way to solve this problem is by using a wait command to wait until the queue you
submitted the commands to is idle, and you will often see this in tutorial code as its a bit easier to wrap your
head around. Unfortunately, this approach does not scale well in the real world; waiting for a queue to be idle
means you cannot submit more work to that queue in the meantime (lest you wind up waiting forever).

The better way of dealing with GPU-to-CPU synchronization is with a fence. Fences can be submitted along with a command
buffer as part of the `VkQueueSubmit` function, and this fence will later be signaled on the *CPU* in a way that is
visible and waitable via `vkWaitForFences`.

## GPU-to-GPU synchronization

We covered this briefly in part I, specifically when we needed to address describing a memory dependency across different
queues. However, sometimes, we need synchronization between different queues on the GPU for something that is not
described easily as a memory dependency. The most common example of this is managing the rendering of a frame, and its
presentation on a different queue (if the graphics and present queues wind up being different). The noun used to describe
this is a `VkSemaphore` and it works very similarly to the `VkFence`. It is submitted to the queue as part of the
`VkSubmitInfo` struct, but instead of waiting for it on the CPU, it is waited on by a different queue (specified in a
different part of the `VkSubmitInfo` struct).

## Events

Vulkan events resemble the "split barrier" which is also available in D3D12. Imagine if you could split the pipeline
barrier into two parts. The first part emits a signal, and the second part waits for the signal with the appropriate
masks to indicate which pipeline stages need to flush what memory. This can be more performant that a a normal barrier
in the event that the barrier includes too much. For example, imagine you had 3 commands you wanted to submit, `A`, `B`,
and `C`, and `C` is dependent on `A`. Suppose further that `A` and `B` were recorded on a separate thread into a
secondary command buffer. How should you synchronize `A` and `C`? One thing you could do is insert a pipeline barrier
between `B` and `C` to flush the memory written by `A`. However, if `B` also writes memory in this pipeline stage,
you'll end up waiting longer to flush more than you might have otherwise. You like the fact that you could record
`A`/`B` and `C` independently on two separate threads on the host CPU, but you don't like that the barrier between
`B` and `C` synchronizes too much. Alternatively, putting a barrier between `A` and `B` may prevent parallelism between
`A` and `B` since they don't technically have any hard dependency between them. The solution is the `VkEvent` which
is more granular. The thread recording `A` and `B` can insert an event signal using `vkCmdEventSignal` between `A` and
`B`, while the thread recording `C` can insert a `vkCmdEventWaits` just before `C` solving all problems as stated.

Events are very powerful, but I strongly encourage first leveraging the render pass and subpass abstractions first,
as these map well to hardware. In addition, fine grained events can occasionally perform worse than a single barrier,
so consider a simpler architecture at first. Remember that if you guard every resource with barriers and fences out
the wazoo, you're slinking back to pre-OpenGL performance and will likely not get much of what Vulkan can offer.

## Conclusion

This concludes part II and likely the final part for now on Vulkan synchronization. It is not a *hard* topic, in my
opinion, but it is certainly opaque and broad. Unlike the CPU, you have to contend with asynchronous dispatch, multiple
programmable pipeline stages (with a distinct order), multiple dispatch queues, and multiple types of memory writes.
As a result, the surface area for describing synchronization dependencies is vast and the API is relatively large
also. It has been my experience though that once you get the *gist* of what the problem is and why the API needs to be
what it is, the documentation and usage of the API becomes much more approachable. As always, if you have any feedback
or corrections, feel free to email or message me using the links below.
