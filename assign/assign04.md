---
layout: default
title: "Assignment 4: Parallel merge sort"
---

*Note: this is a preliminary/incomplete assignment description. Major details
could change.*

**Due**: Wednesday, Nov 9th by 11 pm Baltimore time

Assignment type: **Pair**, you may work with one partner

## Getting started

Download [csf\_assign04.zip](csf_assign04.zip) and unzip it.

You will modify the code in the `parsort.c` source file. You can compile
the program using the provided `Makefile`.  To run the program,
the invocation is

<div class="highlighter-rouge"><pre>
./parsort <i>filename</i> <i>threshold</i>
</pre></div>

where *filename* is the file containing the data to sort, and *threshold*
is the number of elements below which the program should use a sequential
sort.

## Your task

Your task is to write a program that will sort 64-bit signed integers
(stored in a file in little-endian binary format), using a variation
of [Merge sort](https://en.wikipedia.org/wiki/Merge_sort), *modifying*
the data in the file so that the original data values are in sorted order
from least to greatest.

In addition,

* you will *parallelize* the computation with a *fork/join* style computation
  using child processes, and
* your program will access the file data using memory-mapped file I/O

This might sound complicated! Fortunately, this program can be implemented
quite easily in about 200 lines of C code.

