---
layout: post
title: How to Learn Vulkan
date: 2018-03-26 00:11:00 -0800
categories: C++ vulkan graphics rendering
redirect_from:
  - /c++/vulkan/graphics/rendering/2018/03/26/how-to-learn-vulkan.html
---

This blog post is a meta post on the general act of going through the motions in learning Vulkan, and outlines what is hopefully an effective strategy for newer practicitioners. I'll do my best to outline major pitfalls that I encountered on my own, and where I recommend spending the bulk of your time, as well as a rough "timeline." For people already familiar with OpenGL and DirectX, I hope to also explain in plain terms what functionality you were relying on the driver for previous that you are now responsible for (and what that means). I'm _not_ going to try to explain how to solve each one of the problems you will encounter, as this post will get unbearably long as to not serve any real purpose. I _will_ endeavor to provide links to good resources in the community already _and_ explain how to read/utilize them. This blog post is meant to be read/skimmed once, and then bookmarked for reference as you proceed with your journey. If there were resources you think I missed that you think may be worth including, feel free to tweet me (twitter link at the bottom of this page)! So without further ado...

# Assumptions

First, some quick assumptions. I'm going to assume you use C or C++. I'm going to assume you've worked with Direct3D (aka DirectX) or OpenGL before, but not necessarily Direct3D 12 or Metal. I'm going to assume you are interested in learning Vulkan because you heard it performs better, because you want a more viable cross-platform solution (especially now that MoltenSDK is open sourced), or because you like learning shiny things (hey, who doesn't). I'm also going to assume you've done multithreaded programming before (you know what a mutex is, and you've dealt with race conditions and deadlocks). At least the basics.

#### A quick note for people with 0 graphics experience or OpenGL/Direct3D experience

Would I recommend Vulkan for such people? Probably not. There are other blogposts that recommend the contrary, and this is entirely subjective, but in such a case, I believe true beginners in graphics are much more likely to get frustrated programming in Vulkan, not knowing what they are aiming for. Vulkan is less about graphics, and more about GPU drivers at its core, meaning that if your primary goal is understanding how a cascaded shadow map works, or how indirect lighting is done, or screen-space-reflections, you gain nothing by using Vulkan.

Exceptions to the above caveat are people who are convinced they want to do this professionally, in which case a holistic understanding can only help you (although I still strongly recommend learning Vulkan in parallel with running experiments using simpler APIs). Also, people that have masochistic tendencies and/or have a very strong abstraction muscle in their brain will not find too much difficulty. Vulkan is very broad and deep but it is logical. Engineers that don't find themselves struggling when jumping into a very large codebase for the first time will similarly not find too many difficulties here.

#### A quick note for people who aren't C/C++ programmers and want to use Vulkan

I generally advise against this, if only because lower-level explicit control means more foreign-function calls into C-land. For languages that are interpreted languages (excluding Rust, for example), the overhead of the extra function calls will add up and potentially offset the performance gains you may have realized with Vulkan in the first place. This is likely why something like Vulkan is unsuitable as a basis for any future `WebGL` efforts, but don't quote me on this.

# Viable Learning Mentality

First, a few words on the mentality I recommend when approaching Vulkan. You probably already have a good degree of apprehension to learning it, or maybe you've attempted to learn Vulkan before and gave up half-way through. Alternatively, you may have gone through a few tutorials but still feel a bit shaky on concepts that are still in the realm of "black magic." I have a few recommendations on how to adjust your mentality to best maximize your chance for success.

#### Do not assume you aren't making progress if you aren't making _visible_ progress

As graphics engineers, we're used to getting a triangle on screen relatively quickly, and go from there (xform matrices, texture sampling, some simple lighting, and **bam** graphics engine). Most new Vulkan users that have already tried a tutorial or two will likely encounter this mental roadblock and find themselves asking, "why can't I see anything yet!" and get frustrated. Better is to take notes and tally up what you _have_ done. For example:

- I can create a logical representation of the device
- I can create a swapchain
- I can allocate memory on the GPU
- I can express that this chunk of memory in this region should be interpreted as an image in this layout

With Vulkan, you won't be "drawing" things as much as being extremely explicit about all the state necessary for the simple `draw` command at the end to consume all that state and perform its magic. Don't be discouraged if getting to this `draw` takes 5 to 10 times longer than you're used to! Vulkan does not provide "sensible defaults" and makes absolutely zero assumptions about what you as the programmer intends on doing. This is, of course, to bolster performance, as everything is opt-in.

#### You will forget why you did things, or how something works as you learn more

