---
layout: post
title: "Better Vertex Cache Optimised Index Buffer Compression"
description: "Fast index buffer compression revisited"
category: Graphics
tags: [Graphics, Compression]
---
{% include JB/setup %}
I've had a second pass at the compression/decompression algorithm now, after some discussion from davean on #flipCode. The new iteration of the algorithm (which is available alongside the original) is a fair bit faster for decompression, slightly faster for compression and has a slightly better compression ratio. The updated algorithm doesn't however support degenerate triangles (triangles that have a duplicate vertex indice), where as the original does. 

The original is discussed [here](http://conorstokes.github.io/graphics/2014/09/28/vertex-cache-optimised-index-buffer-compression/).

The main change to the algorithm is to change from per vertex codes to a single composite triangle code. Currently for edge FIFO hits there are two 2 bits codes and for other cases, three 2 bit codes. There are a lot of redundant codes here, because not all combinations are valid and some code combinations are rotations of each other. The new algorithm rotates the vertex order of triangles to reduce the number of codes, which lead to one 4 bit code per triangle. This nets us a performance improvement and a mild compression gain. 

There were also some left over codes, so I added a special encoding that took the edge + new vertex triangle case and special cased it for the 0th and 1st edges in the FIFOs. This could potentially net very big wins if you restrict your ordering to always use a triangle adjacent to the last one, trading vertex cache optimisation for compression.

Here's the performance of the old algorithm:

MODEL&nbsp;&nbsp;  |  &nbsp;&nbsp;TRIANGLES&nbsp;&nbsp;  |  &nbsp;&nbsp;UNCOMPRESSED&nbsp;&nbsp;  |  &nbsp;&nbsp;COMPRESSED&nbsp;&nbsp;  |  &nbsp;&nbsp;COMPRESSION TIME&nbsp;&nbsp;  | &nbsp;&nbsp;DECOMPRESSION TIME 
:-- | :-- | :-- | :-- | :-- | :-- |
Bunny | &nbsp;&nbsp;69,630 | &nbsp;&nbsp;835,560 | &nbsp;&nbsp;111,786 | &nbsp;&nbsp;3.7ms | &nbsp;&nbsp;1.45ms
Armadillo | &nbsp;&nbsp;345,944 | &nbsp;&nbsp;4,151,328 | &nbsp;&nbsp;563,122 | &nbsp;&nbsp;17ms | &nbsp;&nbsp;6.5ms
Dragon | &nbsp;&nbsp;871,306 | &nbsp;&nbsp;10,455,672 | &nbsp;&nbsp;1,410,502 | &nbsp;&nbsp;44.8ms | &nbsp;&nbsp;16.6ms
Buddha | &nbsp;&nbsp;1,087,474 | &nbsp;&nbsp;13,049,688 | &nbsp;&nbsp;1,764,509 | &nbsp;&nbsp;54ms | &nbsp;&nbsp;20.8ms
  |   |   |   |   | 

And here is the performance of the new one:

MODEL&nbsp;&nbsp;  |  &nbsp;&nbsp;TRIANGLES&nbsp;&nbsp;  |  &nbsp;&nbsp;UNCOMPRESSED&nbsp;&nbsp;  |  &nbsp;&nbsp;COMPRESSED&nbsp;&nbsp;  |  &nbsp;&nbsp;COMPRESSION TIME&nbsp;&nbsp;  | &nbsp;&nbsp;DECOMPRESSION TIME 
:-- | :-- | :-- | :-- | :-- | :-- |
Bunny | &nbsp;&nbsp;69,630 | &nbsp;&nbsp;835,560 | &nbsp;&nbsp;108,908 | &nbsp;&nbsp;2.85ms | &nbsp;&nbsp;0.92ms
Armadillo | &nbsp;&nbsp;345,944 | &nbsp;&nbsp;4,151,328 | &nbsp;&nbsp;547,781 | &nbsp;&nbsp;15.9s | &nbsp;&nbsp;4.6ms
Dragon | &nbsp;&nbsp;871,306 | &nbsp;&nbsp;10,455,672 | &nbsp;&nbsp;1,363,265 | &nbsp;&nbsp;36.3ms | &nbsp;&nbsp;11.8ms
Buddha | &nbsp;&nbsp;1,087,474 | &nbsp;&nbsp;13,049,688 | &nbsp;&nbsp;1,703,928 | &nbsp;&nbsp;46ms | &nbsp;&nbsp;14.7ms
  |   |   |   |   | 

Overall, there seems to be a 30%-40% speedup on decompression and a modest 3% improvement on compression.

Also, I'd like to thank Branimir Karadžić, whose changes I back-ported, which should hopefully add support for 16bit indices and MSVC 2008 (unless I've broken them with my changes - I don't have MSVC 2008 installed to test).

If you haven't checked out the source, it's available on [GitHub](https://github.com/ConorStokes/IndexBufferCompression).