---
layout: post
title: How I Evaluate Game Engines
date: 2023-09-15 00:00
categories:
- Game Engines
---

I've had to evaluate and understand different game engines and game architectures for myriad purposes in my career. In no particular order, I've evaluated engines to:

- Understand a custom engine as part of my onboarding to a new role
- Weigh different tradeoffs for a new title
- Evaluate an existing choice for a title in-progress

While I'll refrain from commenting on specific commercial engines (e.g. Unreal, Unity), open source engines (e.g. Godot, Bevy, O3DE), and custom engines (to remain unnamed),
I wanted to jot down my thoughts on how I evaluate game engines. If you haven't done this exercise yourself, you might be surprised at what a "game engine veteran"
believes is important compared to your original assumptions.

Before we continue, let's get some disclaimers out of the way. My background is primarily AA and AAA, working with teams that accommodated dozens to hundreds of engineers,
and often more artists in a 3:1 ratio or higher. While I primarily focus on graphics and GPUs (3D in particular), I've also been in positions where I've needed to evaluate game engines
holistically as a generalist, considering not only technical factors, but business and operations also. To the extent that your needs, cost structure, organization, and
goals differ to mine, you should absolutely use your own judgement. In other words, consider my points, and then decide for yourself how much weight to assign each one
before making important decisions. Finally, as with most of my other posts, this post focuses on the technical aspects, which you will need to weigh against merits and demerits
with respect to engine licenser, terms of service, pricing, and other factors. As far as technical aspects go, I'm going to assume that your candidate engines can do roughly
all the things you need it to do, and runs on your target platform. We're interested in evaluating one engine against another, assuming that such prerequisites are met.
With that out of the way, let's get started with what I consider to be the most important factor.

## Production scalability

Scalability comes in many shapes and sizes. Often, when engineers think about scalability, we think of it in a computing sense.
How well does this algorithm scale as the input size changes? How well does my web application service requests when I add more instances?
By a game engine's "production scale," what I mean is, how well does the production efficiency scale, as more engineers, artists, and designers are added to the team?
This matters, of course, because just like film, typical game development cycles operate on schedules negotiated between studios and publishers.
Even if the studio self-publishes a game, capital available to build a game is rarely infinite.

If you aren't in the games industry yourself, or maybe just starting, you might think that such a concern is "not really an engineer's concern."
In fact, nothing could be further from the truth. Game engines live or die by their ability to ship games after all, and the ability for a game engine
to support the constant churn of gameplay iteration, content editions, level expansion, and more, is one of its most critical value propositions.

To put it in perspective, consider a "level" or "room" in any recent game you might have played. Chances are, this level underwent changes
from individuals occupying the following roles:

1. Level Designer
2. Environment Artist
3. Lighting Artist
4. Gameplay Designer

The assets associated with the level might be:

1. Static geometry/props
2. Static and dynamic lights
3. Animated/scripted props
4. Navigation mesh
5. Reflection probes (cubemap captures)
6. Proxy geometry
7. Scripted triggers/sequences
8. Textures
9. Heightmaps
10. Skybox

and more. As levels get more complex, more people and teams get involved. Regardless of which game engine you pick, you need to be able to define
a workflow and asset system that can accommodate all these individuals working in tandem without everyone stepping on each others toes, and without
pulling your hair out.

This problem may be easier for some studios based on size and level complexity, and much harder for others. Off the top of my head, if your game requires:

1. Streaming (e.g. open world)
2. Destruction
3. Time-of-day
4. Procedurally-generated content
5. User-generated content
6. Heavy simulation elements

...expect to expend far more energy ensuring your workflows make sense. The interactions between the asset types and individuals working in the engine
are complicated, problems _will_ crop up in spite of your best efforts. Here are some example pain points I've encountered over the years:

1. Material shaders taking so long to build, the studio-wide shader cache is perpetually out-of-date.
2. Lighting artists unable to get anything done because of iteration on the scene exposure controls, forcing constant recalibration.
3. Long level build times due to very conservatively cached Houdini procgen

So, what makes one game engine exhibit "production scale" in a greater capacity than another? When I'm evaluating this, the main thing
I want to understand is how the game engine _structures the world into files_. It's really not much more complicated than that. The "file" (binary, text, or otherwise)
is the atomic unit that artists, engineers, and designers alike check into source control.
These files correspond to the assets I mentioned above (not in one-to-one fashion of course) and are frequently added, removed, and modified.
Crucially, these files reference each other, often forming a sort of hierarchy of assets.

Before embarking on developing a game with a game engine, _you want to understand this asset structure well_. Without a good mental model
of how things are tied together, you won't be able to understand where future pitfalls may lie. In contrast, a good mental model will inform
you, in advance, how well the engine maps to the type of content you want to build, with the team and outsourcers you intend on deploying.

With that understanding, you can now anticipate failure modes during production. How is referential integrity maintained? How are conflicts resolved?
How are VCS-locked files handled? How much synchronization do you anticipate your studio needing?

Additional things I pay particularly close attention to are how textures, geometry, and shaders are managed, partially because that's in my domain expertise, but also because
those three categories account for the lion's share of data and processing time across the entire studio. When I first got into game engine programming,
someone told me that "game engine programming is like building the rocket while traveling in it." Extrapolating from that statement, it's definitely helpful
if a good chunk of that rocket is already set up more or less how you need it.

## Iteration time

In the same vein as production scale, another important metric is iteration time. This is pretty self-explanatory, but good and bad iteration times can manifest in different ways.