Even for engineers with excellent memory and recall capabilities, you will likely need to constantly backtrack and review why something was necessary or how something was done. The graphics pipeline is extremely deep, and it's easy to lose the big picture once you get stuck in the weeds. The best way to combat this is to proactively pause and remind yourself what you have done and accomplished in order to get to where you currently are. This is especially difficult in the very beginning as a lot of the concepts are orthogonal and have no interdependency. It will be easy to get caught up asking "does X actually need to happen before Y." This is pretty normal also, so just know to expect it and do your best to capture in notes what's going on in broad phases.

#### Expect that there are tons of ways to do the same thing, and that experts haven't necessarily settled on the best ways

With more explicit control, developers in Vulkan have a ton of freedom in expressing how a thing should be drawn (or computation performed). Many different resources will take different approaches, and without understanding the fundamentals, you will not understand what tradeoffs are being made. Keep in mind also that many existing frameworks and engines _ported_ existing functionality to Vulkan, so occasionally, they make abstraction choices to be compatibile with Vulkan. _These decisions are not necessarily optimal!_ I can't emphasize enough how much you must attempt to learn the mental model yourself, or you may find yourself an expert in a super verbose version of OpenGL.

#### You should refer to the spec early and often

It is never to early to refer to the [spec][spec]. When you download the SDK, this is included in the `docs` folder so just bookmark it in your browser to access it quickly. If you haven't done this yet, do it now! You want accessing this to be as low-cost as possible to train yourself to refer to it. I recommend accessing it as you encounter new concepts or Vulkan types (in other words, treat it as a reference). Many sections feel "dry" but compared to reference documentation for other APIs or programming languages, I have found the density of useful information in the Vulkan specification to be quite high and well worth the time invested.

# Preliminaries

- Remind yourself how the graphics pipeline works
  - [Trip through the graphics pipeline][trip]: An excellent must-read tour through the graphics pipeline
  - The graphics rendering pipeline chapter in [Real Time Rendering][rtr] (note: if you don't already have this book, hold off on buying it as the 4th edition is coming out soon!)
- Read [Vulkan in 30 minutes][30min] by baldurk (author of renderdoc, a must-have tool for gfx debugging)
  - Try to map the concepts mentioned in this article to your understanding of the graphics pipeline
- Install all the things!
  - Latest [VulkanSDK][sdk] (run vulkaninfo to convince yourself you have a compatible gfx card and driver)
  - [RenderDoc][renderdoc]
  - If it's your cup of tea, you can also get nVidia nsight or Radeon GPU Profiler (RGP) for perf testing (but I don't believe this is necessary for learning at all)
- Bookmark all the things. You'll be referring to these a LOT so save those clicks
  - [API without Secrets][intel]: Intel's excellent "Hello, world" tutorial with a healthy amount of background information
  - [Vulkan Tutorial][vktut]: Another great Vulkan "Hello, world"
  - [Vulkan Spec][spec]: The Vulkan specification
  - [Vulkan Examples][vkex]: A collection of demos featuring various Vulkan features in a concise and understandable manner
  - [Awesome Vulkan][awesome]: A compendium of Vulkan projects, articles, tutorials, and more
  - [Vulkan Synchronization Primer][primer]: Self-authored short two-part primer for when you want to delve into the topic of Vulkan synchronization more
- Consider subscribing or joining the following Vulkan communities
  - [Vulkan Subreddit][vkreddit]
  - [Vulkan Discord][vkdiscord]

# Vulkan Mental Model

#### Shaders

Before starting any tutorials, take a look at the following vertex shader.

```glsl
#version 450

// Rather contrived shader for illustrative purposes

layout (location = 0) in vec3 position;

layout (location = 1) in vec2 uv;

layout (binding = 0, set = 3) uniform UBO
{
  mat4 projection;
} ubo[4];

layout (binding = 1, set = 1) uniform sampler2D noise_textures[3];

layout (binding = 2) buffer Buffer
{
  mat4 something;
} buffer;

layout (push_constant) uniform Camera {
  vec2 pos;
} camera;

layout (location = 0) out vec3 out_uv;

out gl_PerVertex
{
  vec4 gl_Position;
};

void main()
{
  // shader code
}
```

