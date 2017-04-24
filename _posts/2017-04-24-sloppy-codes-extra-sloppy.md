---
layout: post
title: "Sloppy Codes, Extra Sloppy"
description: ""
category: "Compression"
tags: [Compression, Sloppy, SSE]
---
{% include JB/setup %}
Between the Marlin paper coming out and a contract I'm currently working on, I started thinking about compression again a bit more recently. Friday night, I had an idea (it was quite a bad idea) about using a 16 entry Tunstall dictionary for binary alphabet entropy coding (with variable length output up to 8 bits in length) and using a byte shuffle to lookup the codes from a register in parallel when decoding, throwing in an adaption step.

My initial thought was that maybe it would be a good way to get a cheap compression win for encoding if the next input was a match or a literal in a fast LZ (and using a count trailing bits to count runs of multiple literals). I started to dismiss the idea as particularly useful by Saturday morning, thinking it was the product of too much Friday night cheese and craft beer after thinking about compression all week. However (quite quixotically), I decided to implement it anyway (standalone), because I had some neat ideas for the decoding implementation in particular that I wanted to try out. The implementation turned out quite interesting and I thought it would be neat to share the techniques involved. I also think it's good to explore bad technical ideas sometimes in case you are surprised or discover something new, and to talk about the things that don't work out.

