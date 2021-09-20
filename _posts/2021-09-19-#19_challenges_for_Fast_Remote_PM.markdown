---
title:  "#19 Challenges and Solutions for Fast Remote Persistent Memory Access"
paper_link: https://dl.acm.org/doi/10.1145/3419111.3421294
paper_year: SSoC 2020
---

# Summary
This paper performs an empirical study on NVM performance in a distributed network setting. The key insight is that current CPU
caches are not designed to optimize NVM performance. The paper then presents several solutions to improve performance
for current NVM network systems. This paper is similar in spirit to Swanson's "An Empirical Guide to the Behavior and 
Use of Scalable Persistent Memory", but is for NVM in network context.

# Details
This paper evaluates remote NVM performance under two networking approaches: RDMA and RPC. 
RDMA stands for remote DMA. RDMA is one-sided communication, where a client is trying to write to a remote server's NVM.
The client would interact directly with the remove NVM. No server processor involved.
RPC stands for remote procedure calls, which is two sided communication. Client sends write request to server. Server receives data
and stores them in volatile memory, then write to NVM. Finally the server sends ACK back.

Two key metrics: **single write latency and bulk write bandwidth**. 
- Single write latency: RDMA is better than RPC for DRAM, because it only requires one sided communication. 
However, RDMA's advantage is reduced for remote NVM. This is because for each remote NVM persist, 
we need to perform a remote NVM write AND a remote NVM read. 
The read is required to flush the server NIC and PCIe buffers in order to make sure the data is indeed persisted. We also 
need to turn off DDIO to bypass the cache and write directly to NVM. This **extra read is costly**. The paper does not propose a
solution to this overhead.
    - DDIO is Intel's Data Direct IO which allows network data to go into the cache. 
- Bulk write bandwidth: key takeaway is that we should turn off DDIO for better RDMA bulk write performance. This is because
the cache turns originally sequential writes to near **random writes** into NVM. Recall that NVM performs better with sequential
writes. The other aspect is write amplification. Cache line is 64B but NVM write is 256B. On a cache eviction, 
if the cache coalease fails, the controller performs a 256B read-modify-write, causing write amplification. The proposed solution is
to disable DDIO to preserver sequential writes.

# Questions

# Comments
