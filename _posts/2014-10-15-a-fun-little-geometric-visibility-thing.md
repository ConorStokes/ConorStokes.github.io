---
layout: post
title: "A Fun Little Geometric Visibility Thing"
description: "I solved a visisbility query with this one weird trick!"
category: Geometry
tags: [Geometry, Visibility]
---
{% include JB/setup %}
This is just an interesting little tidbit that I stumbled across a fair few years ago (2006 or so I think) and have been meaning to blog about (I might have and forgotten) ever since. There is a neat trick of geometry that allows you to solve yes-no visibility queries from a point for polygon soups with nothing more than ray-casting and a few tricks of geometry, with no polygon clipping and no sorting.

I'm not claiming this will actually be useful to anyone, in fact, floating point precision problems make it difficult to do in practice. But, it may come in handy for someone and it may well be practical for drastically simplified versions of a scene.

Firstly, we'll talk about visibility queries without any view restrictions (omni-directional from a point). A simple assertion; for non intersecting geometry (intersecting geometry can be handled, with the caveat that the line segments where surfaces intersect must be treated as an edge), a polygon or polygonal object is visible if a ray from the query point passing through 2 edges on the silhouette of any object in the scene hits the object without being occluded. 

If you are familiar with the rule for visibility queries between polygons; that there must exist an extremal stabbing line going through 4 edges between 2 polygons for them to be visible to each other, this is a natural extension to that, because the constraint of the point in the query (the eye, the camera, or whatever you want to call it) is the equivalent to the constraint of 2 edges. So we are left with the point and 2 edges to define an extremal stabbing line; of which one must exist for the visibility query to return a positive result.

Now, there is the case of determining what edges, in particular, will provide us with a ray to cast and what the ray will be. Luckily, there is a straight forward answer to this:

 1. They must be on the silhouette of an object and if the object is a polygon connected to other polygons, this includes the edges of the polygon in the query.
 2.  For such a ray to exist for 2 edges and point, each edge must straddle the plane through the other edge and the query point.
 3. The ray going through these 2 edges is the intersection of these planes; so it must be perpendicular to both planes. This means the direction of the ray is the cross product of the normals of the planes going through each edge and the query point. 
 4. The origin of the ray is obviously the query point.
 5. This ray must obviously intersect with the query object too (if one or more of the edges is on the query object, this is a given).

There is of course a special case to this; 2 edges often intersect at a vertex, in which case there will exist a ray going through the query point and the vertex. 

Anyway, to solve the query, you enumerate all the query point silhouette edge-edge cases and see if you find one that hits the query object without being occluded by another object. If such a ray exists, then the object is visible. You could do a whole scene at once by enumerating all the query point edge-edge cases and knowing that only the objects that got hit were visible.

You could, potentially, do something similar to a beam-tree to accelerate finding which edges straddled the planes of other edges. If I were to make a terrible pun, I would say this is a corner case of a beam-tree (in that, the rays in question are essentially the corners of significant plane intersections in the beam-tree). 

This also works in reverse for the view frustum and portal cases; in this case the ray through the query point and edges must pass through the portals and the view frustum. This allows you to perform portal queries without clipping the portals.

There is also, I suspect, a fair bit of temporal coherence for the edges involved in a visibility query; if the query point moves only slightly, it is likely the same edges will produce an extermal stabbing line that proves visibility.