Specifically, pay attention to the various layout options you have at your disposal. There are `location` specifiers (you'll recognize the inputs to the vertex shader corresponding to vertex buffers and vertex attributes), the `binding` and `set` specifiers, and the `push_constant`. Also notice that some of these layout variables refer to arrays of data (aka 3 `noise_textures` samplers and 4 `uniform` buffer objects). One of the more confusing aspects of Vulkan to the uninitiated is what these all mean, and which Vulkan calls correspond to which. In general, when you learn a new API call or object related to descriptor binding (i.e. [VkWriteDescriptorSet][writedset]), a parameter passed will correspond to one of those `binding`, `set`, or array index numbers. The thing that you bind to that location will be specified as a `buffer`, `uniform`, `sampler2D`, `sampler2Darray`, or what have you depending on the Vulkan object you specify. Existing documentation does **NOT** do a great job of mapping the API parameters/objects to the shader you will ultimately write, so just keep this in mind and cross reference tutorial code with the tutorial shaders. Because tutorials often use 0-indexed everything, you might inadvertently conflate one parameter's value with the value in the shader. After you get something working, try experimenting to make sure you got it. As another tip, when you read the documentation, read words like "binding" and "set" rather literally and consider how you would access them in a shader as a way of "grounding" your understanding.

Later on, when you write your engine, game, demo, or whatever, you'll have a lot of flexibility in how to bind data to shaders, and also how to provision and allocate this memory. As I mentioned before, there are tradeoffs to the various approaches, so don't be afraid do try something new or reimagine how it should work for your workload. Even if you're wrong, the experience doing so will be useful!

#### Render passes and pipelines

Another important concept to understand is the song and dance related to render passes and pipelines. These types of objects were very common to have abstractions for in game and graphics engines already. Chances are, the abstraction you're already used to matches the Vulkan one pretty well, but don't make that assumption. A pipeline here is literally the graphics pipeline. A full specification of all the various stages of the graphics pipeline ([link again][trip]) like what shaders you want to use, what the vertex input format is, if depth testing is enabled, what blend operation you want, etc. The pipeline _depends on_ the render pass because the pipeline needs to know what the required input and output attachments are. Render passes in Vulkan support a handy feature called `subpasses` which allow you to specify in advance multiple stages of rendering that have some set of dependencies amongst each other. This will allow the GPU to schedule non-dependent subpasses independently for some transparent speed-gains. In general, the entire configuration of render passes you provide, their subpasses, and the pipelines (read. pipeline states) that you bind during the execution of each render subpass dictates however all the draws in your frame will occur.

As an aside, a framebuffer is a collection of attachments that a render pass emits to (could be a color attachment, color + depth, multiple color attachments etc). A framebuffer may contain the final swapchain image you want to present, but this doesn't have to be the case. You would use this if you wanted to implement deferred rendering for example, or any rendering technique that has multiple render passes (or if you wanted to emit data to a surface that wasn't just the presented surface). If you are a graphics engineer, you likely know this already, but it can be helpful to know that Vulkan is pretty literal about its definitions.

#### Memory

In C or C++, you have `malloc` which you can use to get a block of memory. Vulkan's memory model is a bit more complicated because there are more types of memory (device visible, coherent, local, etc). Beyond that though, you recite an incantation and now have a block of memory. The type you choose depends on whether you expect the memory to be written to each frame, if you need to to be GPU writeable, etc. Afterwards, to use said memory, you need to associate it with a buffer or image (or any number of either). For the buffers and images, you need to put one final layer of buffer views or image views on top to be usable in a shader. At each one of these layers of abstraction, you have options! You can make a separate memory allocation for each buffer and image, and a corresponding view for each one (the least efficient way), or you could go on the other extreme and back your entire engine with a single memory allocation and heavy use of memory aliasing with views out the wazoo referencing this single region (hint, in practice people strive to this extreme but end up to the left). Just as you are expected to `free` code you have allocated with `malloc`, you must also reclaim memory in Vulkan. Of course, you also need to make sure you aren't using it anymore, and I'll talk about ways you can address this soon. Pay attention to the layout alignment rules, as messing up may land you in difficult to debug situations.

#### Synchronization

The topic Vulkan beginners likely dread the most. If you've already coded multithreaded code in C or C++, don't sweat this too much (pay attention to it obviously, but you already know most of the concepts you need). Treat the GPU as a separate thread of execution conceptually. Treat memory you allocate as shared. Don't write to that memory if it's being used. Don't free that memory if it's being used. You can do this in a very heavy handed way if you like (literally flush the graphics queue so you know nothing is in use at all when you do your thing). Alternatively, you can come up with a double buffering scheme if you want, or inject explicit fences so you know exactly when a particular command you issued to the GPU has finished. If you have different GPU commands that depend on each other (e.g. a compute job is depending on the output of a render job) use semaphores. If there are dependencies you can articulate with render subpasses, do so. For controlling access to memory when they change ownership, layouts, or visibility, use a memory barrier. In general though, just imagine the GPU is a really fat thread that's sharing memory with you and that you need to play nice (and stretching this one step further, the various graphics, transfer, and compute queues can be thought of as separate "threads" within the GPU). Prior to Vulkan, the driver did a whole lot of ref-counting to hide this from you, but now you get to be the boss! A really nice trick for reclaiming resources, for example, is to just wait a frame or so (at which point you're drawing with all the newly allocated things). But there are a lot of approaches just as there are plenty of ways of skinning a cat in C++ multithreading land.

