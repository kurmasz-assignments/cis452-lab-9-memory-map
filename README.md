CIS 452 Lab 9:  Memory Map
------------------------------------------------------------------------

### Overview

The main purpose of this lab is to investigate the cooperative role played by
the compiler and the operating system in the organization of a process's
logical address space.
It is designed to improve your understanding of memory allocation protocols.
A secondary purpose is to introduce a tool for detecting a common memory error
-- a memory bound violation.

### Activities

The first section of the lab introduces a tool to help you test your program
for memory leaks.

The rest of the lab consists of an open-ended investigation into the memory
mapping of the current EOS system environment
(the `gcc` compiler running under Linux on an Intel processor).

Submit all findings:

- your annotated diagram
    * include a brief explanation of how you found the information,
      possibly referencing the relevant source code
- your source code
- answers to all questions

### Memory Bound Protection

One of the common problems operating systems face is incorrect memory usage by
user programs.
This problem can be hard to detect --
most often, systems will "allow" a process to make memory errors,
provided they do not corrupt memory belonging to the system or to another
process.
Basically, a high-performance language like C/C++ will not take the time to
check every memory access to ensure it is correct as that would incur a
performance hit.
Instead, the operating system will intervene only to protect itself and other
users from a misbehaving program.

This section introduces a tool for performing dynamic memory debugging that
will allow a user to validate a program's use of memory while it is executing.
It is most often used together with a debugger.

#### Buffer Overruns

The scenario induced in Sample Program 1 illustrates a common memory problem:
the buffer overrun.
The problem will be examined using a tool called Electric Fence.
Consider the following sample code:

*Sample Program 1*

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define SIZE 16

int main()
{
    char *data1;

    data1 = malloc (SIZE);
    printf ("Please input username: ");
    scanf ("%s", data1);
    printf ("you entered: %s\n", data1);
    free (data1);
    return 0;
}
```

Although the program appears to run correctly (i.e. it does not abort),
it includes a potentially dangerous error that can be detected by using
**Electric Fence**.

Study the code carefully.
Review the man pages on **Electric Fence** (`man efence`),
then perform the following:

* compile and run the program
    * when prompted, enter the username `username`
    * re-run, entering the username `notarealusername`

* re-compile the program,
  this time using the debug option and linking in the **Electric Fence**
  library
    -   i.e. use `-g` and `-lefence`

* re-run the program
    * when prompted, enter the username `username`
    * re-run, entering the username `notarealusername`

**Electric Fence** places a barrier around a process's allocated memory.
This will cause any memory problems in an executing program to suffer a
segmentation fault.
To find the exact location of the problem,
use a debugger to step through the code to find the offending line.

1. Describe the error precisely (nature of problem, location).  Fix the problem and submit your corrected source code along with a screenshot of the execution. Develop a robust solution that allows the user to enter any size username (within reason) without massively over allocating memory. 

### Memory Management Overview

The second part of this lab examines memory management from the system
point-of-view.
Modern operating systems provide the user with a logical view of memory as an
extremely large, contiguous address space.
This address space is usually partitioned conceptually into segments:
functional partitions of logical memory.
As discussed in class, the operating system,
together with the memory management hardware (the MMU),
transparently maps the logical pages of a process into physical memory frames.

Programs consist fundamentally of code and data.
However, there are several other distinct regions of user-mode logical memory:

- *program text* - this constitutes the machine instructions or program code. 
  It is read-only and of fixed size and initially resides on disk as part of the
  executable (i.e. the `a.out`).
  The size is determined at compile-time and is communicated to the operating
  system via the header of the executable,
  so that it can be loaded into the correct amount of memory when run.
- *initialized data* - this data segment holds persistent objects (i.e.
  globals) that have been initialized with values.
  Since the data object is to be initialized with a value,
  the value must be stored as part of the executable.
- *uninitialized data* - this segment holds static (global) objects that have
  been declared but not initialized.
  The space for these objects is constructed at run-time by the kernel and
  initialized to 0 or NULL.
- *run-time heap space* - this refers to heap space used for dynamic memory
  allocation.
  Heap space fluctuates during execution as memory is obtained via `new()` or
  `malloc()`,
  and released via `delete()` or `free()`.
  See the `brk()` and `sbrk()` system calls for more detail.
- *run-time stack space* - there is a run-time stack associated with each
  executing process. 
  It contains stack frames for process context and includes all automatic
  variables
  (e.g. non-static data objects such as function parameters and local
  variables).
- *shared objects* - external functions and variables loaded at runtime from the C shared
  libraries (`*.so` files).

The compiler partitions the logical view of your program into these respective
regions as it creates the format of the executable.
It also places information regarding the sizes of these regions into the
program header of your executable.
Note that the dynamic regions
(stack, heap, uninitialized data)
are not actually created until run-time.

These regions each have their own specific locations in virtual memory. 
As an example, consider Linux memory management.
The simplified logical address space of an executing process typically looks
like this:

<pl-figure file-name="memory-map-skeleton.gif" directory="clientFilesQuestion"
width="400px"></pl-figure>

The text and data regions are static in size and are created by the operating
system at program load time using the information inserted into the program
header by the compiler.
The dynamic regions are created and managed at run-time in response to function
calls,
system calls and process resource requests.
Memory management hardware and software cooperate to implement the mapping.
For example, using a page table, page #2 of the program code
(in the text region)
might be mapped into frame #`0x400006` of physical memory.

### Memory Mapping Exercise

**Perform the following operations:**

* create an annotated memory map of GNU/Linux/Intel virtual memory organization
    * include **all six** of the segments described above
    * determine the direction of growth of the dynamic segments
    * specify the approximate location of each of the segments for a sample
      program you have written

**Hints:**
* think about, and create, the type of information each segment stores
* it is possible to obtain the logical address of any data object using
  the "address of" operator (`&`).
    - Note: the `%p` format modifier can be used to print addresses (in
      hexadecimal)
* it is possible to obtain the address of a function in the same way
* access system variables that contain pointers to specific memory areas
* creating multiple objects will indicate their direction of growth
* do *not* compile your program with Electric Fence included;
  it changes the location of heap variables

Most of what you need can be found by writing your program in a careful way and
printing the addresses of interest.
If there is anything you are unable to determine this way,
or if you are simply curious,
the following options are available to get more information about the address
space of a process.

Alternative mapping options: 
* memory mapping information is available from the kernel
    * pause your process and look in `/proc/PID`
    * the `maps` file will provide memory mapping information
* some utilities (e.g.  `readelf`) interpret binary headers
  (i.e. they parse an executable).
  You may need to page through them or use `grep` to find what you are
  interested in
* some utilities (e.g. `ldd`) give information about libraries and executables
* the `pmap` command and the `valgrind` utility can be used to determine
  process memory usage

Synthesize all of this information to help create your annotated diagram. Make sure the output 
of your program agrees with the information from other sources, such as `/proc/PID/maps`.

2. Add your diagram to your report.
   Include enough details (and possibly source code)
   so that I know how you found the information.

Upload your writeup with the answers to the numbered questions, code samples,
and screenshots of your running program below.