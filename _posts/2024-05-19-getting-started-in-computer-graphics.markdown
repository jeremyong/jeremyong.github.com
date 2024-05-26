---
layout: post
title: Getting Started In Computer Graphics
date: 2024-05-19 00:00
categories:
  - graphics
---

Years ago, I wrote a somewhat popular ~~Twitter~~ X thread on recommendations for getting started
in the field of computer graphics. I deleted my account some time ago, but have been asked more
than a few times for the same advice, and while I'm happy to offer whatever help I can, I figured
it would be a decent idea to publish my recommendations in blog form this time (FAQ style).

As a disclaimer, this blog post will skew heavily game-centric, since that's where most of my
experience comes from. Graphics programming is a highly sought-after skill in other domains as well,
but you may need to supplement the information in this post with external resources to get a
fuller picture beyond games. More broadly speaking, this post is built on personal experience and
I can't claim to embody the totality of experiences for all graphics programmers. Where possible,
I've tried to incorporate input and feedback from other professionals in the field, but just keep
in mind that there is a spectrum of experiences, and this is just one (medium-sized) data point.

## Why might you want to be a graphics programmer?

The first question to navigate is whether graphics programming is "the field for you." While nobody
can answer this but yourself, I can certainly share what it is _I_ find compelling about graphics,
and perhaps some of this will resonate, and some of it won't.

Graphics programming is _creative_. While there are common solutions to common problems in rendering,
you often end up in situations where existing approaches don't cut it. One day, you're working on a
project with bespoke creative needs that demand a different approach. Or maybe you need a more performant
approximation of a common technique due to other constraints. Maybe you actually have time in the
frame to "juice" things more than what standard techniques provide? Either way, the inputs and outputs
of the problem space are creative in nature, so to be an effective graphics programmer, you too, must
be a creative problem solver.

Graphics programming is _interdisciplinary_. The rendering equation governs a physical process
by which light refracts, reflects, or is otherwise absorbed by participating media in the scene.
Physics dictates how the light propagates. Math drives the approximations, modeling, and solutions
to this equation. Artistry defines the scene itself, backed by tooling to describe the totality
of all light sources and matter in the scene, each and every frame. The toolchain optimizes data
throughout the pipeline, encompassing ideas from compression, compiler optimization, image processing,
and more.

Graphics programming is _evolving_. Computer graphics researchers and practitioners are constantly
assessing and reassessing techniques used throughout the rendering pipeline. Hardware and API generations
come and go. With each inflection point, the field collectively learns about faster and more accurate
approaches to different problems. There isn't any sign of this progress stopping any time soon. While
many problems have been "solved" to a degree, each solution opens new questions, and we still have
plenty of longstanding unsolved problems yet to be cracked.

