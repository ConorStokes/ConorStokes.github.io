---
layout: post
title: "An LZ Codec Designed for SSE Decompression"
description: "A fun exercise in building something for a particular platform."
category: Compression
tags: [Compression, SSE, Optimization]
---
{% include JB/setup %}

For quite some time, I've been keenly following developments in the world of compression, reading blog posts and lurking in forums. It's always been something I've found interesting, but I haven't really had much practical experience implementing general purpose compressors (most of the things I've done have been quite special purpose). I had a few weeks off over the Christmas break, so I decided to have a play with some fairly traditional LZ77/LZSS compressors, just to familiarize myself a bit with details of the implementation.

The work done over the break was fairly simplistic and well trodden ground. Sometimes it's good just to instrument code like this, using simple histograms that you can read in the debugger/dump to the console and get a feel for what's going on. Experimenting a little, trying a few things and building up more of an understanding.

About two weeks ago a few ideas clicked together and I decided to start a new experiment, building an SSE/x64 oriented [LZSS](https://en.wikipedia.org/wiki/Lempel%E2%80%93Ziv%E2%80%93Storer%E2%80%93Szymanski) format, with a decoder that can branchlessly decode up-to 32 matches per iteration, with a fair bit of the upfront logic done in parallel using SIMD. This is what I'll be sharing in this blog post, along with associated code for three variants (LZSSE2, LZSSE4 and LZSSE8). 

Note, this is an experiment mainly focused on the decompression, the compressors I've implemented aren't great in terms of speed yet, because I haven't really put any focus on optimizing them (not being the goal of the experiment). The LZSSE4 and LZSSE8 of the compressors aren't too slow, but the LZSSE2 compressor is. It uses an [optimal parse](http://www.cbloom.com/algs/dictionary.html#optimal) to test the compression levels achievable with the format, but the binary tree based matching it uses has some terrible pathological cases.

The ideas that clicked together were the threshold based encoding talked about by Charles Bloom [here](http://cbloomrants.blogspot.com.au/2012/09/09-02-12-encoding-values-in-bytes-part-2.html), the way Yann Collet's [LZ4 block format](https://github.com/Cyan4973/lz4/blob/master/lz4_Block_format.md) uses the highest value in fields to do variable length encoding and the way parallel SSE pop count implementations split apart the high and low nibbles of bytes and then use them to look-up values inside a register using the PSHUFB instruction (note, only LZSSE8 actually uses the PSHUFB to implement the logic I was originally thinking about, the other variants use binary logic).

## Format Overview ##
The three formats implemented have a lot in common. They all use 32 nibble control words packed continuously in the stream, followed by either the offsets or literals for each word. Each nibble either flags a sequence of literals, or a match, based on a threshold. The length for the sequence of literals is encoded if below the threshold, the length of the match is encoded if above. When the nibble is set to 15, that indicates that the following nibble extends the match (and can't encode literals). I've also used a small 64k window, with fixed 16 bit offsets. Each stream also must end with 16 literals (which are not encoded with controls).

One of the goals is that each control word maps to at most one SSE register width worth of data to copy, so that we don't need an inner loop (or rep movsb) to copy data. The corresponding challenge, and the reason there are three variants, is trying to get each step in the decoding loop to shift as many bytes as possible on average to keep performance high (which goes hand in hand with compression ratios, as it reduces the relative overhead of the control nibbles).

In LZSSE2, the control values 0 and 1 represent a sequence of 1 and 2 literals respectively, while the control values 2 through to 15 represent a match length of 3 to 16. If the value of the match control is 15, the next nibble represents an additional 0 to 15 bytes that will be appended to the match (and the same with the next nibble after that and so on). This means that each literal has an overhead of either 2 or 4 bits, but that we can represent shorter matches (3 to 15 bytes) in 20 bits. These kind of matches tend to be quite common in a lot of data, especially text. The overhead for longer matches means we limit the maximal achievable compression ratio to a limit approaching 30x. Because there are only a couple of literals per control, this format isn't great for data that doesn't have a lot of matches, or that has long strings of literals between matches.

In LZSSE4, the control values 0 through to 3 represent sequences of 1 to 4 literals and the control values 4 through 15 represent matches of 4 to 15 bytes, but otherwise the format is the same. This format does better with longer strings of literals and is also more suitable to be paired up with a faster/lower ratio compressor. The longer minimum match length works better with less exhaustive hash based matching.

LZSSE8 is aimed at lower compression scenarios with more literals again, the control values 0 to 7 represent runs of 1 to 8 literals and the control values 8 to 15 represent matches of 4 to 11 bytes. Also, LZSSE8 needs match offsets greater than 8 bytes from the current cursor (LZSSE8 still beats LZSSE4 on some files in compression ratio, despite the extra restrictions). Again, other than that, the format is the same.

Offsets and literals use an xor encoding, where offsets are xor'd against the previously read offset and literals are xor'd against the data at the previous match offset in the decompressed stream. This allows us to do some optimizations that we'll talk about later on.

We also restrict the kind of overlapping matches allowed. Matches are only allowed to overlap if the offset is greater than one SSE register width away from the current output cursor.

## The Compressors ##
The compressor for LZSSE2 uses a variant of the Storer-Szymanski optimal parse algorithm and a binary search tree for finding matches. The implementation is quite naive, but apart from quirks with short offsets, it should always find the longest match. It does have some pathological *binary tree problems* when you have long runs of the same character, a well known problem, so some files compress *very* slowly. At it's maximum level, it should achieve the best possible compression for this format (although, there are some caveats with very small offsets). Note that this slow performance is not indicative of anything inherent in LZSSE2, a better matching algorithm would allow the codec to be competitive for compression speed.

The LZSSE4 and LZSSE8 compressors uses a single entry hash and a greedy parse, they're designed to be sloppy and fast. That said, there are some files where matches are rare enough that the sloppy matching/parses of LZSSE8 actually beat the LZSSE2 optimal parse on compression ratio.

Neither is particularly optimized for speed at the moment (although, SSE is used for testing matches). These are basically built for testing out the performance of the format and the decompression implementations.

## The Decompression Implementation ##
This is where things get interesting. Firstly, a comment on this code; it uses macros and gotos, which are things that I wouldn't usually recommend. It also has largish unrolled loops, which aren't always wise (but work well in this case). Finally, it's quite tricky to follow and beaten to make the compiler (in this case, Visual Studio 2015; note, this code has not been tested yet with any other compiler and is definitely not ready for them yet) do what I want, which is no register spills and some quite specific instructions being generated (with an eye on the assembly listings). It's also some of the code I've had the most fun writing in a long time, because it was a [good puzzle](https://en.wikipedia.org/wiki/House_(TV_series)).

It's also important to note, this code is targeted at 64bit/SSE 4.1 and it eats pretty much all of the registers (and narrowly avoids spills). You'd probably want to do the implementation differently for targeting 32bit x86. I'm sure there are some wins to be had in the implementation. You could definitely trim the implementations back to only require SSSE3 (needed for the 8 bit variable shuffle) though, if you were keen (extracts and blends are fairly simple to emulate with SSE2 instructions).

One of the first things you might notice is that the main loop is implemented 3 times. The first loop has buffer checks for under-flowing the output buffer based on matches before the data. The second loop doesn't have buffer checks and the third loop has both the underflow checks and overflow checks for the input and output buffers. The loops themselves have conditions on them that mean it's impossible to overflow/underflow in the particular loop innards, because they are a "safe" distance away.  

We'll focus on the LZSSE2 implementation, because it's a little bit simpler to explain. LZSSE4 and 8 are well commented though, so it should be possible to grok what's going on.

Each iteration of the loop deals with 32 control nibbles, which is exactly what fits in one SSE register. We load that at the start of the loop, then split it into high and low nibbles:

    __m128i controlBlock = _mm_loadu_si128( reinterpret_cast<const __m128i*>( inputCursor ) );
    __m128i controlHi    = _mm_and_si128( _mm_srli_epi32( controlBlock, CONTROL_BITS ), nibbleMask );
    __m128i controlLo    = _mm_and_si128( controlBlock, nibbleMask );

Then we threshold the control values to work out which is a literal or match (note, later this will possibly be ignored in the event of an extended match):

	__m128i isLiteralHi = _mm_cmplt_epi8( controlHi, literalsPerControl );
	__m128i isLiteralLo = _mm_cmplt_epi8( controlLo, literalsPerControl );

We also work out if the controls will cause the next control to be an extended match. We call this the "carry", because it essentially shifts up to the next control.

	__m128i carryLo = _mm_cmpeq_epi8( controlLo, nibbleMask );
	__m128i carryHi = _mm_cmpeq_epi8( controlHi, nibbleMask );

Now, the low carry will be used with the high controls and the high carry will be used with the low controls, but we'll need to shift the high carry along one to be used with the low controls *after* the high ones. Because this would leave a gap of one, we also need a memory to remember the previous set of high carries.

	__m128i shiftedCarryHi = _mm_alignr_epi8( carryHi, previousCarryHi, 15 );
    
    previousCarryHi = carryHi;

Next, we'll calculate the number of bytes that we'll be outputting per control. To do this, we take the control and add one to it if the carry is not set. We actually use a subtraction of negative one, because SSE masks are the same as negative one. Note, in LZSSE8 the bytes out and bytes read stages are done via a lookup using the PSHUFB, where the values are stored in the high and low values of a register. 

	__m128i bytesOutLo = _mm_sub_epi8( controlLo, _mm_xor_si128( shiftedCarryHi, allSet ) );
	__m128i bytesOutHi = _mm_sub_epi8( controlHi, _mm_xor_si128( carryLo, allSet ) );

Now we'll calculate how many extra bytes we're going to read from the input stream for each control. That will either be two for matches (the size of the offset), one or two for literals and zero for extended matches (when the carry is set). Note that the "literalsPerControl" value here is two, so we're re-using it.

	__m128i streamBytesReadLo = _mm_andnot_si128( shiftedCarryHi, _mm_min_epi8( literalsPerControl, bytesOutLo ) );
	__m128i streamBytesReadHi = _mm_andnot_si128( carryLo, _mm_min_epi8( literalsPerControl, bytesOutHi ) );

This is followed by simple masks for whether we're going to update the offset value (we will for initial matches, we won't for literals or the extended match) and whether we are writing out literal data or match data:

	__m128i readOffsetLo = _mm_xor_si128( _mm_or_si128( isLiteralLo, shiftedCarryHi ), allSet );
	__m128i readOffsetHi = _mm_xor_si128( _mm_or_si128( isLiteralHi, carryLo ), allSet );
	
	__m128i fromLiteralLo = _mm_andnot_si128( shiftedCarryHi, isLiteralLo );
	__m128i fromLiteralHi = _mm_andnot_si128( carryLo, isLiteralHi );

After this point, we're going to need some of these values in general purpose registers instead of SSE registers, so we read read out the bottom 64 bits (we'll read out the top 64 bits in a second version of this later). In fact, when we do the step for handling each individual control, we'll read the bottom 8 bits of these registers, then shift them 8 bits along (we'll avoid angering the partial register stall demon by getting the compiler to generate sign and zero extension instructions).

Speaking of which, this is implemented with macros (because we do it 32 times in a loop), but now we'll get on to the step for an individual control decoding (we'll not include the buffer checks used in some loops). Here we  show the step for low controls, high is very similar. Low and high steps are interleaved.

Part of the goal here is to provide enough interleaved chains of instructions that the compiler always has something to chew on, especially when we have to eat some latency on reads and writes.

First we read from the input stream (we're doing this unconditionally to avoid a branch). We treat it as a 16 bit integer and then zero-extend it to size_t and load it into an SSE register. Note, there will be some latency between the two in the time the value takes to come from the (hopefully) L1 cache. We also load the value into an SSE register to be used as literals:

    size_t  inputWord = *reinterpret_cast<const uint16_t*>( inputCursor );                                                  
    __m128i literals  = _mm_cvtsi64_si128( inputWord );                                                                      
 
Next, there is this beauty/monstrosity, which takes the bottom 8 bits of the "readOffsetLo" mask that we've copied into a GPR, sign extends it, converts it back to an unsigned integer, uses it to mask the inputWord, which could potentially be the offset, which is then xor'd with the previous offset. This will select the previous offset or update to the new offset, without a branch, and ends up being quicker than a mask and a cmov instruction, because it's a shorter dependency chain from the load on the input word. This is why we require the offset to be pre-xor'd with the previous offset in the compression stage.

    offset ^= static_cast<size_t>( static_cast<ptrdiff_t>( static_cast<int8_t>( readOffsetHalfLo ) ) ) & inputWord;   
                                                                                               
    readOffsetHalfLo >>= 8;    

Here we broadcast the low byte of the from literal register (which we're also shifting right by one byte each step) to get a mask across the entire width of a register:

	__m128i fromLiteral = _mm_shuffle_epi8( fromLiteralLo, _mm_setzero_si128() );                                       

    fromLiteralLo   = _mm_srli_si128( fromLiteralLo, 1 );                                                           

Load the match data:

    const uint8_t* matchPointer = reinterpret_cast<const uint8_t*>( outputCursor - offset );
    
	__m128i matchData = _mm_loadu_si128( reinterpret_cast<const __m128i*>( matchPointer ) );                      

Mask out the literals if we are doing a match and xor them against the potential match data. Note, the literals have been xor'd previously against the match data, so this is essentially an operation choosing between the literals and the match data. Then we'll store the data to the output, note we're always storing 16 bytes, regardless of the length of the actual output.

	literals = _mm_and_si128( fromLiteral, literals );                                                                      
	
	__m128i toStore = _mm_xor_si128( matchData, literals );   
	
	_mm_storeu_si128( reinterpret_cast<__m128i*>( outputCursor ), toStore );                                              
  
  Finally, we bump along the input and output streams:
                                                                                                                                
	outputCursor += static_cast< uint8_t >( bytesOutHalfLo );
	inputCursor  += static_cast< uint8_t >( streamBytesReadHalfLo );                                                    
                                                                                                                                
	bytesOutHalfLo        >>= 8;                                                                                        
	streamBytesReadHalfLo >>= 8;       

Here's a direct link to the [code](https://github.com/ConorStokes/LZSSE/blob/master/lzsse2/lzsse2.cpp).

## Results ##
To compare this to other similar codecs, I needed some benchmarks. To do this, I quickly ported [lzbench](https://github.com/inikep/lzbench) to Visual Studio, dropping any codec I couldn't easily get to compile. The changes required to get lzbench working with Visual Studio were fairly minor. The benchmarks here will be enwik8, because there are lots of publicly available results in the [Large Text Compression Benchmark](http://mattmahoney.net/dc/text.html) and the Silesia corpus (both as a tarball and the individual components). Note that we'll mainly focus on decompression speed vs compression ratio here, because that's the main target of this experiment at the moment. There is also a smaller benchmark done on enwik9, with competing codecs other than LZ4 variants dropped because enwik9 is a much larger file and ran out of benchmarking time. The specific machine used was a Haswell i7 4790 at stock clocks, running Windows 10 64 bit.

Here's the results for enwik8, along with all the other compressors that would work with the Visual Studio lzbench build. LZSSE does very well on relatively compressible text like this, which will be one of the common themes in the results. LZSSE2 with the optimal parse provides particularly fast decompression relative to decompression ratio on these kind of files.

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      | 11392 MB/s | 11411 MB/s |   100000000 |100.00 |
| blosclz 2015-11-10 level 1  |  1097 MB/s | 11492 MB/s |   100000000 |100.00 |
| blosclz 2015-11-10 level 3  |   567 MB/s | 11348 MB/s |    99889340 | 99.89 |
| blosclz 2015-11-10 level 6  |   212 MB/s |   758 MB/s |    55174611 | 55.17 |
| blosclz 2015-11-10 level 9  |   213 MB/s |   760 MB/s |    55123815 | 55.12 |
| brieflz 1.1.0               |    95 MB/s |   148 MB/s |    43196165 | 43.20 |
| crush 1.0 level 0           |    44 MB/s |   235 MB/s |    37294882 | 37.29 |
| crush 1.0 level 1           |  3.03 MB/s |   262 MB/s |    32847751 | 32.85 |
| crush 1.0 level 2           |  0.62 MB/s |   268 MB/s |    31699549 | 31.70 |
| fastlz 0.1 level 1          |   249 MB/s |   512 MB/s |    55239233 | 55.24 |
| fastlz 0.1 level 2          |   228 MB/s |   501 MB/s |    54163013 | 54.16 |
| lz4 r131                    |   440 MB/s |  2623 MB/s |    57262281 | 57.26 |
| lz4fast r131 level 3        |   522 MB/s |  2525 MB/s |    63557747 | 63.56 |
| lz4fast r131 level 17       |  1157 MB/s |  3916 MB/s |    86208275 | 86.21 |
| lz4hc r131 level 1          |   119 MB/s |  2389 MB/s |    48870227 | 48.87 |
| lz4hc r131 level 4          |    63 MB/s |  2560 MB/s |    43399178 | 43.40 |
| lz4hc r131 level 9          |    33 MB/s |  2612 MB/s |    42210185 | 42.21 |
| lz4hc r131 level 12         |    31 MB/s |  2591 MB/s |    42196886 | 42.20 |
| lz4hc r131 level 16         |    31 MB/s |  2605 MB/s |    42196873 | 42.20 |
| lzf 3.6 level 0             |   238 MB/s |   542 MB/s |    57695415 | 57.70 |
| lzf 3.6 level 1             |   243 MB/s |   576 MB/s |    53945381 | 53.95 |
| lzg 1.0.8 level 1           |    63 MB/s |   439 MB/s |    61116518 | 61.12 |
| lzg 1.0.8 level 4           |    36 MB/s |   433 MB/s |    54351710 | 54.35 |
| lzg 1.0.8 level 6           |    17 MB/s |   459 MB/s |    50253081 | 50.25 |
| lzg 1.0.8 level 8           |  6.13 MB/s |   517 MB/s |    46199002 | 46.20 |
| lzham 1.0 -d26 level 0      |  8.68 MB/s |   183 MB/s |    37932690 | 37.93 |
| lzham 1.0 -d26 level 1      |  2.48 MB/s |   265 MB/s |    29545382 | 29.55 |
| lzjb 2010                   |   239 MB/s |   418 MB/s |    68711273 | 68.71 |
| lzma 9.38 level 0           |    19 MB/s |    62 MB/s |    36836091 | 36.84 |
| lzma 9.38 level 2           |    16 MB/s |    75 MB/s |    33411186 | 33.41 |
| lzma 9.38 level 4           |  9.86 MB/s |    82 MB/s |    32197921 | 32.20 |
| lzma 9.38 level 5           |  1.71 MB/s |    99 MB/s |    25895800 | 25.90 |
| lzrw 15-Jul-1991 level 1    |   228 MB/s |   427 MB/s |    59669043 | 59.67 |
| lzrw 15-Jul-1991 level 2    |   217 MB/s |   484 MB/s |    59448006 | 59.45 |
| lzrw 15-Jul-1991 level 3    |   235 MB/s |   489 MB/s |    55312164 | 55.31 |
| lzrw 15-Jul-1991 level 4    |   251 MB/s |   458 MB/s |    52468327 | 52.47 |
| lzrw 15-Jul-1991 level 5    |   123 MB/s |   467 MB/s |    47874203 | 47.87 |
| lzsse4 0.1                  |   212 MB/s |  3128 MB/s |    47108403 | 47.11 |
| lzsse8 0.1                  |   214 MB/s |  3107 MB/s |    47249359 | 47.25 |
| lzsse2 0.1                  |  2.88 MB/s |  3759 MB/s |    38060594 | 38.06 |
| quicklz 1.5.0 level 1       |   328 MB/s |   515 MB/s |    52334371 | 52.33 |
| quicklz 1.5.0 level 2       |   168 MB/s |   497 MB/s |    45883075 | 45.88 |
| quicklz 1.5.0 level 3       |    43 MB/s |   689 MB/s |    44789793 | 44.79 |
| yalz77 2015-09-19 level 1   |    71 MB/s |   326 MB/s |    53016707 | 53.02 |
| yalz77 2015-09-19 level 4   |    47 MB/s |   342 MB/s |    48214192 | 48.21 |
| yalz77 2015-09-19 level 8   |    29 MB/s |   334 MB/s |    46089022 | 46.09 |
| yalz77 2015-09-19 level 12  |    21 MB/s |   324 MB/s |    44959032 | 44.96 |
| yappy 2014-03-22 level 1    |   117 MB/s |  1767 MB/s |    54965417 | 54.97 |
| yappy 2014-03-22 level 10   |    95 MB/s |  1811 MB/s |    53844726 | 53.84 |
| yappy 2014-03-22 level 100  |    88 MB/s |  1817 MB/s |    53654121 | 53.65 |
| zlib 1.2.8 level 1          |    65 MB/s |   303 MB/s |    42298774 | 42.30 |
| zlib 1.2.8 level 6          |    21 MB/s |   309 MB/s |    36548921 | 36.55 |
| zlib 1.2.8 level 9          |    17 MB/s |   308 MB/s |    36475792 | 36.48 |
| zstd v0.4.1 level 1         |   262 MB/s |   493 MB/s |    40822932 | 40.82 |
| zstd v0.4.1 level 2         |   192 MB/s |   393 MB/s |    37789941 | 37.79 |
| zstd v0.4.1 level 5         |    91 MB/s |   341 MB/s |    35228432 | 35.23 |
| zstd v0.4.1 level 9         |    25 MB/s |   382 MB/s |    31789698 | 31.79 |
| zstd v0.4.1 level 13        |  7.98 MB/s |   382 MB/s |    30761914 | 30.76 |
| zstd v0.4.1 level 17        |  3.58 MB/s |   340 MB/s |    28907539 | 28.91 |
| zstd v0.4.1 level 20        |  2.20 MB/s |   241 MB/s |    27195437 | 27.20 |
| shrinker 0.1                |   296 MB/s |   806 MB/s |    51493020 | 51.49 |
| wflz 2015-09-16             |   193 MB/s |   740 MB/s |    63521814 | 63.52 |
| lzmat 1.01                  |    32 MB/s |   374 MB/s |    41270839 | 41.27 |
|                             |            |            |             |       |


For some more mixed results, the [silesia corpus](http://sun.aei.polsl.pl/~sdeor/index.php?page=silesia) as a tar shows that on quite varied data the decompression speed vs compression ratio is still pretty good. We'll break these down into the individual components for a little analysis.  

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      | 11336 MB/s | 11513 MB/s |   211948032 |100.00 |
| blosclz 2015-11-10 level 1  |  1204 MB/s | 11317 MB/s |   211644466 | 99.86 |
| blosclz 2015-11-10 level 3  |   635 MB/s |  9196 MB/s |   204503525 | 96.49 |
| blosclz 2015-11-10 level 6  |   299 MB/s |  1298 MB/s |   113321280 | 53.47 |
| blosclz 2015-11-10 level 9  |   279 MB/s |   969 MB/s |   102813688 | 48.51 |
| brieflz 1.1.0               |   127 MB/s |   209 MB/s |    81984968 | 38.68 |
| crush 1.0 level 0           |    39 MB/s |   285 MB/s |    73066686 | 34.47 |
| crush 1.0 level 1           |  6.22 MB/s |   320 MB/s |    66499720 | 31.38 |
| crush 1.0 level 2           |  0.83 MB/s |   327 MB/s |    63751752 | 30.08 |
| fastlz 0.1 level 1          |   305 MB/s |   728 MB/s |   104627960 | 49.36 |
| fastlz 0.1 level 2          |   308 MB/s |   712 MB/s |   100905960 | 47.61 |
| lz4 r131                    |   597 MB/s |  3022 MB/s |   100880708 | 47.60 |
| lz4fast r131 level 3        |   696 MB/s |  3050 MB/s |   107066575 | 50.52 |
| lz4fast r131 level 17       |  1072 MB/s |  3659 MB/s |   131732847 | 62.15 |
| lz4hc r131 level 1          |   138 MB/s |  2743 MB/s |    89227407 | 42.10 |
| lz4hc r131 level 4          |    76 MB/s |  2943 MB/s |    80485966 | 37.97 |
| lz4hc r131 level 9          |    32 MB/s |  3018 MB/s |    77919239 | 36.76 |
| lz4hc r131 level 12         |    25 MB/s |  3009 MB/s |    77852883 | 36.73 |
| lz4hc r131 level 16         |    16 MB/s |  3031 MB/s |    77841832 | 36.73 |
| lzf 3.6 level 0             |   327 MB/s |   774 MB/s |   105681992 | 49.86 |
| lzf 3.6 level 1             |   328 MB/s |   791 MB/s |   102041053 | 48.14 |
| lzg 1.0.8 level 1           |    63 MB/s |   590 MB/s |   108553615 | 51.22 |
| lzg 1.0.8 level 4           |    41 MB/s |   603 MB/s |    95930522 | 45.26 |
| lzg 1.0.8 level 6           |    25 MB/s |   639 MB/s |    89490172 | 42.22 |
| lzg 1.0.8 level 8           |  9.75 MB/s |   695 MB/s |    83606917 | 39.45 |
| lzham 1.0 -d26 level 0      |  9.97 MB/s |   224 MB/s |    64120667 | 30.25 |
| lzham 1.0 -d26 level 1      |  2.89 MB/s |   294 MB/s |    54759533 | 25.84 |
| lzjb 2010                   |   308 MB/s |   539 MB/s |   122671547 | 57.88 |
| lzma 9.38 level 0           |    23 MB/s |    73 MB/s |    64014180 | 30.20 |
| lzma 9.38 level 2           |    20 MB/s |    83 MB/s |    58868127 | 27.77 |
| lzma 9.38 level 4           |    12 MB/s |    90 MB/s |    57202325 | 26.99 |
| lzma 9.38 level 5           |  3.05 MB/s |    98 MB/s |    49722627 | 23.46 |
| lzrw 15-Jul-1991 level 1    |   279 MB/s |   569 MB/s |   113761751 | 53.67 |
| lzrw 15-Jul-1991 level 2    |   258 MB/s |   666 MB/s |   112344625 | 53.01 |
| lzrw 15-Jul-1991 level 3    |   327 MB/s |   670 MB/s |   105422213 | 49.74 |
| lzrw 15-Jul-1991 level 4    |   346 MB/s |   578 MB/s |   100128419 | 47.24 |
| lzrw 15-Jul-1991 level 5    |   149 MB/s |   611 MB/s |    90816056 | 42.85 |
| lzsse4 0.1                  |   269 MB/s |  3192 MB/s |    95918518 | 45.26 |
| lzsse8 0.1                  |   273 MB/s |  3417 MB/s |    94938891 | 44.79 |
| lzsse2 0.1                  |  0.08 MB/s |  3604 MB/s |    75668535 | 35.70 |
| quicklz 1.5.0 level 1       |   471 MB/s |   615 MB/s |    94723809 | 44.69 |
| quicklz 1.5.0 level 2       |   227 MB/s |   558 MB/s |    84563229 | 39.90 |
| quicklz 1.5.0 level 3       |    55 MB/s |   921 MB/s |    81821485 | 38.60 |
| yalz77 2015-09-19 level 1   |    84 MB/s |   488 MB/s |    93943755 | 44.32 |
| yalz77 2015-09-19 level 4   |    54 MB/s |   494 MB/s |    87396316 | 41.23 |
| yalz77 2015-09-19 level 8   |    34 MB/s |   492 MB/s |    85165013 | 40.18 |
| yalz77 2015-09-19 level 12  |    24 MB/s |   488 MB/s |    84061090 | 39.66 |
| yappy 2014-03-22 level 1    |   140 MB/s |  2199 MB/s |   105751224 | 49.89 |
| yappy 2014-03-22 level 10   |   109 MB/s |  2433 MB/s |   100018782 | 47.19 |
| yappy 2014-03-22 level 100  |    79 MB/s |  2515 MB/s |    98672575 | 46.56 |
| zlib 1.2.8 level 1          |    76 MB/s |   352 MB/s |    77260018 | 36.45 |
| zlib 1.2.8 level 6          |    26 MB/s |   376 MB/s |    68228464 | 32.19 |
| zlib 1.2.8 level 9          |    10 MB/s |   379 MB/s |    67643845 | 31.92 |
| zstd v0.4.1 level 1         |   348 MB/s |   603 MB/s |    73803509 | 34.82 |
| zstd v0.4.1 level 2         |   269 MB/s |   523 MB/s |    70269474 | 33.15 |
| zstd v0.4.1 level 5         |   112 MB/s |   468 MB/s |    65419600 | 30.87 |
| zstd v0.4.1 level 9         |    34 MB/s |   507 MB/s |    60463173 | 28.53 |
| zstd v0.4.1 level 13        |    14 MB/s |   513 MB/s |    58851341 | 27.77 |
| zstd v0.4.1 level 17        |  6.10 MB/s |   505 MB/s |    56904225 | 26.85 |
| zstd v0.4.1 level 20        |  4.04 MB/s |   449 MB/s |    56182980 | 26.51 |
| shrinker 0.1                | 11329 MB/s | 11290 MB/s |   211948032 |100.00 |
| wflz 2015-09-16             |   259 MB/s |  1116 MB/s |   109605178 | 51.71 |
| lzmat 1.01                  |    34 MB/s |   484 MB/s |    76486014 | 36.09 |
|                             |            |            |             |       |


The Dickens file in the corpus is the collective work of Charles Dickens. This kind of fairly compressible text works well with LZSSE2, where relatively long matches mean a lot of copied bytes per step, for relatively good performance. Also, LZSSE2 works well in terms of the probability for control words too, because there are relatively short literal runs per number of matches.

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      | 13905 MB/s | 13681 MB/s |    10192446 |100.00 |
| blosclz 2015-11-10 level 1  |  1141 MB/s | 13517 MB/s |    10192446 |100.00 |
| blosclz 2015-11-10 level 3  |   594 MB/s | 13886 MB/s |    10192446 |100.00 |
| blosclz 2015-11-10 level 6  |   209 MB/s |  1137 MB/s |     7582534 | 74.39 |
| blosclz 2015-11-10 level 9  |   209 MB/s |   740 MB/s |     6097250 | 59.82 |
| brieflz 1.1.0               |    95 MB/s |   140 MB/s |     4647459 | 45.60 |
| crush 1.0 level 0           |    51 MB/s |   233 MB/s |     4144644 | 40.66 |
| crush 1.0 level 1           |  2.44 MB/s |   280 MB/s |     3467976 | 34.02 |
| crush 1.0 level 2           |  0.33 MB/s |   290 MB/s |     3350089 | 32.87 |
| fastlz 0.1 level 1          |   257 MB/s |   507 MB/s |     6057734 | 59.43 |
| fastlz 0.1 level 2          |   238 MB/s |   498 MB/s |     5995417 | 58.82 |
| lz4 r131                    |   394 MB/s |  2574 MB/s |     6428742 | 63.07 |
| lz4fast r131 level 3        |   471 MB/s |  2484 MB/s |     7082423 | 69.49 |
| lz4fast r131 level 17       |  1074 MB/s |  3581 MB/s |     9215415 | 90.41 |
| lz4hc r131 level 1          |   115 MB/s |  2282 MB/s |     5525869 | 54.22 |
| lz4hc r131 level 4          |    53 MB/s |  2539 MB/s |     4684564 | 45.96 |
| lz4hc r131 level 9          |    21 MB/s |  2605 MB/s |     4434445 | 43.51 |
| lz4hc r131 level 12         |    20 MB/s |  2579 MB/s |     4431257 | 43.48 |
| lz4hc r131 level 16         |    20 MB/s |  2633 MB/s |     4431257 | 43.48 |
| lzf 3.6 level 0             |   243 MB/s |   530 MB/s |     6329084 | 62.10 |
| lzf 3.6 level 1             |   243 MB/s |   577 MB/s |     5850777 | 57.40 |
| lzg 1.0.8 level 1           |    57 MB/s |   380 MB/s |     6937973 | 68.07 |
| lzg 1.0.8 level 4           |    29 MB/s |   383 MB/s |     6238963 | 61.21 |
| lzg 1.0.8 level 6           |    13 MB/s |   448 MB/s |     5663702 | 55.57 |
| lzg 1.0.8 level 8           |  4.54 MB/s |   557 MB/s |     5021592 | 49.27 |
| lzham 1.0 -d26 level 0      |  8.31 MB/s |   178 MB/s |     4248673 | 41.68 |
| lzham 1.0 -d26 level 1      |  2.59 MB/s |   288 MB/s |     3172720 | 31.13 |
| lzjb 2010                   |   211 MB/s |   372 MB/s |     7728055 | 75.82 |
| lzma 9.38 level 0           |    17 MB/s |    61 MB/s |     4044850 | 39.68 |
| lzma 9.38 level 2           |    14 MB/s |    77 MB/s |     3613945 | 35.46 |
| lzma 9.38 level 4           |    10 MB/s |    80 MB/s |     3525850 | 34.59 |
| lzma 9.38 level 5           |  1.84 MB/s |   107 MB/s |     2829392 | 27.76 |
| lzrw 15-Jul-1991 level 1    |   226 MB/s |   399 MB/s |     6595660 | 64.71 |
| lzrw 15-Jul-1991 level 2    |   228 MB/s |   461 MB/s |     6590368 | 64.66 |
| lzrw 15-Jul-1991 level 3    |   233 MB/s |   494 MB/s |     6157185 | 60.41 |
| lzrw 15-Jul-1991 level 4    |   265 MB/s |   483 MB/s |     5813134 | 57.03 |
| lzrw 15-Jul-1991 level 5    |   113 MB/s |   482 MB/s |     5128916 | 50.32 |
| lzsse4 0.1                  |   213 MB/s |  3017 MB/s |     5161515 | 50.64 |
| lzsse8 0.1                  |   215 MB/s |  2961 MB/s |     5174322 | 50.77 |
| lzsse2 0.1                  |  2.90 MB/s |  3941 MB/s |     3872251 | 37.99 |
| quicklz 1.5.0 level 1       |   340 MB/s |   533 MB/s |     5831353 | 57.21 |
| quicklz 1.5.0 level 2       |   162 MB/s |   480 MB/s |     4953617 | 48.60 |
| quicklz 1.5.0 level 3       |    39 MB/s |   720 MB/s |     5099490 | 50.03 |
| yalz77 2015-09-19 level 1   |    80 MB/s |   321 MB/s |     5634109 | 55.28 |
| yalz77 2015-09-19 level 4   |    51 MB/s |   341 MB/s |     5030032 | 49.35 |
| yalz77 2015-09-19 level 8   |    35 MB/s |   348 MB/s |     4842921 | 47.51 |
| yalz77 2015-09-19 level 12  |    27 MB/s |   353 MB/s |     4759454 | 46.70 |
| yappy 2014-03-22 level 1    |   112 MB/s |  1460 MB/s |     6013187 | 59.00 |
| yappy 2014-03-22 level 10   |    89 MB/s |  1498 MB/s |     5872799 | 57.62 |
| yappy 2014-03-22 level 100  |    84 MB/s |  1505 MB/s |     5846776 | 57.36 |
| zlib 1.2.8 level 1          |    65 MB/s |   310 MB/s |     4585618 | 44.99 |
| zlib 1.2.8 level 6          |    16 MB/s |   323 MB/s |     3871634 | 37.99 |
| zlib 1.2.8 level 9          |    12 MB/s |   325 MB/s |     3854735 | 37.82 |
| zstd v0.4.1 level 1         |   242 MB/s |   473 MB/s |     4280491 | 42.00 |
| zstd v0.4.1 level 2         |   172 MB/s |   356 MB/s |     3910652 | 38.37 |
| zstd v0.4.1 level 5         |    90 MB/s |   322 MB/s |     3722585 | 36.52 |
| zstd v0.4.1 level 9         |    23 MB/s |   374 MB/s |     3351613 | 32.88 |
| zstd v0.4.1 level 13        |  6.45 MB/s |   381 MB/s |     3214233 | 31.54 |
| zstd v0.4.1 level 17        |  3.45 MB/s |   383 MB/s |     3054092 | 29.96 |
| zstd v0.4.1 level 20        |  3.11 MB/s |   382 MB/s |     3036816 | 29.79 |
| shrinker 0.1                |   316 MB/s |   740 MB/s |     5903074 | 57.92 |
| wflz 2015-09-16             |   189 MB/s |   671 MB/s |     7275900 | 71.39 |
| lzmat 1.01                  |    23 MB/s |   380 MB/s |     4501480 | 44.16 |
|                             |            |            |             |       |


Mozilla is the tarred binaries of the Mozilla 1.0 distribution. Things here aren't as rosy as we have shorter matches and lots of literal runs, although there are some long runs of the same character. The LZSSE2 compressor hits the pathological case for it's binary tree based matcher, slowing to an absolute crawl on compression. We're still competitive with LZ4 HC for compression ratio vs decompression performance.

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      | 11349 MB/s | 11374 MB/s |    51220480 |100.00 |
| blosclz 2015-11-10 level 1  |  1355 MB/s | 11321 MB/s |    51220480 |100.00 |
| blosclz 2015-11-10 level 3  |   709 MB/s | 10787 MB/s |    50949746 | 99.47 |
| blosclz 2015-11-10 level 6  |   302 MB/s |  1446 MB/s |    32729305 | 63.90 |
| blosclz 2015-11-10 level 9  |   282 MB/s |   938 MB/s |    27353816 | 53.40 |
| brieflz 1.1.0               |   127 MB/s |   229 MB/s |    22261084 | 43.46 |
| crush 1.0 level 0           |    34 MB/s |   259 MB/s |    20573545 | 40.17 |
| crush 1.0 level 1           |  9.79 MB/s |   274 MB/s |    19750089 | 38.56 |
| crush 1.0 level 2           |  1.51 MB/s |   283 MB/s |    18760569 | 36.63 |
| fastlz 0.1 level 1          |   294 MB/s |   782 MB/s |    27610306 | 53.90 |
| fastlz 0.1 level 2          |   293 MB/s |   754 MB/s |    27070715 | 52.85 |
| lz4 r131                    |   637 MB/s |  2929 MB/s |    26435667 | 51.61 |
| lz4fast r131 level 3        |   745 MB/s |  3019 MB/s |    27973598 | 54.61 |
| lz4fast r131 level 17       |  1116 MB/s |  3749 MB/s |    34039558 | 66.46 |
| lz4hc r131 level 1          |   132 MB/s |  2765 MB/s |    23886540 | 46.63 |
| lz4hc r131 level 4          |    85 MB/s |  2982 MB/s |    22513368 | 43.95 |
| lz4hc r131 level 9          |    44 MB/s |  3067 MB/s |    22092109 | 43.13 |
| lz4hc r131 level 12         |    28 MB/s |  3004 MB/s |    22071142 | 43.09 |
| lz4hc r131 level 16         |    11 MB/s |  3017 MB/s |    22062995 | 43.07 |
| lzf 3.6 level 0             |   339 MB/s |   806 MB/s |    26982485 | 52.68 |
| lzf 3.6 level 1             |   324 MB/s |   800 MB/s |    26687518 | 52.10 |
| lzg 1.0.8 level 1           |    51 MB/s |   625 MB/s |    26280004 | 51.31 |
| lzg 1.0.8 level 4           |    40 MB/s |   631 MB/s |    24613811 | 48.05 |
| lzg 1.0.8 level 6           |    29 MB/s |   634 MB/s |    23802178 | 46.47 |
| lzg 1.0.8 level 8           |    15 MB/s |   640 MB/s |    22980170 | 44.87 |
| lzham 1.0 -d26 level 0      |  9.38 MB/s |   211 MB/s |    16337459 | 31.90 |
| lzham 1.0 -d26 level 1      |  2.66 MB/s |   246 MB/s |    14749315 | 28.80 |
| lzjb 2010                   |   335 MB/s |   583 MB/s |    28867642 | 56.36 |
| lzma 9.38 level 0           |    22 MB/s |    65 MB/s |    16425272 | 32.07 |
| lzma 9.38 level 2           |    19 MB/s |    72 MB/s |    15390824 | 30.05 |
| lzma 9.38 level 4           |    12 MB/s |    77 MB/s |    14878636 | 29.05 |
| lzma 9.38 level 5           |  3.12 MB/s |    82 MB/s |    13512367 | 26.38 |
| lzrw 15-Jul-1991 level 1    |   295 MB/s |   622 MB/s |    27333863 | 53.37 |
| lzrw 15-Jul-1991 level 2    |   260 MB/s |   742 MB/s |    27187667 | 53.08 |
| lzrw 15-Jul-1991 level 3    |   346 MB/s |   697 MB/s |    26210993 | 51.17 |
| lzrw 15-Jul-1991 level 4    |   349 MB/s |   557 MB/s |    25578346 | 49.94 |
| lzrw 15-Jul-1991 level 5    |   154 MB/s |   570 MB/s |    24126074 | 47.10 |
| lzsse4 0.1                  |   258 MB/s |  2635 MB/s |    27406939 | 53.51 |
| lzsse8 0.1                  |   262 MB/s |  2818 MB/s |    26993974 | 52.70 |
| lzsse2 0.1                  |  0.03 MB/s |  3030 MB/s |    22452327 | 43.83 |
| quicklz 1.5.0 level 1       |   478 MB/s |   579 MB/s |    24756819 | 48.33 |
| quicklz 1.5.0 level 2       |   223 MB/s |   503 MB/s |    23337850 | 45.56 |
| quicklz 1.5.0 level 3       |    56 MB/s |   831 MB/s |    22240936 | 43.42 |
| yalz77 2015-09-19 level 1   |    69 MB/s |   501 MB/s |    25454532 | 49.70 |
| yalz77 2015-09-19 level 4   |    46 MB/s |   491 MB/s |    24449446 | 47.73 |
| yalz77 2015-09-19 level 8   |    30 MB/s |   490 MB/s |    24076623 | 47.01 |
| yalz77 2015-09-19 level 12  |    20 MB/s |   483 MB/s |    23881699 | 46.63 |
| yappy 2014-03-22 level 1    |   142 MB/s |  2330 MB/s |    26958469 | 52.63 |
| yappy 2014-03-22 level 10   |   109 MB/s |  2662 MB/s |    25715135 | 50.20 |
| yappy 2014-03-22 level 100  |    77 MB/s |  2707 MB/s |    25271700 | 49.34 |
| zlib 1.2.8 level 1          |    66 MB/s |   317 MB/s |    20577226 | 40.17 |
| zlib 1.2.8 level 6          |    23 MB/s |   336 MB/s |    19089124 | 37.27 |
| zlib 1.2.8 level 9          |  6.36 MB/s |   339 MB/s |    19044396 | 37.18 |
| zstd v0.4.1 level 1         |   345 MB/s |   556 MB/s |    20173050 | 39.38 |
| zstd v0.4.1 level 2         |   280 MB/s |   526 MB/s |    19244936 | 37.57 |
| zstd v0.4.1 level 5         |   110 MB/s |   487 MB/s |    18239976 | 35.61 |
| zstd v0.4.1 level 9         |    37 MB/s |   518 MB/s |    17155767 | 33.49 |
| zstd v0.4.1 level 13        |    20 MB/s |   516 MB/s |    16862530 | 32.92 |
| zstd v0.4.1 level 17        |  8.46 MB/s |   503 MB/s |    16552487 | 32.32 |
| zstd v0.4.1 level 20        |  5.83 MB/s |   435 MB/s |    16397038 | 32.01 |
| shrinker 0.1                |   397 MB/s |  1162 MB/s |    23797180 | 46.46 |
| wflz 2015-09-16             |   249 MB/s |  1270 MB/s |    28757022 | 56.14 |
| lzmat 1.01                  |    29 MB/s |   458 MB/s |    20575801 | 40.17 |
|                             |            |            |             |       |


Silesia's mr is a magnetic resonance image. Again, we have pathological compression performance for LZSEE2. LZSSE8 does well here because of the longer literal runs, but LZSEE2's stronger matching puts it ahead.

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
 ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      | 14023 MB/s | 13695 MB/s |     9970564 |100.00 |
| blosclz 2015-11-10 level 1  |  1531 MB/s | 13473 MB/s |     9970564 |100.00 |
| blosclz 2015-11-10 level 3  |   818 MB/s | 13455 MB/s |     9970564 |100.00 |
| blosclz 2015-11-10 level 6  |   343 MB/s |  1081 MB/s |     5842843 | 58.60 |
| blosclz 2015-11-10 level 9  |   340 MB/s |   858 MB/s |     5122078 | 51.37 |
| brieflz 1.1.0               |   129 MB/s |   177 MB/s |     4393337 | 44.06 |
| crush 1.0 level 0           |    51 MB/s |   280 MB/s |     4151312 | 41.64 |
| crush 1.0 level 1           |  4.74 MB/s |   312 MB/s |     3691296 | 37.02 |
| crush 1.0 level 2           |  0.42 MB/s |   301 MB/s |     3532781 | 35.43 |
| fastlz 0.1 level 1          |   456 MB/s |  1032 MB/s |     5102964 | 51.18 |
| fastlz 0.1 level 2          |   375 MB/s |  1028 MB/s |     5073384 | 50.88 |
| lz4 r131                    |   608 MB/s |  3157 MB/s |     5440937 | 54.57 |
| lz4fast r131 level 3        |   751 MB/s |  3243 MB/s |     5590948 | 56.07 |
| lz4fast r131 level 17       |  1100 MB/s |  3899 MB/s |     6040318 | 60.58 |
| lz4hc r131 level 1          |   142 MB/s |  2558 MB/s |     5077585 | 50.93 |
| lz4hc r131 level 4          |    68 MB/s |  2826 MB/s |     4529845 | 45.43 |
| lz4hc r131 level 9          |    19 MB/s |  2965 MB/s |     4247505 | 42.60 |
| lz4hc r131 level 12         |    14 MB/s |  2998 MB/s |     4244773 | 42.57 |
| lz4hc r131 level 16         |  4.96 MB/s |  2970 MB/s |     4244224 | 42.57 |
| lzf 3.6 level 0             |   424 MB/s |   801 MB/s |     5388536 | 54.04 |
| lzf 3.6 level 1             |   450 MB/s |   858 MB/s |     4999202 | 50.14 |
| lzg 1.0.8 level 1           |    70 MB/s |   552 MB/s |     5494713 | 55.11 |
| lzg 1.0.8 level 4           |    37 MB/s |   545 MB/s |     5262984 | 52.79 |
| lzg 1.0.8 level 6           |    16 MB/s |   569 MB/s |     5056631 | 50.72 |
| lzg 1.0.8 level 8           |  6.39 MB/s |   614 MB/s |     4807004 | 48.21 |
| lzham 1.0 -d26 level 0      |  8.64 MB/s |   176 MB/s |     3155756 | 31.65 |
| lzham 1.0 -d26 level 1      |  3.34 MB/s |   232 MB/s |     2985074 | 29.94 |
| lzjb 2010                   |   336 MB/s |   633 MB/s |     5854361 | 58.72 |
| lzma 9.38 level 0           |    21 MB/s |    69 MB/s |     3157626 | 31.67 |
| lzma 9.38 level 2           |    18 MB/s |    76 MB/s |     3092086 | 31.01 |
| lzma 9.38 level 4           |    12 MB/s |    81 MB/s |     3040649 | 30.50 |
| lzma 9.38 level 5           |  2.92 MB/s |    84 MB/s |     2751595 | 27.60 |
| lzrw 15-Jul-1991 level 1    |   406 MB/s |   770 MB/s |     5492440 | 55.09 |
| lzrw 15-Jul-1991 level 2    |   383 MB/s |   970 MB/s |     5453392 | 54.69 |
| lzrw 15-Jul-1991 level 3    |   470 MB/s |   960 MB/s |     5187387 | 52.03 |
| lzrw 15-Jul-1991 level 4    |   484 MB/s |   796 MB/s |     5013906 | 50.29 |
| lzrw 15-Jul-1991 level 5    |   172 MB/s |   776 MB/s |     4629636 | 46.43 |
| lzsse4 0.1                  |   254 MB/s |  2589 MB/s |     7034206 | 70.55 |
| lzsse8 0.1                  |   281 MB/s |  3031 MB/s |     6856430 | 68.77 |
| lzsse2 0.1                  |  0.03 MB/s |  3315 MB/s |     4006788 | 40.19 |
| quicklz 1.5.0 level 1       |   636 MB/s |   526 MB/s |     4778194 | 47.92 |
| quicklz 1.5.0 level 2       |   277 MB/s |   486 MB/s |     4317635 | 43.30 |
| quicklz 1.5.0 level 3       |    56 MB/s |   956 MB/s |     4559691 | 45.73 |
| yalz77 2015-09-19 level 1   |    79 MB/s |   484 MB/s |     5269368 | 52.85 |
| yalz77 2015-09-19 level 4   |    56 MB/s |   473 MB/s |     4928226 | 49.43 |
| yalz77 2015-09-19 level 8   |    37 MB/s |   467 MB/s |     4792581 | 48.07 |
| yalz77 2015-09-19 level 12  |    28 MB/s |   468 MB/s |     4730734 | 47.45 |
| yappy 2014-03-22 level 1    |   142 MB/s |  1692 MB/s |     6113745 | 61.32 |
| yappy 2014-03-22 level 10   |   108 MB/s |  2492 MB/s |     5258774 | 52.74 |
| yappy 2014-03-22 level 100  |    68 MB/s |  2903 MB/s |     5016379 | 50.31 |
| zlib 1.2.8 level 1          |    81 MB/s |   364 MB/s |     3828366 | 38.40 |
| zlib 1.2.8 level 6          |    16 MB/s |   342 MB/s |     3675941 | 36.87 |
| zlib 1.2.8 level 9          |  6.01 MB/s |   354 MB/s |     3660794 | 36.72 |
| zstd v0.4.1 level 1         |   313 MB/s |   643 MB/s |     3832967 | 38.44 |
| zstd v0.4.1 level 2         |   229 MB/s |   493 MB/s |     3611124 | 36.22 |
| zstd v0.4.1 level 5         |    98 MB/s |   373 MB/s |     3455926 | 34.66 |
| zstd v0.4.1 level 9         |    27 MB/s |   382 MB/s |     3360748 | 33.71 |
| zstd v0.4.1 level 13        |  9.32 MB/s |   388 MB/s |     3298241 | 33.08 |
| zstd v0.4.1 level 17        |  5.19 MB/s |   382 MB/s |     3238037 | 32.48 |
| zstd v0.4.1 level 20        |  4.62 MB/s |   380 MB/s |     3232535 | 32.42 |
| shrinker 0.1                |   440 MB/s |   938 MB/s |     5348926 | 53.65 |
| wflz 2015-09-16             |   317 MB/s |  1580 MB/s |     6128033 | 61.46 |
| lzmat 1.01                  |    40 MB/s |   468 MB/s |     4211586 | 42.24 |
|                             |            |            |             |       |


Silesia's nci is a database of chemical structures and is highly compressible. LZSSE loves this file, especially LZSSE2, where we get over 60% of memcpy speed for relatively competitive compression ratios (yes, that is over 7GB/s). The main thing here is that we get a lot of bytes copied per step, which increases the relative efficiency of the decompression.

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      | 11459 MB/s | 11455 MB/s |    33553445 |100.00 |
| blosclz 2015-11-10 level 1  |  1122 MB/s | 11374 MB/s |    33553445 |100.00 |
| blosclz 2015-11-10 level 3  |   609 MB/s |  5875 MB/s |    28744881 | 85.67 |
| blosclz 2015-11-10 level 6  |   588 MB/s |  1622 MB/s |     6809799 | 20.30 |
| blosclz 2015-11-10 level 9  |   589 MB/s |  1622 MB/s |     6809799 | 20.30 |
| brieflz 1.1.0               |   246 MB/s |   392 MB/s |     4559358 | 13.59 |
| crush 1.0 level 0           |   132 MB/s |   643 MB/s |     3668407 | 10.93 |
| crush 1.0 level 1           |    19 MB/s |   848 MB/s |     2806144 |  8.36 |
| crush 1.0 level 2           |  1.97 MB/s |   912 MB/s |     2624033 |  7.82 |
| fastlz 0.1 level 1          |   701 MB/s |  1134 MB/s |     6947358 | 20.71 |
| fastlz 0.1 level 2          |   636 MB/s |  1117 MB/s |     6583289 | 19.62 |
| lz4 r131                    |  1103 MB/s |  4133 MB/s |     5533040 | 16.49 |
| lz4fast r131 level 3        |  1176 MB/s |  4068 MB/s |     5684097 | 16.94 |
| lz4fast r131 level 17       |  1295 MB/s |  3883 MB/s |     7172738 | 21.38 |
| lz4hc r131 level 1          |   270 MB/s |  3783 MB/s |     5059842 | 15.08 |
| lz4hc r131 level 4          |   132 MB/s |  4359 MB/s |     4078371 | 12.15 |
| lz4hc r131 level 9          |    28 MB/s |  4758 MB/s |     3679323 | 10.97 |
| lz4hc r131 level 12         |    20 MB/s |  4853 MB/s |     3662990 | 10.92 |
| lz4hc r131 level 16         |    18 MB/s |  4836 MB/s |     3661231 | 10.91 |
| lzf 3.6 level 0             |   679 MB/s |  1458 MB/s |     7263009 | 21.65 |
| lzf 3.6 level 1             |   691 MB/s |  1559 MB/s |     6936021 | 20.67 |
| lzg 1.0.8 level 1           |   114 MB/s |   962 MB/s |     8218641 | 24.49 |
| lzg 1.0.8 level 4           |    72 MB/s |  1082 MB/s |     5991131 | 17.86 |
| lzg 1.0.8 level 6           |    50 MB/s |  1186 MB/s |     5304820 | 15.81 |
| lzg 1.0.8 level 8           |    19 MB/s |  1330 MB/s |     4613756 | 13.75 |
| lzham 1.0 -d26 level 0      |    15 MB/s |   566 MB/s |     2813533 |  8.39 |
| lzham 1.0 -d26 level 1      |  4.35 MB/s |   933 MB/s |     2135880 |  6.37 |
| lzjb 2010                   |   487 MB/s |   776 MB/s |     8714416 | 25.97 |
| lzma 9.38 level 0           |    56 MB/s |   242 MB/s |     2777997 |  8.28 |
| lzma 9.38 level 2           |    58 MB/s |   301 MB/s |     2487371 |  7.41 |
| lzma 9.38 level 4           |    48 MB/s |   331 MB/s |     2398761 |  7.15 |
| lzma 9.38 level 5           |  6.52 MB/s |   362 MB/s |     1978318 |  5.90 |
| lzrw 15-Jul-1991 level 1    |   511 MB/s |   796 MB/s |    10423020 | 31.06 |
| lzrw 15-Jul-1991 level 2    |   524 MB/s |   935 MB/s |     9698071 | 28.90 |
| lzrw 15-Jul-1991 level 3    |   547 MB/s |   976 MB/s |     8630804 | 25.72 |
| lzrw 15-Jul-1991 level 4    |   593 MB/s |   905 MB/s |     8412942 | 25.07 |
| lzrw 15-Jul-1991 level 5    |   190 MB/s |  1081 MB/s |     6556582 | 19.54 |
| lzsse4 0.1                  |   497 MB/s |  5503 MB/s |     5711408 | 17.02 |
| lzsse8 0.1                  |   509 MB/s |  5254 MB/s |     5796797 | 17.28 |
| lzsse2 0.1                  |  1.85 MB/s |  7195 MB/s |     3703275 | 11.04 |
| quicklz 1.5.0 level 1       |   847 MB/s |  1343 MB/s |     6160636 | 18.36 |
| quicklz 1.5.0 level 2       |   466 MB/s |  1468 MB/s |     4867322 | 14.51 |
| quicklz 1.5.0 level 3       |    96 MB/s |  1945 MB/s |     4482913 | 13.36 |
| yalz77 2015-09-19 level 1   |   288 MB/s |   902 MB/s |     5050596 | 15.05 |
| yalz77 2015-09-19 level 4   |   168 MB/s |  1006 MB/s |     4192253 | 12.49 |
| yalz77 2015-09-19 level 8   |   113 MB/s |  1017 MB/s |     3910381 | 11.65 |
| yalz77 2015-09-19 level 12  |    89 MB/s |  1013 MB/s |     3767494 | 11.23 |
| yappy 2014-03-22 level 1    |   249 MB/s |  2719 MB/s |     8224487 | 24.51 |
| yappy 2014-03-22 level 10   |   169 MB/s |  3380 MB/s |     6569703 | 19.58 |
| yappy 2014-03-22 level 100  |    65 MB/s |  3505 MB/s |     6280019 | 18.72 |
| zlib 1.2.8 level 1          |   183 MB/s |   674 MB/s |     4624597 | 13.78 |
| zlib 1.2.8 level 6          |    70 MB/s |   809 MB/s |     3200188 |  9.54 |
| zlib 1.2.8 level 9          |    13 MB/s |   840 MB/s |     2988001 |  8.91 |
| zstd v0.4.1 level 1         |   732 MB/s |  1047 MB/s |     2876326 |  8.57 |
| zstd v0.4.1 level 2         |   688 MB/s |  1018 MB/s |     2937436 |  8.75 |
| zstd v0.4.1 level 5         |   278 MB/s |  1015 MB/s |     2810373 |  8.38 |
| zstd v0.4.1 level 9         |   110 MB/s |  1317 MB/s |     2360355 |  7.03 |
| zstd v0.4.1 level 13        |    46 MB/s |  1459 MB/s |     2197208 |  6.55 |
| zstd v0.4.1 level 17        |  7.67 MB/s |  1561 MB/s |     1938899 |  5.78 |
| zstd v0.4.1 level 20        |  3.85 MB/s |  1418 MB/s |     1802903 |  5.37 |
| shrinker 0.1                |   885 MB/s |  1892 MB/s |     5350052 | 15.94 |
| wflz 2015-09-16             |   778 MB/s |  2114 MB/s |     6316882 | 18.83 |
| lzmat 1.01                  |    45 MB/s |   904 MB/s |     4046914 | 12.06 |
|                             |            |            |             |       |


Ooffice is a dll from open office. Again with this kind of binary LZSSE does not do as well, although LZSSE8 does a lot better here. Performance is much closer to LZ4.

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      | 17628 MB/s | 17832 MB/s |     6152192 |100.00 |
| blosclz 2015-11-10 level 1  |  1358 MB/s | 17281 MB/s |     6152192 |100.00 |
| blosclz 2015-11-10 level 3  |   699 MB/s | 17330 MB/s |     6152192 |100.00 |
| blosclz 2015-11-10 level 6  |   235 MB/s |  5961 MB/s |     5790799 | 94.13 |
| blosclz 2015-11-10 level 9  |   196 MB/s |   722 MB/s |     4297469 | 69.85 |
| brieflz 1.1.0               |    94 MB/s |   172 MB/s |     3543046 | 57.59 |
| crush 1.0 level 0           |    24 MB/s |   190 MB/s |     3188991 | 51.84 |
| crush 1.0 level 1           |  5.94 MB/s |   195 MB/s |     3077918 | 50.03 |
| crush 1.0 level 2           |  1.35 MB/s |   202 MB/s |     2958514 | 48.09 |
| fastlz 0.1 level 1          |   197 MB/s |   538 MB/s |     4301654 | 69.92 |
| fastlz 0.1 level 2          |   201 MB/s |   515 MB/s |     4259180 | 69.23 |
| lz4 r131                    |   514 MB/s |  2649 MB/s |     4338918 | 70.53 |
| lz4fast r131 level 3        |   746 MB/s |  3142 MB/s |     4733421 | 76.94 |
| lz4fast r131 level 17       |  1661 MB/s |  6115 MB/s |     5528929 | 89.87 |
| lz4hc r131 level 1          |    94 MB/s |  2439 MB/s |     3823662 | 62.15 |
| lz4hc r131 level 4          |    57 MB/s |  2544 MB/s |     3589528 | 58.35 |
| lz4hc r131 level 9          |    38 MB/s |  2562 MB/s |     3544431 | 57.61 |
| lz4hc r131 level 12         |    33 MB/s |  2586 MB/s |     3543539 | 57.60 |
| lz4hc r131 level 16         |    28 MB/s |  2581 MB/s |     3543401 | 57.60 |
| lzf 3.6 level 0             |   234 MB/s |   565 MB/s |     4297787 | 69.86 |
| lzf 3.6 level 1             |   223 MB/s |   554 MB/s |     4213660 | 68.49 |
| lzg 1.0.8 level 1           |    37 MB/s |   486 MB/s |     4139615 | 67.29 |
| lzg 1.0.8 level 4           |    27 MB/s |   471 MB/s |     3931101 | 63.90 |
| lzg 1.0.8 level 6           |    18 MB/s |   458 MB/s |     3812065 | 61.96 |
| lzg 1.0.8 level 8           |  8.66 MB/s |   444 MB/s |     3674340 | 59.72 |
| lzham 1.0 -d26 level 0      |  7.42 MB/s |   132 MB/s |     2822729 | 45.88 |
| lzham 1.0 -d26 level 1      |  2.85 MB/s |   162 MB/s |     2578108 | 41.91 |
| lzjb 2010                   |   264 MB/s |   472 MB/s |     4600800 | 74.78 |
| lzma 9.38 level 0           |    16 MB/s |    46 MB/s |     2841578 | 46.19 |
| lzma 9.38 level 2           |    13 MB/s |    51 MB/s |     2703265 | 43.94 |
| lzma 9.38 level 4           |  8.58 MB/s |    54 MB/s |     2637526 | 42.87 |
| lzma 9.38 level 5           |  3.23 MB/s |    57 MB/s |     2428736 | 39.48 |
| lzrw 15-Jul-1991 level 1    |   200 MB/s |   472 MB/s |     4269971 | 69.41 |
| lzrw 15-Jul-1991 level 2    |   180 MB/s |   552 MB/s |     4256502 | 69.19 |
| lzrw 15-Jul-1991 level 3    |   257 MB/s |   521 MB/s |     4138957 | 67.28 |
| lzrw 15-Jul-1991 level 4    |   251 MB/s |   421 MB/s |     4036479 | 65.61 |
| lzrw 15-Jul-1991 level 5    |   121 MB/s |   411 MB/s |     3847820 | 62.54 |
| lzsse4 0.1                  |   164 MB/s |  2433 MB/s |     3972687 | 64.57 |
| lzsse8 0.1                  |   162 MB/s |  2710 MB/s |     3922013 | 63.75 |
| lzsse2 0.1                  |  0.03 MB/s |  2478 MB/s |     3492808 | 56.77 |
| quicklz 1.5.0 level 1       |   339 MB/s |   433 MB/s |     4013859 | 65.24 |
| quicklz 1.5.0 level 2       |   167 MB/s |   381 MB/s |     3722542 | 60.51 |
| quicklz 1.5.0 level 3       |    45 MB/s |   579 MB/s |     3548660 | 57.68 |
| yalz77 2015-09-19 level 1   |    48 MB/s |   435 MB/s |     4125570 | 67.06 |
| yalz77 2015-09-19 level 4   |    33 MB/s |   388 MB/s |     3996893 | 64.97 |
| yalz77 2015-09-19 level 8   |    24 MB/s |   377 MB/s |     3953444 | 64.26 |
| yalz77 2015-09-19 level 12  |    17 MB/s |   373 MB/s |     3929331 | 63.87 |
| yappy 2014-03-22 level 1    |   109 MB/s |  2458 MB/s |     4175016 | 67.86 |
| yappy 2014-03-22 level 10   |    93 MB/s |  2602 MB/s |     4105418 | 66.73 |
| yappy 2014-03-22 level 100  |    85 MB/s |  2611 MB/s |     4090885 | 66.49 |
| zlib 1.2.8 level 1          |    48 MB/s |   224 MB/s |     3290532 | 53.49 |
| zlib 1.2.8 level 6          |    18 MB/s |   244 MB/s |     3097294 | 50.34 |
| zlib 1.2.8 level 9          |    10 MB/s |   242 MB/s |     3092926 | 50.27 |
| zstd v0.4.1 level 1         |   268 MB/s |   535 MB/s |     3585831 | 58.29 |
| zstd v0.4.1 level 2         |   202 MB/s |   442 MB/s |     3329621 | 54.12 |
| zstd v0.4.1 level 5         |    73 MB/s |   361 MB/s |     3049253 | 49.56 |
| zstd v0.4.1 level 9         |    21 MB/s |   365 MB/s |     2891406 | 47.00 |
| zstd v0.4.1 level 13        |    11 MB/s |   365 MB/s |     2856608 | 46.43 |
| zstd v0.4.1 level 17        |  8.28 MB/s |   372 MB/s |     2839843 | 46.16 |
| zstd v0.4.1 level 20        |  7.80 MB/s |   368 MB/s |     2838729 | 46.14 |
| shrinker 0.1                |   299 MB/s |   902 MB/s |     3868354 | 62.88 |
| wflz 2015-09-16             |   152 MB/s |  1051 MB/s |     4507764 | 73.27 |
| lzmat 1.01                  |    32 MB/s |   300 MB/s |     3389975 | 55.10 |
|                             |            |            |             |       |


The next cab off the rank is osdb, which is a MySQL database. Here we do better than we did on the binaries, with LZSSE8 doing particularly well.

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      | 13816 MB/s | 13797 MB/s |    10085684 |100.00 |
| blosclz 2015-11-10 level 1  |  1653 MB/s | 13740 MB/s |    10085684 |100.00 |
| blosclz 2015-11-10 level 3  |   831 MB/s | 13556 MB/s |    10085684 |100.00 |
| blosclz 2015-11-10 level 6  |   267 MB/s |  1273 MB/s |     5736204 | 56.87 |
| blosclz 2015-11-10 level 9  |   267 MB/s |  1276 MB/s |     5736204 | 56.87 |
| brieflz 1.1.0               |   131 MB/s |   241 MB/s |     4261872 | 42.26 |
| crush 1.0 level 0           |    26 MB/s |   302 MB/s |     3944111 | 39.11 |
| crush 1.0 level 1           |  4.72 MB/s |   329 MB/s |     3649422 | 36.18 |
| crush 1.0 level 2           |  1.25 MB/s |   344 MB/s |     3545628 | 35.16 |
| fastlz 0.1 level 1          |   205 MB/s |   812 MB/s |     6716718 | 66.60 |
| fastlz 0.1 level 2          |   279 MB/s |   775 MB/s |     5391583 | 53.46 |
| lz4 r131                    |   544 MB/s |  2800 MB/s |     5256666 | 52.12 |
| lz4fast r131 level 3        |   685 MB/s |  2796 MB/s |     5864711 | 58.15 |
| lz4fast r131 level 17       |  1217 MB/s |  4188 MB/s |     8089261 | 80.21 |
| lz4hc r131 level 1          |   145 MB/s |  3158 MB/s |     4280365 | 42.44 |
| lz4hc r131 level 4          |    80 MB/s |  3150 MB/s |     4017760 | 39.84 |
| lz4hc r131 level 9          |    53 MB/s |  3144 MB/s |     3977505 | 39.44 |
| lz4hc r131 level 12         |    53 MB/s |  3145 MB/s |     3977501 | 39.44 |
| lz4hc r131 level 16         |    53 MB/s |  3202 MB/s |     3977501 | 39.44 |
| lzf 3.6 level 0             |   256 MB/s |   879 MB/s |     6602260 | 65.46 |
| lzf 3.6 level 1             |   282 MB/s |   891 MB/s |     6418449 | 63.64 |
| lzg 1.0.8 level 1           |    44 MB/s |   725 MB/s |     8130169 | 80.61 |
| lzg 1.0.8 level 4           |    37 MB/s |   730 MB/s |     5330956 | 52.86 |
| lzg 1.0.8 level 6           |    27 MB/s |   869 MB/s |     4413239 | 43.76 |
| lzg 1.0.8 level 8           |    12 MB/s |   891 MB/s |     4206158 | 41.70 |
| lzham 1.0 -d26 level 0      |  9.17 MB/s |   231 MB/s |     3530290 | 35.00 |
| lzham 1.0 -d26 level 1      |  2.57 MB/s |   283 MB/s |     3152108 | 31.25 |
| lzjb 2010                   |   307 MB/s |   620 MB/s |     9545173 | 94.64 |
| lzma 9.38 level 0           |    20 MB/s |    57 MB/s |     3988823 | 39.55 |
| lzma 9.38 level 2           |    17 MB/s |    73 MB/s |     3307207 | 32.79 |
| lzma 9.38 level 4           |    11 MB/s |    74 MB/s |     3370412 | 33.42 |
| lzma 9.38 level 5           |  2.98 MB/s |    83 MB/s |     2870328 | 28.46 |
| lzrw 15-Jul-1991 level 1    |   182 MB/s |   560 MB/s |     8129348 | 80.60 |
| lzrw 15-Jul-1991 level 2    |   151 MB/s |   700 MB/s |     8070717 | 80.02 |
| lzrw 15-Jul-1991 level 3    |   306 MB/s |   652 MB/s |     7300607 | 72.39 |
| lzrw 15-Jul-1991 level 4    |   325 MB/s |   507 MB/s |     5704498 | 56.56 |
| lzrw 15-Jul-1991 level 5    |   151 MB/s |   552 MB/s |     5333359 | 52.88 |
| lzsse4 0.1                  |   260 MB/s |  4066 MB/s |     4262592 | 42.26 |
| lzsse8 0.1                  |   261 MB/s |  4512 MB/s |     4175474 | 41.40 |
| lzsse2 0.1                  |  2.89 MB/s |  3538 MB/s |     3957567 | 39.24 |
| quicklz 1.5.0 level 1       |   435 MB/s |   541 MB/s |     5496443 | 54.50 |
| quicklz 1.5.0 level 2       |   225 MB/s |   530 MB/s |     4532176 | 44.94 |
| quicklz 1.5.0 level 3       |    53 MB/s |  1168 MB/s |     4259234 | 42.23 |
| yalz77 2015-09-19 level 1   |    77 MB/s |   676 MB/s |     4570193 | 45.31 |
| yalz77 2015-09-19 level 4   |    49 MB/s |   690 MB/s |     4465943 | 44.28 |
| yalz77 2015-09-19 level 8   |    31 MB/s |   718 MB/s |     4384305 | 43.47 |
| yalz77 2015-09-19 level 12  |    21 MB/s |   728 MB/s |     4336873 | 43.00 |
| yappy 2014-03-22 level 1    |    98 MB/s |  2640 MB/s |     7445962 | 73.83 |
| yappy 2014-03-22 level 10   |    93 MB/s |  2685 MB/s |     7372889 | 73.10 |
| yappy 2014-03-22 level 100  |    93 MB/s |  2688 MB/s |     7370336 | 73.08 |
| zlib 1.2.8 level 1          |    69 MB/s |   344 MB/s |     4076391 | 40.42 |
| zlib 1.2.8 level 6          |    31 MB/s |   368 MB/s |     3695170 | 36.64 |
| zlib 1.2.8 level 9          |    23 MB/s |   376 MB/s |     3673177 | 36.42 |
| zstd v0.4.1 level 1         |   333 MB/s |   622 MB/s |     3790691 | 37.58 |
| zstd v0.4.1 level 2         |   271 MB/s |   574 MB/s |     3517954 | 34.88 |
| zstd v0.4.1 level 5         |    91 MB/s |   529 MB/s |     3503732 | 34.74 |
| zstd v0.4.1 level 9         |    33 MB/s |   544 MB/s |     3380393 | 33.52 |
| zstd v0.4.1 level 13        |    14 MB/s |   550 MB/s |     3286962 | 32.59 |
| zstd v0.4.1 level 17        |  5.74 MB/s |   604 MB/s |     3133720 | 31.07 |
| zstd v0.4.1 level 20        |  5.30 MB/s |   595 MB/s |     3128197 | 31.02 |
| shrinker 0.1                |   545 MB/s |  1721 MB/s |     4641152 | 46.02 |
| wflz 2015-09-16             |   321 MB/s |  1569 MB/s |     4768614 | 47.28 |
| lzmat 1.01                  |    64 MB/s |   522 MB/s |     4238262 | 42.02 |
|                             |            |            |             |       |


Reymont is a Polish literature work in pdf, this time LZSSE2 does well again.

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      | 16651 MB/s | 17036 MB/s |     6627202 |100.00 |
| blosclz 2015-11-10 level 1  |  1112 MB/s | 16609 MB/s |     6627202 |100.00 |
| blosclz 2015-11-10 level 3  |   575 MB/s | 16820 MB/s |     6627202 |100.00 |
| blosclz 2015-11-10 level 6  |   240 MB/s |   793 MB/s |     3259380 | 49.18 |
| blosclz 2015-11-10 level 9  |   240 MB/s |   786 MB/s |     3243988 | 48.95 |
| brieflz 1.1.0               |   119 MB/s |   168 MB/s |     2493374 | 37.62 |
| crush 1.0 level 0           |    63 MB/s |   248 MB/s |     2190579 | 33.05 |
| crush 1.0 level 1           |  4.12 MB/s |   321 MB/s |     1758150 | 26.53 |
| crush 1.0 level 2           |  0.25 MB/s |   355 MB/s |     1644697 | 24.82 |
| fastlz 0.1 level 1          |   276 MB/s |   556 MB/s |     3335571 | 50.33 |
| fastlz 0.1 level 2          |   271 MB/s |   546 MB/s |     3186529 | 48.08 |
| lz4 r131                    |   397 MB/s |  2531 MB/s |     3181387 | 48.00 |
| lz4fast r131 level 3        |   417 MB/s |  2514 MB/s |     3322020 | 50.13 |
| lz4fast r131 level 17       |   568 MB/s |  2509 MB/s |     4403444 | 66.44 |
| lz4hc r131 level 1          |   131 MB/s |  2422 MB/s |     2944201 | 44.43 |
| lz4hc r131 level 4          |    62 MB/s |  2738 MB/s |     2361684 | 35.64 |
| lz4hc r131 level 9          |    15 MB/s |  2826 MB/s |     2114022 | 31.90 |
| lz4hc r131 level 12         |    11 MB/s |  2835 MB/s |     2106571 | 31.79 |
| lz4hc r131 level 16         |    12 MB/s |  2836 MB/s |     2106571 | 31.79 |
| lzf 3.6 level 0             |   277 MB/s |   579 MB/s |     3365987 | 50.79 |
| lzf 3.6 level 1             |   273 MB/s |   595 MB/s |     3253836 | 49.10 |
| lzg 1.0.8 level 1           |    55 MB/s |   416 MB/s |     3326973 | 50.20 |
| lzg 1.0.8 level 4           |    33 MB/s |   486 MB/s |     2948111 | 44.49 |
| lzg 1.0.8 level 6           |    19 MB/s |   556 MB/s |     2715211 | 40.97 |
| lzg 1.0.8 level 8           |  6.11 MB/s |   679 MB/s |     2427424 | 36.63 |
| lzham 1.0 -d26 level 0      |  9.12 MB/s |   203 MB/s |     2111331 | 31.86 |
| lzham 1.0 -d26 level 1      |  2.80 MB/s |   335 MB/s |     1544771 | 23.31 |
| lzjb 2010                   |   219 MB/s |   368 MB/s |     3889268 | 58.69 |
| lzma 9.38 level 0           |    20 MB/s |    78 MB/s |     1921954 | 29.00 |
| lzma 9.38 level 2           |    17 MB/s |    90 MB/s |     1807903 | 27.28 |
| lzma 9.38 level 4           |    14 MB/s |    94 MB/s |     1777103 | 26.82 |
| lzma 9.38 level 5           |  2.14 MB/s |   136 MB/s |     1340625 | 20.23 |
| lzrw 15-Jul-1991 level 1    |   268 MB/s |   464 MB/s |     3524660 | 53.18 |
| lzrw 15-Jul-1991 level 2    |   269 MB/s |   522 MB/s |     3512256 | 53.00 |
| lzrw 15-Jul-1991 level 3    |   274 MB/s |   562 MB/s |     3224618 | 48.66 |
| lzrw 15-Jul-1991 level 4    |   310 MB/s |   561 MB/s |     3043004 | 45.92 |
| lzrw 15-Jul-1991 level 5    |   136 MB/s |   629 MB/s |     2502161 | 37.76 |
| lzsse4 0.1                  |   251 MB/s |  3020 MB/s |     2925190 | 44.14 |
| lzsse8 0.1                  |   251 MB/s |  2946 MB/s |     2952727 | 44.55 |
| lzsse2 0.1                  |  2.59 MB/s |  4960 MB/s |     1852623 | 27.95 |
| quicklz 1.5.0 level 1       |   408 MB/s |   613 MB/s |     3003825 | 45.33 |
| quicklz 1.5.0 level 2       |   200 MB/s |   573 MB/s |     2560936 | 38.64 |
| quicklz 1.5.0 level 3       |    46 MB/s |   774 MB/s |     2448354 | 36.94 |
| yalz77 2015-09-19 level 1   |   101 MB/s |   372 MB/s |     3017083 | 45.53 |
| yalz77 2015-09-19 level 4   |    63 MB/s |   405 MB/s |     2679592 | 40.43 |
| yalz77 2015-09-19 level 8   |    43 MB/s |   420 MB/s |     2569444 | 38.77 |
| yalz77 2015-09-19 level 12  |    35 MB/s |   428 MB/s |     2520600 | 38.03 |
| yappy 2014-03-22 level 1    |   146 MB/s |  1780 MB/s |     3011002 | 45.43 |
| yappy 2014-03-22 level 10   |    89 MB/s |  1896 MB/s |     2781083 | 41.96 |
| yappy 2014-03-22 level 100  |    72 MB/s |  1924 MB/s |     2738695 | 41.33 |
| zlib 1.2.8 level 1          |    76 MB/s |   348 MB/s |     2376430 | 35.86 |
| zlib 1.2.8 level 6          |    17 MB/s |   381 MB/s |     1860871 | 28.08 |
| zlib 1.2.8 level 9          |  5.81 MB/s |   380 MB/s |     1823196 | 27.51 |
| zstd v0.4.1 level 1         |   260 MB/s |   454 MB/s |     2169769 | 32.74 |
| zstd v0.4.1 level 2         |   221 MB/s |   387 MB/s |     2100979 | 31.70 |
| zstd v0.4.1 level 5         |   107 MB/s |   388 MB/s |     1974996 | 29.80 |
| zstd v0.4.1 level 9         |    33 MB/s |   477 MB/s |     1724763 | 26.03 |
| zstd v0.4.1 level 13        |    10 MB/s |   509 MB/s |     1620588 | 24.45 |
| zstd v0.4.1 level 17        |  4.55 MB/s |   540 MB/s |     1480040 | 22.33 |
| zstd v0.4.1 level 20        |  3.43 MB/s |   549 MB/s |     1451022 | 21.89 |
| shrinker 0.1                |   347 MB/s |   742 MB/s |     3157154 | 47.64 |
| wflz 2015-09-16             |   252 MB/s |   682 MB/s |     4035470 | 60.89 |
| lzmat 1.01                  |    17 MB/s |   469 MB/s |     2124786 | 32.06 | 
|                             |            |            |             |       |


The samba source code, similar performance characteristics to other text:

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      | 11729 MB/s | 11492 MB/s |    21606400 |100.00 |
| blosclz 2015-11-10 level 1  |  1144 MB/s | 11359 MB/s |    21503746 | 99.52 |
| blosclz 2015-11-10 level 3  |   603 MB/s |  9883 MB/s |    21074313 | 97.54 |
| blosclz 2015-11-10 level 6  |   363 MB/s |  1321 MB/s |     8066597 | 37.33 |
| blosclz 2015-11-10 level 9  |   353 MB/s |  1315 MB/s |     7953097 | 36.81 |
| brieflz 1.1.0               |   161 MB/s |   272 MB/s |     6175378 | 28.58 |
| crush 1.0 level 0           |    64 MB/s |   366 MB/s |     5531441 | 25.60 |
| crush 1.0 level 1           |  9.78 MB/s |   411 MB/s |     5046297 | 23.36 |
| crush 1.0 level 2           |  1.44 MB/s |   429 MB/s |     4912137 | 22.73 |
| fastlz 0.1 level 1          |   369 MB/s |   803 MB/s |     8327938 | 38.54 |
| fastlz 0.1 level 2          |   390 MB/s |   828 MB/s |     7651048 | 35.41 |
| lz4 r131                    |   679 MB/s |  3275 MB/s |     7716839 | 35.72 |
| lz4fast r131 level 3        |   724 MB/s |  3200 MB/s |     8250343 | 38.18 |
| lz4fast r131 level 17       |  1036 MB/s |  3584 MB/s |    11109978 | 51.42 |
| lz4hc r131 level 1          |   177 MB/s |  3213 MB/s |     6858151 | 31.74 |
| lz4hc r131 level 4          |   102 MB/s |  3390 MB/s |     6286421 | 29.10 |
| lz4hc r131 level 9          |    46 MB/s |  3448 MB/s |     6142054 | 28.43 |
| lz4hc r131 level 12         |    36 MB/s |  3451 MB/s |     6139201 | 28.41 |
| lz4hc r131 level 16         |    30 MB/s |  3486 MB/s |     6138755 | 28.41 |
| lzf 3.6 level 0             |   375 MB/s |   872 MB/s |     8616496 | 39.88 |
| lzf 3.6 level 1             |   390 MB/s |   914 MB/s |     8208471 | 37.99 |
| lzg 1.0.8 level 1           |    69 MB/s |   664 MB/s |     9073050 | 41.99 |
| lzg 1.0.8 level 4           |    49 MB/s |   711 MB/s |     7759679 | 35.91 |
| lzg 1.0.8 level 6           |    32 MB/s |   776 MB/s |     6981326 | 32.31 |
| lzg 1.0.8 level 8           |    12 MB/s |   846 MB/s |     6485251 | 30.02 |
| lzham 1.0 -d26 level 0      |    11 MB/s |   332 MB/s |     5194325 | 24.04 |
| lzham 1.0 -d26 level 1      |  4.03 MB/s |   446 MB/s |     4282964 | 19.82 |
| lzjb 2010                   |   350 MB/s |   591 MB/s |    10566148 | 48.90 |
| lzma 9.38 level 0           |    28 MB/s |    89 MB/s |     5338935 | 24.71 |
| lzma 9.38 level 2           |    26 MB/s |   108 MB/s |     4635279 | 21.45 |
| lzma 9.38 level 4           |    19 MB/s |   117 MB/s |     4361856 | 20.19 |
| lzma 9.38 level 5           |  4.31 MB/s |   129 MB/s |     3859221 | 17.86 |
| lzrw 15-Jul-1991 level 1    |   337 MB/s |   645 MB/s |     9682113 | 44.81 |
| lzrw 15-Jul-1991 level 2    |   304 MB/s |   747 MB/s |     9550240 | 44.20 |
| lzrw 15-Jul-1991 level 3    |   374 MB/s |   743 MB/s |     8708966 | 40.31 |
| lzrw 15-Jul-1991 level 4    |   403 MB/s |   658 MB/s |     8241669 | 38.14 |
| lzrw 15-Jul-1991 level 5    |   168 MB/s |   710 MB/s |     7486315 | 34.65 |
| lzsse4 0.1                  |   297 MB/s |  4045 MB/s |     7601765 | 35.18 |
| lzsse8 0.1                  |   297 MB/s |  4185 MB/s |     7582300 | 35.09 |
| lzsse2 0.1                  |  0.21 MB/s |  4400 MB/s |     6088709 | 28.18 |
| quicklz 1.5.0 level 1       |   555 MB/s |   792 MB/s |     7309452 | 33.83 |
| quicklz 1.5.0 level 2       |   270 MB/s |   735 MB/s |     6548190 | 30.31 |
| quicklz 1.5.0 level 3       |    64 MB/s |  1176 MB/s |     6369074 | 29.48 |
| yalz77 2015-09-19 level 1   |   116 MB/s |   560 MB/s |     7098899 | 32.86 |
| yalz77 2015-09-19 level 4   |    71 MB/s |   598 MB/s |     6590006 | 30.50 |
| yalz77 2015-09-19 level 8   |    48 MB/s |   610 MB/s |     6446898 | 29.84 |
| yalz77 2015-09-19 level 12  |    37 MB/s |   618 MB/s |     6361237 | 29.44 |
| yappy 2014-03-22 level 1    |   162 MB/s |  2318 MB/s |     8788099 | 40.67 |
| yappy 2014-03-22 level 10   |   120 MB/s |  2635 MB/s |     8169046 | 37.81 |
| yappy 2014-03-22 level 100  |    90 MB/s |  2682 MB/s |     8021698 | 37.13 |
| zlib 1.2.8 level 1          |    96 MB/s |   438 MB/s |     6329455 | 29.29 |
| zlib 1.2.8 level 6          |    37 MB/s |   472 MB/s |     5451405 | 25.23 |
| zlib 1.2.8 level 9          |    19 MB/s |   475 MB/s |     5402710 | 25.01 |
| zstd v0.4.1 level 1         |   422 MB/s |   720 MB/s |     5573442 | 25.80 |
| zstd v0.4.1 level 2         |   337 MB/s |   656 MB/s |     5418700 | 25.08 |
| zstd v0.4.1 level 5         |   149 MB/s |   652 MB/s |     4953932 | 22.93 |
| zstd v0.4.1 level 9         |    56 MB/s |   755 MB/s |     4503155 | 20.84 |
| zstd v0.4.1 level 13        |    24 MB/s |   778 MB/s |     4360991 | 20.18 |
| zstd v0.4.1 level 17        |  8.40 MB/s |   809 MB/s |     4204351 | 19.46 |
| zstd v0.4.1 level 20        |  4.21 MB/s |   794 MB/s |     4171774 | 19.31 |
| shrinker 0.1                |   490 MB/s |  1294 MB/s |     7317434 | 33.87 |
| wflz 2015-09-16             |   344 MB/s |  1167 MB/s |     8856272 | 40.99 |
| lzmat 1.01                  |    47 MB/s |   646 MB/s |     5783582 | 26.77 |
|                             |            |            |             |       |



The SAO star catalogue is less compressible and suits LZSSE8 very well, but LZSSE2 fairs quite poorly, even with the optimal parse advantage.

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      | 16079 MB/s | 16115 MB/s |     7251944 |100.00 |
| blosclz 2015-11-10 level 1  |  1660 MB/s | 16044 MB/s |     7251944 |100.00 |
| blosclz 2015-11-10 level 3  |   861 MB/s | 15629 MB/s |     7251944 |100.00 |
| blosclz 2015-11-10 level 6  |   289 MB/s | 15629 MB/s |     7251944 |100.00 |
| blosclz 2015-11-10 level 9  |   192 MB/s |   935 MB/s |     6541044 | 90.20 |
| brieflz 1.1.0               |    92 MB/s |   194 MB/s |     6139160 | 84.66 |
| crush 1.0 level 0           |    16 MB/s |   183 MB/s |     5758539 | 79.41 |
| crush 1.0 level 1           |  3.86 MB/s |   192 MB/s |     5573768 | 76.86 |
| crush 1.0 level 2           |  0.68 MB/s |   201 MB/s |     5472031 | 75.46 |
| fastlz 0.1 level 1          |   159 MB/s |   715 MB/s |     6525733 | 89.99 |
| fastlz 0.1 level 2          |   220 MB/s |   699 MB/s |     6526165 | 89.99 |
| lz4 r131                    |   504 MB/s |  3393 MB/s |     6790273 | 93.63 |
| lz4fast r131 level 3        |   944 MB/s |  4339 MB/s |     6930690 | 95.57 |
| lz4fast r131 level 17       |  2160 MB/s |  8364 MB/s |     7183536 | 99.06 |
| lz4hc r131 level 1          |    81 MB/s |  2131 MB/s |     6240508 | 86.05 |
| lz4hc r131 level 4          |    45 MB/s |  2291 MB/s |     5859003 | 80.79 |
| lz4hc r131 level 9          |    29 MB/s |  2404 MB/s |     5735467 | 79.09 |
| lz4hc r131 level 12         |    29 MB/s |  2403 MB/s |     5735136 | 79.08 |
| lz4hc r131 level 16         |    29 MB/s |  2402 MB/s |     5735136 | 79.08 |
| lzf 3.6 level 0             |   229 MB/s |   708 MB/s |     6325572 | 87.23 |
| lzf 3.6 level 1             |   219 MB/s |   697 MB/s |     6280222 | 86.60 |
| lzg 1.0.8 level 1           |    26 MB/s |   517 MB/s |     6357698 | 87.67 |
| lzg 1.0.8 level 4           |    19 MB/s |   493 MB/s |     6217135 | 85.73 |
| lzg 1.0.8 level 6           |    13 MB/s |   516 MB/s |     6086665 | 83.93 |
| lzg 1.0.8 level 8           |  6.36 MB/s |   555 MB/s |     5930187 | 81.77 |
| lzham 1.0 -d26 level 0      |  6.91 MB/s |   132 MB/s |     4990889 | 68.82 |
| lzham 1.0 -d26 level 1      |  2.16 MB/s |   153 MB/s |     4734194 | 65.28 |
| lzjb 2010                   |   282 MB/s |   556 MB/s |     7005137 | 96.60 |
| lzma 9.38 level 0           |    12 MB/s |    32 MB/s |     4923529 | 67.89 |
| lzma 9.38 level 2           |  9.73 MB/s |    33 MB/s |     4892916 | 67.47 |
| lzma 9.38 level 4           |  5.77 MB/s |    33 MB/s |     4894122 | 67.49 |
| lzma 9.38 level 5           |  3.04 MB/s |    36 MB/s |     4418197 | 60.92 |
| lzrw 15-Jul-1991 level 1    |   164 MB/s |   507 MB/s |     6582297 | 90.77 |
| lzrw 15-Jul-1991 level 2    |   138 MB/s |   622 MB/s |     6582297 | 90.77 |
| lzrw 15-Jul-1991 level 3    |   268 MB/s |   582 MB/s |     6540282 | 90.19 |
| lzrw 15-Jul-1991 level 4    |   269 MB/s |   403 MB/s |     6477885 | 89.33 |
| lzrw 15-Jul-1991 level 5    |   110 MB/s |   426 MB/s |     6178268 | 85.19 |
| lzsse4 0.1                  |   168 MB/s |  2432 MB/s |     6305407 | 86.95 |
| lzsse8 0.1                  |   166 MB/s |  3274 MB/s |     6045723 | 83.37 |
| lzsse2 0.1                  |  2.59 MB/s |  1863 MB/s |     6066923 | 83.66 |
| quicklz 1.5.0 level 1       |   359 MB/s |   353 MB/s |     6498301 | 89.61 |
| quicklz 1.5.0 level 2       |   142 MB/s |   271 MB/s |     6121309 | 84.41 |
| quicklz 1.5.0 level 3       |    37 MB/s |   751 MB/s |     6073127 | 83.74 |
| yalz77 2015-09-19 level 1   |    38 MB/s |   611 MB/s |     6299030 | 86.86 |
| yalz77 2015-09-19 level 4   |    24 MB/s |   568 MB/s |     6084431 | 83.90 |
| yalz77 2015-09-19 level 8   |    17 MB/s |   548 MB/s |     6042253 | 83.32 |
| yalz77 2015-09-19 level 12  |    12 MB/s |   543 MB/s |     6021554 | 83.03 |
| yappy 2014-03-22 level 1    |    94 MB/s |  2367 MB/s |     6153132 | 84.85 |
| yappy 2014-03-22 level 10   |    86 MB/s |  2383 MB/s |     6096705 | 84.07 |
| yappy 2014-03-22 level 100  |    85 MB/s |  2389 MB/s |     6091380 | 84.00 |
| zlib 1.2.8 level 1          |    37 MB/s |   246 MB/s |     5567774 | 76.78 |
| zlib 1.2.8 level 6          |    16 MB/s |   271 MB/s |     5331799 | 73.52 |
| zlib 1.2.8 level 9          |    13 MB/s |   274 MB/s |     5325920 | 73.44 |
| zstd v0.4.1 level 1         |   235 MB/s |   482 MB/s |     6255977 | 86.27 |
| zstd v0.4.1 level 2         |   169 MB/s |   418 MB/s |     5818661 | 80.24 |
| zstd v0.4.1 level 5         |    53 MB/s |   355 MB/s |     5403988 | 74.52 |
| zstd v0.4.1 level 9         |    14 MB/s |   365 MB/s |     5228930 | 72.10 |
| zstd v0.4.1 level 13        |  7.85 MB/s |   357 MB/s |     5200485 | 71.71 |
| zstd v0.4.1 level 17        |  6.44 MB/s |   351 MB/s |     5202330 | 71.74 |
| zstd v0.4.1 level 20        |  6.40 MB/s |   351 MB/s |     5201103 | 71.72 |
| shrinker 0.1                |   333 MB/s |  1409 MB/s |     6394653 | 88.18 |
| wflz 2015-09-16             |   131 MB/s |  1405 MB/s |     6677960 | 92.09 |
| lzmat 1.01                  |    32 MB/s |   341 MB/s |     5862040 | 80.83 |
|                             |            |            |             |       |



The 1913 Webster dictionary works well with LZSSE2. There is a definite trend towards LZSSE2 for the more compressible text.

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      | 11704 MB/s | 11681 MB/s |    41458703 |100.00 |
| blosclz 2015-11-10 level 1  |  1087 MB/s | 11355 MB/s |    41458703 |100.00 |
| blosclz 2015-11-10 level 3  |   567 MB/s | 11358 MB/s |    41458703 |100.00 |
| blosclz 2015-11-10 level 6  |   242 MB/s |   854 MB/s |    20100893 | 48.48 |
| blosclz 2015-11-10 level 9  |   242 MB/s |   855 MB/s |    20100893 | 48.48 |
| brieflz 1.1.0               |   112 MB/s |   166 MB/s |    15088514 | 36.39 |
| crush 1.0 level 0           |    55 MB/s |   270 MB/s |    12752811 | 30.76 |
| crush 1.0 level 1           |  4.04 MB/s |   322 MB/s |    10826131 | 26.11 |
| crush 1.0 level 2           |  0.50 MB/s |   335 MB/s |    10430224 | 25.16 |
| fastlz 0.1 level 1          |   287 MB/s |   563 MB/s |    20113352 | 48.51 |
| fastlz 0.1 level 2          |   267 MB/s |   554 MB/s |    19693994 | 47.50 |
| lz4 r131                    |   454 MB/s |  2566 MB/s |    20139988 | 48.58 |
| lz4fast r131 level 3        |   507 MB/s |  2538 MB/s |    21871634 | 52.76 |
| lz4fast r131 level 17       |   819 MB/s |  2961 MB/s |    28765737 | 69.38 |
| lz4hc r131 level 1          |   136 MB/s |  2370 MB/s |    16861690 | 40.67 |
| lz4hc r131 level 4          |    70 MB/s |  2499 MB/s |    14557596 | 35.11 |
| lz4hc r131 level 9          |    30 MB/s |  2536 MB/s |    14006633 | 33.78 |
| lz4hc r131 level 12         |    27 MB/s |  2536 MB/s |    13995894 | 33.76 |
| lz4hc r131 level 16         |    27 MB/s |  2537 MB/s |    13995894 | 33.76 |
| lzf 3.6 level 0             |   275 MB/s |   602 MB/s |    20788247 | 50.14 |
| lzf 3.6 level 1             |   276 MB/s |   635 MB/s |    19701097 | 47.52 |
| lzg 1.0.8 level 1           |    70 MB/s |   500 MB/s |    21258007 | 51.28 |
| lzg 1.0.8 level 4           |    41 MB/s |   503 MB/s |    18766104 | 45.26 |
| lzg 1.0.8 level 6           |    21 MB/s |   548 MB/s |    17083163 | 41.21 |
| lzg 1.0.8 level 8           |  7.18 MB/s |   642 MB/s |    15354723 | 37.04 |
| lzham 1.0 -d26 level 0      |  9.65 MB/s |   216 MB/s |    13254015 | 31.97 |
| lzham 1.0 -d26 level 1      |  2.79 MB/s |   332 MB/s |     9988909 | 24.09 |
| lzjb 2010                   |   257 MB/s |   442 MB/s |    25415740 | 61.30 |
| lzma 9.38 level 0           |    22 MB/s |    75 MB/s |    12704878 | 30.64 |
| lzma 9.38 level 2           |    19 MB/s |    94 MB/s |    11298267 | 27.25 |
| lzma 9.38 level 4           |    12 MB/s |   101 MB/s |    10831475 | 26.13 |
| lzma 9.38 level 5           |  2.13 MB/s |   122 MB/s |     8797488 | 21.22 |
| lzrw 15-Jul-1991 level 1    |   253 MB/s |   465 MB/s |    22060671 | 53.21 |
| lzrw 15-Jul-1991 level 2    |   250 MB/s |   524 MB/s |    21825578 | 52.64 |
| lzrw 15-Jul-1991 level 3    |   263 MB/s |   546 MB/s |    19983920 | 48.20 |
| lzrw 15-Jul-1991 level 4    |   294 MB/s |   537 MB/s |    18773214 | 45.28 |
| lzrw 15-Jul-1991 level 5    |   136 MB/s |   565 MB/s |    16407820 | 39.58 |
| lzsse4 0.1                  |   242 MB/s |  3541 MB/s |    16613479 | 40.07 |
| lzsse8 0.1                  |   241 MB/s |  3378 MB/s |    16775799 | 40.46 |
| lzsse2 0.1                  |  2.62 MB/s |  4496 MB/s |    12372209 | 29.84 |
| quicklz 1.5.0 level 1       |   390 MB/s |   611 MB/s |    18315816 | 44.18 |
| quicklz 1.5.0 level 2       |   195 MB/s |   601 MB/s |    15734418 | 37.95 |
| quicklz 1.5.0 level 3       |    48 MB/s |   820 MB/s |    15101357 | 36.43 |
| yalz77 2015-09-19 level 1   |    93 MB/s |   364 MB/s |    18435248 | 44.47 |
| yalz77 2015-09-19 level 4   |    58 MB/s |   393 MB/s |    16188981 | 39.05 |
| yalz77 2015-09-19 level 8   |    36 MB/s |   389 MB/s |    15382920 | 37.10 |
| yalz77 2015-09-19 level 12  |    27 MB/s |   370 MB/s |    15006287 | 36.20 |
| yappy 2014-03-22 level 1    |   135 MB/s |  1872 MB/s |    19174395 | 46.25 |
| yappy 2014-03-22 level 10   |   101 MB/s |  1935 MB/s |    18513087 | 44.65 |
| yappy 2014-03-22 level 100  |    88 MB/s |  1947 MB/s |    18410318 | 44.41 |
| zlib 1.2.8 level 1          |    78 MB/s |   346 MB/s |    14991242 | 36.16 |
| zlib 1.2.8 level 6          |    26 MB/s |   366 MB/s |    12213947 | 29.46 |
| zlib 1.2.8 level 9          |    16 MB/s |   369 MB/s |    12073475 | 29.12 |
| zstd v0.4.1 level 1         |   284 MB/s |   519 MB/s |    13759779 | 33.19 |
| zstd v0.4.1 level 2         |   211 MB/s |   432 MB/s |    12990370 | 31.33 |
| zstd v0.4.1 level 5         |   111 MB/s |   401 MB/s |    11917853 | 28.75 |
| zstd v0.4.1 level 9         |    31 MB/s |   463 MB/s |    10591896 | 25.55 |
| zstd v0.4.1 level 13        |    10 MB/s |   484 MB/s |    10118655 | 24.41 |
| zstd v0.4.1 level 17        |  4.25 MB/s |   459 MB/s |     9454323 | 22.80 |
| zstd v0.4.1 level 20        |  2.70 MB/s |   344 MB/s |     9120434 | 22.00 |
| shrinker 0.1                |   333 MB/s |   861 MB/s |    18510142 | 44.65 |
| wflz 2015-09-16             |   228 MB/s |   762 MB/s |    22825692 | 55.06 |
| lzmat 1.01                  |    32 MB/s |   446 MB/s |    14027055 | 33.83 |
|                             |            |            |             |       |


Collected xml files, the optimal parse and the focus on matches instead of literals means LZSSE2 again performs very well, copying a lot of bytes per step and the control word probabilities suiting it well:

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      | 19297 MB/s | 19297 MB/s |     5345280 |100.00 |
| blosclz 2015-11-10 level 1  |  1110 MB/s | 18755 MB/s |     5345280 |100.00 |
| blosclz 2015-11-10 level 3  |   671 MB/s |  5292 MB/s |     3573039 | 66.84 |
| blosclz 2015-11-10 level 6  |   470 MB/s |  1552 MB/s |     1334659 | 24.97 |
| blosclz 2015-11-10 level 9  |   472 MB/s |  1559 MB/s |     1334659 | 24.97 |
| brieflz 1.1.0               |   196 MB/s |   303 MB/s |     1087960 | 20.35 |
| crush 1.0 level 0           |   116 MB/s |   518 MB/s |      772420 | 14.45 |
| crush 1.0 level 1           |    24 MB/s |   676 MB/s |      597351 | 11.18 |
| crush 1.0 level 2           |  3.58 MB/s |   715 MB/s |      563740 | 10.55 |
| fastlz 0.1 level 1          |   536 MB/s |   951 MB/s |     1385060 | 25.91 |
| fastlz 0.1 level 2          |   525 MB/s |   966 MB/s |     1273649 | 23.83 |
| lz4 r131                    |   867 MB/s |  3287 MB/s |     1227495 | 22.96 |
| lz4fast r131 level 3        |   920 MB/s |  3297 MB/s |     1294713 | 24.22 |
| lz4fast r131 level 17       |  1141 MB/s |  3400 MB/s |     1671390 | 31.27 |
| lz4hc r131 level 1          |   233 MB/s |  3351 MB/s |     1098791 | 20.56 |
| lz4hc r131 level 4          |   137 MB/s |  4096 MB/s |      829982 | 15.53 |
| lz4hc r131 level 9          |    56 MB/s |  4454 MB/s |      770453 | 14.41 |
| lz4hc r131 level 12         |    49 MB/s |  4465 MB/s |      769600 | 14.40 |
| lz4hc r131 level 16         |    49 MB/s |  4461 MB/s |      769600 | 14.40 |
| lzf 3.6 level 0             |   521 MB/s |  1083 MB/s |     1438227 | 26.91 |
| lzf 3.6 level 1             |   526 MB/s |  1137 MB/s |     1379833 | 25.81 |
| lzg 1.0.8 level 1           |    97 MB/s |   813 MB/s |     1490854 | 27.89 |
| lzg 1.0.8 level 4           |    65 MB/s |   919 MB/s |     1132076 | 21.18 |
| lzg 1.0.8 level 6           |    42 MB/s |  1015 MB/s |     1017213 | 19.03 |
| lzg 1.0.8 level 8           |    17 MB/s |  1199 MB/s |      849051 | 15.88 |
| lzham 1.0 -d26 level 0      |    13 MB/s |   503 MB/s |      722379 | 13.51 |
| lzham 1.0 -d26 level 1      |  3.93 MB/s |   766 MB/s |      533740 |  9.99 |
| lzjb 2010                   |   361 MB/s |   582 MB/s |     2130319 | 39.85 |
| lzma 9.38 level 0           |    44 MB/s |   170 MB/s |      691236 | 12.93 |
| lzma 9.38 level 2           |    44 MB/s |   222 MB/s |      570904 | 10.68 |
| lzma 9.38 level 4           |    38 MB/s |   230 MB/s |      559244 | 10.46 |
| lzma 9.38 level 5           |  6.88 MB/s |   259 MB/s |      484357 |  9.06 |
| lzrw 15-Jul-1991 level 1    |   429 MB/s |   711 MB/s |     1844800 | 34.51 |
| lzrw 15-Jul-1991 level 2    |   425 MB/s |   803 MB/s |     1794772 | 33.58 |
| lzrw 15-Jul-1991 level 3    |   465 MB/s |   862 MB/s |     1574249 | 29.45 |
| lzrw 15-Jul-1991 level 4    |   504 MB/s |   822 MB/s |     1501348 | 28.09 |
| lzrw 15-Jul-1991 level 5    |   204 MB/s |   934 MB/s |     1232257 | 23.05 |
| lzsse4 0.1                  |   383 MB/s |  4403 MB/s |     1388503 | 25.98 |
| lzsse8 0.1                  |   381 MB/s |  4179 MB/s |     1406119 | 26.31 |
| lzsse2 0.1                  |  0.41 MB/s |  6723 MB/s |      767398 | 14.36 |
| quicklz 1.5.0 level 1       |   739 MB/s |  1189 MB/s |     1124708 | 21.04 |
| quicklz 1.5.0 level 2       |   422 MB/s |  1332 MB/s |      897312 | 16.79 |
| quicklz 1.5.0 level 3       |    89 MB/s |  1575 MB/s |      918580 | 17.18 |
| yalz77 2015-09-19 level 1   |   213 MB/s |   765 MB/s |     1067378 | 19.97 |
| yalz77 2015-09-19 level 4   |   149 MB/s |   913 MB/s |      892051 | 16.69 |
| yalz77 2015-09-19 level 8   |   114 MB/s |   949 MB/s |      850394 | 15.91 |
| yalz77 2015-09-19 level 12  |    96 MB/s |   962 MB/s |      834608 | 15.61 |
| yappy 2014-03-22 level 1    |   210 MB/s |  2825 MB/s |     1361662 | 25.47 |
| yappy 2014-03-22 level 10   |   135 MB/s |  3059 MB/s |     1234356 | 23.09 |
| yappy 2014-03-22 level 100  |    98 MB/s |  3227 MB/s |     1205281 | 22.55 |
| zlib 1.2.8 level 1          |   147 MB/s |   564 MB/s |      965248 | 18.06 |
| zlib 1.2.8 level 6          |    53 MB/s |   683 MB/s |      687997 | 12.87 |
| zlib 1.2.8 level 9          |    28 MB/s |   702 MB/s |      658905 | 12.33 |
| zstd v0.4.1 level 1         |   603 MB/s |   955 MB/s |      709132 | 13.27 |
| zstd v0.4.1 level 2         |   523 MB/s |   874 MB/s |      695991 | 13.02 |
| zstd v0.4.1 level 5         |   213 MB/s |   905 MB/s |      629082 | 11.77 |
| zstd v0.4.1 level 9         |    90 MB/s |  1122 MB/s |      537682 | 10.06 |
| zstd v0.4.1 level 13        |    46 MB/s |  1209 MB/s |      512754 |  9.59 |
| zstd v0.4.1 level 17        |  8.47 MB/s |  1288 MB/s |      491938 |  9.20 |
| zstd v0.4.1 level 20        |  7.37 MB/s |  1298 MB/s |      488349 |  9.14 |
| shrinker 0.1                |   644 MB/s |  1448 MB/s |     1242035 | 23.24 |
| wflz 2015-09-16             |   536 MB/s |  1408 MB/s |     1447468 | 27.08 |
| lzmat 1.01                  |    81 MB/s |   835 MB/s |      769801 | 14.40 |
|                             |            |            |             |       |



The last file of Silesia, an x-ray medical image, is less compressible and suits LZSSE8.

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      | 15105 MB/s | 15186 MB/s |     8474240 |100.00 |
| blosclz 2015-11-10 level 1  |  1864 MB/s | 15078 MB/s |     8474240 |100.00 |
| blosclz 2015-11-10 level 3  |   980 MB/s | 14815 MB/s |     8474240 |100.00 |
| blosclz 2015-11-10 level 6  |   333 MB/s | 14815 MB/s |     8474240 |100.00 |
| blosclz 2015-11-10 level 9  |   206 MB/s |   672 MB/s |     8216913 | 96.96 |
| brieflz 1.1.0               |    76 MB/s |   133 MB/s |     7333725 | 86.54 |
| crush 1.0 level 0           |    17 MB/s |   197 MB/s |     6390926 | 75.42 |
| crush 1.0 level 1           |  8.38 MB/s |   201 MB/s |     6256115 | 73.83 |
| crush 1.0 level 2           |  4.37 MB/s |   188 MB/s |     5958599 | 70.31 |
| fastlz 0.1 level 1          |   206 MB/s |   698 MB/s |     8202940 | 96.80 |
| fastlz 0.1 level 2          |   228 MB/s |   702 MB/s |     8200360 | 96.77 |
| lz4 r131                    |  1321 MB/s |  6763 MB/s |     8390195 | 99.01 |
| lz4fast r131 level 3        |  1912 MB/s |  8093 MB/s |     8460144 | 99.83 |
| lz4fast r131 level 17       |  3609 MB/s |  8976 MB/s |     8503227 |100.34 |
| lz4hc r131 level 1          |    82 MB/s |  2488 MB/s |     7569783 | 89.33 |
| lz4hc r131 level 4          |    45 MB/s |  2255 MB/s |     7177516 | 84.70 |
| lz4hc r131 level 9          |    44 MB/s |  2255 MB/s |     7175001 | 84.67 |
| lz4hc r131 level 12         |    44 MB/s |  2254 MB/s |     7175001 | 84.67 |
| lz4hc r131 level 16         |    44 MB/s |  2256 MB/s |     7175001 | 84.67 |
| lzf 3.6 level 0             |   265 MB/s |   845 MB/s |     8283610 | 97.75 |
| lzf 3.6 level 1             |   260 MB/s |   743 MB/s |     8111360 | 95.72 |
| lzg 1.0.8 level 1           |    52 MB/s |   564 MB/s |     7811567 | 92.18 |
| lzg 1.0.8 level 4           |    32 MB/s |   527 MB/s |     7710338 | 90.99 |
| lzg 1.0.8 level 6           |    15 MB/s |   490 MB/s |     7531355 | 88.87 |
| lzg 1.0.8 level 8           |  5.44 MB/s |   475 MB/s |     7239622 | 85.43 |
| lzham 1.0 -d26 level 0      |  7.46 MB/s |   120 MB/s |     4622273 | 54.54 |
| lzham 1.0 -d26 level 1      |  2.54 MB/s |   128 MB/s |     4549082 | 53.68 |
| lzjb 2010                   |   280 MB/s |   602 MB/s |     8353724 | 98.58 |
| lzma 9.38 level 0           |    14 MB/s |    36 MB/s |     5198894 | 61.35 |
| lzma 9.38 level 2           |    11 MB/s |    41 MB/s |     5069114 | 59.82 |
| lzma 9.38 level 4           |  5.25 MB/s |    47 MB/s |     4923749 | 58.10 |
| lzma 9.38 level 5           |  3.18 MB/s |    40 MB/s |     4487613 | 52.96 |
| lzrw 15-Jul-1991 level 1    |   188 MB/s |   559 MB/s |     7821233 | 92.29 |
| lzrw 15-Jul-1991 level 2    |   161 MB/s |   708 MB/s |     7821232 | 92.29 |
| lzrw 15-Jul-1991 level 3    |   273 MB/s |   706 MB/s |     7763663 | 91.61 |
| lzrw 15-Jul-1991 level 4    |   274 MB/s |   468 MB/s |     7533798 | 88.90 |
| lzrw 15-Jul-1991 level 5    |   116 MB/s |   476 MB/s |     7375665 | 87.04 |
| lzsse4 0.1                  |   177 MB/s |  2383 MB/s |     7525821 | 88.81 |
| lzsse8 0.1                  |   174 MB/s |  3074 MB/s |     7248659 | 85.54 |
| lzsse2 0.1                  |  2.91 MB/s |  1626 MB/s |     7035769 | 83.03 |
| quicklz 1.5.0 level 1       |   386 MB/s |   330 MB/s |     7440632 | 87.80 |
| quicklz 1.5.0 level 2       |   152 MB/s |   285 MB/s |     6952012 | 82.04 |
| quicklz 1.5.0 level 3       |    39 MB/s |   590 MB/s |     6713171 | 79.22 |
| yalz77 2015-09-19 level 1   |    37 MB/s |   592 MB/s |     7933653 | 93.62 |
| yalz77 2015-09-19 level 4   |    27 MB/s |   394 MB/s |     7893419 | 93.15 |
| yalz77 2015-09-19 level 8   |    20 MB/s |   353 MB/s |     7914786 | 93.40 |
| yalz77 2015-09-19 level 12  |    15 MB/s |   347 MB/s |     7904338 | 93.27 |
| yappy 2014-03-22 level 1    |    89 MB/s |  4859 MB/s |     8328020 | 98.27 |
| yappy 2014-03-22 level 10   |    89 MB/s |  4801 MB/s |     8327929 | 98.27 |
| yappy 2014-03-22 level 100  |    89 MB/s |  4798 MB/s |     8327919 | 98.27 |
| zlib 1.2.8 level 1          |    44 MB/s |   229 MB/s |     6033932 | 71.20 |
| zlib 1.2.8 level 6          |    23 MB/s |   219 MB/s |     6045117 | 71.34 |
| zlib 1.2.8 level 9          |    23 MB/s |   219 MB/s |     6045038 | 71.33 |
| zstd v0.4.1 level 1         |   593 MB/s |   680 MB/s |     6772806 | 79.92 |
| zstd v0.4.1 level 2         |   269 MB/s |   596 MB/s |     6688571 | 78.93 |
| zstd v0.4.1 level 5         |    57 MB/s |   275 MB/s |     5748916 | 67.84 |
| zstd v0.4.1 level 9         |    14 MB/s |   229 MB/s |     5368140 | 63.35 |
| zstd v0.4.1 level 13        |  8.18 MB/s |   223 MB/s |     5317081 | 62.74 |
| zstd v0.4.1 level 17        |  5.96 MB/s |   215 MB/s |     5306998 | 62.63 |
| zstd v0.4.1 level 20        |  5.89 MB/s |   217 MB/s |     5306994 | 62.63 |
| shrinker 0.1                |   310 MB/s |  1332 MB/s |     7709609 | 90.98 |
| wflz 2015-09-16             |   126 MB/s |  1531 MB/s |     8007717 | 94.49 |
| lzmat 1.01                  |    45 MB/s |   334 MB/s |     6953823 | 82.06 |
|                             |            |            |             |       |


Here's enwik9 from the Large Text Compression Benchmark, mainly for completeness, as there is a quite large number of published results for this benchmark elsewhere. Curiously here, we actually beat memcpy, probably due to some OS related memory management shenanigans:

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      |  3606 MB/s |  3627 MB/s |  1000000000 |100.00 |
| lz4 r131                    |   478 MB/s |  2648 MB/s |   509203962 | 50.92 |
| lz4hc r131 level 1          |   130 MB/s |  2508 MB/s |   433450594 | 43.35 |
| lz4hc r131 level 4          |    73 MB/s |  2663 MB/s |   384213411 | 38.42 |
| lz4hc r131 level 12         |    36 MB/s |  2706 MB/s |   374049977 | 37.40 |
| lz4hc r131 level 16         |    36 MB/s |  2715 MB/s |   374049340 | 37.40 |
| lzsse2 0.1                  |  0.14 MB/s |  3836 MB/s |   340268143 | 34.03 |
| lzsse4 0.1                  |   220 MB/s |  3322 MB/s |   425848263 | 42.58 |
| lzsse8 0.1                  |   225 MB/s |  3214 MB/s |   427471454 | 42.75 |
|                             |            |            |             |       |


## Future Work ##
Currently, this experiment is pretty young and I'm sure there are plenty of potential improvements and optimizations I haven't yet thought of. 

Here are a few I have:

 - An adaptive threshold that tries to balance between match and literal lengths in controls, to be able to copy the maximum amount of bytes on average per decompression step, as well to get some wins in terms of compression ratio. This is critically important as it will lead to a single codec that is more generically applicable, as well as performing better.
   
 - Try and implement something closer to LZ4 with alternating literal length/match sequences instead of the threshold encoding. Should give better performance on literal heavy data, but would require using the carry value for both literals and matches.

 - Implement a much faster matching algorithm for the optimal parse (suffix array or a much better tree implementation) and add an optimal parse implementation for all variants. Also, generally make the compressors perform better.
 
 - Get this working and producing decent machine code on other compilers and OSs (at least clang, other versions of Visual Studio and gcc).
 
 - Turning this into an actual production quality library if people are interested.