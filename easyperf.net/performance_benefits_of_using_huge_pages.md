
# Performance Benefits of Using Huge Pages for Code.
https://easyperf.net/blog/2022/09/01/Utilizing-Huge-Pages-For-Code

## Recap on Memory Pages
Applications operate on virtual addresses for reasons of
- protection 
- effective physical memory management 

Memory can be divided into pages. \
Every address is the address of the page + the offset on that page.
 
Default page size 
- x86: 4KB, 
- ARM: 16KB. 

E.g. on a 4KB page
*page address*: the 52 rightmost bits\
*offset*: The 12 leftmost bits 

You only need to translate the page address since the page offset doesn’t change.

**Translations**
User-space applications only know virtual addresses.\
To access data in memory, those must be translated to actual physical adresses.\
The kernel maintains a page table, which holds address mappings from virtual pages into physical pages. 

Without HW support for translations, every time you need to do a load or store, you would
1. interrupt the process
2. kernel handles that interrupt 
3. interrupt service routine “walks” the page table and retrieves the translation. 

Would be 10000+ cycles - unbearable.

**TLBs**
The page table rarely changes, so it’s a good idea to cache some of the most frequent ones in HW.\
Done in every modern CPU - TLB (Translation Lookaside Buffer) keeps the most recent translations.
 
Hierarchy of
- L1 ITLB (Instructions)
- L1 DTLB (Data)
- L2 STLB (Shared - instructions and data). 
 
L1 can hold up to a few hundred recent translations.\
L2 can hold a few thousand. 

With a page size of 4KB, every such entry in the TLB is a mapping for a 4KB page.\
Thus L1 TLB can cover up to 1MB of memory; L2 up to 10 MB.

**Page Walks** 
The 10MB of memory space covered by L2 STLB sounds like it should be enough for many applications.

Consider a TLB miss: 
Because the HW knows the format of the page table, it can search for translations by itself, without waking up the kernel, as interrupting the process would be very expensive (“HW page walker”).\
 I.e. it will issue all the necessary instructions to find the required address translation.\
Much faster than an interruption, but still very expensive.

## Huge Pages for data AND code
Large pages for data are commonly used for data.\
Any algorithm that does random accesses into a large memory region will likely suffer from TLB misses.\
E.g. binary search in a big array, large hash tables, histogram-like algorithms, etc.\
Because the size of a page is relatively small (4KB), there is a high chance that the page you will access next is not in the TLB cache.

Can be solved with *Huge Pages*.\
On x86 you can allocate 2MB and 1GB pages.\
With just one 2MB page you can cover the same amount of memory as with 512 4KB pages.\
You need fewer translation entries in the TLB caches.\ 
Doesn't eliminate TLB misses completely, but greatly increases the chance of a TLB hit. 
 
Example: speedup Clang compilation
Well-known Clang compiler example.\
Very flat performance profile, i.e. has no clear hotspots.\
Many functions take 1-2% of the total execution time.\
The complete section of statically built clang binary on Linux has a code section of ~60MB - does not fit into L2 STLB. 

It is likely that multiple hot functions are scattered all around that 60MB memory space, and very rarely do they share the same memory page of 4KB.\
When they begin frequently calling each other, they start competing for translation entries in the ITLB.\
 Since the space in L2 STLB is limited, this may become a serious bottleneck.
 
 ### Baseline
 
Building the compiler itself for the x86 target:
```
$ cmake -G Ninja -DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_PROJECTS="clang" -DLLVM_TARGETS_TO_BUILD=X86 ../llvm
```

Compile LLVM sources and collect ITLB misses:
```
$ perf stat -e iTLB-loads,iTLB-load-misses ../llvm-project/build/bin/clang++ -c -O3 <other options> ../llvm-project/llvm/lib/Transforms/Vectorize/LoopVectorize.cpp
```

Iodlr can be used to estimate the fraction of cycles the CPU was stalled due to instruction TLB misses.\
 Gives an intuition of how much potential speedup you can achieve by tackling this issue.

```
$ measure-perf-metric.sh -m itlb_stalls -e ../llvm-project/build/bin/clang++ -c -O3 <other options> ../llvm-project/llvm/lib/Transforms/Vectorize/LoopVectorize.cpp
```


7% of cycles that Clang spent compiling LoopVectorize.cpp was wasted on demanding page walks and populating TLB entries - a significant number, so there is something to improve.

You can continue the analysis by adding the -r option to measure-perf-metric.sh. This will sample on icache_64b.iftag_stall event to locate the place where the TLB stalls are coming from. 

## Preventing ILTB misses

### Option1: align code section at 2MB boundary

