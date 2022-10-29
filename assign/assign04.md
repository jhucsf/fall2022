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

Let's say that you want to map the contents of a file into memory so you can sort it.
First, you will need to use the [open](https://man7.org/linux/man-pages/man2/open.2.html)
syscall to open the file in read-write mode and get a file descriptor:

```
int fd = open(filename, O_RDWR);
```

TODO: talk about getting file size using fstat system call
Next, `mmap` will need to know how many bytes of data the file has. This can be
accomplished using the [fstat](https://man7.org/linux/man-pages/man3/fstat.3p.html) system
call:

```
struct stat statbuf;
int rc = fstat(fd, &statbuf);
if (rc != 0) {
    // handle fstat error and exit
}
size_t file_size_in_bytes = statbuf.st_size;
```

TODO: mention any details that might be necessary for calling mmap
Once the program knows the size of the file, creating a shared read-write mapping will
allow the program, and all its descendants, to modify the file in-place in memory:

```
int64_t *data = mmap(NULL, file_size_in_bytes, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0)
if (data == MAP_FAILED) {
    // handle mmap error and exit
}
```

Passing in `NULL` for the requested mapping address gives `mmap` complete freedom to
choose any address in memory to map. Since we don't care where the file ends up in memory,
so long as we can access it, this is the correct semantics. Similarly, we want to map the
entire file, so we set the offset to zero.

Note: Don't forget to call [munmap](https://man7.org/linux/man-pages/man2/mmap.2.html) and
[close](https://man7.org/linux/man-pages/man2/close.2.html) before returning from the
topmost process in your program before returning to prevent leaking resources. Note that
closing a file descriptor _does not_ `unmap` the memory; both calls must be used.


### Creating child processes

TODO: talk about using fork, emphasizing the importance of making sure
that they exit (and don't start executing code that should only
be executed by the parent process.)

The `fork()` call can be used to spawn child processes from the current process. The child
will start executing at the point of the fork call, and will share its initial memory
space with the parent. Recall that the fork call will always return the `pid` zero to the
newly started subprocess, and the actual `pid` to the parent process:

```
pid_t pid = fork()
if (pid == -1) {
    // fork failed to start a new process
    // handle the error and exit
} else if (pid == 0) {
    // this is now in the child process
}
// if pid is not 0, we are in the parent process
```

You must make sure that the child branch exits after it has completed its work. Failure to
make the child exit will allow it to continue executing through the parent's code path,
which will lead to memory corruption and other difficult-to-debug behaviors. We highly
recommend that you hand over control to a function and exit the child process immediately
afterwards:

```
if (pid == 0) {
    int retcode = do_child_work();
    exit(retcode);
    // everything past here is now unreachable in the child
}
```

TODO: talk about waiting for child processes to complete using
waitpid, and finding their exit status using WIFEXITED and
WEXITSTATUS.

To pause program execution until a child process has completed, we recommend using the
[waitpid](https://man7.org/linux/man-pages/man3/wait.3p.html) call:

```
int wstatus;
// blocks until the process indentified by pid_to_wait_for completes
pid_t actual_pid = waitpid(pid_to_wait_for, &wstatus, 0);
if (actual_pid == -1) {
    // handle waitpid failure
}
```

The `wstatus` argument provides an opaque handle that can be used with special macros to
query information about the how the subprocess exited. The `WIFEXITED(wstatus)` macro
will evaluate to a true value if the subprocess exited normally, and the
`WEXITSTATUS(wstatus)` macro can be used to retrieve the return code that the subprocess
exited with:

```
if (!WIFEXITED(wstatus)) {
    // subprocess crashed, was interrupted, or did not exit normally
    // handle as error
}
if (WEXITSTATUS(wstatus) != 0) {
    // subprocess returned a non-zero exit code
}
```

Thus, the subprocess and notify its parent if its operation succeeded by returning a return code.

Note: You must wait on every new process you start. This means that every fork call should
have a corresponding `waitpid` call. Failure to due this in a long-running process creates
a "pid leak", and can lead to pid exhaustion and the inability to start any new processes
on the system due to the accumulation of the "zombie" processes (yes this is the technical
term). While the kernel and the `init` process do clean up zombies after the parent exits,
it is a good practice to ensure that you promptly deal with zombie program in your
program. We will be manually checking your code to ensure that you don't leave zombies
around while your program executes.

### Generating test data, running the program

TODO: describe how to create test data using dd and /dev/urandom.
Suggest using /tmp (but emphasize not to leave very large files in
/tmp.)
You can create some random test data using the `dd` command:

```
dd if=/dev/urandom of=/path/to/output/file bs=8 count=number of integers to generate
```

e.g. to generate a file named `test.in` n the /tmp directory of 1000 integers, you can use
the following command:

```
dd if = /dev/urandom of=/tmp/test.in bs=8 count=1000
```

Please be very careful when executing `dd`. It will happily fill up you entire disk if you
make a typo in either the counts or the blocksize. Similarly, double check your output
path before you run `dd`, as it will overwrite anything there without any warning.

We suggest using the `/tmp` directory (the system temporary directory) to create your test
files to prevent accumulating many small test files alongside your assignment. However, if
you do decide to use `/tmp` please note the following points:

* `/tmp` is shared amongst all users of the system, so you should probably create a
    subfolder that will be reasonably unique to you.
* `/tmp` is local to the current machine. E.g. `/tmp` on ugrad1 is independent of `/tmp`
    on ugrad2.
* Since `/tmp` is shared, you **must ensure** that you set file permissions correctly on
  your created directory (`chmod -R 700 /tmp/mydir`). You should also keep your actual
  implementation out of `/tmp`.
* You must not create every large files in `/tmp` and you must ensure that you delete
    everything you leave there _before logging off_.

To check that your sort program works correctly, you can use the `is_sorted` program we have
included in the starer code:

```
# generate the file with 1000 integers
dd if=/dev/urandom of=data.in bs=8 count=1000
# sort the file
./parsort data.in 500
# verify that the file is sorted correctly
./is_sorted data.in
```

If the file is correctly sorted, `is_sorted` will print "Data values are sorted!\n",
otherwise, it will print an informative error message.

TODO: emphasize that when working on a shared system, be careful about
using system resources responsibly.

Remember that you are writing a parallel program that can consume resources at an
exponential rate. If you are working on a shared system (e.g. the ugrad machines), you
must ensure that you test in a responsible manner. If your program appears to be frozen,
or taking an inordinate amount of time, you should immediately terminate it. You should
test your program on small inputs first, before moving on to larger ones to contain the
blast radius of any potential programming mistakes that you might have made. Do not
suspend your program using `ctrl-z`, you must use `ctrl-c` to ensure that the entire
process tree receives an interrupt signal and is terminated. Estimate the number of
processes that your program will attempt to spawn with the given parameters before running
the command. You should never try spawning more than a hundred processes at your highest
limits on a shared system.

TODO: explain how to run experiments, collect timing data.

You can collect timing info for a given command be prefixing it with the time command:

```
time ./parsort test.in 1000
```

This will report you timing information in the following format:

```
real    0m0.010s
user    0m0.002s
sys     0m0.001s
```

You should use the `sys` time reported to eliminate scheduling variance. Since this will
still be sensitive to system load, you should run each experiment multiple times, and
eliminate any clear outliers before including a result in your report. You will need to
tweak the amount of data you test against until you are able to distinguish results
between different threshold values on the same data size.

### Report on speedup

We will be requiring you to submit a brief report on you implementation in your README.txt
file that includes the following information:

* Test results from running your parallel sort program on inputs of varying sizes.
* Test results from running your parallel sort program with a threshold set high enough to
  sort the file sequentially, and subsequent lower thresholds to test speedup as
  parallelism increases.
* Justification on why you obtained the results you did.

If you implemented your program correctly, you should see execution time decrease to an
asymptotic limit as you decrease the threshold to a certain point (increasing
parallelism), then start increasing again as the process creation overhead outweighs the
speedup provided by splitting up the work. If you see no benefit to any level of increased
process-parallelism on a reasonably large dataset, or you see the parallel execution
always take longer, you probably have made an implementation mistake.

### Note on the autograder

Passing the autograder will be a necessary but insufficient condition for full credit.
This means that you may still lose functionality points, even if you pass all of the
autograder tests. Due to the nature of the problem, there will be a significant number of
points up for manual review, so please structure your code accordingly. Some of the things
we may manually verify (bot not limited to) are:

* Ensuring that your implementation is actually parallel
* Ensuring that you did not leave zombies around during execution
* Ensuring that the correct number of children are created for a given threshold and data
  size value.

## Submitting

Edit the `README.txt` file to include the report and summarize each team member's contributions.

You can create a zipfile of your solution using the command `make solution.zip`

Submit your zipfile to **Assignment 4**.
