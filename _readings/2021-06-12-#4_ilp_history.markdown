---
title:  "#4 Instruction-Level Parallel Processing: History, Overview and Perspective"
paper_link: https://www.hpl.hp.com/techreports/92/HPL-92-132.pdf
paper_year: 1992
---

# Summary

# Details
Instruction Level Parallelism (ILP), as its name suggests, refers to parallelism obtained by executing multiple instructions 
(ex. add, load, multiply) concurrently. This can be achieved in two ways: 1) utilizing multiple independent functional units (FU)
2) FU pipelining. The paper identifies three classes of ILP: sequential arhictectures, dependent arhictectures, and independent arhictectures.
The difference between each class is whether parallelism information is embedded in the program binary or deduced by the hardware.

## Sequential Arhictectures
The representative use of sequential arhictecture is superscalar processor. The compiler/application binary provides no parallelism information. It is completely
up to the hardware to exploit ILP at runtime. Classic superscalar algorithms include scoreboard and Tomasulo's algorithm.

## Dependence Arhictectures
The representative use of dependence arhictecture is dataflow processor. The compiler or the programmer specifies the dependence between operations.
A dataflow processor executes an operation as soon as all of its input operands and the required FU are available (hence dataflow). However,
traditional dataflow processors cannot take advantage of speculative execution. Thus, dataflow processors may gain an edge if the program contains 
few branch instructions.

## Independence Arhictectures
The representative use of independence arhictecture is Very Large Instruction Word (VLIW). The compiler identifies parallelism within the program 
and communicates this by specifying which operations are independent. Note that in a typical program, it is impractical to specify all operations
independent from a particular operation. Thus, the compiler only specify independent operations that it thinks will provide the greatest performance 
benefits. A VLIW program specifies exactly when (which cycle) and where (which FU) each operation should be executed. With this information, the 
hardware can choose to make no runtime decisions at all (no brain mode). On the other hand, a particular VLIW processor implementation can choose
to still make some runtime decisions (ignoring or overriding some compiler specified parallelisms). With VLIW architectures, an operation is not 
equivalent to an instruction. An operation is a unit of compute such as add, load, branch. An instruction is a set of operations intended to be 
issued concurrently. In VLIW, an instruction contains multiple operations (hence Very Long)

## Hardware ILP Techniques
To support ILP, the hardware should either have multiple parallel FUs or pipelined FUs (often times both). In principle, pipelining is more efficient
than parallel FUs because adding pipeline stages is relatively cheap (registers). However, more pipelining results in clock skew, setup/hold time
violations, which limits the maximum operation frequency, and large latencies, which limits performance when running mainly sequential programs. 
On the other hand, adding parallel FUs requires larger interconnects, whose cost increases proportional to the square of number of FUs (assuming
all to all interconnect). 

In addition, more parallel FUs require more register file read and write ports. The chip area of a multi-port register 
file is proportional to the product of the number of the read ports and number of write ports. The access time also degrades with more ports.
One solution to improve register file scaling is (just like memory banks, file systems, caches, you name it) hierarchical design. We can group FUs
into clusters. All FUs in a cluster share the same multi-port register file. Inter-cluster communication is slow, so the compiler must partition 
data intelligently so that the cluster can exploit the benefits of locality. 

One of the common obstacle of ILP is small basic blocks. A basic block is a portion of the program that contains no branching. If basic blocks 
are small, ILP opportunities within each basic block is scarce, so we need to find parallelism across branch instructions. One common technique
is speculative execution. Another technique is predication, where the processor executes both branches and discard the wrong branch once the 
condition is resolved. Predication essentially converts control dependencies into data dependencies.

## Software ILP Techniques
coming soon... 

## ILP Limit Studies (and Their Limitations)
The author defines ILP limit studies as "throwaway all considerations of hardware and compiler practicality and measure the greatest possible 
amount of ILP inherent in a program." The major shortcoming of existing ILP limit studies is that they do not consider compiler transformations.
That is, they assume the application binary is fixed. The authors argue that once we consider compiler transformations, there are no limits to the
ILP speed up. Consider a fatuous architecture where we replace all compute with look up tables. For example, to perform sorting, we recording 
every possible input sequence and store their results in the gigantic look up table. This is of course not practical, but ILP limit study's 
very definition emphasizes the need to discard practicality. As a result of this shortcoming, early ILP limit studies found pessimistic results
for ILP speedup potentials (ex. Flynn bottleneck says that ILP can only speedup real world applications by 2-3x.)

# Thoughts and Comments

# Questions
