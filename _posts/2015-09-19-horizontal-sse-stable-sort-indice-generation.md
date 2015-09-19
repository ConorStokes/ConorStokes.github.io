---
layout: post
title: "Horizontal SSE Stable Sort Indice Generation"
description: "Generating Stable Sort Indices for an SSE Register"
category: [Optimization]
tags: [SSE, Optimization]
---
{% include JB/setup %}

Well, the topic of this blog post is a deceptively simple one; given 4 float values (although you could implement it for integers), in an SSE register, we're going to perform a stable sort and output the destination indices (from 0 to 3) for the sorted elements in another SSE register, without branches. This came about because I thought it might be useful for some [QBVH](https://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/QBVH.pdf) nearest neighbour distance queries I'm potentially going to be implementing.

I haven't yet thought about if or how this would be extended to a wider instruction set like AVX, but I don't think it would be impossible (just hard).

## The Basic Premise ##
Firstly, in this case, just sorting the keys themselves isn't enough, we actually want to be able to treat them as keys to re-order *another set of values*. To achieve this, we're going to be producing destination indices (i.e. each original key will have a new "destination" in the sorted output), which can be used either to re-order the original keys, or other values stored in the same order as the original keys. 

Instead of using swap operations, as you would in a regular sorting network implementation, we're instead going to build the indices using a system where all keys start at index 0, then we increase the index by 1 if it has a "losing" comparison against another key. So, if a key loses a comparison against all 3 other keys, its index will be 3.

Obviously, a naive implementation of this would have problems with keys that are equal. To get around this, we're going to implement a simple tie-breaking rule, which is that in the case that the two keys are equal, then the key with the original highest place in the list will lose. This will also give the sort the property of being stable, so values that are equal will maintain their original relative ordering.

One of the interesting things about this is that because the operation of changing the relative orders is associative (adding 1 to the index), it does not matter if we use the same key in multiple comparisons at once, because we can combine the offsets to the indices at the end. 

So, assuming an ascending sort, we'll do two "rounds" of comparisons (considering we can do 4 at once). Each comparison will be a greater than (with the value with the lower "original" index being the left operand and the higher the second operand). This means if the left value is greater than the first, its output index will be incremented by one, but if the right value is greater than or equal to the first, its output index will be incremented.

The first round looks like this:

	key[ 0 ] > key[ 1 ]
	key[ 1 ] > key[ 2 ]
	key[ 2 ] > key[ 3 ]
	key[ 0 ] > key[ 3 ]  

The second round looks like this:

	key[ 0 ] > key[ 2 ]
	key[ 1 ] > key[ 3 ]
 
## The Implementation

The total implementation (for the actual sorting algorithm, not counting loading the input, storing the output indices or loading constants, because they are assumed to be done out of loop) comes in at 14 operations and only requires SSE2. The operation counts are:

 - 2 adds
 - 2 and
 - 1 and not
 - 5 shuffles
 - 2 greater than compares
 - 2 xors

We'll explain the implementation step by step...

First, we load the constants. My assumption for the algorithm is that these will be done out of loop and be kept in registers (which will be the case for my implementation, but may not always be true):

    __m128i flip1 = _mm_set_epi32( 0xFFFFFFFF, 0, 0, 0 );
    __m128i flip2 = _mm_set_epi32( 0xFFFFFFFF, 0xFFFFFFFF, 0, 0 );
    __m128i mask1 = _mm_set1_epi32( 1 );

Now for the actual sorting algorithm itself. First we shuffle the input to setup the first round of comparisons:

    __m128 left1  = _mm_shuffle_ps( input, input, _MM_SHUFFLE( 0, 2, 1, 0 ) );
    __m128 right1 = _mm_shuffle_ps( input, input, _MM_SHUFFLE( 3, 3, 2, 1 ) );

After that, we compare the first values and increment the indices of the losers. Note, there are 8 possible losers in the set (and 4 actual losers), so we will have two 4 element registers as the output for this step (with each element either being 1 or 0). Because there are instances where we reference the same key twice (on both sides) in this set of comparisons and we want to do the increments for the "losing" keys in parallel (where we can't add 2 source elements from the register into one destination), we use an xor to flip the bits for the last element, effectively switching the potential loser at element 3, with the potential loser at element 0. This also avoids a shuffle on the first set of loser increments, because they are already arranged in the right order. Note that the losers on one side of the comparison match up with the winners on the other (we just need to shuffle them to the appropriate places).

    __m128i tournament1 = _mm_castps_si128( _mm_cmpgt_ps( left1, right1 ) ); 
    __m128i losers1     = _mm_xor_si128( tournament1, flip1 );
    __m128i winners2    = _mm_shuffle_epi32( losers1, _MM_SHUFFLE( 2, 1, 0, 3 ) );

Because our comparisons result in 0xFFFFFFFF instead of 1, we mask out everything but the bottom bit (because we only want to increment by 1). We convert the winners to losers (with a complement) and mask out the bits in a single step using the and-not instrinsic/instruction.

    __m128i maskedLosers1 = _mm_and_si128( losers1, mask1 );
    __m128i maskedLosers2 = _mm_andnot_si128( winners2, mask1 );

Next, we need to do the second round of comparisons. This time we only have 2 comparisons and 4 possible losers (of which 2 will actually be losers). Also luckily, the four possible losers are each one of the four keys. This time what we're going to do is double up on the comparisons (doing each one twice, but all the results in one register), but then flip the results for the top two, before masking the values to 1.

    __m128i tournament2   = _mm_castps_si128( _mm_cmpgt_ps( left2, right2 ) );
    __m128i losers3       = _mm_xor_si128( tournament2, flip2 );
    __m128i maskedLosers3 = _mm_and_si128( losers3, mask1 );

Finally, we will add all the masked losers together, like a failed superhero convention:

    __m128i indices = 
	    _mm_add_epi32( 
	        _mm_add_epi32( maskedLosers1, 
	                       maskedLosers2 ), 
	        maskedLosers3 );

Now if you want to, you can take 4 unsorted values (in the same order as the keys) and scatter them using these indices. Unfortunately SSE doesn't have any gather/scatter functionality, so you'll have to do that bit the old fashioned way.

## Have You Tested It?

Why yes! I have enumerated all possible ordering permutations with repetition (using the values 0.0, 1.0, 2.0 and 3.0). Early performance testing implies low to mid single digit nanoseconds per sort.

## What about NaNs?

NaNs aren't possible in my current use case, so I haven't walked through all the possibilities yet.