Graphics programming can be _analytical_. To understand the entire soup-to-nuts pipeline from source art
content to final pixels, you'd need to understand (at a minimum) vector calculus, linear algebra,
differential geometry, statistics, etc. While you could be a perfectly productive graphics programmer
without a complete knowledge of the entire picture (that's most of us), it isn't an exaggeration to
say that these concepts _do_ come up, and knowledge in these fields is a legitimate advantage.

Graphics programming can be _low level_. While graphics programmers seldomly dip into actual hardware
engineering, we work at a scale where small inefficiencies really add up. While we don't operate
under "hard realtime" constraints like some embedded engineers do, we (in games) certainly have
soft realtime deadlines we need to respect to produce a high-quality experience. Some graphics programmers
work hard to improve perf to improve production times for artists working on films. Still others
specialize to squeeze maximal performance out of a particular console. Others build abstractions
designed to scale well on a variety of hardware platforms.

In short, graphics programming is a profession that combines different aspects of creativity
and artistry with low-level quantitative thinking. While challenging, it also offers a unique reward for
your efforts in the form of compelling interactive imagery. Note that the points above
apply in varying amounts depending on the _type_ of graphics programming work you do. I describe
a few of the different roles in real-time graphics [here](https://www.jeremyong.com/graphics/interviewing/2023/08/05/graphics-programmer-interviewing/),
but in reality there are far more specializations that even what is in that article.

On a practical level, I believe graphics programmers have extremely transferable skills. We work
with large amounts of data at scale. We can write code that targets a wide range of hardware and
platforms. We typically have the analytical skills needed to learn and apply new algorithms. We
can debug difficult-to-diagnose problems. Whether we'd _want_ to transfer the skills to a different
domain is definitely a different story.

Day-to-day, graphics engineers often get to work with the best creative and artistic minds in the
world also. There's nothing quite like authoring a system and seeing artists and designers bring it
to life.

## What about the downsides?

I'd be doing you a disservice if I painted a completely rosy picture of graphics engineering.
Plenty of days pass without solving glamorous math problems to produce gorgeous pictures in
code. As a hint of what's in store, you'll have days where:

- You're stuck watching the paint dry (engine compilation, shader compilation, texture compression,
  and mesh baking are just a few processes you'll iterate on and run time and time again).
- You have a valid solution but need to find a different solution anyways due to driver bugs or
  some other limitation outside your control.
- You have a problem and no hints as to how to proceed because the problem is either very difficult
  to reproduce, or there is insufficient debugging data, or both.
- You can solve the problem but the solution amounts to menial data plumbing through obtuse APIs.
- You waste hours trying to apply a technique only to realize that there was a bug in the
  documentation.
- You do normal mundane software things (like fighting your version control system or other infrastructure woes).

Of course, many of these issues are par for the course in software engineering,
and while these days of frustration do exist, there are days we can spend improving tooling,
documentation, or processes to make the "bad days" more bearable in the future.

Another downside is that most people will not understand what it is you do. Fortunately for
me, I don't care much at all. When I tell people I'm a "graphics programmer," they usually assume
I'm an artist of some sort, and I pretty much never have the energy to correct them. I suspect
this is true for most other specialized disciplines as well -- your average layperson won't
know what an "algebraic topologist" does either.

There are some practical downsides also. Graphics programming is a difficult field to get
started in. For the most part, C++ competency is a given and from there, the path to becoming
a competent graphics programmer is a very real grind. While it's very unlikely that anyone will
ask you to work overtime, I think it's just as unlikely that average graphics programmers
truly constrain their working hours to standard 40 hour weeks, at least when starting out.
I am _definitely_ not saying you need to work overtime to succeed. However, I want to be transparent
and mention that I personally have worked overtime, and so have most of the people I have worked with.
Empirically, the field appears to attract individuals that lack the discipline to maintain
"work-life" balance, although I have no real numbers to back this up.

Finally, as a game-specific downside, it's quite possible to do an amazing job as a graphics
programmer, and still ship a title that doesn't succeed. Graphics is one component of many, and
will often not lie directly on the critical path to success.

## What are the prerequisites?

While doing research-level graphics work is certainly not for the faint of heart, you certainly
don't need all that mathematical machinery to get started. At a minimum, I would suggest you:

- Have a decent grasp of C++ (the "lingua franca" of graphics programming)
- Grok linear algebra to a certain degree

While you can certainly learn graphics programming with whatever programming language you please,
inevitably, you will run into projects, work, or research authored in C++. That's not going to
change any time soon, so make of that what you will. (I have little interest in waging programming
language wars, and if I did, I suspect I'd be in a different line of work entirely). Learning and
using C++ is a pragmatic decision that will get you to pushing triangles and textures through a
GPU to your screen, and even if you _know_ you'll be working on a non-C++ project, it will still
be useful to _read_ work authored in C++ by others.

By "grok linear algebra," I just mean that you understand matrices as composable linear maps, which
you'll initially use to model everything in the scene (camera, objects, light sources, etc.).
Your bread-and-butter matrices will typically be orthogonal transforms and projections (or compositions
thereof).

It's certainly helpful but not required to have a decent grasp of calculus as well. Derivatives
and integrals show up all over the place in computer graphics. Derivatives are often taken using
finite differences to filter textures. We use them also to compute surface orientations and define
surface parameterizations. The rendering equation itself is an integral (a multidimensional recursive
one at that), and more specialized integrals abound for solving specific cases, like approximating
an environment light computation, subsurface scattering, cloud or fluids rendering, and more.

## What are the hallmarks of a good graphics engineer?

This is a broad enough question to warrant a dedicated post, but aside from general traits any good
engineer should possess, excellent graphics engineers tend to exhibit other traits:

### Ability to reason about deep pipelines

In graphics programming, there is a lot of "time and space" between the code expressing that we "need to do something"
and the thing "actually happening." The pipelining is necessary because we are juggling multiple timelines (CPU threads, GPU)
and for efficiency. Being able to mentally organize how the pipeline is constructed, make new pipelines, and refine them
as needed is an important skill to avoid being lost. Note that working with deep pipelines requires a knack for debugging
or writing tools that aid debugging, as graphics bugs are anything but a breakpoint or printf away from a solution (usually).

### Ability to communicate

While being able to communicate to other engineers is a must, we must also learn to communicate with artists.
Our customers are often artists, who themselves need to understand constraints that we establish.
Through documentation, verbal communication, tooling, and other mechanisms, part of our role is to
ensure that artists understand how to maximize the tech at their disposal, without running afoul
of scalability footguns or other problems that present as visual artifacts.

### Grit

There are bugs in graphics programming that require serious amounts of patience and dedication to
solve. I have worked on bugs that took me weeks to resolve to satisfaction, and this isn't a unique
experience. Getting "stuck" is quite common when tackling a problem, and it's important to know how
to ask the right questions in order to make forward progress.

### Ability to think in abstractions

As with other complicated engineering systems, "the renderer" is a beast that's often too big to hold
in one's head in its entirety. To be effective, we need to be able to treat various subsystems as opaque
boxes, reasoning primarily based on inputs/outputs and execution properties. Knowing when to operate
at what level of abstraction is a key efficiency boost. Conversely, over-abstracting or not diving below
an abstraction to drive a concrete solution home are also problems good graphics engineers avoid.

## How should I get started?

There are at least two possible routes here that a beginner could take. On the one extreme, you could
work on a software path tracer or rasterizer to develop an understanding of lighting models,
texture filtering, bireflectance distribution functions, etc. On the other extreme, you could work
on a realtime renderer to learn how to drive a GPU with data and shaders, developing an understanding
of the GPU memory model, execution model, and systems. Both approaches are complementary, or you could
combine them into a single (ambitious) project.

For the first method, do yourself a favor and _work through_ [Physcially Based Rendering: From Theory to Implementation](https://pbrt.org/),
implementing at least the first half or so. The full contents are available for free online, and it
really is the most comprehensive treatment of the subject. I have owned three different physical copies
of this book over the years, and years after the first graphics program I wrote, I still routinely
keep PBRT handy on my desk to refer to from time to time. While a software path tracer is of limited
use in real-time games, I still think the experience is valuable to gain a good intuition on the algorithms
and math behind lighting models, without getting too caught up in the details of "programming a GPU."

For the second method, you have a number of options, but my recommendation is to run with MSFT's
[DirectX-Graphics-Samples](https://github.com/microsoft/DirectX-Graphics-Samples) repo. The samples
are easy to build on Windows (double-click a solution file, run and compile). Specifically, start
with the [Hello World](https://github.com/microsoft/DirectX-Graphics-Samples/tree/master/Samples/Desktop/D3D12HelloWorld)
sample and try to follow the code and understand how it comes together. I recommend also checking out
the helpful tutorial [Using D3D12 in 2022](https://alextardif.com/DX12Tutorial.html) from Alex Tardiff.

In reality, you can pursue both methods concurrently, in accordance to your interests and motivations.
To see some examples of projects that resemble a slice of what a "real game renderer" might look like,
you can check out the MSFT [MiniEngine](https://github.com/microsoft/DirectX-Graphics-Samples/tree/master/MiniEngine),
AMD's [Cauldron](https://gpuopen.com/cauldron-framework/), or NVIDIA's [Falcor](https://developer.nvidia.com/falcor).

### Aside: Which API and shading language?

Do you start with DX12, DX11, Vulkan (VK), Metal, OpenGL, or something else? Personally, I would start with
DX12 or VK because I have a penchant for pain and I very much want to learn how to control all aspects
of synchronization between the CPU and GPU and memory management. Also, I want access to more recent features.
If you want to use ray-tracing APIs, DX11 is immediately off the table for example. Furthermore, DX12/VK
are APIs that are (a bit) closer to console APIs (if that's what you're interested in).

DX12 and VK impose a higher upfront learning tax to get things going though, so if you're the type to get
impatient working through what feels like boilerplate to get something onscreen, you may find yourself
losing interest quickly. Hence, I recommend working from sample code first instead of trying to stand
everything up yourself (although that is a very valuable experience that will teach you how to debug and
diagnose common issues).

If you use a Mac, use Metal. In my specific segment of the industry, note that the demand for Mac-focused
graphics programmers is very small relative to Windows and console programmers.

(Warning, strong opinions follow). With respect to shading languages, I think GLSL is a dead shading language
(and OpenGL is a dying/dead API).  I've used both professionally, and while they served important roles at
various times in history, you are much better off sticking to HLSL and DX12/VK in my opinion.

Regardless of what API and shader language you choose, however, you _will_ learn and use more APIs in time,
so don't sweat your initial choices too much. What's more important is gaining an appreciation of the
general programming model. In particular, you should try and get a feel for:

- How memory is allocated on the GPU
- How memory is allocated by the driver
- How memory in shaders are bound and referenced via descriptors/views
- How vertices and pixels are computed on in parallel
- How to configure render targets, blend state, depth/stencil state, and other bits of the fixed function pipeline
- How to synchronize memory usage between GPU and CPU timelines
- How different lanes of execution within a shader communicate and synchronize (at different granularities)

Every API and shading language provides different means of expressing the ideas above, so after learning
one mechanism, it usually isn't too hard to transfer that knowledge laterally.

### What projects would you recommend?

The fun thing about graphics is that there's no shortage of interesting demos you can put together. After
you have some of the basics down, I recommend picking up [Real-Time Rendering](https://www.amazon.com/Real-Time-Rendering-Fourth-Tomas-Akenine-M%C3%B6ller/dp/1138627003).
This book is an excellent survey of techniques, many of which are used in the field today, with many
helpful references for further reading. If you're just starting out, I recommend trying to reproduce
existing techniques as opposed to trying to invent something new from the outset. By getting a feel for
how the industry approaches surface modeling, lighting, shadows, GI, texture streaming, and other such
problems, you'll develop an intuition for what works, what doesn't, and why. Eventually, you should find
that rarely are there "slam dunk" techniques. Often, we're navigating the [Pareto front](https://en.wikipedia.org/wiki/Pareto_front),
balancing memory usage, runtime performance, artist flexibility, and other such tradeoffs.

Here are some (_some!_) high-level descriptions of some fundamental problems that graphics programmers face.
I'm not going to even mention what the high level solutions are here, mind you, but hopefully understanding
the problems themselves will serve as a useful motivator to go in one direction or another.

#### Efficient scene submission

Given one or more camera poses, how do we efficiently submit the geometry visible to the scene?
How do we filter geometry outside of the view frustum? How do we filter geometry that is occluded
by other geometry (occlusion)? How do we efficiently sort geometry that needs to be rendered in
a particular order (i.e. as needed in transparency or decal rendering)? Can we leverage a similar
approach for shadow views? How can we efficiently update the scene dynamically? How do we balance
memory usage and IO in cases where the world is too big to be fully resident in memory? Are there
ways we can store representations of our scene on disk to accelerate scene submission at runtime?
Should we have different representations for different aspects of the scene, such as terrain?

#### Material and lighting models

Given any physical object, how should we best represent its surface properties in a way that allows
us to model its appearance? How might we support appearance parameters that update dynamically, or
are defined procedurally? Are there approximations to the model we can accommodate for more efficient
rendering at a distance? How do we handle surfaces that change at runtime in a general way (e.g. supporting
snow or dirt deposits that accumulate on other surfaces)? What about particularly unique surfaces
that exhibit complex scattering like eyes, hair, and skin? How can we represent materials such that
different materials can be tiled and composed to form composite materials?

#### Shadows

Rendering shadows requires solving a many-to-many visibility problem. When and how should we compute
these visibility tests? If using the rasterizer, how should we store the results? At what resolution
should we store them, and how should we sample the results to reduce projective and perspective aliasing?
How can we model penumbras that are present in shadows when the light source is partially occluded?
How do we represent partial occlusion from complex light shapes? Are there efficient ways to store
and compress shadow queries offline in cases where neither shadow casters nor receivers move at runtime?
Are there stochastic approaches (e.g. with ray tracing) that allow us to sample visibility at lower frequency?
What denoising techniques do I need to make stochastic approaches tenable? For soft shadows, are there
mechanisms by which we can store a compact representation of the depth _distribution_ to quickly reconstruct
an estimate of occlusion?

#### Color treatment

Given the wide range of viewing environments, display types, user settings, and users, how should
we treat the color pipeline to ensure a consistent and high quality viewing experience? While content
is being mastered, how should we handle data interchange between various digital content creation (DCC)
tools to ensure artist work remains calibrated? How do we handle in-scene exposure to capture the
dynamic range and compress it into what the display is capable of? How do we clip or otherwise handle
colors that move out of our target gamut? What gamut should we even be rendering in? When applying
our display-out color transforms, how do we avoid objectionable hue shifts? How do we maintain hues
that are desirable for artistic reasons (bucking the "path to white")? In what cases does dithering
make sense to break up banding issues?

#### Streaming

Mesh and texture data for most shipping titles can't reside entirely in VRAM for most target consumer
cards. How do we determine what mesh levels-of-detail (LODs) or texture MIPs are needed? How do we
evict old data? How do we handle memory fragmentation? How are we ensuring that memory we recycle
isn't currently in use on the GPU timeline? Are there ways to avoid more CPU overhead? How do we
transition between LOD levels? How should coarser mesh LODs be generated? How should texture MIPs be 
filtered to produce coarser MIPs? Should we filter different types of image data differently?

#### Indirect lighting

Indirect lighting (aka global illumination) accounts for scene lighting from non-primary light
sources (i.e. light bounced off from other surfaces in the scene). If we choose to cache irradiance,
where and how should we store this? How should we sample and reconstruct radiance later? If we choose
to bake indirect lighting offline, what representation should we use? If we opt for a runtime solution,
how do we ensure our solution scales on different hardware? When sampling our irradiance cache, how
do we account for geometry superimposed with the cache (i.e. how do we avoid "light leaking")? Alternatively,
how might we represent our cache such that light leaking isn't an issue? How do we handle cache invalidation
in the presence of dynamic geometry or light sources?

#### Volumetrics

Smoke, clouds, water, and other fluids all represent a significant rendering challenge. How should
we represent the data? If marching through a volume, how can we reduce the sample count or step count
without overly impacting quality? How do we account for light energy being injected from multiple
directions and sources into the volume? How should we model multiple-scattering?

#### Anti-aliasing

AA is needed anywhere we sample signals containing frequencies greater than the [Nyquist frequency](https://en.wikipedia.org/wiki/Nyquist_frequency).
Without AA, we see stairstepping, noise, and other objectionable artifacts because our sampling
rate is often coupled with our framebuffer resolution. How should we combat aliasing? How do we handle
different sources of aliasing? How can AA be applied without losing clarity in the scene?

In time, you'll come to see and appreciate many different answers to all the questions above and more.
Each answer will have its own implementation quirks, advantages, and disadvantages. It is important
to _implement_ at least some of these solutions yourself, because only through implementation will
you gain a deep intuition about the problem.

## How do I get into the games industry?

The general recommendation is to develop a project or demo that you can show off. It could be a
general purpose renderer that has some basic handling for materials, shadows, and lighting. It
could demonstrate a specific technique (possibly in the vein of some of the problems mentioned above).
It might demonstrate your particular set of choices for abstracting different APIs/platforms. Either
way, the general consensus is that starting off as a graphics programmer without any concrete relevant
experience you can show off is likely a non-starter. Aside from giving you a "hook" you can talk about
in the interview, working on a demo is a good way to validate for yourself that you find the work
interesting.

AAA studios are admittedly more competitive than smaller studios. While there are more junior positions
available, such positions are definitely more scarce. This isn't to say that starting off in AAA is
impossible, but expect that your portfolio and experience will need more to stand out among other
applicants.

Regardless of which studios you apply for however, a common experience among graphics engineers is
the rather large upfront investment in time needed to accrue enough relevant experience. Much of what
I've talked about thus far isn't really covered in schools, unless you happened to major in computer
graphics or game programminng at a university that covered such topics.

## Should I get into the games industry?

The games industry has an understandably bad rap for:

- Bad working hours
- Worse compensation
- Misogyny

There is an element of truth to this, and while I have (mostly) avoided many of these problems in my
career, such problems do exist and it's something to be aware of. If there was a silver lining, it's
that not _all_ studios or teams exhibit these problems. Just as there's a high degree of variability
within the rest of the tech industry, so to is there variability across game studios. If you survey
others in games, you're likely to encounter a range of experiences, and your particular experience
will likely be something entirely unique.

With respect to overtime, game developers are a hard-working bunch, and there are cases where while
there is no explicit overtime expectation, the general working environment can result in a
social pressure to commit overtime hours. This affects different people differently, and I have also
worked in studios that actively try to combat this mentality.

With respect to compensation, the data points I have indicate that graphics engineers enjoy higher-than-average
compensation. This is mainly a consequence of the relative scarcity of good graphics engineers.

All the (potential) negatives aside though, you should absolutely consider the industry if enabling
creators and creating games is something you'd love doing.

## Can I leave the games industry after getting started?

For sure. This does happen from time-to-time, and as I mentioned, your skills developing and debugging
complex applications will serve you well. The most natural lateral move would be from games to film,
but transitioning to high-performance computing, robotics, or other disciplines is definitely viable.

## Should I learn Unity or Unreal?

Getting some familiarity with the tooling is definitely recommended, if only to understand how game
engines choose to structure things and expose features to artists. Both Unity and Unreal make "choices" (TM)
that may be good or bad depending on the needs of the project, and whether you work at a studio that
uses one of these engines or an entirely custom engine, your knowlege of how things work in "an engine"
will help ground your understanding.

## How do I get into console programming?

Console devkits, APIs, and documentation are available under NDA only, so trying to "learn console programming"
outside of a game studio isn't really viable. Generally, it's best to treat low-level graphics APIs like DX12
as a coarse approximation of what console graphics programming is like, and imagine having lower-level access
to data structures that are typically opaque (for platform independence) in addition to working with a unified
memory architecture (UMA). Most studios will expect that hires without prior graphics programming experience
will need to pick up the console-programming bits on-the-job.

# Closing thoughts

Hopefully this blog/FAQ gave you enough information to understand the nature and types of problems 
graphics engineers work on from day-to-day. While the role certainly isn't for everyone, it's perfect
for me, and there are very few roles that are like it. I encourage newcomers to "dive right in" and not
spend too much time being intimidated. Be patient with yourself, disciplined, and consistent. The payoff
of delighting users and empowering artists will come with time, and in my opinion at least, is well worth
the effort.

_Special thanks also to Alex Tardiff, Niklas Lundberg, Matthäus Chajdas, Johan Andersson, Nathan Reed, and
Matt Pettineo for their input as I was drafting this post._