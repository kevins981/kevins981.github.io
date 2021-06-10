---
title:  "#2 High-Performance Transactions for Persistent Memories"
paper_link: https://dl.acm.org/doi/10.1145/2872362.2872381
paper_year: ASPLOS 2016
---

## Problem
Atomic and durable transactions are a widely used mechanism to ensure data consistency on NVMs. However, naive 
implementations of NVM transaction is low performance due to the unnessesary persist dependencies.

## Summary
The paper proposes defered commit transactions (DCT) to remove unnecessary persist dependencies enforced by the naive
transaction implementation, synchronous commit transactions (SCT). Specifically, DCT defers the transaction's commit step
until after the lock is released. The paper then evaluations DCT across three memory persistency models and show its 
superior runtime over SCT.

## Details
This paper builds upon ideas from the Memory Persistency paper by analyzing how to best utilize the persistency models
when designing transactions. The paper makes the observation that "most of the unnecessary dependencies arise as a consequence of
performing the commit step of the transaction while locks are held," so naturally (for the reader, perhaps not obvious for
the researchers when performing the studies), the proposed solution is to release the locks before commit. 

The paper assumes that transactions use lock for concurrency control, but does not provide background on this topic. Maybe 
the readers are this expected to know this concept. My educated guess is that the NVM address space is divided into lock regions,
each controlled by a lock. To modify data within a lock region, a thread must first obtain its lock (pretty much how regular volatile
locks work). 

The authors breaks down an undo transaction into five steps: acquire all locks (L), allocate and write to log (P), 
modify the data in NVM (M), commit transcation (C), release all locks (U). Only P, M, and C are persist operations,
while L and U are volatile operations. The paper proceeds to describle the minimum set of persist dependencies between
conflicting transactions (transactions that modifies overlapping data). First, P\<M\<C. This is true even with non-conflicting
transactions. Second, if thread m acquires the conflicting lock(s) before thread n, then P_m < P_n and M_m < M_n and C_m < C_n.
This is because we would like the data modification and log entry ordering to match the lock acquisition ordering (why?).
- One question I had was: why do we not need C_m < P_n? Doesn't thread n need to wait for thread m to release the lock before
starting its transaction? I believe the answer is we are considering persist ordering and not volatile ordering. Lock 
aquisitions and releases occur in volatile memory. We indeed must have C_m < P_n in volatile memory order (ie. thread n's P 
must be issued after thread m's C, but we can relax the ordering constraints when these write requests reach the NVM.

In this model, for x conflicting transactions, the persist critical path is x+2 persist operations.

The paper then review the three persistency models presented in Memory Persistency: strict, epoch, and strand. 
Furthermore, the authors attempt to formalize Intel's x86 persistency model (the only commercially available NVM platform): eager sync.
We can order two persists to A and B with the following: STORE A; CLWB A; SFENCE; PCOMMIT; SFENCE; STORE B;
HOWEVER, the PCOMMIT instruction has been deprecated by Intel as per https://software.intel.com/content/www/us/en/develop/blogs/deprecate-pcommit-instruction.html.
The original intent of PCOMMIT was to ensure data that reached the NVM memory controller's write pending queue are flushed into persistent domain.
This is because there was a concern that not many hardware platforms would support ADR, where the persistence domain includes the write queue.
It turned out that many platforms planned to support ADR, so PCOMMIT is not needed anymore since data that reached the write queue is guaranteed
to be persisted, even during a power failure. Thus, the latest code to order two persists is: STORE A; CLWB A; SFENCE; STORE B; where the CLWB ensures A 
is flushed to the NVM write queue. SFENCE ensures every store before the fence are globally visible before any stores after the fence. 
- This is a bit confusing. Our goal is for A to persist before B, but it sounds like SFENCE only enforces global visibility ordering and not persist ordering. I don't think visibility 
and persistence are the same thing. The SFENCE ensures the store to A is visible to other processors before store to B, but does that guarantee A is persisted
before B? The Programming PM book seems to suggest that the SFENCE instruction ensures that A is flushed from cache before B. Does this mean that the order of flushes
is equal to the order of persists? How does global visibility relate to flushing? Are they the same thing? That's a lot of questions...

The paper proceeds to show that SCT enforces unnessesary persist constraints even under relaxed persistence models. Under epoch persistency, we need to
insert 4 epoch barriers to ensure correctness. I am not sure why the lock aquisition and release are a part of persist ordering constraints (see questions).
Through bunch of equations, the paper shows that SCT results in serialized transactions under epoch persistency for both non-conflicting and conflicting transactions.
SCT under strand persistency is a bit better, as non-conflicting transactions are allowed to execute independently, which is the ideal case. However conflicting transactions
are still serialized.

DCT releases the lock before performing the commit step (commit deferred). This means that for conflicting transactions, P_n does not have to wait for C_m anymore.
However, this means that the commit ordering is not guaranteed anymore (Eq 5), so the paper introduces new mechanism to ensure this. With epoch persistency, 
conflicting transactions under DCT now only require the minimal set of persist ordering constraints. I don't understand the reasoning behind this. I think I need to first 
understand why SCT under epoch persistency works the way it does.


## Thoughts and Comments

## Questions
- What exactly does SFENCE guarantee? See details section.
- I don't quite understand inter-thrasaction dependencies. Ex. exactly how does Figure 3 conflicting case work?
    - From section 4.2, I believe that epoch barriers only order persists from the same thread. Ordering across 
    threads are NOT controlled by epoch barriers but by strong persist atomicity.
- "Atomic, durable transactions are a widely used abstraction to enforce such consistency." What other abstractions 
are there? Alternatives to transactions?
- What are per-thread distributed logs and centralized logs?
- What is checksum-based log entry validation? How does it enable parallel log writes?
- Is this claim true? How so? "Most of the unnecessary dependencies arise as a consequence of performing the commit 
step of the transaction while locks are held." 
- For the minimal persist dependencies in conflicting transactions, why do the inter-transaction ordering have to 
be the same as lock acquisition order?
- I do not understand the right hand side of the first equation in section 5.1. It seems to be describing the fact that
lockDS_n may not persist before unlockDS_m. But the locks are held in volatile memory, so why do we have a persist ordering constraint?