Tell the loader/kernel to place the code section of an application onto preallocated (explicit) Huge Pages. 
Key requirement is that the code section must be aligned at the Huge Page boundary, e.g. 2MB. 
Requires that you relink your binary.

Rebuild the clang compiler with the code section aligned at the 2MB boundary
```
$ cmake -G Ninja -DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_PROJECTS="clang" -DLLVM_TARGETS_TO_BUILD=X86 ../llvm -DCMAKE_CXX_LINK_FLAGS="-Wl,-zcommon-page-size=2097152 -Wl,-zmax-page-size=2097152" 
$ ninja -j `nproc` clang
```

Difference visible in the ELF binary  
```
readelf -S --wide file  
```

In the 2MB aligned case, the PROGBITS section starts from the offset 0xe00000, which equals 14MB - a multiple of 2MB.\
In the baseline case, the .init section starts right after the .rela.plt section with a few padding bytes added.\
The size of the .text section didn’t change – it’s the same code, only the offset has changed.

The downside is that the binary size gets larger.\
The clang-16 executable increased from 111MB to 114MB.\
Since the problem with ITLB misses usually arises for large applications, an additional 2-4MB of padded bytes isn't too dramatic tho.

### Configuring the target machine

We need to reconfigure the machine which will use our “improved” clang compiler. 

```
$ sudo apt install libhugetlbfs-bin
$ sudo hugeadm --create-global-mounts
$ sudo hugeadm --pool-pages-min 2M:128
```


Why 128 huge pages? The size of the code section of the executable is 0x3b6b7a4 (see article) - roughly 60MB. Less could have been allocated, but it doesn't mean much with 16GB RAM or more.\
Reserving 128 explicit huge pages increased memory usage from 0.9Gb to 1.15G.\
That space is unavailable for other applications not utilizing huge pages. 

You can also check the effect of allocating explicit huge pages with:
```
$ watch -n1 "cat /proc/meminfo  | grep huge -i"
```

### Running the relinked binary

```
$ hugectl --text ../llvm-project/build_huge/bin/clang++ -c -O3 <other options> ../llvm-project/llvm/lib/Transforms/Vectorize/LoopVectorize.cpp
```

hugectl --text requests remapping of the program text.

We dont need hugectl if we set the special bit in the ELF header that determines if the text segment by default is backed with huge pages. 
A similar bit exists for data segments (not set below). 

```
$ hugeedit --text ../llvm-project/build_huge/bin/clang-16
```

Comparing to the baseline, we have 7 times less iTLB misses (12M -> 1.6M), resulting in a 5% faster compiler time (15.4s -> 14.7s). 
There's still 1.6M iTLB misses, accounting for 4.1% of all cycles stalled (down from 7% in the baseline).

/proc/meminfo can also be used to observe usage of huge pages


### Option2: remap the code section at runtime

iodlr library can automatically remap code from the default pages onto huge pages - avoids recompilation.\
Build the liblppreload.so library and preload it when running your application.

```
$ cd iodlr/large_page-c
$ make -f Makefile.preload
$ sudo cp liblppreload.so /usr/lib64/
$ LD_PRELOAD=/usr/lib64/liblppreload.so ../llvm-project/build/bin/clang++ -c -O3 <other options> ../llvm-project/llvm/lib/Transforms/Vectorize/LoopVectorize.cpp
```

Allows you to speed up existing applications when you don’t have access to the source code.

The liblppreload.so library works both with explicit (EHP) and transparent huge pages (THP).\
By default it will use THP, so they must be enabled  (/sys/kernel/mm/transparent_hugepage/enabled should be "always" or "madvise"). 

Use /proc/meminfo like above to detect usage of THP as well.

### Benchmarks

Using Huge pages gives roughly 5% faster compile times for the recent Clang compiler.\
Option 2 (iodlr version) is faster than option 1 - no good explanation.
 
 
## Closing thoughts


Remapping .text onto Huge Pages is not free and takes additional execution time.\
May hurt rather than help short-running programs. Always Measure.

Transparent Huge Pages suffer from non-deterministic allocation latency and memory fragmentation.\
To satisfy a Huge Page allocation request at runtime, the Linux kernel needs to find a contiguous chunk of 2MB.\
If unable to do so, it needs to reorganize the pages, resulting in significantly longer allocation latency.\
In contrast, Explicit Huge Pages are allocated in advance and are not prone to such problems.

Another way to attack iTLB misses is to BOLT your application.\
Will likely group all the hot functions, which should also drastically reduce the iTLB bottleneck.\
Huge Pages is a more fundamental solution to the problem since it doesn’t adapt to the particular behavior of the program.\
You can try using Huge Pages after bolting your application to see if there are any gains to be made.