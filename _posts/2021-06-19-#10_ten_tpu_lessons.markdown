---
title:  "#10 Ten Lessons From Three Generations Shaped Google's TPUv4i"
paper_link: https://conferences.computer.org/iscapub/pdfs/ISCA2021-4ghucdBnCWYB7ES2Pe4YdT/333300a001/333300a001.pdf
paper_year: ISCA 2021
---

# Details 

## Introduction to TPUs
Skipping this for now. Should learn about TPUs from other papers dedicated to this topic. 
Ex. https://ieeexplore.ieee.org/document/9351692 

## The Ten Lessons
### 1. Logic, wire, SRAM, and DRAM improve unequally
From 45nm to 7nm, the energy of SRAM improved 1.3-2.4x (desnsity scaling slowing down), DRAM 6.3x (HMB), unit wire <2x,
adder 2.4-4.3x, multiplier 2.1-3.2x. Logic has been improving faster than SRAM and wire, so logic is relatively "free" in
terms of energy cost. DRAM was able to avoid this due to innovations in HBM.
### 2. Leverage existing compiler optimizations
The TPU uses VLIW instructions and the XLA (Accelerated Linear Algebra) compiler. Recall that VLIW relies on the compiler 
the identify parallelization opportunities. TPUv4 enhanced the XLA compiler for TPUv2.
### 3. Design for perf/TCO instead of perf/CapEx
TCO stands for total cost of ownership. Capital Expense (CapEx) is the price of an item, and Operation Expense (OpEx) is the
cost of operation such as electricity. TCO amortizes CapEx over 3 years, so TCO = CapEx + 3\*OpEx. For products such as the
TPU, the paper argues that companies should care more about perf/TCO than perf/CapEx. DSAs should aim for good perf/TCO over
its full lifetime.
### 4. Backwards ML compatibility enables rapid deployment of trained DNNs
Timeliness can be important for DNNs, and having backwards compatibility will help this. The authors define backward ML
compatibility as TPU generations producing exactly the same training/inference results. The achieve this, new TPUs must
provide the same numerics such as bf16 and fp32. Floating point addition complicate this, as it is not associative. That is,
(a+b)+c != a+(b+c). This can prevent TPUs to procude bit-identical results. Constraining the compiler to a specific order 
degrades performance. The paper's solution is to use the compiler from different generations in a similar way and
making the TPU appear simillar from the compiler's perspective (this is pretty vague. Not sure what they mean).
### 5. Inference DSAs need air cooling for global scale
TPUv3 (450W) uses liquid cooling, which requires placing multiple TPUs in adjacent racks to ammortize the cooling 
infrastructure. This is too restrictive for small datacenters and datacenters that are already packed.
### 6. Some inference apps need floating point arithmetic
Quantization is nice for power, area, and speed, but not all applications can afford to use it. Figure 1 shows the decrease
in accuracy caused by quantization. The lesson is DSAs may offer quantization, but should not require it.

{:refdef: style="text-align: center;"}
![](/assets/images/posts/ten_lessons_tpu/quantize_not_ok.jpg){: width="450" } 
{: refdef}
{:refdef: style="text-align: center;"}
*Figure 1: Example of quantization error in outline cropping [1].*
{: refdef}

### 7. Production inference normally needs multi-tenancy
Multi-tenancy means multiple models sharing the same physical TPU. This can lower cost and reduce latency for applications
that require multiple models. Application developers often require fast switching time between models. Achieving this
by loading weights from the CPU on the spot is not realistic under current memory bandwidths.
### 8. DNNs grow ~1.5x annually in memory and compute
Self explanatory. Architectures need to leave room for improvement to keep up.
### 9. DNN workloads evolve with DNN breakthroughs
Self explanatory. The paper emphasizes the importance of programmability and flexibility to keep up.
### 10. The inference SLO is P99 latency, not batch size
SLO stands for Service Level Objective, which is a means of measuring the service provider's performance [2]. 
P99 latency is the latency at which 99% of the requests should be faster than [3]. The authors disagree with the fact that 
recent DSA papers associated latency limit in terms of batch size (often set at 1). Thus, future DSAs should take advantage
of larger batch sizes.

## TPUv4i Design Decisions and Performance Analysis
The rest of this paper goes into details about TPUv4's implementation and performance analysis. I found them difficult to 
understand without some background readings first. Will comback to this.

# Thoughts and Comments
- The authors mention that the goal of backward ML compatibility is for TPU generations to produce exactly the same 
training/inference results. This sounds very strict in general. Imagine porting a floating point application from CPU
to GPU and requiring bit-identical results. Sounds very difficult due to the difference in architecture. Maybe it is not
that bad for TPU generations since, well, they are all TPUs and Google made all of them.

# Questions

# Sources
[1] Shih, Y., Lai, W.S. and Liang, C.K., 2019. Distortion-free wide-angle portraits on camera phones. ACM Transactions on
Graphics, 38(4), pp.1-12.

[2] https://en.wikipedia.org/wiki/Service-level_objective

[3] https://stackoverflow.com/questions/12808934/what-is-p99-latency
 
