---
title:  "#1 Memory Persistency"
paper_link: https://web.eecs.umich.edu/~twenisch/papers/isca14.pdf
paper_year: ISCA 2014
---

## Summary
This paper proposes memory persistency, a theoretical framework used to describe persist ordering constraints. 
The paper introduces three memory persistency models: strict, epoch, and strand persistency. 
Experiments show that relaxe persistency models can achieve higher throughput than strict models by reducing persist ordering constraints.

## Details
Memory consistency models are formal specifications on how the memory hardware should behave given a program. 
The key realization is that the hardware execution order is often times not the same as the program order, due to optimizations such as 
out-of-order execution, cache evictions, and memory controller reorderings. Hence, the programmer must know to what extend the hardware 
will reorder memory operations in order to write functionally correct code. 

Memory persistency is memory consistency for persistent memory. The same realization applies: the NVM persist order does not need to be 
the same as the program order. Similarly, NVM programmers cannot write recoverable code is persist ordering is not guaranteed. The goal of
memory persistency models are two folds. First, the persistency model should enable high performance. This paper lists two specific optimizations:
persist buffering and persist coalescing. For persist buffering, we would like to enable instruction level parallelism by having multiple NVM write
requests in flight while potentially performing other useful work. For persist coalescing, we would like to combine multiple small persists into 
a single persist operation (usually 8 bytes). How does persist reordering enable the above two optimizations? My understanding is that with relaxed 
consistency models, we do not have to pessimistically wait for the previous persist to finish before issuing the next persist. Second, the persistency 
model should be programmer friendly. The paper mentions the importance of the ease of programmability, and I believe it is crucial.

**Strict persistency** equates persist ordering with memory ordering. The NVM acts just as another processor. It will perform the persists in the exact order
it sees the memory operations. This is the most programmer friendly persistence model, as the programmer does not need to add any persist ordering annotations.
A naive strict persistency implementation's performance is teeerrible. With every persist operation, the CPU must stall untill the persist completes. The paper
does propose one optimization, buffered strict persistency, which allows somoe degree of instruction level parallelism. **Epoch persistency** divides the program 
into epochs. Persists within the same epoch may be reordered. Persists from different epochs must follow epoch order. Lastly, **strand persistency** further divides
the program into strands. Strands should be logically independent. Thus, persists from different strand may be reordered (since they are logically independent). Note
that one can still use epoch barriers in strand persistency. 


## Thoughts and Comments
- The strict persistency baseline is weak. The paper is aware of this, as they discuss the buffered strict persistency optimization. It would be interesting to
see strict persistency's optimized performance since it requires the least amount of programmer annotations.

## Questions
- I don't really understand what is going on in multiprocessor scenarios. Ex. strong persist atomicity, persist-epoch race