# Reading the [Vulkan Tutorial][vktut] or [API without Secrets][intel]

My recommendation is the following:

1. Don't be afraid to copy-paste.
2. Don't try to get too clever and define your own functions, classes, etc.
3. Read the spec as you introduce new concepts from the tutorial.

My reasoning is that mindless typing doesn't actually help you. You shouldn't be trying to memorize the API since after using it "for real" you will pick that up surprisingly quickly once you get your bearings around the concepts. Also, because you are a bit new to the API, even if you _think_ you know what abstractions you'll need, you may be mistaken and shuffling around code is not a great usage of time. You _should_, however, be trying to learn concepts (image layout transitions, command execution, memory allocation, etc). You're also trying to get the gist of a Vulkan application, while noting that there are substantial differences between what you ultimately write, and the Vulkan Tutorial. Some key points to keep in mind:

- The tutorial prerecords commands for each framebuffer in the swapchain and just replays them every frame. This is a highly unrealistic workflow as most applications need to draw things dynamically
- The tutorial has a very linear order to what Vulkan objects get created but in some cases, this is not necessary. Try to take notes of the timeline as a tree so you understand where there are actual dependencies and where the author just happened to do a thing first
- The tutorial uses the C API and I don't like the C API (subjective of course ^^)

As for choosing between the these two tutorials, they are both well written although I find the Intel one slightly more comprehensive. It also makes you do a bit more work so if you're looking for a faster path to just getting the gist and then striking off on your own, the former tutorial may be more up your alley.

# Doing your own thing

Supposing you've gone through and finished the Vulkan tutorial, I recommend trying to implement your own renderer. This is a good time to experiment with pipeline derivatives, secondary command buffers, alternative memory layout schemes, push constants, using the compute queue, and more. For allocating/freeing memory, I highly recommend the open sourced memory allocator from AMD ([link][allocator]). It might feel like "cheating" but you'll find that even with this allocator, there are plenty of knobs to turn and levers to pull to tune your game or engine as you see fit. I do recommend at some point trying to write a simpler allocator for learning purposes, and you will find that it is essentially an exercise in writing `malloc`.

As you try to use a feature, you will likely find an appropriate example in Sascha Willem's excellent [Vulkan examples repo][vkex]. Keep in mind that the project organization places shaders in a `data` subfolder from the root of the repository when perusing the examples.

Try to be more aggressive than you even think is possible in terms of draw call minimization. For example, lots of indie games that I see could likely be drawn entirely with a couple draw calls and careful bookkeeping. Once you start to grasp how this is possible with the resources at your disposal, stop and congratulate yourself! You've started to grok Vulkan B).

# Conclusion

Hopefully, this post helped clear up some misconceptions and gave helpful pointers in making efficient and effective forward progress in your Vulkan-learning process. Feel free to tweet me or email if you have any feedback or suggestions!

[spec]: https://www.khronos.org/registry/vulkan/specs/1.1/html/vkspec.html
[trip]: https://fgiesen.wordpress.com/2011/07/09/a-trip-through-the-graphics-pipeline-2011-index
[rtr]: http://www.realtimerendering.com
[30min]: https://renderdoc.org/vulkan-in-30-minutes.html
[sdk]: https://vulkan.lunarg.com/sdk/home
[renderdoc]: https://renderdoc.org
[writedset]: https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkWriteDescriptorSet.html
[vktut]: https://vulkan-tutorial.com
[vkex]: https://github.com/SaschaWillems/Vulkan
[allocator]: https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator
[vkreddit]: https://www.reddit.com/r/vulkan
[vkdiscord]: https://discord.gg/JyJDbyH
[intel]: https://software.intel.com/en-us/articles/api-without-secrets-introduction-to-vulkan-part-1
[awesome]: https://github.com/vinjn/awesome-vulkan
[primer]: https://www.jeremyong.com/c++/graphics/gpu/vulkan/2018/11/22/vulkan-synchronization-primer.html
