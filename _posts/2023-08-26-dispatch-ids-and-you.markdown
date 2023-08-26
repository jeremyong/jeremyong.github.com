---
layout: post
title: Dispatch IDs and you
date: 2023-08-26 00:00
categories:
- Graphics
katex: true
---

Do you frequently find yourself struggling to remember how the various built-in compute shader semantic values work by glancing at the image below?

![Thread Group IDs](/images/dispatches/threadgroupids.png)

You can find this image on the various `SV_*` Win32 documentation pages such as [this one](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/sv-dispatchthreadid).

It's easy to get confused as to when to use which semantic value if you're not used to it, so here's how I remember things.
The main thing to remember is that both `SV_GroupID` and `SV_GroupIndex` are _poorly named_, or at the very least, are inconsistently named.
After we examine the different compute shader semantics available, you'll see what I mean (or you might not, this is just how I tend to think of things).
You're also welcome to skip to the summary table below.

## Compute shader semantics

There are three IDs we need to consider:

* `SV_DispatchThreadID`
* `SV_GroupThreadID`
* `SV_GroupID`

Recall that compute dispatches are specified in a two-level hierarchy (ignoring the notion of [`WaveSize`](https://microsoft.github.io/DirectX-Specs/d3d/HLSL_SM_6_6_WaveSize.html) for now).
First, threads are grouped into, well, groups. The size of these groups is given by the three arguments to the `numthreads` attribute attached to your compute shader entry point.
The groups themselves are launched on the host using a call to `Dispatch`, which form the next layer of the hierarchy.
The number of threads in a group is independent of the number of groups dispatched.

So, if you had an entry function declaration like so:

```hlsl
[numthreads(5, 5, 1)]
void main();
```

the group size would be 25 threads in total, spanning a 5x5x1 conceptual volume, where each cell in that volume is a separate thread or invocation.
The address of a thread within a group is given by an `SV_GroupThreadID` value, which is a `uint3` as you might expect.
Crucially, the `SV_GroupThreadID` has _nothing_ to do with _how many_ groups are present in a given dispatch.

In contrast, `SV_DispatchThreadID` is also a "thread ID" so we expect the value to identify a thread somehow.
The prefix `Dispatch` here gives us a hint. Whereas the `SV_GroupThreadID` identifies a thread within a group scope,
the `SV_DispatchThreadID` instead identifies a thread across the entire dispatch.

That is, if we invoked `Dispatch(3, 2, 4)` with the entry point above, we would be dispatching a total of $3 \times 2\times 4 = 24$ groups total,
for a total of $24 * 25 = 600$ total threads, and the `SV_DispatchThreadID` will thus take on 600 unique values while the shader executes.
As with threads within a group, groups within a dispatch are addressed using 3 coordinates, so combining the group address and the address of a thread
within a group, it should be clear why an `SV_DispatchThreadID` has a `uint3` type, just like the `SV_GroupThreadID` does.

This brings us to the `SV_GroupID` value. As you might expect, unlike the `SV_*ThreadID` values which identify threads within some scope,
the `SV_GroupID` identifies a group instead. Here, the scope is omitted because only one scope is really possible.
Namely, the `SV_GroupID` value identifies a group within the dispatch. Given the example again of invoking `Dispatch(3, 2, 4)`, the `SV_GroupID` would
take on `uint3` values spanning the 24-cell volume of groups (each containing a 5x5x1 set of threads). The reason I dislike the name
personally is because while the scope of the ID is unambiguous (it has to be a dispatch after all), I would have preferred `SV_DispatchGroupID` to
mirror the naming convention used for the other two values we've seen so far. If you have trouble remembering which is which, it might be helpful
to think of `SV_GroupID` as `SV_DispatchGroupID` instead to recover the logical consistency with the other terms.

With those terms out of the way, we arrive at the last value to remember:`SV_GroupIndex`. Unlike `SV_GroupID`, whose values are mapped to groups,
`SV_GroupIndex` values are mapped to _threads_. Confusing, right? Specifically, `SV_GroupIndex` values map to "threads within a group".
Given the main function above with attribute `numthreads(5, 5, 1)`, `SV_GroupID` would take on values from 0 to 24 inclusive.
The naming of this value is particularly egregious, because while the index describes "threads", the term "thread" doesn't appear in the name itself.

Note that there is no `SV_DispatchIndex` value which would hypothetically be a flattened thread index unique across the entire dispatch.
This value is sometimes useful in compute shaders, and you generally compute it yourself by flattening an `SV_DispatchThreadID`.

## Summary

To summarize, here's a table with the terms in the leftmost column, a description in the center column, and what I think they should have been named on the rightmost column.
The suggested renamings are mainly here as a mnemonic device, and in some cases as discussed, the original names are fine.

| Semantic | Description | Preferred spelling |
| -------- | ----------- | ---------------- |
| `SV_DispatchThreadID` | Unique `uint3` ID of a thread within the dispatch | `SV_DispatchThreadID` |
| `SV_GroupThreadID`    | Unique `uint3` ID of a thread within a group | `SV_GroupThreadID` |
| `SV_GroupID`          | Unique `uint3` ID of a group within a dispatch | `SV_DispatchGroupID` |
| `SV_GroupIndex`       | Unique `uint` index of a thread within a group | `SV_GroupThreadIndex` |
| Doesn't exist         | Unique `uint` index of a thread within a dispatch | `SV_DispatchThreadIndex` |