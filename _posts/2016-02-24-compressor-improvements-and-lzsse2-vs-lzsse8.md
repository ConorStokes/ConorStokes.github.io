---
layout: post
title: "Compressor Improvements and LZSSE2 vs LZSSE8"
description: "Evolving LZSSE"
category: Compression
tags: [Compression, LZSSE, Optimization]
---
{% include JB/setup %}

One of the things that I hadn't put a lot of time into with [LZSSE](conorstokes.github.io/compression/2016/02/15/an-LZ-codec-designed-for-SSE-decompression), when I shoved it out into the open last week, was the compressor side of things. The compressors were mainly there as a way to show that the codec/decompression functioned and although it was pretty young and experimental, I thought it was worth sharing and getting some feedback on.

One of the things that became fairly apparent (and anyone who eyeballed the original benchmarks would've seen) is that the compression speed for LZSSE2's optimal parser was terrible. Thanks to some pointers from the community, particularly from Charles Bloom, the match finding has been replaced with something better (an implementation of the same algorithm used in LzFind) and compression speed improved substantially. There is a very minor cost in compression ratio though on some files, although some was gained back due to fixing a bug in the cost calculations for matches. I've also created an LZSSE8 optimal parsing variant, LZSSE4's is on the way.

One of the changes, to prevent really bad performance with long matches (especially of the same character), is to skip completely over match finding after long matches. This can have some minor impact on compression ratio, so there is the option to disable this by pushing the compression level up to "17" on the optimal parser, which is otherwise the same as level 16. Levels 0 through to 15 just check through progressively less matches as the level gets lower (amortization). 

The compression changes to LZSSE8 proved rather important, because there are some files LZSSE2 really struggled with, particularly machine code binaries and less compressible data. With the compressors for LZSSE2 and LZSSE8 previously not being on equal footing, it wasn't possible to show how much better LZSSE8 worked on those files. The quality of the compression really has quite a large impact on the performance of the decompression on those files too. 

Part of the end goal is to make a codec that either adapts per iteration (32 steps), or allows changing between the different codecs per block to get the ideal compression ratio and performance on a wider range of files without changing codec. This is not there yet, but the results below make a strong case for it. 

I was lucky enough to get some code contributed by Brian Marshall that got things working with Gcc and Clang (although, it was ported elsewhere as part of [turbobench](https://github.com/powturbo/TurboBench)). Performance of the decompression on Gcc lags Visual Studio by over 10% (sometimes even higher) and investigating why that is will be an interesting experiment in and of itself. The code is the same for both, there is nothing compiler specific in the decompression.

I'm still intending to release a stand-alone single file utility (both as a binary and on GitHub), but there are still some changes I want to make there before it's public.

Another note, there has been a few problems with the blog comments, if you make a comment and it doesn't show up, I haven't deleted it. I'm still investigating why this is.

## Updated Results ##

Note, the decompression implementation hasn't changed here. I'm using the same method for benchmarks as before (same stock clocked i7 4790, Windows 10 machine, same build settings, same lzbench code), but with the results reduced to LZSSE variants, because I mainly want to compare LZSSE2 against LZSSE8 here (the results for the other compressors haven't changed, if you want the comparisons).

The tl;dr version is that LZSSE8 lags a little on text, but is a better general purpose option, when used with the new optimal parser, than LZSSE2. LZSSE8 isn't that far behind on text, but where LZSSE2 does badly (less compressible data, machine code and long literal runs), LZSSE8 does a lot better (this makes a convincing case for an adaptive codec). Compression speed, especially on the previous terrible cases, is a lot better.

First up, enwik8; our new LZSSE2 compression implementation has given up a few bytes here and the new optimal parser for LZSSE8 is a little bit behind. Compression speed is significantly improved. No real surprises here.

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      | 11511 MB/s | 11562 MB/s |   100000000 |100.00 |
| lzsse2 0.1 level 1          |    18 MB/s |  3138 MB/s |    45854485 | 45.85 |
| lzsse2 0.1 level 2          |    13 MB/s |  3403 MB/s |    41941798 | 41.94 |
| lzsse2 0.1 level 4          |  9.37 MB/s |  3739 MB/s |    38121396 | 38.12 |
| lzsse2 0.1 level 8          |  9.30 MB/s |  3720 MB/s |    38068526 | 38.07 |
| lzsse2 0.1 level 12         |  9.28 MB/s |  3718 MB/s |    38068509 | 38.07 |
| lzsse2 0.1 level 16         |  9.29 MB/s |  3748 MB/s |    38068509 | 38.07 |
| lzsse2 0.1 level 17         |  9.05 MB/s |  3717 MB/s |    38063182 | 38.06 |
| lzsse4 0.1                  |   213 MB/s |  3122 MB/s |    47108403 | 47.11 |
| lzsse8f 0.1                 |   212 MB/s |  3108 MB/s |    47249359 | 47.25 |
| lzsse8o 0.1 level 1         |    15 MB/s |  3434 MB/s |    41889439 | 41.89 |
| lzsse8o 0.1 level 2         |    12 MB/s |  3559 MB/s |    40129972 | 40.13 |
| lzsse8o 0.1 level 4         |    10 MB/s |  3668 MB/s |    38745570 | 38.75 |
| lzsse8o 0.1 level 8         |    10 MB/s |  3671 MB/s |    38721328 | 38.72 |
| lzsse8o 0.1 level 12        |    10 MB/s |  3670 MB/s |    38721328 | 38.72 |
| lzsse8o 0.1 level 16        |    10 MB/s |  3698 MB/s |    38721328 | 38.72 |
| lzsse8o 0.1 level 17        |    10 MB/s |  3673 MB/s |    38716643 | 38.72 |
|                             |            |            |             |       |

LZSSE8 shows that with an optimal parser that it's a better generalist on the tar'd silesia corpus than LZSSE2 (which has poor handling of hard to compress data), with both stronger compression and significantly faster decompression. Compression speed again is significantly improved:

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      | 11406 MB/s | 11347 MB/s |   211948032 |100.00 |
| lzsse2 0.1 level 1          |    19 MB/s |  3182 MB/s |    87976190 | 41.51 |
| lzsse2 0.1 level 2          |    14 MB/s |  3392 MB/s |    82172004 | 38.77 |
| lzsse2 0.1 level 4          |  9.57 MB/s |  3636 MB/s |    76093504 | 35.90 |
| lzsse2 0.1 level 8          |  8.62 MB/s |  3637 MB/s |    75830436 | 35.78 |
| lzsse2 0.1 level 12         |  8.56 MB/s |  3662 MB/s |    75830178 | 35.78 |
| lzsse2 0.1 level 16         |  8.77 MB/s |  3663 MB/s |    75830178 | 35.78 |
| lzsse2 0.1 level 17         |  3.42 MB/s |  3669 MB/s |    75685829 | 35.71 |
| lzsse4 0.1                  |   275 MB/s |  3243 MB/s |    95918518 | 45.26 |
| lzsse8f 0.1                 |   276 MB/s |  3405 MB/s |    94938891 | 44.79 |
| lzsse8o 0.1 level 1         |    17 MB/s |  3932 MB/s |    81866286 | 38.63 |
| lzsse8o 0.1 level 2         |    14 MB/s |  4085 MB/s |    78727052 | 37.14 |
| lzsse8o 0.1 level 4         |    10 MB/s |  4262 MB/s |    78723935 | 37.14 |
| lzsse8o 0.1 level 8         |  9.77 MB/s |  4284 MB/s |    75464683 | 35.61 |
| lzsse8o 0.1 level 12        |  9.70 MB/s |  4283 MB/s |    75464456 | 35.61 |
| lzsse8o 0.1 level 16        |  9.68 MB/s |  4283 MB/s |    75464456 | 35.61 |
| lzsse8o 0.1 level 17        |  3.50 MB/s |  4289 MB/s |    75328279 | 35.54 |
|                             |            |            |             |       |

Dickens favours LZSSE2, but LZSSE8 is not that far behind:

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      | 13773 MB/s | 13754 MB/s |    10192446 |100.00 |
| lzsse2 0.1 level 1          |    19 MB/s |  2938 MB/s |     5259008 | 51.60 |
| lzsse2 0.1 level 2          |    13 MB/s |  3326 MB/s |     4626067 | 45.39 |
| lzsse2 0.1 level 4          |  7.96 MB/s |  3935 MB/s |     3878822 | 38.06 |
| lzsse2 0.1 level 8          |  7.92 MB/s |  3932 MB/s |     3872650 | 38.00 |
| lzsse2 0.1 level 12         |  7.90 MB/s |  3876 MB/s |     3872650 | 38.00 |
| lzsse2 0.1 level 16         |  7.90 MB/s |  3944 MB/s |     3872650 | 38.00 |
| lzsse2 0.1 level 17         |  7.78 MB/s |  3942 MB/s |     3872373 | 37.99 |
| lzsse4 0.1                  |   209 MB/s |  2948 MB/s |     5161515 | 50.64 |
| lzsse8f 0.1                 |   209 MB/s |  2953 MB/s |     5174322 | 50.77 |
| lzsse8o 0.1 level 1         |    15 MB/s |  3256 MB/s |     4515350 | 44.30 |
| lzsse8o 0.1 level 2         |    12 MB/s |  3448 MB/s |     4207915 | 41.28 |
| lzsse8o 0.1 level 4         |  9.12 MB/s |  3734 MB/s |     3922639 | 38.49 |
| lzsse8o 0.1 level 8         |  9.15 MB/s |  3752 MB/s |     3920885 | 38.47 |
| lzsse8o 0.1 level 12        |  9.15 MB/s |  3741 MB/s |     3920885 | 38.47 |
| lzsse8o 0.1 level 16        |  9.26 MB/s |  3640 MB/s |     3920885 | 38.47 |
| lzsse8o 0.1 level 17        |  8.98 MB/s |  3751 MB/s |     3920616 | 38.47 |
|                             |            |            |             |       |

Mozilla, LZSSE8 provides a significant decompression performance advantage and a slightly better compression ratio:

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      | 11448 MB/s | 11433 MB/s |    51220480 |100.00 |
| lzsse2 0.1 level 1          |    21 MB/s |  2744 MB/s |    24591826 | 48.01 |
| lzsse2 0.1 level 2          |    16 MB/s |  2868 MB/s |    23637008 | 46.15 |
| lzsse2 0.1 level 4          |    11 MB/s |  3037 MB/s |    22584848 | 44.09 |
| lzsse2 0.1 level 8          |  9.97 MB/s |  3063 MB/s |    22474739 | 43.88 |
| lzsse2 0.1 level 12         |  9.93 MB/s |  3064 MB/s |    22474508 | 43.88 |
| lzsse2 0.1 level 16         |  9.82 MB/s |  3063 MB/s |    22474508 | 43.88 |
| lzsse2 0.1 level 17         |  2.01 MB/s |  3038 MB/s |    22460261 | 43.85 |
| lzsse4 0.1                  |   257 MB/s |  2635 MB/s |    27406939 | 53.51 |
| lzsse8f 0.1                 |   261 MB/s |  2817 MB/s |    26993974 | 52.70 |
| lzsse8o 0.1 level 1         |    18 MB/s |  3369 MB/s |    23512213 | 45.90 |
| lzsse8o 0.1 level 2         |    15 MB/s |  3477 MB/s |    22941395 | 44.79 |
| lzsse8o 0.1 level 4         |    12 MB/s |  3646 MB/s |    22243004 | 43.43 |
| lzsse8o 0.1 level 8         |    10 MB/s |  3647 MB/s |    22148536 | 43.24 |
| lzsse8o 0.1 level 12        |    10 MB/s |  3688 MB/s |    22148366 | 43.24 |
| lzsse8o 0.1 level 16        |    10 MB/s |  3688 MB/s |    22148366 | 43.24 |
| lzsse8o 0.1 level 17        |  1.96 MB/s |  3689 MB/s |    22135467 | 43.22 |
|                             |            |            |             |       |

LZSSE8 again shows that it can significantly best LZSSE2 on some files on mr, the magnetic resonance image:

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      | 13848 MB/s | 13925 MB/s |     9970564 |100.00 |
| lzsse2 0.1 level 1          |    30 MB/s |  2801 MB/s |     4847277 | 48.62 |
| lzsse2 0.1 level 2          |    21 MB/s |  3004 MB/s |     4506789 | 45.20 |
| lzsse2 0.1 level 4          |    12 MB/s |  3250 MB/s |     4018986 | 40.31 |
| lzsse2 0.1 level 8          |    11 MB/s |  3263 MB/s |     4013037 | 40.25 |
| lzsse2 0.1 level 12         |    11 MB/s |  3263 MB/s |     4013037 | 40.25 |
| lzsse2 0.1 level 16         |    11 MB/s |  3263 MB/s |     4013037 | 40.25 |
| lzsse2 0.1 level 17         |  1.89 MB/s |  3333 MB/s |     4007016 | 40.19 |
| lzsse4 0.1                  |   249 MB/s |  2560 MB/s |     7034206 | 70.55 |
| lzsse8f 0.1                 |   279 MB/s |  2990 MB/s |     6856430 | 68.77 |
| lzsse8o 0.1 level 1         |    27 MB/s |  3421 MB/s |     4697617 | 47.11 |
| lzsse8o 0.1 level 2         |    20 MB/s |  3677 MB/s |     4384872 | 43.98 |
| lzsse8o 0.1 level 4         |    14 MB/s |  4058 MB/s |     4037906 | 40.50 |
| lzsse8o 0.1 level 8         |    14 MB/s |  4010 MB/s |     4033597 | 40.46 |
| lzsse8o 0.1 level 12        |    14 MB/s |  4138 MB/s |     4033597 | 40.46 |
| lzsse8o 0.1 level 16        |    14 MB/s |  4009 MB/s |     4033597 | 40.46 |
| lzsse8o 0.1 level 17        |  1.87 MB/s |  4026 MB/s |     4029062 | 40.41 |
|                             |            |            |             |       |

The nci database of chemical structures shows that LZSSE2 still definitely has some things it's better at:

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      | 11550 MB/s | 11642 MB/s |    33553445 |100.00 |
| lzsse2 0.1 level 1          |    20 MB/s |  5616 MB/s |     5295570 | 15.78 |
| lzsse2 0.1 level 2          |    14 MB/s |  6368 MB/s |     4423662 | 13.18 |
| lzsse2 0.1 level 4          |  7.86 MB/s |  6981 MB/s |     3810991 | 11.36 |
| lzsse2 0.1 level 8          |  6.17 MB/s |  7110 MB/s |     3749513 | 11.17 |
| lzsse2 0.1 level 12         |  6.16 MB/s |  7178 MB/s |     3749513 | 11.17 |
| lzsse2 0.1 level 16         |  6.17 MB/s |  7107 MB/s |     3749513 | 11.17 |
| lzsse2 0.1 level 17         |  3.00 MB/s |  7171 MB/s |     3703443 | 11.04 |
| lzsse4 0.1                  |   497 MB/s |  5524 MB/s |     5711408 | 17.02 |
| lzsse8f 0.1                 |   501 MB/s |  5325 MB/s |     5796797 | 17.28 |
| lzsse8o 0.1 level 1         |    18 MB/s |  5975 MB/s |     4558097 | 13.58 |
| lzsse8o 0.1 level 2         |    14 MB/s |  6225 MB/s |     4202797 | 12.53 |
| lzsse8o 0.1 level 4         |  7.94 MB/s |  6538 MB/s |     3903767 | 11.63 |
| lzsse8o 0.1 level 8         |  6.48 MB/s |  6601 MB/s |     3872165 | 11.54 |
| lzsse8o 0.1 level 12        |  6.48 MB/s |  6601 MB/s |     3872165 | 11.54 |
| lzsse8o 0.1 level 16        |  6.54 MB/s |  6601 MB/s |     3872165 | 11.54 |
| lzsse8o 0.1 level 17        |  3.09 MB/s |  6638 MB/s |     3828113 | 11.41 |
|                             |            |            |             |       |

LZSSE8 again shows it's strength on binaries with the ooffice dll:

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      | 17379 MB/s | 17379 MB/s |     6152192 |100.00 |
| lzsse2 0.1 level 1          |    21 MB/s |  2205 MB/s |     3821230 | 62.11 |
| lzsse2 0.1 level 2          |    17 MB/s |  2297 MB/s |     3681963 | 59.85 |
| lzsse2 0.1 level 4          |    12 MB/s |  2444 MB/s |     3505982 | 56.99 |
| lzsse2 0.1 level 8          |    12 MB/s |  2424 MB/s |     3496155 | 56.83 |
| lzsse2 0.1 level 12         |    12 MB/s |  2457 MB/s |     3496154 | 56.83 |
| lzsse2 0.1 level 16         |    12 MB/s |  2455 MB/s |     3496154 | 56.83 |
| lzsse2 0.1 level 17         |  7.03 MB/s |  2457 MB/s |     3495182 | 56.81 |
| lzsse4 0.1                  |   166 MB/s |  2438 MB/s |     3972687 | 64.57 |
| lzsse8f 0.1                 |   162 MB/s |  2712 MB/s |     3922013 | 63.75 |
| lzsse8o 0.1 level 1         |    19 MB/s |  2936 MB/s |     3635572 | 59.09 |
| lzsse8o 0.1 level 2         |    17 MB/s |  3041 MB/s |     3564756 | 57.94 |
| lzsse8o 0.1 level 4         |    14 MB/s |  3063 MB/s |     3499888 | 56.89 |
| lzsse8o 0.1 level 8         |    14 MB/s |  3097 MB/s |     3494693 | 56.80 |
| lzsse8o 0.1 level 12        |    14 MB/s |  3099 MB/s |     3494693 | 56.80 |
| lzsse8o 0.1 level 16        |    14 MB/s |  3062 MB/s |     3494693 | 56.80 |
| lzsse8o 0.1 level 17        |  7.25 MB/s |  3100 MB/s |     3493814 | 56.79 |
|                             |            |            |             |       |

The MySQL database, osdb, strongly favours LZSSE8 as well:

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      | 14086 MB/s | 14185 MB/s |    10085684 |100.00 |
| lzsse2 0.1 level 1          |    18 MB/s |  3200 MB/s |     4333712 | 42.97 |
| lzsse2 0.1 level 2          |    14 MB/s |  3308 MB/s |     4172322 | 41.37 |
| lzsse2 0.1 level 4          |    11 MB/s |  3546 MB/s |     3960769 | 39.27 |
| lzsse2 0.1 level 8          |    11 MB/s |  3555 MB/s |     3957654 | 39.24 |
| lzsse2 0.1 level 12         |    11 MB/s |  3556 MB/s |     3957654 | 39.24 |
| lzsse2 0.1 level 16         |    11 MB/s |  3557 MB/s |     3957654 | 39.24 |
| lzsse2 0.1 level 17         |    11 MB/s |  3558 MB/s |     3957654 | 39.24 |
| lzsse4 0.1                  |   265 MB/s |  4098 MB/s |     4262592 | 42.26 |
| lzsse8f 0.1                 |   259 MB/s |  4484 MB/s |     4175474 | 41.40 |
| lzsse8o 0.1 level 1         |    16 MB/s |  4628 MB/s |     3962308 | 39.29 |
| lzsse8o 0.1 level 2         |    13 MB/s |  4673 MB/s |     3882195 | 38.49 |
| lzsse8o 0.1 level 4         |    11 MB/s |  4723 MB/s |     3839571 | 38.07 |
| lzsse8o 0.1 level 8         |    11 MB/s |  4723 MB/s |     3839569 | 38.07 |
| lzsse8o 0.1 level 12        |    11 MB/s |  4717 MB/s |     3839569 | 38.07 |
| lzsse8o 0.1 level 16        |    11 MB/s |  4719 MB/s |     3839569 | 38.07 |
| lzsse8o 0.1 level 17        |    11 MB/s |  4721 MB/s |     3839569 | 38.07 |
|                             |            |            |             |       |

The Polish literature PDF Reymont definitely favours LZSSE2 though, but LZSSE8 isn't that far out:

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      | 16693 MB/s | 16906 MB/s |     6627202 |100.00 |
| lzsse2 0.1 level 1          |    19 MB/s |  3416 MB/s |     2657481 | 40.10 |
| lzsse2 0.1 level 2          |    13 MB/s |  4004 MB/s |     2309022 | 34.84 |
| lzsse2 0.1 level 4          |  7.55 MB/s |  4883 MB/s |     1867895 | 28.19 |
| lzsse2 0.1 level 8          |  7.32 MB/s |  4967 MB/s |     1852687 | 27.96 |
| lzsse2 0.1 level 12         |  7.30 MB/s |  4994 MB/s |     1852684 | 27.96 |
| lzsse2 0.1 level 16         |  7.30 MB/s |  4990 MB/s |     1852684 | 27.96 |
| lzsse2 0.1 level 17         |  7.26 MB/s |  4960 MB/s |     1852682 | 27.96 |
| lzsse8f 0.1                 |   252 MB/s |  2989 MB/s |     2952727 | 44.55 |
| lzsse4 0.1                  |   251 MB/s |  3020 MB/s |     2925190 | 44.14 |
| lzsse8o 0.1 level 1         |    16 MB/s |  3659 MB/s |     2402352 | 36.25 |
| lzsse8o 0.1 level 2         |    12 MB/s |  4001 MB/s |     2179138 | 32.88 |
| lzsse8o 0.1 level 4         |  8.10 MB/s |  4409 MB/s |     1905807 | 28.76 |
| lzsse8o 0.1 level 8         |  8.00 MB/s |  4394 MB/s |     1901961 | 28.70 |
| lzsse8o 0.1 level 12        |  8.01 MB/s |  4391 MB/s |     1901958 | 28.70 |
| lzsse8o 0.1 level 16        |  8.01 MB/s |  4415 MB/s |     1901958 | 28.70 |
| lzsse8o 0.1 level 17        |  7.97 MB/s |  4415 MB/s |     1901955 | 28.70 |
|                             |            |            |             |       | 

The samba source code offers a bit of a surprise, also doing better on LZSSE8. I haven't studied the compression characteristics of source code too much, but expected it to favour LZSSE2 due to it's favouring matches over literals. 

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      | 11781 MB/s | 11781 MB/s |    21606400 |100.00 |
| lzsse2 0.1 level 1          |    20 MB/s |  4017 MB/s |     6883306 | 31.86 |
| lzsse2 0.1 level 2          |    16 MB/s |  4201 MB/s |     6511387 | 30.14 |
| lzsse2 0.1 level 4          |    12 MB/s |  4330 MB/s |     6177369 | 28.59 |
| lzsse2 0.1 level 8          |  9.45 MB/s |  4336 MB/s |     6161485 | 28.52 |
| lzsse2 0.1 level 12         |  8.56 MB/s |  4333 MB/s |     6161465 | 28.52 |
| lzsse2 0.1 level 16         |  8.54 MB/s |  4292 MB/s |     6161465 | 28.52 |
| lzsse2 0.1 level 17         |  2.57 MB/s |  4379 MB/s |     6093318 | 28.20 |
| lzsse4 0.1                  |   300 MB/s |  3997 MB/s |     7601765 | 35.18 |
| lzsse8f 0.1                 |   298 MB/s |  4136 MB/s |     7582300 | 35.09 |
| lzsse8o 0.1 level 1         |    18 MB/s |  4656 MB/s |     6433602 | 29.78 |
| lzsse8o 0.1 level 2         |    15 MB/s |  4761 MB/s |     6220494 | 28.79 |
| lzsse8o 0.1 level 4         |    12 MB/s |  4860 MB/s |     6043129 | 27.97 |
| lzsse8o 0.1 level 8         |    10 MB/s |  4880 MB/s |     6030715 | 27.91 |
| lzsse8o 0.1 level 12        |  9.57 MB/s |  4931 MB/s |     6030693 | 27.91 |
| lzsse8o 0.1 level 16        |  9.58 MB/s |  4930 MB/s |     6030693 | 27.91 |
| lzsse8o 0.1 level 17        |  2.58 MB/s |  4956 MB/s |     5965309 | 27.61 |
|                             |            |            |             |       |

The less compressible SAO star catalog is less compressible and LZSSE2 does very poorly compared to LZSSE8 here:

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      | 15938 MB/s | 16044 MB/s |     7251944 |100.00 |
| lzsse2 0.1 level 1          |    24 MB/s |  1785 MB/s |     6487976 | 89.47 |
| lzsse2 0.1 level 2          |    20 MB/s |  1817 MB/s |     6304480 | 86.94 |
| lzsse2 0.1 level 4          |    15 MB/s |  1888 MB/s |     6068279 | 83.68 |
| lzsse2 0.1 level 8          |    15 MB/s |  1885 MB/s |     6066923 | 83.66 |
| lzsse2 0.1 level 12         |    15 MB/s |  1903 MB/s |     6066923 | 83.66 |
| lzsse2 0.1 level 16         |    15 MB/s |  1870 MB/s |     6066923 | 83.66 |
| lzsse2 0.1 level 17         |    15 MB/s |  1887 MB/s |     6066923 | 83.66 |
| lzsse4 0.1                  |   167 MB/s |  2433 MB/s |     6305407 | 86.95 |
| lzsse8f 0.1                 |   167 MB/s |  3268 MB/s |     6045723 | 83.37 |
| lzsse8o 0.1 level 1         |    23 MB/s |  3486 MB/s |     5826841 | 80.35 |
| lzsse8o 0.1 level 2         |    20 MB/s |  3684 MB/s |     5713888 | 78.79 |
| lzsse8o 0.1 level 4         |    17 MB/s |  3875 MB/s |     5575656 | 76.88 |
| lzsse8o 0.1 level 8         |    16 MB/s |  3857 MB/s |     5575361 | 76.88 |
| lzsse8o 0.1 level 12        |    17 MB/s |  3832 MB/s |     5575361 | 76.88 |
| lzsse8o 0.1 level 16        |    17 MB/s |  3804 MB/s |     5575361 | 76.88 |
| lzsse8o 0.1 level 17        |    17 MB/s |  3861 MB/s |     5575361 | 76.88 |
|                             |            |            |             |       |

The webster dictionary has LZSSE2 edging out LZSSE8, but not by much:

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      | 11596 MB/s | 11545 MB/s |    41458703 |100.00 |
| lzsse2 0.1 level 1          |    17 MB/s |  3703 MB/s |    15546126 | 37.50 |
| lzsse2 0.1 level 2          |    12 MB/s |  4087 MB/s |    13932531 | 33.61 |
| lzsse2 0.1 level 4          |  8.30 MB/s |  4479 MB/s |    12405168 | 29.92 |
| lzsse2 0.1 level 8          |  8.12 MB/s |  4477 MB/s |    12372822 | 29.84 |
| lzsse2 0.1 level 12         |  8.12 MB/s |  4483 MB/s |    12372822 | 29.84 |
| lzsse2 0.1 level 16         |  8.12 MB/s |  4441 MB/s |    12372822 | 29.84 |
| lzsse2 0.1 level 17         |  8.12 MB/s |  4444 MB/s |    12372794 | 29.84 |
| lzsse4 0.1                  |   242 MB/s |  3542 MB/s |    16613479 | 40.07 |
| lzsse8f 0.1                 |   238 MB/s |  3376 MB/s |    16775799 | 40.46 |
| lzsse8o 0.1 level 1         |    14 MB/s |  3910 MB/s |    14218534 | 34.30 |
| lzsse8o 0.1 level 2         |    11 MB/s |  4069 MB/s |    13404951 | 32.33 |
| lzsse8o 0.1 level 4         |  9.12 MB/s |  4191 MB/s |    12700224 | 30.63 |
| lzsse8o 0.1 level 8         |  8.98 MB/s |  4191 MB/s |    12685372 | 30.60 |
| lzsse8o 0.1 level 12        |  8.98 MB/s |  4192 MB/s |    12685372 | 30.60 |
| lzsse8o 0.1 level 16        |  8.98 MB/s |  4229 MB/s |    12685372 | 30.60 |
| lzsse8o 0.1 level 17        |  9.01 MB/s |  4196 MB/s |    12685361 | 30.60 |
|                             |            |            |             |       | 

LZSSE2 edges out LZSSE8 on xml, but it is pretty close:

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      | 18624 MB/s | 18624 MB/s |     5345280 |100.00 |
| lzsse2 0.1 level 1          |    21 MB/s |  5873 MB/s |     1003195 | 18.77 |
| lzsse2 0.1 level 2          |    15 MB/s |  6303 MB/s |      885118 | 16.56 |
| lzsse2 0.1 level 4          |  9.64 MB/s |  6706 MB/s |      778294 | 14.56 |
| lzsse2 0.1 level 8          |  9.49 MB/s |  6706 MB/s |      776774 | 14.53 |
| lzsse2 0.1 level 12         |  9.40 MB/s |  6706 MB/s |      776774 | 14.53 |
| lzsse2 0.1 level 16         |  9.42 MB/s |  6706 MB/s |      776774 | 14.53 |
| lzsse2 0.1 level 17         |  3.23 MB/s |  6766 MB/s |      768186 | 14.37 |
| lzsse4 0.1                  |   380 MB/s |  4413 MB/s |     1388503 | 25.98 |
| lzsse8f 0.1                 |   377 MB/s |  4189 MB/s |     1406119 | 26.31 |
| lzsse8o 0.1 level 1         |    19 MB/s |  5932 MB/s |      919525 | 17.20 |
| lzsse8o 0.1 level 2         |    15 MB/s |  6215 MB/s |      849715 | 15.90 |
| lzsse8o 0.1 level 4         |    10 MB/s |  6424 MB/s |      793567 | 14.85 |
| lzsse8o 0.1 level 8         |    10 MB/s |  6455 MB/s |      792369 | 14.82 |
| lzsse8o 0.1 level 12        |    10 MB/s |  6455 MB/s |      792369 | 14.82 |
| lzsse8o 0.1 level 16        |    10 MB/s |  6447 MB/s |      792369 | 14.82 |
| lzsse8o 0.1 level 17        |  3.27 MB/s |  6494 MB/s |      784226 | 14.67 |
|                             |            |            |             |       |

The x-ray file is interesting, it favours LZSSE2 on compression ratio and LZSSE8 is significantly faster on decompression speed (which you would expect for this less compressible data):

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      | 15186 MB/s | 15268 MB/s |     8474240 |100.00 |
| lzsse2 0.1 level 1          |    25 MB/s |  1617 MB/s |     7249233 | 85.54 |
| lzsse2 0.1 level 2          |    19 MB/s |  1623 MB/s |     7181547 | 84.75 |
| lzsse2 0.1 level 4          |    15 MB/s |  1637 MB/s |     7036083 | 83.03 |
| lzsse2 0.1 level 8          |    15 MB/s |  1637 MB/s |     7036044 | 83.03 |
| lzsse2 0.1 level 12         |    15 MB/s |  1625 MB/s |     7036044 | 83.03 |
| lzsse2 0.1 level 16         |    15 MB/s |  1624 MB/s |     7036044 | 83.03 |
| lzsse2 0.1 level 17         |    15 MB/s |  1624 MB/s |     7036044 | 83.03 |
| lzsse4 0.1                  |   175 MB/s |  2361 MB/s |     7525821 | 88.81 |
| lzsse8f 0.1                 |   172 MB/s |  3078 MB/s |     7248659 | 85.54 |
| lzsse8o 0.1 level 1         |    27 MB/s |  3020 MB/s |     7183700 | 84.77 |
| lzsse8o 0.1 level 2         |    26 MB/s |  3053 MB/s |     7171372 | 84.63 |
| lzsse8o 0.1 level 4         |    25 MB/s |  3058 MB/s |     7169088 | 84.60 |
| lzsse8o 0.1 level 8         |    26 MB/s |  3052 MB/s |     7169084 | 84.60 |
| lzsse8o 0.1 level 12        |    25 MB/s |  3061 MB/s |     7169084 | 84.60 |
| lzsse8o 0.1 level 16        |    25 MB/s |  3060 MB/s |     7169084 | 84.60 |
| lzsse8o 0.1 level 17        |    25 MB/s |  3051 MB/s |     7169084 | 84.60 |
|                             |            |            |             |       |

The enwik9 text heavy benchmark again favours LZSSE2, but not by much:

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      |  3644 MB/s |  3699 MB/s |  1000000000 |100.00 |
| lzsse2 0.1 level 1          |    18 MB/s |  3424 MB/s |   405899410 | 40.59 |
| lzsse2 0.1 level 2          |    13 MB/s |  3685 MB/s |   372353993 | 37.24 |
| lzsse2 0.1 level 4          |  9.49 MB/s |  3935 MB/s |   340990265 | 34.10 |
| lzsse2 0.1 level 8          |  9.24 MB/s |  3929 MB/s |   340559550 | 34.06 |
| lzsse2 0.1 level 12         |  9.34 MB/s |  3980 MB/s |   340558897 | 34.06 |
| lzsse2 0.1 level 16         |  9.34 MB/s |  3908 MB/s |   340558894 | 34.06 |
| lzsse2 0.1 level 17         |  7.89 MB/s |  3954 MB/s |   340293998 | 34.03 |
| lzsse4 0.1                  |   221 MB/s |  3344 MB/s |   425848263 | 42.58 |
| lzsse8f 0.1                 |   216 MB/s |  3281 MB/s |   427471454 | 42.75 |
| lzsse8o 0.1 level 1         |    15 MB/s |  3689 MB/s |   374183459 | 37.42 |
| lzsse8o 0.1 level 2         |    12 MB/s |  3643 MB/s |   358878280 | 35.89 |
| lzsse8o 0.1 level 4         |    10 MB/s |  3926 MB/s |   347111696 | 34.71 |
| lzsse8o 0.1 level 8         |    10 MB/s |  3905 MB/s |   346907396 | 34.69 |
| lzsse8o 0.1 level 12        |    10 MB/s |  3923 MB/s |   346907021 | 34.69 |
| lzsse8o 0.1 level 16        |    10 MB/s |  3937 MB/s |   346907020 | 34.69 |
| lzsse8o 0.1 level 17        |  9.25 MB/s |  3926 MB/s |   346660075 | 34.67 |
|                             |            |            |             |       |

Finally, this is a new benchmark, the app3 file from [compression ratings](http://www.compressionratings.com/). It was brought to my attention on encode.ru. LZSSE2 does very poorly on this file, while LZSSE8 does significantly better in terms of compression ratio and decompression speed (note, I've included more results here because this wasn't in the previous post):

| Compressor name             | &nbsp;&nbsp;Compression&nbsp;&nbsp;| &nbsp;&nbsp;Decompress.&nbsp;&nbsp;| &nbsp;&nbsp;Compr. size&nbsp;&nbsp; | &nbsp;&nbsp;Ratio&nbsp;&nbsp; |
| ---------------             | -----------| -----------| ----------- | ----- |
| memcpy                      | 11442 MB/s | 11390 MB/s |   100098560 |100.00 |
| blosclz 2015-11-10 level 1  |  1369 MB/s | 10755 MB/s |    99783925 | 99.69 |
| blosclz 2015-11-10 level 3  |   733 MB/s | 10115 MB/s |    98845453 | 98.75 |
| blosclz 2015-11-10 level 6  |   311 MB/s |  2129 MB/s |    72486401 | 72.42 |
| blosclz 2015-11-10 level 9  |   267 MB/s |  1166 MB/s |    62088582 | 62.03 |
| brieflz 1.1.0               |   124 MB/s |   260 MB/s |    56188157 | 56.13 |
| crush 1.0 level 0           |    27 MB/s |   244 MB/s |    52205980 | 52.15 |
| crush 1.0 level 1           |  9.85 MB/s |   256 MB/s |    50786667 | 50.74 |
| crush 1.0 level 2           |  2.35 MB/s |   260 MB/s |    49887073 | 49.84 |
| fastlz 0.1 level 1          |   239 MB/s |   766 MB/s |    63274586 | 63.21 |
| fastlz 0.1 level 2          |   291 MB/s |   752 MB/s |    61682776 | 61.62 |
| lz4 r131                    |   731 MB/s |  3361 MB/s |    61938601 | 61.88 |
| lz4fast r131 level 3        |   891 MB/s |  3567 MB/s |    65030520 | 64.97 |
| lz4fast r131 level 17       |  1494 MB/s |  4653 MB/s |    76451494 | 76.38 |
| lz4hc r131 level 1          |   115 MB/s |  3143 MB/s |    57430191 | 57.37 |
| lz4hc r131 level 4          |    69 MB/s |  3330 MB/s |    54955555 | 54.90 |
| lz4hc r131 level 9          |    44 MB/s |  3367 MB/s |    54423515 | 54.37 |
| lz4hc r131 level 12         |    30 MB/s |  3381 MB/s |    54396882 | 54.34 |
| lz4hc r131 level 16         |    17 MB/s |  3346 MB/s |    54387597 | 54.33 |
| lzf 3.6 level 0             |   294 MB/s |   833 MB/s |    63635905 | 63.57 |
| lzf 3.6 level 1             |   291 MB/s |   840 MB/s |    62477667 | 62.42 |
| lzg 1.0.8 level 1           |    34 MB/s |   619 MB/s |    63535191 | 63.47 |
| lzg 1.0.8 level 4           |    30 MB/s |   633 MB/s |    59258839 | 59.20 |
| lzg 1.0.8 level 6           |    23 MB/s |   641 MB/s |    57202956 | 57.15 |
| lzg 1.0.8 level 8           |    13 MB/s |   659 MB/s |    55031252 | 54.98 |
| lzham 1.0 -d26 level 0      |  9.64 MB/s |   203 MB/s |    47035821 | 46.99 |
| lzham 1.0 -d26 level 1      |  2.42 MB/s |   314 MB/s |    34502612 | 34.47 |
| lzjb 2010                   |   313 MB/s |   574 MB/s |    71511986 | 71.44 |
| lzma 9.38 level 0           |    18 MB/s |    47 MB/s |    48231641 | 48.18 |
| lzma 9.38 level 2           |    15 MB/s |    51 MB/s |    45721294 | 45.68 |
| lzma 9.38 level 4           |  8.74 MB/s |    54 MB/s |    44384338 | 44.34 |
| lzma 9.38 level 5           |  3.79 MB/s |    56 MB/s |    41396555 | 41.36 |
| lzrw 15-Jul-1991 level 1    |   231 MB/s |   580 MB/s |    68040073 | 67.97 |
| lzrw 15-Jul-1991 level 2    |   191 MB/s |   700 MB/s |    67669680 | 67.60 |
| lzrw 15-Jul-1991 level 3    |   329 MB/s |   665 MB/s |    65336925 | 65.27 |
| lzrw 15-Jul-1991 level 4    |   329 MB/s |   488 MB/s |    63955770 | 63.89 |
| lzrw 15-Jul-1991 level 5    |   125 MB/s |   487 MB/s |    61433299 | 61.37 |
| lzsse2 0.1 level 1          |    21 MB/s |  2245 MB/s |    62862916 | 62.80 |
| lzsse2 0.1 level 2          |    18 MB/s |  2320 MB/s |    61190732 | 61.13 |
| lzsse2 0.1 level 4          |    13 MB/s |  2384 MB/s |    59622902 | 59.56 |
| lzsse2 0.1 level 8          |    12 MB/s |  2393 MB/s |    59490753 | 59.43 |
| lzsse2 0.1 level 12         |    12 MB/s |  2394 MB/s |    59490213 | 59.43 |
| lzsse2 0.1 level 16         |    12 MB/s |  2393 MB/s |    59490213 | 59.43 |
| lzsse2 0.1 level 17         |  2.77 MB/s |  2396 MB/s |    59403900 | 59.35 |
| lzsse4 0.1                  |   241 MB/s |  2751 MB/s |    64507471 | 64.44 |
| lzsse8f 0.1                 |   244 MB/s |  3480 MB/s |    62616595 | 62.55 |
| lzsse8o 0.1 level 1         |    20 MB/s |  3846 MB/s |    57314135 | 57.26 |
| lzsse8o 0.1 level 2         |    17 MB/s |  3936 MB/s |    56417687 | 56.36 |
| lzsse8o 0.1 level 4         |    14 MB/s |  4015 MB/s |    55656294 | 55.60 |
| lzsse8o 0.1 level 8         |    13 MB/s |  4033 MB/s |    55563258 | 55.51 |
| lzsse8o 0.1 level 12        |    13 MB/s |  4035 MB/s |    55562882 | 55.51 |
| lzsse8o 0.1 level 16        |    13 MB/s |  4032 MB/s |    55562882 | 55.51 |
| lzsse8o 0.1 level 17        |  2.76 MB/s |  4039 MB/s |    55482271 | 55.43 |
| quicklz 1.5.0 level 1       |   437 MB/s |   487 MB/s |    62035785 | 61.97 |
| quicklz 1.5.0 level 2       |   190 MB/s |   394 MB/s |    59010932 | 58.95 |
| quicklz 1.5.0 level 3       |    50 MB/s |   918 MB/s |    56738552 | 56.68 |
| yalz77 2015-09-19 level 1   |    57 MB/s |   583 MB/s |    58727991 | 58.67 |
| yalz77 2015-09-19 level 4   |    35 MB/s |   577 MB/s |    56893671 | 56.84 |
| yalz77 2015-09-19 level 8   |    24 MB/s |   580 MB/s |    56199900 | 56.14 |
| yalz77 2015-09-19 level 12  |    17 MB/s |   575 MB/s |    55916030 | 55.86 |
| yappy 2014-03-22 level 1    |   120 MB/s |  2722 MB/s |    64009309 | 63.95 |
| yappy 2014-03-22 level 10   |   102 MB/s |  3031 MB/s |    62084911 | 62.02 |
| yappy 2014-03-22 level 100  |    84 MB/s |  3116 MB/s |    61608971 | 61.55 |
| zlib 1.2.8 level 1          |    55 MB/s |   309 MB/s |    52928473 | 52.88 |
| zlib 1.2.8 level 6          |    28 MB/s |   328 MB/s |    49962674 | 49.91 |
| zlib 1.2.8 level 9          |    12 MB/s |   329 MB/s |    49860696 | 49.81 |
| zstd v0.4.1 level 1         |   377 MB/s |   710 MB/s |    53079608 | 53.03 |
| zstd v0.4.1 level 2         |   298 MB/s |   648 MB/s |    51425209 | 51.37 |
| zstd v0.4.1 level 5         |   110 MB/s |   604 MB/s |    48827923 | 48.78 |
| zstd v0.4.1 level 9         |    36 MB/s |   644 MB/s |    46859280 | 46.81 |
| zstd v0.4.1 level 13        |    19 MB/s |   644 MB/s |    46356567 | 46.31 |
| zstd v0.4.1 level 17        |  7.83 MB/s |   643 MB/s |    45814107 | 45.77 |
| zstd v0.4.1 level 20        |  1.48 MB/s |   849 MB/s |    35601905 | 35.57 |
| shrinker 0.1                |   389 MB/s |  1342 MB/s |    58260187 | 58.20 |
| wflz 2015-09-16             |   194 MB/s |  1246 MB/s |    65416104 | 65.35 |
| lzmat 1.01                  |    38 MB/s |   525 MB/s |    52250313 | 52.20 |
|                             |            |            |             |       |

 