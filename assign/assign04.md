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

### Fork/join computation

The [fork/join](https://en.wikipedia.org/wiki/Fork%E2%80%93join_model) model
of parallel computation is a technique for parallelizing divide and conquer
algorithms.

The outline of a fork/join computation is the following:

```
if (problem is small enough)
  solve the problem sequentially
else {
  in parallel {
    solve the left half of the problem
    solve the right half of the problem
  }
  combine the solutions to the left/right halves of the problem
}
```

In the case of merge sort, a fork/join approach will look something
like this:

```
if (number of elements is at or below the threshold)
  sort the elements sequentially
else {
  in parallel {
    recursively sort the left half of the sequence
    recursively sort the right half of the sequence
  }
  merge the sorted sequences
}
```

In your program, "sort the elements sequentially" can be delegated to the
[qsort function](https://man7.org/linux/man-pages/man3/qsort.3.html).

Recursively sorting in parallel can be implemented by using
[fork](https://man7.org/linux/man-pages/man2/fork.2.html) two times to create two
child processes, and having each one recursively sort half of
the array.

Merging the elements sequentially can be implemented using
a function call to a `merge` function (which you will need to
implement.)

### Memory-mapped file I/O

The [mmap](https://man7.org/linux/man-pages/man2/mmap.2.html) system call allows
a process to map file data into its address space. If the process
passes the `PROT_READ|PROT_WRITE` options for the *prot* argument and
`MAP_SHARED` option to the *options* argument, then any modifications
the process makes to the memory within the file mapping will be written
back to the actual file.

TODO: talk about opening a file using open system call

TODO: talk about getting file size using fstat system call

TODO: mention any details that might be necessary for calling mmap

### Creating child processes

TODO: talk about using fork, emphasizing the importance of making sure
that they exit (and don't start executing code that should only
be executed by the parent process.)

TODO: talk about waiting for child processes to complete using
waitpid, and finding their exit status using WIFEXITED and
WEXITSTATUS.

### Generating test data, running the program

TODO: describe how to create test data using dd and /dev/urandom.
Suggest using /tmp (but emphasize not to leave very large files in
/tmp.)

TODO: emphasize that when working on a shared system, be careful about
using system resources responsibly.

TODO: explain how to run experiments, collect timing data.
