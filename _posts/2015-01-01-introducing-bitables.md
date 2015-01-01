---
layout: post
title: "Introducing Bitables"
description: ""
category: "Data Structures"
tags: [Storage, "Data Structures", "Key/value"]
---
{% include JB/setup %}
# An Introduction to Bitables and the Bitable Library #

Bitables are a storage oriented data structure designed to exploit the benefits of OS optimization to provide a low overhead mechanism for fast range queries and ordered scans on an immutable key-value data-set. The immutability means they can be queried from multiple threads without locking. They're also designed to be written out fast, using sequential writes, even with large amounts of keys and data. You can think of them as a hybrid between an SSTable and a b+ tree (although, currently they do not allow duplicate keys). 

You can call this a thought experiment, because as I can imagine lots of different use cases for bitables, I have not had time yet to adequately explore them. However, preliminary benchmarks indicate that writing out a bitable is fast enough that they can be used for relatively dynamic structures, not just static ones. Some potential uses would be:

 - A lucene like inverted index (with segments and merging), when combined with something like [roaring bitmaps](http://roaringbitmap.org/).
 - A de-amortized log structured COLA like structure (with bitables at deeper levels and the higher levels stored in memory).
 - A static index where range or prefix searches are required.

Merging bitables is quite a quick operation, so they are suitable many places large SSTables would be used.

The basic premise is that bitables are built by adding keys/value pairs in key sorted order (like an SSTable), but they progressively build up a hierarchical index during this process (using very little in the way of extra resources to do so). Note that the library assumes you appending keys in an already sorted order. Writes of the hierarchical index are amortized across many keys and the operating system is left to buffer them up (the index being substantially smaller than the main data) to do as a batch. 

There is a bitable library implementation, written in C (although, the included example is C++). This library serves as a relatively simple reference implementation, although it is designed to be usable for real life use. The library has both posix and win32 implementations, although it has only currently been tested and built on Windows and Linux (ubuntu). I don't currently claim that the library is production ready, as it hasn't been used fully in anger yet.

Here is an example of using the library to search for a key in a bitable, then scan forwards, reading 100 keys and values:

    BitableCursor cursor;
    BitableResult searchResult = bitable_find( &cursor, table, &searchKey, BFO_LOWER );
    
    for ( int where = 0; 
          searchResult == BR_SUCCESS && where < 100; 
          searchResult = bitable_next( &cursor, table ), ++where )  
    {
		BitableValue key;
		BitableValue value;

		bitable_key_value_pair( &cursor, &key, &value );

		/* Do something here with the key and value */   
    }

## How does it work? ##
Bitables consist of multiple levels:

 - ***The leaf level***, which is the main ordered store for keys and small values.
 - ***The branch level(s)***, which are a hierarchical sorted index into the leaf levels.
 - ***The large value store***, which stores values too large to fit in the leaf level, in sorted order.

Currently each level is stored in a separate file. When building the bitable, each file is only ever appended to (except when we write a final header page on completion), so writes to the individual files are sequential. The operating system is left to actually write these files to disk as it chooses.

The leaf and branch levels are organised into pages. Within each page, there is a sorted set of keys (and values for leaf pages), which can be searched using a binary search. 

Building the bitable uses an algorithm similar to a bulk insertion into a b+ tree. Keys and small values are added to a leaf page (buffered in memory), as they are appended and when the page is full, the buffer is appended to the file and the next page begun. When a new page begun, the first key is added to the level above it. New levels are created when the highest current level grows to more than one page. 

Similar to a b+ tree, the first key in each branch level page is implicit and not stored. Unlike a b+ tree, branch level pages do not have a child pointer for every key, but only to the first child page. Because the pages in the level below are in order, we can work out the index to the page based on the result of the binary search. 

Because we don't have to worry about future inserts or page splitting, every page is as full as possible given the ordering, which should lead to the minimum possible number of levels for the data (within constraints). This keeps the number of pages needing to be touched for any one query at a minimum, reducing I/O, page faults, TLB misses and increasing the efficiency of the page cache. 

The implementation of bitables uses a trick that many b+ tree implementations use and has offsets to keys (and possibly values) at the front of each page and key/value data at the back. This allows each page to be built up quickly, growing inwards from both the back and the front until it is full. However, it has some disadvantages for processor cache access patterns for searching (sorted order binary searches are not very good for processor cache access patterns). In future, for search heavy workloads, it might be reasonable to make branch levels use an internal Van Emde Boas layout. 

## The Implementation ##

The implementation is available [on GitHub](https://github.com/ConorStokes/bitable) and the documentation is available [here](/bitabledocs). A premake4 build file is included (and the Windows premake4 binary I used to build it). On Ubuntu, I built it with the premake4 package (from apt-get) and Gcc. Let me know if there are any issues, or any other platforms you have tried it on.

The reference implementation relies on the operating system where possible (memory mapped files, caching) to do the heavy lifting for things like caching and ordering writes in a smart way. It is a bit naive in that the buffers it uses for writing out bitables are only one page in size (per level), so the number of system calls is probably a reasonable amount higher than it needs to be. 

Reading of the bitable uses a zero copy memory mapping system, where keys and values are mapped directly from the file (although, the mapping should use read only page protection, so it shouldn't be possible to hose the data). Power of 2 alignments are supported for keys and values, specified when initializing the table for creation, so you can cast and access data structures in place. Currently we do not support omitting key or value sizes for fixed size keys/values (but this seems like a good future optimisation). 

Apart from initializing and opening the bitable, for read operations a few rules are followed:

 - No heap allocation is done (although, memory may be allocated into the page cache for demand caching by the operating system).
 - Cursors and statistics structures may be allocated on the stack.
 - No locking is required (including internally).
 - No system calls made (memory mapped IO).
 - There is no mutation of the bitable data structures/no reliance on internal shared mutable state.
 - All the code is re-entrant and there is no reliance on statics or global mutable state.

In the reference library, the bitable write completion process is atomic (the table won't be readable unless it has been fully written completely). It also contains durable and non durable modes for finishing the bitable.  The durable mode completes writes and flushes (including through the on-disk cache) in a particular order to make sure the bitable is completely on storage (preserving the atomiticity guarantee), the non durable mode does the writes in order but does not perform the flushing (for use cases where a bitable is only used within the current power cycle).

The implementation has been designed so that the reading and writing parts of the bitable code are not dependent on each other (although, they are dependent on some small common parts), even though they are currently built into the library together. Although, the library is small enough (it can fit in the instruction cache entirely) that you don't really need to worry about this unless you're really trying to make your code super compact.

The implementation also uses UTF-8 for all file paths (including on Windows). 

Note that this is a side project for me and has been done in very limited time, so there is plenty of room for improvement, extra work and further rigor to be applied. Also note, the files produced by the bitable system use platform endianness etc, so they may not be portable between platforms.

## Does it actually perform decently? ##

All these benchmarks were performed on my Windows 8.1 64bit machine with 32GB of RAM and a Core i7 4790 and a Samsung 850 Pro 512GB SSD. All tests use 4KB pages and 8 byte key and value alignment. Note that these benchmarks are definitely not the final say on performance and I haven't had time to perform adequate bench-marking for multi-threaded reads, 

Our test dataset is using 129,672,136 24 byte keys (a latitude and longitude pair of 2 doubles, as well as a 64bit unix millisecond time stamp), matching up to an 8 byte value . It's 3.86 GiB, packed end to end.

Reading the data set from a warm memory mapped file and creating the bitable takes (averaged over 4 runs) 11.01 seconds in the non durable mode and 13.08 seconds in the durable mode. The leaf level is 4.84 GiB and the 3 branch levels are 34 MiB, 240 Kib and 4 Kib respectively. 

From this dataset, we then selected keys 2,026,127 keys evenly distributed across the dataset, then randomly sorted them. We then used these for queries point queries (on a single thread), which on a cold read of the bitable averaged 7.51 seconds (averaged over 4 runs again) and on warm runs 1.81 seconds (keys and values were read and verified). That's well over a million random key point queries per second (ordered) on a single thread. 