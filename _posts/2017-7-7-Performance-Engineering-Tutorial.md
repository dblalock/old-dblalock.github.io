
<!-- ^ Note that above quotes the whole section of Knuth's paper, which is actually a really good + short commentary about when + what to optimize -->

If you do machine learning, you're probably familiar with the pain of waiting a long time for your code to run. This post is about making your code faster.

Specifically, I'm going to talk about performance engineering on CPUs with an emphasis on ideas most relevant for machine learning and data mining. GPUs are of course also important, but I won't address them since:

 1. If I did, this post would be too long
 2. If you use TensorFlow, Torch, etc (which you hopefully do), most of the details relevant for performance tuning are beyond your control anyway

<!-- If you're not in the mood for reading, I would at least recommend scrolling to the bottom and checking out some of the links to. -->

<!-- This is just a short intro about some of the basic concepts with an emphasis on programming for CPUs. -->

For brevity and readability, this post contains many simplifications. For a more exhaustive treatment, see the slides for MIT's excellent [performance engineering class](https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-172-performance-engineering-of-software-systems-fall-2010/).

## When and What to Optimize

<!-- No post about performance engineering would be complete without noting that, m -->
Most of the time, you shouldn't be optimizing for performance. And when you actually do have a performance problem, step 1 isn't making everything faster---it's figuring out what part of your code is slow. Only once you know what the bottlenecks are should you spend time optimizing. See this discussion of the classic maxim, ["Premature optimization is the root of all evil"](https://softwareengineering.stackexchange.com/a/215671).

