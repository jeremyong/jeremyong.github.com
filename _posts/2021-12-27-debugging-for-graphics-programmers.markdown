---
layout: post
title: 'Debugging For Graphics Programmers'
subtitle: Got a graphics bug? Things to check
categories:
  - graphics
---

When I encounter a bug while doing graphics programming work or research, I have a mental checklist I work through to diagnose and solve the issue.

This post is about the contents of that mental checklist. This list intentionally excludes things specific to a given algorithm. For example, specific debugging tricks or things to look out for when
implementing, say, exponential shadow maps, or tiled light culling, or light probes, etc.

## Bug Zoo

Even if we restrict ourselves to the class of "general bugs" one might encounter in graphics programming, I doubt this will be exhaustive but will keep adding to this list as I think of more items. Without further ado, and in no particular order, here are some of the things I consider when debugging graphics bugs.

### Floating point precision

Consider floating point precision if you observing banding or Moiré patterns indicitave of precision issues.

- Consider extreme ranges for the relevant quantities. Is your computation resilient, and can you check that the values are within the expected ranges?
- Are your computations subject to catastrophic cancellation?
- In the context of shadows and lighting, are you considering the effects of geometric aliasing or perspective aliasing?

### Improper pipeline state

Sometimes the result are completely incorrect and wildly outside your expectation of what should occur. In this case, it's worth sanity checking the overall pipeline state.

- Is it possible that your binding the incorrect resource?
- The wrong shader module?
- Draw constants/uniforms correct?
- Are sample counts what you expect?
- Correct winding order? Topology?
- Correct blend function?
- Correct depth state (depth bounds, test operation, etc.)?
- Is your shader compiler emitting code that is actually incorrect (possible with bleeding edge versions or when using newer compiler features)?

### Glitchy bugs

Sometimes things are "wonky" in a way that doesn't line up with an obvious shader bug or anything. These bugs might be better/worse on different drivers, or may even be transient as you move a camera around.

- Do you have divergent control flow in a shader and if so, are you properly using `NonUniformResourceIndex` when accessing resource arrays with a VGPR index?
- Alternatively, are you using `ddx`, `ddy`, or `Sample` instructions in divergent control flow?
- Are you writing to a UAV in a way that is possibly unsafe?
- Are you missing a resource barrier causing async compute to stomp on memory?
- Alternatively, is a resource barrier misconfigured (e.g. guarding the wrong resource, misplace, wrong pipeline flags or access bits, etc.)?
- Are you aliasing memory that shouldn't be aliased (multiple concurrent writers that should be sequenced)?
- Are you running into an actual driver bug that a driver update would fix?
- Are you using LDS and forgot to initialize the memory?
- Is there a mismatch between the data layout your shader expects vs what your code expects?
  - Check alignment and packing rules (e.g. structured buffer packing rules vs byte address buffer rules)
  - Check that you've allocated sufficient memory
  - Check if your resource views are correctly specified
  - Check row-major vs column-major conventions
- Alternatively, do you have stale state from a previous frame that wasn't cleared properly?
- If using wave/quad intrinsics, are you forgetting to mask the correct threads?
- If your bug appears tile-dependent, look for bugs in all-your tile-based/binning algorithms (e.g. tiled light culling)

### Content bugs

Outside of the realm of exotic bugs, there is a plethora of bugs related to content. Ideally, you would have lots of tools and visualizations used to debug this. Some examples:

- Light leaking could be the result of misplaced volumes used to classify interior/exterior regions
- Objects popping in or out could be the result of misplaced occluders, mismatching static BVH/BSP, or something similar
- Miscoloration of transparents/particles could be due to mishandling of premultiplied alpha

## More Debugging Tips

Again, in no particular order:

- You should have an easy way to programmatically start and stop a capture in every tool in your arsenal (Pix, RenderDoc, NSight, etc.)
- You should be able to freeze the frame at any point (i.e. simulation is paused)
- You need the ability to freely move the camera after everything else is "frozen" so can inspect the systems that are view dependent (e.g. terrain, culling, streaming, etc.)
- Ideally, your resources and render targets are named
- Ideally, your event timeline has user supplied markers that are easy to navigate
- For bugs that "go away" when a capture tool is attached, have modes of rendering where you disable multi-threaded rendering, async compute, or other synchronization-sensitive techniques to narrow down the issue
- Visualizing render targets or other resources in-engine without the need for a capture can be an amazing accelerant
- By the same token, another really useful trick is binding an auxiliary buffer and writing data from shaders (use `InterlockedAdd` to get a free slot in the buffer to write to)
- Nail your workflows for generating shader PDBs and integrating the shader PDB paths with all the tools you use (if there's too much friction in this process, you'll just resort to less efficient debugging techniques)
  - Some tools like Pix support pdb/shader reloading
- Shorten the feedback loop between when you make a change, and when you can determine if the change succeeded or not.
  - For example, if you think the bug is in a shader header that every material uses, clone the header and have only one material reference it so you don't have to recompile all materials when iterating.
  - This idea applies to pretty much every workflow. If you spend too much time between cycles, identify what the slow portion is, and don't be afraid to spend more time than you think is wise to accelerate it (it generally pays off over time)
- RenderDoc has some handy features you should know (outside the basics)
  - Debug pixel and debugging shaders in general (although if you use wave intrinsics, this is off the table)
  - Dumping resource contents to CSV/binary to analyze with a spreadsheet tool or Pix
- NSight is generally the best tool for debugging perf issues (if developing for desktop)
- Never underestimate the power of commenting out code to aggressively simplify the problem
- Gather as much data as possible about bug reproducibility and the conditions needed (e.g. platform, OS version, driver version, GPU vendor, etc.)