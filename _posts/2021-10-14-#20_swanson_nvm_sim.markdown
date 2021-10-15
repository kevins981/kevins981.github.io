---
title:  "#20 Characterizing and Modeling Non-Volatile Memory Systems"
paper_link: https://swanson.ucsd.edu/data/bib/pdfs/MICRO20-LensVans.pdf
paper_year: MICRO 2020
---

# Summary
This paper contains two main components: LENS and VANS. LENS is a profiler used to extract key NVM 
microarchitecture parameters by executing microbenchmarks on real NVMs. VANS uses parameter acquired
from LENS to build a NVM simluator. The authors show that this approach offers higher simulation accuracy
compared to prior models such as DRAM+delays.

# Background


# Details
## Internal Buffers
Based on released Intel documents, the paper identifies several known NVM architecture components.
- Write Pending Queue (WPQ): a queue in the memory controller that stores pending NVM write quests. This
queue is in the ADR domain, meaning that it is power-fail safe.
- Load Store Queue (LSQ): queue that reorders incoming read/write requests. This queue also performs write 
combining. The goal is to minimize read-modify-write operations.
- Read Modify Write buffer (RMW): buffer that performs RMW operations. Recall that the underlying NVM media
has 256 Bytes granularity. Also recall that Intel cache line size is 64 Bytes. If the LSQ fails to combine
multiple 64B requests into a 256B req, then we need to perform **RMW operations**. This makes sense. If we 
are writing less than 256B, we need to know what the original NVM content is. RMW is of course more 
expensive than a write due to the extra read and modify operations.
- Address Indirection Translation buffer (AIT): buffer that performs internal address translation to 
achieve wear-leveling and bad-block management. 

{:refdef: style="text-align: center;"}
![](/assets/images/posts/swanson_nvm_sim/nvm_internal.png){: width="550" }
{: refdef}

## Prior NVM Simulators
Prior emulators work by stalling CPU for additional cycles or injecting software delays. This approach is 
NOT accurate and cannot more NVM's complex performance behaviors. The paper gives two discrepancy examples.
1) inaccurate BW and latency magnitudes 2) real NVMs have unstable read latencies.

## LENS profiler
THe main goal of LENS is to inject specially designed microbenchmarks to reverse engineer NVM architectual
parameters. 

{:refdef: style="text-align: center;"}
![](/assets/images/posts/swanson_nvm_sim/lens.png){: width="550" }
{: refdef}

Benchmarks:
- Pointer chasing: allocates a continuous NVM memory region (PC-Region). Divide it into equal sized 
blocks (PC-Blocks). Perform R/W on all blocks in the region in a random order. Sequentially access data
within each block. Three variants:
    1) Buffer capacity: fix block size, vary region size and collect average latency per cache line
    2) Buffer entry size: fix region size, vary block size
    3) Issue read after write reqs
    - All accesses bypass the cache
- Overwrite: repeated write to a fixed NVM region. Two variants:
    1) collect execution time of each write in fixed region
    2) measure frequency of long tail-latency by changing region size
- Stride: r/w cache lines with fixed stride. Two variants:
    1) fix stride, vary access size, and measure BW
    2) fix access size and vary stride to cahcaterize interleaving


Here is how the paper probes each parameters:
- Buffer capacity: 
    - I do not understand why the load inflection points on Fig. 5a) indicate read buffer sizes.
    - I do not understand why do 16K and 16M not show up in the store line. Aren't the RMW
    and AIT bufs also on the write path?
    - I believe the CPU r/w req issue freqency is much greater than NVM req service frequency (GHz vs MHz).
    This means that if the program is R/W heavy, the NVM internal buffers will be full most of the time.
    - I believe NVM write request are non-blocking, as in the CPU only needs to wait for the req to 
    reach WPQ.
    - At region sizes below 16K, all the requested read data can fit into the RMW buffer. 


- Buffer entry size: 

{:refdef: style="text-align: center;"}
![](/assets/images/posts/swanson_nvm_sim/fig5.png){: width="950" }
{: refdef}

# Questions
1. If the NVM media is 256-byte access granularity, how is it byte-addressable?
2. Why is the cache connected to RMW in figure 4?
3. The paper claims that buffer overflows will cause dramatic change in access latency. Why is the latency
dramatically different when a buffer overflows?

# Comments



# Sources
[1] https://www.intel.com/content/www/us/en/developer/articles/technical/deprecate-pcommit-instruction.html