If you're coding in C/C++, your IDE probably has good tools for performance monitoring. If you're using Python, [LineProfiler](https://github.com/rkern/line_profiler) can be invaluable. If you're using MATLAB, stop (although I have to admit the built-in profiler is great).

## Computer Architecture in Three Paragraphs

Performance engineering is all about using the hardware more effectively. Unfortunately, if you come from a non-EECS background, you might know about as much about how computers work as I know about pig farming. Here's a quick overview.<sup>[1](#overview)</sup>

Computers have a few basic operations they know how to perform---adding two numbers together, loading in data from memory, computing the logical OR of bits, etc. These operations are done in a section of the chip called the Arithmetic Logic Unit ([ALU](https://en.wikipedia.org/wiki/Arithmetic_logic_unit)). The basic loop the computer carries out is:

 1. Load data from memory into the ALU
 2. Do some operations to that data
 3. Store the results back in memory

For code to be faster, the computer has to spend less time doing one or more of these steps. <!--  Steps 1 and 3 use the memory hierarchy (described below), while step 2 uses the computer's Arithmetic Logic Unit ([ALU](https://en.wikipedia.org/wiki/Arithmetic_logic_unit)), which has the logic for carrying out the various operations. -->

## Memory Hierarchy

As you might have noticed, 2/3 of the above steps involve memory. Consequently, the processor does everything it can to make the memory fast. There are a lot of low-level details that go into making it fast in general, but the trick that matters most for programmers is *caching*.

Caching is when the processor stores data you're likely to access in a special segment of memory called the *cache*. This memory uses faster hardware, but has to be smaller because:

 1. It requires more chip area per bit stored.
 2. The bigger it is, the farther it is (on average) from the computation logic. This matters because electricity doesn't conduct as fast as we would like.

Modern processors don't just have one cache. They have several, each offering a different size/speed tradeoff. These tradeoffs, along with the relevant statistics for other storage media, are shown in Table 1. Registers, described in the first row, are the temporary storage that the ALU operates on directly.

<!-- Here are some (very approximate) statistics about the memory hierarchy on [Haswell](http://www.7-cpu.com/cpu/Haswell.html)). -->

| Level         | Size      | Latency       | Throughput    |
|:--------------|----------:|--------------:|:--------------|
| Register      | 100B      | None          | N/A                           |
| L1 Cache      | 64KB      | 4 cycles      | 2 64b loads/cycle             |
| L2 Cache      | 256KB     | 10 cycles     | 2 cycle read\*, 5 cycle write  |
| L3 Cache      | 10MB      | 40 cycles     | 5 cycle read\*, 10 cycle write |
| Main Memory   | 10GB      | 100 ns        | 7.5 GB/s                      |
| SSD           | 1TB       | 100 us        | 500MB/s                       |
| Disk          | 1TB       | 10 ms         | 100MB/s                       |

\* "Read" is defined as "load into L1 cache"

A few notes:

 - These numbers are approximate and vary from chip to chip
 - The above values have been rounded/approximated significantly for ease of recollection.
 - The latencies and throughputs are for random access; memory will be about 2x faster if read sequentially, and disk will be *much* faster.
 - The main memory, SSD, and disk are mostly independent of the processor, so the above are ballpark figures for comparison.
 - The L1 cache is often split so that half is reserved for storing code, and half for storing "data", defined as anything that isn't the code. This means that it effectively stores at most 32KB of parameters, variables, etc.

When the processor needs to load data, it first checks whether it's in the L1 cache; if not, then it checks the L2 cache; if it's not there, it checks the L3 cache, and so on. Finding the data in any cache is called a *cache hit*, and failing to find it is called a *cache miss*. Finding or failing to find it in a particular cache level N is called an *LN cache {hit,miss}* (e.g., "L1 cache hit", "L2 cache miss").

You can't control what data resides in which level of cache directly, but the processor uses a few simple heuristics that generally work well. They're also simple enough that we can structure our code to help them work even better.

The main heuristic is that, when you access data, it probably gets loaded into L1 or L2 cache, and when data gets kicked out of a given cache level ("evicted") to make room for other data, it probably gets moved to the next level down. This means that you can keep much of your data in a given cache level by only operating on an amount of data that fits in that level. E.g., you could operate on <32KB of data to keep most of it in L1 cache.

Other heuristics are discussed below.

### Cache Lines

Part of the challenge that caching hardware faces is keeping track of what data is in which cache (if any). E.g., in a 256KB cache, there are 256,000 individual bytes it has to know are in there at a given time.

To make this easier, the CPU doesn't allow individual bytes to be cached---it instead operates on *cache lines*, which are blocks of 64 adjacent bytes. Entire cache lines are moved into and out of cache levels as necessary.

Grouping bytes is also a heuristic to improve performance, thanks to [locality of reference](https://en.wikipedia.org/wiki/Locality_of_reference)---i.e., if your program just accessed byte 974, it's likely to access byte 975 (or other bytes near there) in the immediate future. Consequently, by loading all the bytes around byte 974 when you ask for 974, the processor can probably avoid some cache misses.

### Prefetching

Another important heuristic the processor uses is *prefetching*, which can roughly be stated as:

    "If the program just accessed cache lines (i, i+1, i+2)
    in that order, load (i+3) into cache immediately"

The details of how many successive accesses are required and how far ahead the prefetching happens will vary, but the point is that the processor will notice when you scan through data in memory order and will be able to avoid cache misses when this happens.

It can also notice when there are multiple sequences of accesses at once; e.g., if you have code that looks like:

```C
for (int i = 0; i < 100*1000*1000; i++) {
    c[i] = a[i] + b[i];
}
```

The processor can prefetch the memory for all three arrays `a`, `b`, and `c`. Note that this only works for a few sequences at a time (often less than 7). This means that if we needed four more arrays `d`, `e`, `f`, and `g` for this computation, we would probably be better off splitting it into two loops.

If you're doing *extremely* low-level programming and you feel like suffering, you can manually tell the processor to prefetch using builtins such as GCC's [`__builtin_prefetch()`](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html). I've never gotten a performance boost from this, but in theory it's possible.

<!--
A couple notes:

 - The processor can only identify a few (often less than 6) sequences of addresses at once.
 - Can explicitly prefetch, but you almost never want to do this

 --><!-- Note that, at least on recent Intel chips, prefetching doesn't happen across page boundaries.  -->

### Storage Order

2D arrays can be stored in row-major ('C') or column major ('F'/'Fortran') order. The former means that elements within the same row are contiguous in memory, while the latter means that elements in the same column are contiguous in memory. For more than 2D, row-major means that later dimension are closer together (with the final dimension contiguous), while column-major means you're screwed if you need to reason about this.<sup>[2](#colmajor)</sup>

Storage order matters because it affects whether we get the benefits of prefetching and cache lines loading nearby elements. <!-- If we access non-contiguous elements (that are far enough apart that they don't fit in the same cache line) -->

If you iterate through array elements that aren't contiguous, you'll suffer two penalties:

 1. Each cache line will only store one array element you care about. As a result, you'll effectively have a much smaller cache.
 2. You're unlikely to trigger prefetching.

In the case of huge arrays, traversing non-contiguous elements can even result in [cache thrashing](https://en.wikipedia.org/wiki/Thrashing_(computer_science))---i.e., repeatedly cache missing because new data kicks the old data out.

Fortunately, any sane array library will take storage order into account when carrying out its functions, so you typically don't have to worry about this. The two most common cases where it might matter are

 1. Slicing an array. If you want to examine a subset of the rows, it's probably better if the array is in row-major order. Vice-versa for a subset of the columns.

 2. Iterating through array elements in a compiled language. You really want to scan through the elements in memory order. E.g.:

```Cpp
    Matrix<float, RowMajor>(10000, 1000) A;  // row-major 2D array

    // this way is good
    for (size_t i = 0; i < A.rows(); i++) {
        for (size_t j = 0; j < A.cols(); j++) {
            doStuff(A(i, j));
        }
    }
    // reversed the loops--this way is bad
    for (size_t j = 0; j < A.cols(); j++) {
        for (size_t i = 0; i < A.rows(); i++) {
            doStuff(A(i, j));
        }
    }
```

Numpy uses row-major order by default, while Fortran, MATLAB, Julia, R, and TensorFlow<sup>[3](#tf)</sup> use column-major. In Python, you can specify how to store a matrix using the `order` parameter of the [`np.array` function](https://docs.scipy.org/doc/numpy-1.13.0/reference/generated/numpy.array.html), or with the [`np.ascontiguousarray()`](https://docs.scipy.org/doc/numpy-1.13.0/reference/generated/numpy.ascontiguousarray.html) and [`np.asfortranarray()`](https://docs.scipy.org/doc/numpy-1.13.0/reference/generated/numpy.asfortranarray.html) functions.

<!-- If you're slicing an array -->

<!-- Let's say you have a 1e6 x 256 matrix, and you want to compute the mean of the rows. In row-major order, the CPU can look at 256 elements in a row, which all fit in L1 cache and will be prefetched, and compute the sum. In column-major order, each element in the sum will be 1 million elements (4MB) apart. If it tries to compute across each row, it will cache miss repeatedly and go incredibly slowly. What it will actually do (if your library is smart), is just have a 1e6-element column of running sums and scan down each column of the data, incrementing the corresponding elements of the sums. -->

### Loop Tiling

One way to keep your data in cache is to split computations on large blocks of data into several computations on smaller chunks of data. E.g.:

```python
A = -log(abs(B + C))
```

could be replaced with something like:

```python
num_chunks = 10
chunk_size = len(A) / num_chunks
for i in range(num_chunks):
    start, end = i * chunk_size, (i + 1) * chunk_size
    A[start:end] = -log(abs(B[start:end] + C[start:end]))
```

This will result in only loading `chunk_size` rows of each array at once, and then doing the negation, log, absolute value, and addition operations to each chunk in turn. Ideally, the chunks will fit in cache, even though the original arrays wouldn't have. This is unlikely to work as written since the intermediate results (e.g. `B[start:end] + C[start:end]`) will be allocated in new memory, but if you wrote these intermediate results to preallocated storage (see below), this wouldn't be a problem.

In practice, loop tiling is hard to get a performance increase from, since prefetching usually does an extremely good job when operating on large arrays. It's more likely to help when writing [lower-level code](https://en.wikipedia.org/wiki/Loop_nest_optimization) that can use it to keep everything in L1 cache.

### Subtree Clustering / Block Storage

If you're implementing a data structure in a low-level language, you want to ensure that elements are stored contiguously to the greatest extent possible. For linked lists or search trees, this means storing pointers to blocks of elements, not individual elements (see [unrolled linked lists](https://en.wikipedia.org/wiki/Unrolled_linked_list) and [B-Trees](https://en.wikipedia.org/wiki/B-tree)). For trees that need smaller numbers of children, you might want to use [subtree clustering](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/12/ccds.pdf) (see top of page 6).

<!-- ### Key Takeaways -->

## Memory Allocation

When your program starts, it only has access to a little bit of memory, provided by the operating system. To create new objects, load data, etc, it has to ask the operating system for more. In C/C++, you do this using `malloc/calloc`. In a higher-level language, you generally just create objects and let the language make the underlying `malloc/calloc` calls for you.

The overhead of allocating memory isn't terrible, but it's often worth avoiding in critical sections of code. According to [some benchmarks](https://randomascii.wordpress.com/2014/12/10/hidden-costs-of-memory-allocation/), allocating N bytes is about twice as expensive as copying N bytes.

### Reusing Memory

Consider the following operation:
```python
A = -log(abs(B + C))
```

It appears that this creates one array, `A`. In fact, it is likely to create 4 arrays to hold the all the incremental values needed to get to A. I.e., it is probably interpreted as:

```python
temp0 = B + C
temp1 = abs(temp0)
temp2 = log(temp1)
A = -temp2
```

To avoid these extra memory allocations, you can often give functions an "output" argument that tells them where to store their results. In Python, for example, the above can become:

```python
A = np.empty(B.shape)  # allocate storage once
np.add(B, C, out=A)
np.absolute(A, out=A)
np.log(A, out=A)
np.negative(A, out=A)
```

This is less concise, but can be worth it if you find that allocation of temporary arrays is slowing your code down. Note that good C++ array libraries bypass this problem using [expression templates](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Expression-template).<sup>[4](#exprTemplates)</sup>

### Small-object Allocators

One way to reduce the cost of memory allocation is to allocate one large block of it and then hand out pieces of that block yourself. If the pieces can be of all different sizes, it's unlikely you'll be able to do this any more efficiently than just calling into the operating system normally. However, if many of the pieces are small and/or about the same size (probably because they are instances of the same class), it can be easy to keep track of them, and you can manage their storage with far less overhead than the operating system. This is the idea behind writing a [small-object allocator](https://stackoverflow.com/questions/25069570/very-fast-object-allocator-for-small-object-of-same-size).

## Pipelining and Superscalar Execution

Processors have the ability to do several different kinds of operations---adds, subtracts, loads from memory, logical ORs, and dozens of others. Yet each instruction only uses one of these operations. This means that the vast majority of the computation hardware sits idle at any given time.

Or does it?

It turns out that modern CPUs are intelligent about looking ahead at the next few instructions and, whenever possible, executing more than one of them at a time when they use different operations. This is called [superscalar execution](https://en.wikipedia.org/wiki/Superscalar_processor). More concretely, the processor has 5-10 "execution ports", each of which handles a certain set of operations. In the best case, all the ports are always busy, but this is never achieved in practice.

Another trick the processor uses is [pipelining](https://en.wikipedia.org/wiki/Instruction_pipelining). The idea here is to split each operation into multiple sub-operations, and work on sub-operations for different operations in parallel.

If you've ever been to Chipotle, you already understand pipelining. They decompose the process of making a burrito into a "pipeline" of steps:

 1. Getting the tortilla or bowl
 2. Putting rice and meat on it
 3. Putting other ingredients on it
 4. Having you pay

Because they split it into different steps, each employee can just do one step and then pass it on to the next person. To handle high demand, they can have many employees shoulder-to-shoulder cranking out burritos. Replace employees with logic circuits and burritos with operations, and you have CPU pipelining.

<!-- The difference between this and superscalar execution is that the former allows parallel execution of different kinds of operations, while pipelining allows parallel execution of sequential operations -->

### Conditional Branches

Continuing the Chipotle analogy, suppose you have the following situation. A group of people walk in and all get in line. The first one orders a chicken burrito, but his friends say, "if he likes the chicken burrito, we want that; otherwise, we want a veggie bowl."

This is a problem. Why? Because we can't start working on their orders until the first order is done. This means that most of the employees will just be standing around until the first order finishes, instead of assembling pieces of orders. This is called a pipeline "stall".

In code, this situation happens whenever you have an `if` statement (the formal term for which is "conditional branch"). The processor doesn't know whether to execute the `if` block or not until the results of the conditional are known. Consequently, it doesn't get to leverage its pipeline. <!-- It's therefore essential to remove such unpredictability from critical sections of code. -->

Or so it seems.

But consider the following twist to the Chipotle scenario. What if the employees knew that almost everyone likes their chicken burritos? In this case, they could just assume that the first person would like it, and go ahead and start making everyone else's order immediately. They might be wrong, but in the likely scenario that they weren't, they wouldn't have spent any time idle. And if they were wrong, it might cost them some ingredients, but they wouldn't have lost any more time than they would have by waiting.

CPUs employ this exact strategy in the form of "branch prediction." Whenever they encounter an `if` statement, they guess whether it will be true or not and start executing the corresponding instructions. In other words, what hurts you isn't having conditional branches---it's having conditional branches that aren't predictable. <!-- If their predictions were perfect, they would never need to stall the pipeline. -->

What makes a branch predictable? CPUs predict branches in two ways:

 1. Heuristics. CPUs assume that what's in the `if` block will happen, and what's in the `else` block, if one exists, won't happen.
 2. Once they've seen a particular `if` statement, they remember what happened the last few times they encountered it and predict the same thing will happen this time.<sup>[5](#branchPrediction)</sup> This is by far the more common case.

So the takeaway is that, it's not terrible to have an if statements in critical sections of code as long as its condition will tend to be the same many times in a row.

### Loops

A special case of the conditional branch is the loop. Programming languages usually hide this, but loops boil down to checking whether the termination condition is met at the end of the loop body and going back to the top of the body if not. I.e., they're all secretly `while` loops.

You can't usually avoid having a loop, but, fortunately, they tend to be inexpensive for two reasons:

 1. Branch prediction for loops is extremely good. Just assuming that the loop happens again is correct (n-1) out of n times for an n-iteration loop. Furthermore, Intel chips (and likely others) can even avoid being wrong at the very end for n <= 64, if n is known when the loop begins.
 2. Loops with small, constant numbers of iterations are often "unrolled" by the compiler. I.e., it removes the loop and just pastes in the loop body the appropriate number of times. This has the added benefit of removing the conditional branch logic entirely, not just predicting its outcome correctly.

Another consequence of point 2 is that loop tiling with small, fixed-size blocks can often improve performance when using a compiled language. E.g.:

```C
int a[] = some_array();
int length = length_of_a();
const int chunk_size = 4;
int num_chunks = length / chunk_size;
for (int c = 0; c < num_chunks; c++) {
    for (int i = 0; i < chunk_size; i++) {
        a[c * chunk_size + i] += 42;
    }
}
```
Without loop unrolling, the inner loop really says:
```C
    loop_start:
        a[c * chunk_size + i] += 42;
        i++;
        if (i < chunk_size) {
            goto loop_start;
        }
```
With loop unrolling, it says:
```C
    a[c * chunk_size + 0] += 42;
    a[c * chunk_size + 1] += 42;
    a[c * chunk_size + 2] += 42;
    a[c * chunk_size + 3] += 42;
```
which has eliminated all of the loop overhead and should run much faster.

Note that all of this only applies in a compiled language. In a scripting language, evaluating conditionals is more a matter of running code in the interpreter than stalling pipelines. Much the same holds for basically everything else in the section.

<!-- Instead of being able to have employees work on each of their orders in parallel, they now have -->

<!-- When there's a long sequence of predictable instructions---for example, when element-wise adding two long arrays---the processor knows what instructions are coming next, and can "fill" the pipeline. -->

<!-- When the sequence of operations to be performed is fixed, the processor can look ahead at a few dozen of them, intelligently dispatch them to different ports, and -->

<!--  - Backward jumps taken, forward jumps not taken
     - Predicts that loops will execute repeatedly, not just once
     - Predicts the `if` block is more probable
 - Exact branch prediction when number of iterations <= 64 and known at start of loop -->


### Store-to-load Forwarding

One final note about pipelining is that it sometimes lets you avoid touching even the cache when operating on the same data repeatedly. This is done through [store-to-load forwarding](https://en.wikipedia.org/wiki/Memory_disambiguation#Store_to_load_forwarding), which basically means that output from one operation in the pipeline can be fed directly into another operation earlier in the pipeline. <!-- This suggests that putting sequences of operations on the same data nearby could improve -->

### Compiler Intelligence and Front-End Lookahead

Many modern processors will rearrange the sequence of instructions to allow better scheduling in the pipelining. Compilers will also rearrange your instructions to avoid unnecessary code execution and help get them in a decent order for the processor. As a result, it's rarely worth it to think about how you order your lines of code. For example, it's fine to do something like:

```C
M = 20;
for (int i = 0; i < N; i++) {
    int constant = M * M;
    a[i] = constant + i;
}
```
You might worry that it will call `M * M` N times, but any sane compiler will pull that line out of the loop. The exception would be if the constant is determined based on a function call and the compiler doesn't know whether calling the function will have [side effects](https://en.wikipedia.org/wiki/Side_effect_(computer_science)).

<!-- ### Key Takeaways -->


## SIMD Instructions

If you've written code in a scripting language like Python or R, you've probably heard about the importance of using "vectorized" operations instead of loops. <!-- This is good advice. It lets you run optimized C, Fortran, or assembly code instead of code in the scripting language. -->

Part of why this lower-level code is faster is that it avoids the overhead of the scripting language's interpreter and abstractions (e.g., Python `int`s and `float`s are [reference-counted](https://en.wikipedia.org/wiki/Reference_counting) `PyObj` pointers, not raw numbers).

The other part is that it can use more efficient sequences of operations to do a given computation. Some of this is generic compiler optimizations such as [common sub-expression elimination](https://en.wikipedia.org/wiki/Common_subexpression_elimination), but a good deal of it in scientific computing is the use of vector instructions.

Vector instructions (often called [SIMD](https://en.wikipedia.org/wiki/SIMD) instructions) are special CPU operations designed for number crunching. They carry out the same operation on groups of adjacent numbers simultaneously. E.g., using x86 [AVX2 instructions](https://software.intel.com/sites/landingpage/IntrinsicsGuide/#techs=AVX,AVX2), one can replace:
```C
for (int i = 0; i < 8; i++) {
    a[i] = b[i] + c[i];
}
```
with
```C
__m256 b_vect = _mm256_loadu_ps(b); // load 8 floats from b
__m256 c_vect = _mm256_loadu_ps(c); // load 8 floats from c
__m256 sums = _mm256_add_ps(b_vect, c_vect); // sum the floats
_mm256_store_ps(a, sums); // store the results
```
This replaces 16 loads, 8 additions, and 8 stores with 2 loads, 1 addition, and 1 store, offering a huge speedup. In case you're curious, the `m256` means 256-bit (the width of 8 floats) and the `ps` means "packed single-precision".

You likely won't be invoking SIMD instructions directly, so here are tips for making sure that your code can leverage them:

 1. Set array dimensions to multiples of 32 bytes, which corresponds to 8 floats or 4 doubles. SIMD instructions usually operate on either 16B or 32B of data at once, so this ensures that operations can be done entirely using SIMD instructions.
 2. Store your data contiguously. SIMD instructions need the values they operate on to be adjacent in memory.
 3. If using a compiled language, try to have use arrays of sizes that are compile-time constants. This makes it more likely that the compiler will use SIMD instructions to operate on them.

<!-- ### What kinds of instructions are there? -->

<!-- ### Storage Layout (order and contiguity) -->

<!-- Just say that getting stuff contiguous is important. -->

<!-- My experience with rowmajor vs colmajor matrix-vector multiplies -->

<!-- ### Key Takeaways -->


## Threads, Hyperthreads, and Coroutines

<!-- Many scientific computing  -->

<!-- Threading can add a lot of overhead (and distributing across a cluster is even worse). -->

A tutorial on concurrency is beyond the scope of this post, so we're just going to go over some basics to give you a feel for what threads, etc, can and can't do for you. The main lessons are that:

 1. There are often [fundamental limits](https://en.wikipedia.org/wiki/Amdahl%27s_law) to what parallelism can buy you.
 2. It often takes [an *enormous* amount of data](https://www.usenix.org/system/files/conference/hotos15/hotos15-paper-mcsherry.pdf) for multithreading (let alone using multiple machines) to be worth the overhead. <!-- If you read anything about threading, read this link. -->
 3. Unless your problem is [embarrassingly parallel](https://en.wikipedia.org/wiki/Embarrassingly_parallel) and/or you have excellent abstractions, writing concurrent code will probably cost you a great deal of time debugging.

### Definitions

[Core](https://en.wikipedia.org/wiki/Multi-core_processor). You've probably heard of computers having, say, a "four-core" processor. Each core is basically it's own processor, except that the cores share the same memory (and sometimes L3 cache).

[Threads](https://en.wikipedia.org/wiki/Thread_(computing)) are a construct in the operating system that represents a sequence of instructions to be executed. Each core has support for running one thread at a time, in the form of a [hardware thread](https://en.wikipedia.org/wiki/Multithreading_(computer_architecture)). The operating system can have many "OS threads" that take turns using the one hardware thread. Starting a new OS thread is an expensive operation, so it's often better to use a [thread pool](https://en.wikipedia.org/wiki/Thread_pool) rather than start up new threads for new tasks.

[Hyperthreads]((https://en.wikipedia.org/wiki/Hyper-threading)). Remember how I just said each core has one hardware thread? I lied. Sort of. Cores on modern processors often have two hyperthreads instead of one regular hardware thread. The idea of hyperthreads is to feed two sequences of instructions to the same ALU. If neither sequence would be able to utilize half the ALU's throughput (which is common since waiting for memory often takes most of the time; see above), this lets the hardware effectively execute two threads at once. Since the low utilization condition doesn't necessarily hold, having a hyperthread is not as good as having a full core to yourself. This comes up when you use a cloud computing service---one EC2 "vCPU", for example, is just one hyperthread, not one full core.

[Coroutines](https://en.wikipedia.org/wiki/Coroutine), also called "lightweight threads," are thread-like objects implemented in software on top of OS threads. They have far less overhead than OS threads (e.g., you can switch between them in <1000 cycles), and so can substitute for using a thread pool. If you're familiar with Go, its Goroutines are basically coroutines. If you're not using Go or doing systems programming, you probably don't need to think about Coroutines.

## Calling C/C++ code

The best way to call low-level code through a high-level langauge is by using a well-maintained library, such as Numpy. Sometimes, though, there's a critical section of code that can't be implemented efficiently enough using these libraries. At other times, you may have low-level code of your own and want to create an easy interface in a higher-level language. In both cases, you have a few options:

 1. Language-specific [FFI](https://en.wikipedia.org/wiki/Foreign_function_interface). E.g., in Python, this is [ctypes](https://docs.python.org/3/library/ctypes.html). In MATLAB, this is mex files.
 3. [Cython](http://cython.org). This is a language that basically lets you write typed Python code and compile it to an ordinary Python module.
 2. [SWIG](http://www.swig.org). This takes in a set of C/C++ files and automatically generates wrapper functions and classes in any language you would probably care about except MATLAB or Julia.
 4. [Weave](https://docs.scipy.org/doc/scipy-0.18.1/reference/tutorial/weave.html). This lets you write C code in the middle of your Python code.
 5. [BoostPython](https://wiki.python.org/moin/boost.python/GettingStarted) A C++ library that lets your code work as a Python module.

If you aren't using Python, your options are probably either SWIG or your langauge's FFI. I would definitely opt for SWIG if you have to wrap a lot of code, and/or if the code changes regularly, but likely the FFI if you just need something simple.

If you are using Python, I personally just use SWIG for everything, but I also like Cython. I haven't used Weave but I've heard good things. If you want to use SWIG, you should use my [example project](), because otherwise you will suffer greatly.


<!-- I've found Cython to be reasonably good, but a bit annoying in that you have to manually wrap every single class and function. I think ver -->

<!-- (just link to my SWIG example) -->
<!-- also link to example of using Ctypes, weave, and Cython -->

## Bit hacks

There are often sneaky ways to perform computations that are faster than the obvious ways. For example, if you want to compute the minimum of two integers, the obvious way would be:
```C
int min(int x, int y) {
    if (x < y) {
        return x;
    }
    return y;
}
```
This has a conditional branch, and not even a predictable one. A better way is:
```C
int max(int x, int y) {
    return y ^ ((x ^ y) & -(x < y));
}
```
where `^` denotes exclusive or. (See lecture 3 of [6.172](https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-172-performance-engineering-of-software-systems-fall-2010/) for an explanation of why this works.)

When writing C/C++, the compiler is probably better at these sorts of tricks than you, so inserting them manually may not be a good idea. In languages without compilers, however, these tricks are often helpful.

See [here](https://graphics.stanford.edu/~seander/bithacks.html) and [here](https://github.com/hcs0/Hackers-Delight) for the best lists of bit hacks and similar low-level tricks.

<!--
Examples:

 - Zeroing only the lowest bit in a number
 - Integer min and max without branches
 - Zigzag encoding + decoding
 -->

<!-- ## Math / Algorithmic tricks

^ Probably in a separate post

[Gumbel-Max Trick](http://timvieira.github.io/blog/post/2014/07/31/gumbel-max-trick/)
Sampling from a categorical distribution [in sublinear time](https://github.com/obachem/kmc2)
 -->
## Other Topics and Resources

 - The aptly named "[Latency Numbers Every Programmer Should Know](https://gist.github.com/jboner/2841832)"
 - See [here](https://www.anandtech.com/show/8104/intel-ssd-dc-p3700-review-the-pcie-ssd-transition-begins-with-nvme/3) for a much more detailed (but slightly outdated) set of SSD benchmarks.
 - [Profile-guided optimization](https://en.wikipedia.org/wiki/Profile-guided_optimization) (despite the popular misconception, this is *not* referring to manually optimizing based on performance measurements)
 - [Concurrency and parallelism](https://en.wikipedia.org/wiki/Parallel_computing)
 - [GPUs](https://en.wikipedia.org/wiki/Graphics_processing_unit) (in particular [CUDA programming](http://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html))
 <!-- - [Expression templates](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Expression-template) (a major reason compiled languages will always have a speed advantage over scripting languages with compiled back ends)<sup>[4](#exprTemplates)</sup> -->
 - [Godbolt](https://godbolt.org) (beautiful tool for looking at generated assembly)
 - Agner Fog's [website](http://www.agner.org/optimize/) and [instruction tables](http://www.agner.org/optimize/instruction_tables.ods)
 - The [Intel Optimization Manual](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=1&ved=0ahUKEwj8mcfapbDXAhVGRyYKHahoDDsQFggmMAA&url=https%3A%2F%2Fwww.intel.com%2Fcontent%2Fdam%2Fwww%2Fpublic%2Fus%2Fen%2Fdocuments%2Fmanuals%2F64-ia-32-architectures-optimization-manual.pdf&usg=AOvVaw31VXvTeMSxxZ3RqBvbJhZ4)
 <!-- - Agner Fog's [website](http://www.agner.org/optimize/) and [instruction tables](http://www.agner.org/optimize/instruction_tables.pdf) (which are better as [a spreadsheet](http://www.agner.org/optimize/instruction_tables.ods)) -->
 - [x86 SIMD Cheatsheet](https://db.in.tum.de/~finis/x86-intrin-cheatsheet-v2.1.pdf)
 <!-- - The best [list of bit hacks](https://graphics.stanford.edu/~seander/bithacks.html) -->
 <!-- - Last but not least, the [classic resource](https://github.com/hcs0/Hackers-Delight) -->


<hr/>

<a name="overview">1</a>: Of hardware, not pig farming

<a name="colmajor">2</a>: Actually it means that earlier dimensions are closer together, with elements sharing the first dimension being contiguous. But boy does this get confusing as soon as you need to slice anything.

<a name="tf">3</a>:  If you're running code on a GPU, the storage order considerations are different, since GPUs are less dependent on caches and can parallelize many more operations at once.

<a name="exprTemplates)">4</a>: Also the most effective means I know of ensuring that straightforward-looking code will subtly screw you over.

<a name="branchPrediction)">5</a>: It's actually more complicated than this, but what I described is a good approximation of what happens.