1. Code build time (how long after a code modification is the change visible)
2. Asset build time (how long after X asset is exported from Maya/3DSMax/Blender/Substance/Photoshop/etc. is the asset available and observable in engine)
3. Deploy time (how long does it take to see the game on the target desktop/console/device)
4. Scripting (whether gameplay scripting, FX, or some other system, how much behavior is modifiable without any compilation whatsoever)

I have consulted for studios where after making a change, an engineer would literally "watch paint dry" for a solid minute or more for what was a routine task.
This workflow is, in my opinion, not acceptable and would need immediate (or at least high priority) correction. It isn't _quite_ as bad as a production fire where
the game doesn't launch at all or the build is broken of course, but the wasted minutes add up quickly across the studio. In particular for designers and artists,
I believe anecdotally that rapid iteration is needed to facilitate the creative process. Long feedback cycles of any sort results in frustration, and ultimately, a bad product.

## Runtime scalability

Along with production scalability and iteration time, runtime scalability is the other non-negotiable.
By "runtime scalability" what I mean precisely is the degree to which the engine can run the game on a given hardware configuration at the required framerate.
Generally, the content fidelity will be authored to the "top-end" hardware specification, and it is the engine's responsibility to scale down to the title's min-spec.
Common content scalability mechanisms include:

1. Texture MIP or geometry LOD bias
2. Reduced animation playback rates depending on character distance
3. Turning off various features or substituting lower-fidelity but more efficient features

In addition, to various content-oriented knobs as above, game engines should also scale in the sense that they utilize the hardware available.

1. CPU
2. GPU
3. Available system and video memory
4. I/O

I'm not going to even try here to explain how you might evaluate, say, rendering, physics, animation, or any other engine subsystem, so if in doubt,
create a relevant torture test you believe represents what your game may require and measure.

I pay particularly close attention to resource utilization in the ballpark of the current console generation configuration.
Speaking of which, if the game engine doesn't ship with well-tested integrations with available profiling tools or custom profiling tools, I would be suspicious
about performance claims.

There are specific events that, in my experience, game engines tend to really struggle with (for reasons that are well-beyond the scope of this article).

1. Spawning an actor/object in the world
2. Despawning an actor/object from the world
3. Traversing around the world at X velocity

Taking coarse measurements around these types of events tend to give me a sense of how much the engine will struggle, because these events are the precursor
to downstream system work that is difficult to parallelize (updating your scene BSPs/Octrees, updating your physics acceleration structures, updating the navmesh, etc.).
Of course, this type of thing matters a lot more for open world games. For corridor shooters, shoebox fighters, or other game genres, you'd of course key in on the
performance sensitive aspects relative to you.

Speaking of open world games, if you're in the business of trying to make one, do yourself a favor and estimate from your concept art the approximate object density,
set the velocity to the peak traversal speed you imagine designers will need (accounting for vehicles and wind and all that), and take the engine for a spin.
Congratulations, you now have a baseline measurement to indicate how much pain you'll be in store for later 😀.

In addition to profiling to understand broader multi-frame events, it's important to also understand roughly the "anatomy of a frame" in the engine.
Some engines have a global execution graph. Other engines are structured into "phases" which execute sequentially with respect to each other, but process concurrent work within each phase.
Some engines overlap work from the previous frame to the next frame. When I initially profile an engine, my overarching goal is to understand the overall frame structure,
asynchronous mechanisms, how I/O units are fed, and how the GPU is fed. With that understanding, I can usually compose pathological scenarios I imagine the engine will
struggle with. If those pathological scenarios are unlikely to be needed for my game, great, I can move on. Otherwise, further investigation is necessary to test
the hypothesis. None of that analysis can happen without that "bird's eye view" first though.

## Nice-to-haves

So far, I have only talked about what I consider to be absolute non-negotiables. Of course, you can weight the items above accordingly depending on your needs
and studio size. Beyond that, there is a _litany_ of items that might matter to some, and might not matter to others. Indeed, I have often found myself looking
for a particular "non-negotiable" selling point for an engine not listed above, but required for a particular use-case.

In any case, here's a small sampling of things that might or might not matter to you:

1. Specific features (in the realm of graphics, audio, etc.)
2. Integration with a particular DCC tool
3. Usage of a particular programming language
4. Source availability
5. GGPO/rollback networking support
6. Anti-cheat mechanisms

I'm definitely not going to enumerate more, because this post would become unbearably long. But the important point here is that *if* you need some
sort of functionality in the engine, you'll want to be sure that the engine is extensible in the area you need it to be. Every engine I've worked with
has had some form of plugin system or module system. The question is whether the plugin system accommodates your particular needs, for the areas you anticipate needing to augment the engine's capabilities.
In my case, I would be comfortable evaluating an engine's extension capabilities with respect to artist rendering needs, but I would likely defer
to another specialist if I knew the title needed one-off features in a different domain.

## Conclusion

If there was a takeaway, I hope people read this understanding that evaluating an engine goes well beyond just taking a checklist and confirming item-by-item if an engine fits the bill.
In reality, no engine is going to fit your needs exactly, and some effort will be needed to make an engine work for your team and development goals.
I encourage engine evaluators to focus on the fundamentals first, and only after establishing that those needs are met (or confirming that there are ways to mitigate problems that surface during discovery)
look at all the shiny things game engines have to offer. As a final hint, I always look at the biggest titles that ship with an engine, and work backwards from any published figures on
production budget, team size, and production timeline that are available. If the engine in question has games similar to yours that shipped in a timeline you can accommodate (accounting for
engine ramp-up time), you're in good shape (potentially). Otherwise, be prepared to spend more time upfront doing independent evaluation and consulting other expert users you know. It will be time well-spent.