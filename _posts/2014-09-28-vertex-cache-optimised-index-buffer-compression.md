---
layout: post
title: "Vertex Cache Optimised Index Buffer Compression"
description: "Fast index buffer compression"
category: Graphics
tags: [Graphics, Compression]
---
{% include JB/setup %}
A few weeks ago, I saw Fabian Giesen's [Simple lossless(*) index buffer compression](http://fgiesen.wordpress.com/2013/12/14/simple-lossless-index-buffer-compression/) post (and the accompanying code on GitHub) and it made me think about a topic I haven't thought about in a while, index buffer compression. A long time ago, at university, I built a mesh compressor as a project. It had compression rates for triangle lists of less than 2bits a triangle (and did pretty well with vertices too), but it had a few limitations; it completely re-ordered the triangles and it required that at most 2 triangles share an edge. The performance was semi-decent, but it was also relatively heavy and complicated, as well as requiring a fair bit of dynamic allocation for maintaining things like connectivity structures and an edge stack.

Re-ordering the triangles has the disadvantage that it destroys post-transform vertex cache optimisation (although, it did optimise order for the pre-transform vertex cache). Now, the vertex cache utilisation of these algorithms isn't generally catastrophic, as they use locality, but definitely worse than those produced by [Tom Forsyth's algorithm](http://home.comcast.net/~tom_forsyth/papers/fast_vert_cache_opt.html). 

Fabian's algorithm has the advantage of keeping triangle order, while performing better when vertex cache optimisation has been performed. It's also incredibly simple, easy to implement and uses very little in the way of extra resources. The key insight is that triangles that have been vertex cache optimised are likely to share an edge with the previous triangle.

But what if we extend this assumption a little, in that a triangle is also likely to share edges and vertices with other recent triangles. This is pretty obvious as this is exactly what the vertex cache optimisation aims for when it re-orders the triangles. 

## The Algorithm
Post-transform vertex caches in hardware are typically implemented as fixed size FIFOs. Given that is a case we've already optimized for (and fixed sized ring-buffer FIFOs are very cheap to implement), this is the concept we'll use as the basis for our algorithm. As well as a vertex index FIFO cache, we'll also introduce an edge FIFO cache that has the most recent edges.

What about cache misses? Well, these fall into two categories; new vertices that haven't yet been seen in the triangle list and cache misses for repeated vertices. If we re-order vertices so that they are sorted in the order they appear in the triangle list, then we can encode them with a static code and use an internal counter to keep track of the index of these vertices. 

Cache misses are obviously the worst case for the algorithm and we encode these relative to the last seen new vertex, with a v-int encoding.

So, we end up with four codes (which, for a fixed code size, means 2 bits):

 - ***New vertex***, a vertex we haven't seen before.
 - ***Cached edge***, edges that have been seen recently and are in the FIFO. This is followed by a relative index back into the edge FIFO (using a small fixed bit with; 5bits for a 32 entry FIFO).
 - ***Cached vertex***, vertices that been seen recently and are in the FIFO. This is followed by a relative index back into the vertex FIFO (same fixed bit with and FIFO size as the cached edge FIFO).
 - ***Free vertex***, for vertices that have been seen but not recently. This code is followed by a variable length integer encoding (like that seen in protobuf) of the index relative to the most recent new vertex.

Triangles can either consist of two codes, a cached edge followed by one of the vertex codes, or of 3 of the vertex codes. The most common codes in an optimised mesh are generally the cached edge and new vertex codes.

Cached edges are always the first code in any triangle they appear in and may correspond to any edge in the original triangle (we check all the edges against the FIFO). This means that an individual triangle may have its vertices specified in a different order (but in the same winding direction) than the original uncompressed one. It also simplifies the implementation in the decompression. 

While we use a quite simple encoding, with lots of fixed widths, the algorithm has also been designed such that an entropy encoder could be used to improve the compression. 

With the edge FIFO, for performance on decompression, we don't check an edge is already in the FIFO before inserting it, but we don't insert an edge in the case that we encounter a cached edge code. This is the same for cached vertices. This means the decompression algorithm does not have to probe the FIFOs. 

## Results

I've provided a simple proof of concept implementation on [GitHub](https://github.com/ConorStokes/IndexBufferCompression). These results were obtained with a small test harness and this implementation, using Visual Studio 2013 and on my system (with a bog standard i7 4790 processor). 

To make things consistent, I decided to make my results relative to those provided in Fabian's blog, using the same Armadillo mesh. His results use a file that encodes both the positions of the vertices and the indices in a simple binary.

The Armadillo mesh has 345,944 triangles, which is 4,151,328 bytes of index information. After running a vertex cache optimisation (using Tom Forsyth's algorithm) and then compressing with my algorithm, the index information takes up 563,122 bytes, or about 13 bits a triangle. By comparison, just running 7zip on *just* the optimised indices comes in at 906,575 bytes (so, it quite handily beats much slower standard dictionary compression). Compression takes 17 milliseconds and decompression takes 6.5 milliseconds, averaged over 8 runs. Even without the vertex cache optimisation, the algorithm still compresses the mesh (but not well), coming in at 1,948,947 bytes for the index information.

For comparison to Fabian's results and using the same method as him, storing the uncompressed vertices (ZIP and 7z results done with 7-zip):

STAGE  |  &nbsp;&nbsp;SIZE&nbsp;&nbsp;  |  &nbsp;&nbsp;.ZIP SIZE&nbsp;&nbsp;  |  &nbsp;&nbsp;.7Z SIZE
:------------- | :------------- | :----------- | :-----------
Compression (Vertex Optimised)  | &nbsp;&nbsp;2577k  | &nbsp;&nbsp;1518k | &nbsp;&nbsp;1276k
 |  |  |   

Performance wise, the algorithm offers linear scaling, decompressing about 50 million triangles a second and compressing about 20 million across a variety of meshes with my implementation on a single core. For compression, I used a large pre-allocated buffer for the bitstream to avoid the overhead of re-allocations. Compression is also very consistently between 12 and 13bits a triangle.

The compressed and uncompressed columns in the below table refer to sizes in bytes. The models are the typical Stanford scanned test models.

MODEL&nbsp;&nbsp;  |  &nbsp;&nbsp;TRIANGLES&nbsp;&nbsp;  |  &nbsp;&nbsp;UNCOMPRESSED&nbsp;&nbsp;  |  &nbsp;&nbsp;COMPRESSED&nbsp;&nbsp;  |  &nbsp;&nbsp;COMPRESSION TIME&nbsp;&nbsp;  | &nbsp;&nbsp;DECOMPRESSION TIME 
:-- | :-- | :-- | :-- | :-- | :-- |
Bunny | &nbsp;&nbsp;69,630 | &nbsp;&nbsp;835,560 | &nbsp;&nbsp;111,786 | &nbsp;&nbsp;3.7ms | &nbsp;&nbsp;1.45ms
Armadillo | &nbsp;&nbsp;345,944 | &nbsp;&nbsp;4,151,328 | &nbsp;&nbsp;563,122 | &nbsp;&nbsp;17ms | &nbsp;&nbsp;6.5ms
Dragon | &nbsp;&nbsp;871,306 | &nbsp;&nbsp;10,455,672 | &nbsp;&nbsp;1,410,502 | &nbsp;&nbsp;44.8ms | &nbsp;&nbsp;16.6ms
Buddha | &nbsp;&nbsp;1,087,474 | &nbsp;&nbsp;13,049,688 | &nbsp;&nbsp;1,764,509 | &nbsp;&nbsp;54ms | &nbsp;&nbsp;20.8ms
  |   |   |   |   | 


## Notes About the Implementation

Firstly, the implementation is in C++ (but it wouldn't be too hard to move it to straight C, if that is your thing). However, I've eschewed dependencies where-ever possible, including the STL and tried to avoid heap allocation as much as possible (only the bitstream allocates if it needs to grow the buffer). I've currently only compiled the implementation on MSVC, although I have tried to keep other compilers in mind. If you compile somewhere and it doesn't work, let me know and I'll patch it.

The [implementation](https://github.com/ConorStokes/IndexBufferCompression) provides a function that compresses the indices (CompressIndexBuffer) and one that decompresses the indices (DecompressIndexBuffer), located in IndexBufferCompression.cpp/IndexBufferCompression.h and IndexBufferDecompression.cpp/IndexBufferDecompression.h. There's also some shared constants (nothing else is shared between them) and some very simple read and write bitstream implementations. While this is a proof of concept implementation, it's designed such that you can just drop the relevant parts (compression or decompression) into your project without bringing in anything else. Compression doesn't force the flushing of the write stream, so make sure you call WriteBitstream::Finish to flush the stream after compressing, or the last few triangles might be incorrect.

There has been little optimisation work apart from adding some force-inlines due to compiler mischief, so there is probably some room for more speed here. 

I've allocated the edge and vertex FIFOs on the stack and made the compression and decompression functions work on whole mesh at a time. It's possible to implement the decompression to be lazy, decompressing a single (or batch) of triangles at a time. Doing this, you could decompress in-place, as you walked across the mesh. I'm not sure or not if the performance is there for you to do this in real-time and that probably varies on your use case.
