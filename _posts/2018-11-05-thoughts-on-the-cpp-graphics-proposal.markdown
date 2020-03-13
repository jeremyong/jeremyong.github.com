---
layout: post
title: Thoughts on the Cpp Graphics Proposal
categories: C++ graphics
redirect_from:
  - /math/2018/11/05/thoughs-on-the-cpp-graphics-proposal.html
---

Earlier this year (February 2018), I sent [an email](https://groups.google.com/a/isocpp.org/forum/?fromgroups#!topic/sg13/gUr98RZMU7M) to the ISO
SG13 C++ group to the effect of why I felt the C++ graphics proposal was, in short, _not a good idea_. You're welcome to read it if you want,
but this post is an attempt at presenting a more complete and better-organized argument.

## Brief Graphics History Lesson

First, it is important to understand the evolution of graphics APIs as they have evolved over the years. In the last two decades, we've gone through
multiple paradigm shifts over several generations of architecture. As such, reviewing the changes and sheer velocity will give us a good idea of
the challenges any C++ graphics proposal will need to keep in mind.

### Fixed-function Programming

In the very beginning of course, we had a straightfoward fixed-function, immediate mode of rendering. I'm not going to talk about this mode much,
because it is such a violent departure from how modern GPUs (integrated or otherwise) would like to consume data. In short, in this era, programmers
had access to an API that would directly mutate state on the GPU, send data to the GPU, and issue draw commands.

The issue of course is that this style of communication necessitates an absurd amount of round-trip stalls between the GPU and the CPU. Operating
completely in lockstep, bubbles of idle waiting are inevitable on one device or the other.

### Naïve Programmable Pipelines

Next we had what I will call "naïve" programmable pipelines. Not that they aren't productive or a huge improvement over their ancestors; they are. To
wit the mere existence of a "vertex shader" and a "fragment shader" opened up massive opportunities for exploring new shading techniques and
increased productivity. Now, instead of prescribing all operations for each primitive, programmers could create and link a "shader" program which
would later operate on the GPU. Effectively, the innermost loop was offloaded from the CPU to the GPU, removing many round-trip cycles and resultant
latency. This type of evolution would prove to be a common theme for future evolutions of graphics APIs. I would lump into this category of pipeline
OpenGL 2, OpenGLES 2, and DirectX 9.

### "Fatter" and more Robust Programmable Pipelines

The next evolution of graphics APIs brought about yet more expressiveness, primarily in the form of of new ways of reading and writing data between
the GPU and CPU (reads and writes both ways). The main observation here is that graphically impressive content consumed heavily _variable_ data. To
compose a single frame of a film, game, or even application, a program would need to conjure up and stream hundreds of megabytes of vertex data,
texture data, and raw data (for transforming, animating, or blending data). Programmers started to get used to the idea of needing to pack memory
upfront and think about hardware compression availability. We got vertex array objects, texture arrays, more binding points, and more. All this in an
effort to keep the GPU fed. Also, graphics engineers were blessed with far more mechanisms for actually authoring shaders and reading back data from
the GPU. Examples of APIs that allowed this style of code include Opengl 4, OpenGLES 3, and DirectX 11.

### "Approaching Zero Driver Overhead" Style Engines

Of course, that wasn't enough :). Engines and games routinely struggled to hit frame times, partially owing to driver overhead. As materials and
techniques evolved, graphics engineers soon found themselves at the limits of the APIs again, resulting in yet another paradigm of programming. In
the "Approaching Zero Driver Overhead" or AZDO approach , yet more synchronization points between the CPU and GPU were removed. The
previous approach would often access data or mutate a buffer that was not yet streamed to the GPU, or that was currently in-use by the GPU
respectively. Simplifying the problem a bit, protecting against such data-hazard violations were safe-guarded by the driver. As a result, the driver
had to do a fair bit of work through inserted fences, reference counts, hidden copy-on-write accesses, locks, etc. The manner of this overhead also
meant that drivers had a very difficult time scaling to multicore workloads. Trying to opimize draw call submission with an engine architected in the
previous paradigm was generally unproductive. To combat this, AZDO-style engines maximize GPU throughput by queueing work in ways that require as few
touchpoints between the CPU and GPU as possible. For example, dynamic buffer data would be triple buffered to ensure no hidden fences or
copy-on-write behavior would kick in. Furthermore, such buffers would be coalesced into much larger single allocations with offsets passed to the
shader for proper per-object data consumption. For texture data, engines began to turn to either "mega-textures" that were virtually addressed or
arrays of texture-arrays to essentially "bind once" and never require changing texture state bindings again. Finally, draw calls themselves could be
reduced from a draw per-object or per-instance-group to just per-material through usage of "indirect draws," wherein GPU draw commands were codified
in a flat buffer of command data instead of individual function calls.

### Mobile and Desktop Divergence

Meanwhile, mobile hardware and mobile graphics were getting more and more important. Mobile chipsets continued to develop, but were faced with many
challenges to deliver performant hardware accelerated graphics. A primary problem, for example, was that of memory bandwidth. For modern phones, the
pixel density is surprisingly high (especially for high-DPI devices), and drawing the full screen every frame in the traditional "immediate mode
rendering" (IMR) style that discrete graphics cards do would have required more bandwidth than was reasonable for the phone's heat and energy
requirements. As such, phones moved to a new style of rendering known as TBDR (tile-based deferred rendering). In this rendering mode, triangle
fragments were first binned to tiles in screen-space (often small tiles, say 16x16 pixels). In a second pass, all the fragments of each tile would
be shaded, one tile at a tile. This effectively cuts down the amount of memory in the framebuffer that needs to be "hot" at the same time,
especially compared to immediate rendering which may as well invoke random access of framebuffer memory. While this was a great optimization though,
graphics engineers needed to adapt their practices to take this form of rendering into account. It's worth mentioning that prior to mobile, similar
architectures and problems existed in the console space too. Consoles, like mobile, use a unified memory architecture (UMA), and in some cases,
encouraged a tile based rendering approach (e.g. Xbox One's ESRAM was fast but limited in size). In a TBDR world, accidentally introducing a
command such as a readback or data-dependency that could flush the entire pipeline was extremely easy, and often non-trivial to address. In addition,
rendering algorithms began to diverge. Some techniques that performed well on desktop performed poorly on mobile, and vice versa. For example, the
availability of HSR (hidden surface removal) on PowerVR chips (or similar) meant that sorting draws by depth was strictly worse for performance on
mobile. In contrast, IMRs usually benefitted greatly from a depth-prepass to enable fast depth-based culling. Additional considerations were the
usage of the `discard` instruction in a shader program or alpha blending invalidating the HSR optimization.

### "Modern" APIs

I say "modern" with quotes because what is modern today, may not be modern tomorrow. Already we have hardware samplers that are getting ever closer
to a ray-tracing paradigm (and by some definitions, already there depending on your performance target). For the time being, with APIs such as Vulkan, Direct3D 12, and Metal 2, all the implicit memory dependencies and instruction dependencies are more or less in the hands of the developer
(for better or worse). This is, again, a huge paradigm shift which opens up another class of potential rendering architectures. When specifying
fences, memory dependencies, etc becomes the responsibility of the programmer, a large swath of optimizations open up, particular that of multi-
threaded rendering. Locks, fences, and barries no longer implicitly happen, so scaling up rendering across multiple cores is a possibility and
drivers more or less get out of the way (in principal). Of course, this brings about new challenges as engines still often need some form of
backwards compatibility, and a careless approach to injecting the new API may result in _worse_ performance on both fronts. In addition, games
and apps continue to need to account for differences in TBDR and IMR architectures, as well as UMA and non-UMA architectures. For example, render
pass "subpass" dependencies can be used to enable user-land tiled deferred rendering where a gbuffer may have been prohibitively expensive before.
Also, allocation of dynamic buffers needs to account available memory heap types and allocate the correct one for the job.

## History Summary

TL;DR All that is to say that graphics is a volatile space and much has changed in just a couple decades. The progression does not resemble a
guacamole dip where new layers are layered on top. Here, we have entire paradigm shifts, that have often caused entire rewrites of games, graphics
engines, UI frameworks, browser backends, and more. Developers in every one of these disciplines have needed to pour enormous amounts of time and
resources to advance their efforts, while simultaneously needing to support innumerable fallback paths and branching paths for every type of
device under the sun. But back to the subject at hand, we're talking about just a 2D graphics API right? Why does any of the above matter?

## 99 Problems

Let's consider, for just a 2D API, what the C++ standard library now needs to consider for **all** platforms:

- Window surface creation and its configuration
- Creation of a hardware accelerated context of some sort (Assuming you want hardware acceleration? Hopefully?)
- Handling of an event loop so the developer can decide what happens when the user moves the mouse, clicks/taps a pixel, performs a gesture, resizes the windows, etc.
- Actual rasterization of the data, animation, etc.

I think not doing all the above properly will likely result in what I consider a "toy" library, in the sense that I would not ship something to
production relying on it. Handling all the above essentially means replicating software like Skia (but now it's the compiler author's responsibility
to make sure it compiles, runs, and performs well on all supported hardware).

It gets worse though! As the folks that made OpenGL (Khronos) have learned over the past several decades, things get dicey when software
specifications run aground of hardware limitations. That is to say, having a spec is all well and good, but what if the platform you run on can't
support a feature? Here, we get into the murky territory of checking capabilities at runtime and selecting the appropriate code path thereafter.
This is _strange_ for a standardized C++ library, where even SIMD which is nearly ubiquitous is not easily standardizable because of instruction
set availability and variability (ARM NEON vs Intel intrinsics, and then you have the embedded devices that don't support it at all).

Suppose we sat down and scoped it. Minimal window-ing functionality, barebones event-handling, and software rasterization. This could be the
barest lowest common denominator for something we might deem useful. The question is, _do I really want this to bloat every executable I make
and link to the standard library_. Even this minimal cross-section is a fair bit of code, as anyone who has written a software rasterizer will
tell you. And this "small" cross-section actually has, dispproportionately, the _largest_ cross section with OS-level behavior compared to any
other section of the standard. Furthermore, this also introduces a strict _runtime_ dependency on its execution that definitely goes well beyond
any other runtime functionality switching that I'm aware of in the standard.

Now, for a "real" 2D graphics API, it's worth thinking about what your web browser does. To render a typical HTML/CSS page, your browser needs to
know how to handle:

- SVG rendering (for fonts and SVG figures)
- Hardware accelerated animations and transforms
- "Dirty rectangles" layout optimizations (and knowing when applying them is worse)
- Full-fledged event handling for supporting scroll events, gestures, window focus states, backgrounding, etc
- Hit-box detection
- Depth-aware layering so things on top, draw on top
- etc.

And this isn't even accounting for the browser renderer's many mixed-media capabilities (image support out the wazoo, video codecs, an entire
embeddable WebGL-capable canvas, etc). The people that author browser render backends are numerous, skilled, and specialized. They follow the trends I highlighted above, despite not doing
"hardcore 3D rendering" because they have a different set of performance benchmarks and needs. They manage compatibility on an impressive array of
hardware and platforms, and likely maintain hundreds of branching code paths to make any individual feature fast and correct for your device. They
have giant testing suites and compliance tests to make sure they do not break existing functionality. But as great as they are, _they are not your
compiler engineers._

## Conclusion

I think someone should make an "easy" (not necessarily simple, just easy) 2D library that someone can pick up and make buttons, shapes and lines on.
I think that library can introduce concepts that may be useful for a budding graphics engineer down the road. Maybe throw in bezier curves. Maybe
teach something about winding orders. Eventually maybe you kick it up a notch and add some time dependent blending there. In a computational physics
class you learn something about stable numerical integration, what have you.

This thing would be great as a library. And my contention here is just that. Frankly, spending the committee's time trying to even contemplate the
inclusion of graphics into the core of the standard library before fixing other glaring issues feels ridiculous to the point of embarrassment. The greatest strengths
of the language is its ability to compile everywhere, while providing zero-cost abstractions in some cases, and no abstractions at all for the rest.
Even in this respect, however, the language will eventually lose ground if other languages (read. Rust) manage to provide parity in this regard while
improving ergonomics for other things (read. build systems, package management, etc). If the committee really wants to pursue graphics, I think a
better starting point is to first consider the following:

- How should the C++ memory model interoperate with external hardware (possibly not just GPUs, but network cards, audio cards, and more)? This is the
  subject of the "Elsewhere Memory" proposal
- How much "OS" should the C++ standard library actually care about?
- Why is "graphics" in the proposed state so important, and who will maintain it into perpetuity?
- Is the C++ better served with less bloat within the standard itself, and an easier way to access code to integrate with?

As a long-time user of C++, I'm curious as always to see how it develops and have been a longtime proponent of the language in spite of its flaws.
The "spirit of the committee" in this regard though, will be a great tell as to how the language will choose to evolve in the future, and I suspect
play a big part in shaping the language's future demographic as well.
