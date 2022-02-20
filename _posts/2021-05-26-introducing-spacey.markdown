---
layout: post
title: 'Introducing, Spacey'
subtitle: A fast, hardware-accelerated MikkT basis generator.
katex: true
published: false
categories:
  - cpp
  - graphics
---

[MikkTSpace](http://www.mikktspace.com/) is a ubiquitous technique for encoding vertex-frequency tangent spaces.
However, the corresponding [reference implementation](https://github.com/mmikk/MikkTSpace) hasn't changed much
since the original release in 2011. Especially now that geometric detail has exploded in triangle counts, I
figured it was high time to reimplement the computation from the ground up with performance as a goal while
preserving all the desirable properties of the original algorithm. In this post, I'll first describe the
algorithm itself. Then, I'll explain the architecture behind [Spacey](https://github.com/jeremyong/spacey),
a fast MikkT-compatible alternative.

## MikkT Refresher

Let's quickly recap what "mikktspace" even is, why it does what it does, and how it works. The mikktspace
algorithm prescribes a method for assigning, to each vertex in a mesh, a tangent space in a robust manner.
The inputs to the algorithm are a set of vertices connected as quads or triangles. Each vertex must have
an associated normal vector and UV coordinate. The output is a tangent/bitangent pair per vertex such that
the tangent points along the gradient of U and the bitangent points along the gradient of V. As a quick
refresher, such a tangent space is needed to correctly orient normals as sampled from a normal map. The
magnitude of the tangent/bitangent pair can also be retrieved from the algorithm for use in relief-mapping
effects.

Formally, recall that a well-formed mesh is actually a 2D-manifold (embedded in 3D). The per-vertex texture
coordinates provide a mapping from each interpolated position in 3-space to a coordinate pair (which we
typically use to sample a normal map). Crucially, there is no unique way to compute the tangent basis at
a vertex. The vertex itself may have multiple connected elements, each with a different surface gradient.

TODO: Picture/explanation