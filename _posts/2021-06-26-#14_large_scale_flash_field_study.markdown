---
title:  "#14 A Large-Scale Study of Flash Memory Failures in the Field"
paper_link: https://dl.acm.org/doi/10.1145/2796314.2745848
paper_year: SIGMETRICS 2015
---

# Background
- First, some terminologies. The following definitions are taken from different websites, forums, and blogs on the Internet. They are probably not 100% accurate, but we also don't need them to be 
for the purpose of understanding this paper. Solid State Drive (SSD) refers to storage device with no moving parts. Flash memory refers to a memory technology (two main types are NAND and NOR flash). 
Thus, a SSD may or may not implement flash. HOWEVER, for the purpose of this paper, we will refer to flash-based SSD simply as SSD. 
- According to Wikipedia, the name "flash" was used because the erasure process resembled the flash of a camera.
- In general, the increase in flash capacity has decreased flash reliability. The authors expect this trend to continue.
- The bit error rate (BER), as its name suggests, is the rate at which errors occur relative to the total data transfered. That is, BER = erroneous_bits / bits_accessed.
- Flash memory is organized into blocks and pages. A block contains multiple pages. To write to a page, the entire block must be first erased.

# Details 
The goal of this paper is to understand the nature of SSD failures in the field (Facebook servers).
Figure 1 shows the SSD architecture. The SSD controller coordinates transfers between the host and flash chips while performing various performance and reliability optimization.
Data is typically mapped accross channels to enable parallel access (similar to DRAM). The SSD controller performs wear leveling by maintaining a table mapping logical addresses to 
physical addresses (not to be confused with virtual memory). The SSD controller also detect and correct errors for small amounts of data (few bits per KB). For larger errors, the controller
sends the data to a driver running on the host for more complex corrections. The paper refers to these errors as (SSD) uncorrectable errors, and such occurances as SSD failure. 
If the host cannot correct the error either, the data is lost. This paper only collects data for uncorrectable errors.

{:refdef: style="text-align: center;"}
![](/assets/images/posts/14_large_scale_flash_field_study/server_ssd.jpg){: width="450" } 
{: refdef}

This paper reports the following 5 observations.
1. SSDs go through four phases during their lifetimes: early detection, early failure, usable life, and wearout (Figure 2). The traditional bathtub curver (kinda like a parabola
with a flat bottom) does not have the early detection phase. During this period, the failure rate increases. The authors believe this new phase is caused by the two-pool model 
of flash blocks. The flash chip has a weak pool, containing blocks whose error rate increases much faster than the average, and strong pool, which is the opposite. The blocks in the 
weaker pool causes many failures and thus are quickly abandoned by the SSD controller. As blocks in the weaker pool are exhausted, the failure rate starts decreasing. 
2. Reads are not a dominant source of SSD errors.
3. Sparse (non-contiguous) data layout in SSDs lead to high failure rates. 
4. Higher temperatures increase failure rate. Throttling, a technique that aims to reduce SSD temperature, is effective at reducing failure rates.
5. The amount of stored data reported by system software can be higher than the actual amount of physical data written to flash chips. This is due to techniques such as 
write coalescing.

{:refdef: style="text-align: center;"}
![](/assets/images/posts/14_large_scale_flash_field_study/ssd_lifecycle.jpg){: width="350" } 
{: refdef}
{:refdef: style="text-align: center;"}
*Figure 2: SSD lifecycle pattern [1].*
{: refdef}


# Thoughts and Comments
- Hmm, feels like observations 1, 3, and maybe 2 are novel. 4 and 5 do not seem to be surprising.

# Sources
[1] MEZA, J., WU, Q., KUMAR, S., AND MUTLU, O. A large-scale study of flash memory failures in the field. In Proceedings of the 2015 ACM SIGMETRICS International Conference on Measurement 
and Modeling of Computer Systems (New York, NY, USA, 2015), SIGMETRICS '15, ACM, pp. 177-190
