---
title:  "#18 Cores that don't count"
paper_link: https://dl.acm.org/doi/10.1145/3458336.3465297
paper_year: HotOS 2021
---

# One Sentence Summary
This paper describes "mercurial cores", which are CPU cores that produces silent errors that are difficult to detect with traditional methods.

# Details
**CPU cores repeatedly and randomly performs erroneous computations** due to silicon defects. These cores are called "mercurial cores".
The authors encountered this problem at Google, when their massive compute clusters started to produce wrong results after they made an
innocuous code change. This code change caused the servers to use certain instructions more often. Only a small number of server machines are 
responsible for this error.

Even worse, these errors are **silent**, meaning that they could only be detected by checking the results against a golden result.
The paper refers to these errors as corrupt execution errors (CEE). CEEs usually occur within a specific core on multi-core CPUs (I think
this is just a result of mercurial core's low probability).

CEEs have 3 types of impacts in increasing severity:
1. Process crashes
2. Wrong answers
3. Data loss (ex. metadata corruptions), which may cause corruption amplification

The authors point out that this issue has been long known for storage devices and networks, where the data can be corrupted during transit. 
However, we have not considered silent errors for processors. We typically assume processors to be **fail-stop**, meaning that the processor halts
and/or raises signals when an internal error occurs. The processor silent error problem is harder to solve because the golden result is more 
costly to compute, perhaps requiring double the work (storage and network's golden results are the identify function). Furthermore, storage and
network can often amortize their fault tolerance costs since they operate on large amounts of data (ex. blocks and packets). Fault tolerance could 
be more difficult to implement at a per-instruction level.

We are only seeing this problem now partly because of Google's huge server fleet, but more fundamentally because of
1) **increasing CMOS scaling** and 2) **increasing architecture complexity**. The authors believe that we cannot count on chip makers to eliminate 
mercurial cores, especially for defects that occur late in life. Early warning and detection mechanisms need to be in place.

The tricky part about CEEs is that we typically do not have control over, or even understand, most variables that causes CEEs. For example, we
may observe that "this code runs incorrectly on this core but not that core", but we do not know the underlying root cause of the error (perhaps a 
faulty transistor?). 

The first dimension of a solution is detection. To detect CEEs reliably and as early as possible, the authors suggest that we should 
**test CPUs throughout their life cycles.** The other dimension is mitigation, where instead of relying on detection, we hope to eliminate 
CEEs in the first place. For example, we can run a computation on two cores, and if their results differ, rerun on a different pair of cores.
A well-known approach is triple modular redundancy, where the same computation is done three times and the final result is selected via majority
voting. The paper also notes that CEEs also require hardware mitigations, such as exposing testing features to users, designing functional units
that always check its answers, and trading extra area/power for reliability.

# Some English Words
- mercurial: subject to sudden or unpredictable changes of mood or mind. 
- innocuous: not harmful

# Questions
1. What is a code sanitizer?
2. What does end to end checksum mean?
3. What is an invariant?

# Comments
- This is *probably* not an issue for most compute clusters right now, since mercurial cores occur so rarely.
- Hmm, this paper is saying that the current chip verification methods are not good enough. I wonder what the verification community 
thinks.
