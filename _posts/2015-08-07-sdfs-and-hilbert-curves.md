---
layout: post
title: "SDFs and Hilbert Curves"
description: "SDFs and Hilbert curves, a match made in heaven?"
category: "Geometry"
tags: ["SDF", "Graphics", "Hilbert Curve", "Compression", "Geometry"]
---
{% include JB/setup %}
So, this is more of an "thought bubble" blog post than anything concrete yet, but I thought it might be worth some discussion, or at least putting the ideas out there because someone else might find them useful. 

I've been thinking a lot lately about the potentials for combining traversals in a [Hilbert curve](https://en.wikipedia.org/wiki/Hilbert_curve) ordering with regularly sampled SDFs (signed distance field, which is a sampling of the nearest distance to the surface of an object, with a sign derived from whether the sample is inside or outside the object). Using a Hilbert curve is already a well known technique for improving spatial locality for regularly sampled grids, but there are some properties that apply to SDFs with fairly accurate euclidean distances that sit well with Hilbert curves. Note that what I'm talking about is mainly with 3D SDFs/Hilbert curves in mind, but it applies to 2D as well (as a side note for the pedant in me, if we're talking higher dimensions, SDFs aren't really a good idea).

The main property of the Hilbert ordering we're interested in is that unlike other orderings, for example the Morton/Z-Curve, Hilbert provides an ordering where each consecutive grid point is rectilinearly adjacent to the previous one (only a single step in a single axis). This also happens to be the minimum distance possible to move between two grid points (for a grid that is uniformly spaced in each axis). A Hilbert curve isn't the only ordering that provides this property, but it does provide great spatial locality as well.

There is a simple constraint on euclidean distance functions, which is that the magnitude of the difference of the distance function value between two sample points can't be greater than the distance between the sample points. This property also holds for signed distance functions and it holds regardless of the magnitude of the distances.   

Add these two little pieces together and there are some "tricks" you can pull.  

One of the first is that if you traverse a SDF's grid points in Hilbert order, you can delta encode values and know the magnitude of the delta will never be greater than the distance between the two adjacent grid points, meaning that a you can achieve a constant compression ratio that is lossless vs the precision you need to represent the distance field globally (this is without getting into predictors or more advanced transforms). This also works for tiles, where you could represent each tile with one "real" distance value for the first point, along with deltas for the rest of the points, traversing the tile in Hilbert order. This would make the compressed SDF randomly accessible at a tile level (similar to video compression, where you have regular frames that don't depend on any others, so you can reconstruct without decompressing the whole stream), although you would still potentially have to sample multiple tiles if you wanted to interpolate between points. If you wanted to go further, you could use more general segments of the curve instead of tiles.

Another neat trick; if you're building the SDF from base geometry using a search for the nearest piece of geometry at each point and you build traversing in Hilbert, you can both put a low upper bound on the distance you have to search. This is very useful for doing a search in spatial partitions and bounding volume hierarchies, or even probing spatial hashes. You also get the advantage of spatial locality, where you will be touching the same pieces of geometry/your search structure, so they will likely be in your cache (and if you are transforming your geometry to another representation, such as a point based one, you can do it lazily). Also, you can split the work into tiles/segments and process them in parallel. 

This is actually a topic close to my heart, as my first/primary task during my research traineeship at MERL around 15 years ago was investigating robustly turning not-so-well-behaved triangle meshes into a variant of SDFs (adaptively sampled distance fields, or ADFs), which is an area that seems to be drawing a lot of attention again now. [This algorithm](http://www.sciencedirect.com/science/article/pii/S152407031400037X) in particular is a neat new approach to this problem.