As predicted, it ended up being fast (for an adaptive binary entropy coder), but quite terrible at getting much of a compression win. I call the results "Sloppy Coding" (sloppy codes, extra sloppy) and the repository is available [here](https://github.com/ConorStokes/Sloppy-Coding).

Firstly, I wrote a small Tunstall dictionary generator for binary alphabets that generated 16 entry dictionaries for 64 different probabilities and outputted tables for encoding/decoding as C/C++ code, so the tables can be shipped with the code and compiled inline. Each dictionary contains the binary output codes, the length of the codes, an extraction mask for each code and the probability of a bit being on or off in each node. This totals at 64 bytes (one cache line) for an entire dictionary (or 4K for all the dictionaries).

The encoded format itself packs 32 nibble words (dictionary references) into a 16 byte control word. First the low nibbles, then the high nibbles for the word are populated. 

At decode time, we load the control word as well as the dictionary into a SSE registers, then split out the low nibbles:

	__m128i dictionary     = _mm_loadu_si128( reinterpret_cast< const __m128i* >( decodingTable ) + dictionaryIndex * 4 );
	__m128i extract        = _mm_loadu_si128( reinterpret_cast< const __m128i* >( decodingTable ) + dictionaryIndex * 4 + 1 );
	__m128i counts         = _mm_loadu_si128( reinterpret_cast< const __m128i* >( decodingTable ) + dictionaryIndex * 4 + 2 );
	
	__m128i  nibbles       = _mm_loadu_si128( reinterpret_cast< const __m128i* >( compressed ) );

	__m128i  loNibbles     = _mm_and_si128( nibbles, nibbleMask ); 

Then we use PSHUFB to look up the dictionary values, bit lengths and extraction masks. We do a quick horizontal add with PSADBW to sum up the bit-lengths (we do the same for the high nibbles after):

	__m128i  symbols   = _mm_shuffle_epi8( dictionary, loNibbles );
    __m128i  mask      = _mm_shuffle_epi8( extract, loNibbles );
    __m128i  count     = _mm_shuffle_epi8( counts, loNibbles );
    __m128i  sumCounts = _mm_sad_epu8( count, zero );

Then there's an extraction and output macro that extracts a 64 bit word from the symbols, masks, sums and counts, uses PEXT to get rid of the empty spaces between the values coming from the dictionary (this macro is used four times, twice each for the high and low nibbles). After that, it branchlessly outputs the compacted bits. We have at least 16 bits and at most 64 bits extracted at a time (and at most 7 bits left in the bit buffer). Note when repopulating the bit buffer at the end we have a small work around in case we end up with a shift right by 64. The branch-less bit output, as well as the combination of PSHUFB and PEXT are some of the more interesting parts here (and a big part of what I wanted to try out). Similar code could be used to output Huffman codes for a small alphabet:

	uint64_t symbolsExtract  = 
		static_cast< uint64_t >( _mm_extract_epi64( symbols, extractIndex ) );
	uint64_t maskExtract     = 
		static_cast< uint64_t >( _mm_extract_epi64( mask, extractIndex ) );
	uint64_t countExtract    = 
		static_cast< uint64_t >( _mm_extract_epi64( sumCounts, extractIndex ) );
	
	uint64_t compacted  = _pext_u64( symbolsExtract, maskExtract );
	                                                       
	bitBuffer |= ( compacted << ( decompressedBits & 7 ) );
	
	*reinterpret_cast< uint64_t* >( outBuffer + ( decompressedBits >> 3 ) ) = bitBuffer;                               

	decompressedBits += countExtract;                                                                                 
	
	bitBuffer = ( -( ( decompressedBits & 7 ) != 0 ) ) & 
	            ( compacted >> ( countExtract - ( decompressedBits & 7 ) ) );

After we've done all this for the high and low nibbles/high and low words, we adapt and choose a new dictionary. We do this by adding together all the probabilities (which are in the range 0 to 255), summing them together and dividing by 128 (no fancy rounding): 

        __m128i probabilities = _mm_loadu_si128( reinterpret_cast< const __m128i* >( decodingTable ) + dictionaryIndex * 4 + 3 );
        __m128i loProbs       = _mm_shuffle_epi8( probabilities, loNibbles );
        __m128i sumLoProbs    = _mm_sad_epu8( loProbs, zero );
        __m128i hiProbs       = _mm_shuffle_epi8( probabilities, hiNibbles );
        __m128i sumHiProbs    = _mm_sad_epu8( hiProbs, zero );
        __m128i sumProbs      = _mm_add_epi32( sumHiProbs, sumLoProbs );

        dictionaryIndex = ( _mm_extract_epi32( sumProbs, 0 ) + _mm_extract_epi32( sumProbs, 2 ) ) >> 7;

Currently in the code there is no special handling at end of buffer (I just assume that there is extra space at the end of the buffer) because this is more of a (dis)proof of concept than anything else. The last up-to 7 bits in the bit buffer are flushed at the end.

## Results
For testing, I generated 128 megabytes of randomly set bits (multiple times, according to various probability distributions), as well as a one that used a sine wave to vary the probability distribution (to make sure that it was adapting). Compression and decompression speed are in megabytes (the old fashioned 1024<sup>2</sup> variety) averaged over 15 runs. Bench-marking machine is my usual Haswell i7 4790. 

| Probability&nbsp; | Compression Speed&nbsp; | Encoded Size | Ratio&nbsp;&nbsp; | &nbsp;&nbsp;Decompression Speed&nbsp; | 
|-------------|-------------------|--------------|----------|---------------------| 
| 0           | 206.230639        | 67114272     | &nbsp;&nbsp;0.50004  | &nbsp;&nbsp;2158.874927         | 
| 0.066667    | 175.3852          | 79162256     | &nbsp;&nbsp;0.589805 | &nbsp;&nbsp;1831.462974         | 
| 0.133333    | 151.631986        | 91572032     | &nbsp;&nbsp;0.682265 | &nbsp;&nbsp;1586.54503          | 
| 0.2         | 132.081896        | 104868416    | &nbsp;&nbsp;0.781331 | &nbsp;&nbsp;1377.348473         | 
| 0.266667    | 117.898678        | 117748768    | &nbsp;&nbsp;0.877297 | &nbsp;&nbsp;1236.169277         | 
| 0.333333    | 108.803616        | 127463824    | &nbsp;&nbsp;0.949679 | &nbsp;&nbsp;1141.881051         | 
| 0.4         | 104.331363        | 133059200    | &nbsp;&nbsp;0.991368 | &nbsp;&nbsp;1084.070366         | 
| 0.466667    | 102.554293        | 134885904    | &nbsp;&nbsp;1.004978 | &nbsp;&nbsp;1057.176446         | 
| 0.533333    | 102.718932        | 134931136    | &nbsp;&nbsp;1.005315 | &nbsp;&nbsp;1066.073632         | 
| 0.6         | 104.353041        | 133057584    | &nbsp;&nbsp;0.991356 | &nbsp;&nbsp;1092.383074         | 
| 0.666667    | 108.918791        | 127351120    | &nbsp;&nbsp;0.94884  | &nbsp;&nbsp;1143.198439         | 
| 0.733333    | 118.245568        | 117570320    | &nbsp;&nbsp;0.875967 | &nbsp;&nbsp;1226.966374         | 
| 0.8         | 132.864274        | 104712784    | &nbsp;&nbsp;0.780171 | &nbsp;&nbsp;1389.678224         | 
| 0.866667    | 151.684972        | 91559024     | &nbsp;&nbsp;0.682168 | &nbsp;&nbsp;1585.006786         | 
| 0.933333    | 174.659716        | 79162864     | &nbsp;&nbsp;0.589809 | &nbsp;&nbsp;1830.458242         | 
| 1           | 205.484085        | 67108880     | &nbsp;&nbsp;0.5      | &nbsp;&nbsp;2162.718762         | 
| Sine Wave   | 140.850482        | 98377072     | &nbsp;&nbsp;0.732966 | &nbsp;&nbsp;1461.307419         | 
