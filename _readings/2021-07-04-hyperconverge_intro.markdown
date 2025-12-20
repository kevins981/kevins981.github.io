---
title: "Introduction to Hyperconverged Infrastructure"
---

I wanted to learn about this topic because I saw a paper on software-defined persistent memory, whose concept was inspired by 
software-defined storage. It turns out that software-defined storage is a part of a larger infrastructure called HyperConverged 
Infrastructure (HCI). This idea seems to be gaining popularity in datacenters in the past 5-10 years. I am interested to learn
about the idea behind it and potentially how persistent memory can fit into this infrastructure.

# Virtualization [1]
Virtualization allows us to use software to emulate hardware by abstracting the physical hardware from the operating system.
This abstration is done by the **hypervisor**, which allows us to run virtualized servers, called virtual machines (VM).
Great solution! Virtualization enables many benefits, such as ease of management (a single interface), VM portability, availability,
higher hardware utilization etc. 

{:refdef: style="text-align: center;"}
![](/assets/images/posts/hci/hypervisor.jpg){: width="750" } 
{: refdef}
{:refdef: style="text-align: center;"}
*Figure 1: Server virtualization. The hypervisor is running two VMs on top of a single physical server [1].*
{: refdef}

# HCI [1]
At a high level, HCI combines different platforms into a single platform. In terms of server hardware, this means combining 
compute (CPU, memory) and storage (SSD, disk) into a single server. Hmmm, so this is just like our own personal computers?
we are used to this concept, but this is not how datacenters were built. For example, in datacenters that impelment storage
area network (SAN), all the storage devices are combined and placed in one location instead of residing locally beside each CPU.
One of HCI's main goals is to simplify data center management, that is, eliminate siloes of management. Figure 2 shows four silos. 
In this context, silo indicates isolation and separation. 

A feature of HCI is that it scales out all of its resource (ex. CPU, memory, SSD) in portions. That is, even if you only want more 
storage, you have to add a new node, which contains all resources, including storage (I think this is an extreme case where all
the storage spaces on exsiting nodes are 100% full. I imagine typically one can for example upgrade existing SSD DIMMs to achieve 
this).

{:refdef: style="text-align: center;"}
![](/assets/images/posts/hci/silo.jpg){: width="750" } 
{: refdef}
{:refdef: style="text-align: center;"}
*Figure 2: Some silos [2].*
{: refdef}

### Data Center's Cyclical Nature
Similar to clothing fashions, what was once obselete is often improved and brought back as the main stream. Virtualization itself, 
where mutliple users share the same physical hardware echos with the command line only and shared access model of 
main frame computers when they were first invented (although the latter never really became obselete). HCI shares a similar story.
Early data centers consisted of computers and locally attached storage devices. Each physical server ran a single task, and these
servers were managed separately. Then came along shared storage, ex. SAN, where a centralized storage array enables better efficiency
and utilization. HCI brings us back to direct-attached storage. However, the improvement this time is that although the storage devices
are locally attached, software-defined storage (SDS) allows us to manage them as if they are pooled.

### Virtualization + Storage
We saw what virtualization meant for compute (VMs). What does virtualization mean for storage? The key idea is again the abstraction 
of the underlying hardware. The workloads (VMs) are unaware of the underlying resources its consuming. Yes, each server has its own 
local storage, but the workload does not know nor care about whether the data was from local or remote storage. The SDS software, similar
to hypervisor for compute, administers and manages the resources based on policies.

# Sources

[1] J. Green, S. Lowe, and D. Davis. The Fundamentals of Hyperconverged Infrastructure. [Online]. Available: https://www.actualtechmedia.com/wp-content/uploads/2018/04/SCALE-The-Fundamentals-of-Hyperconverged-Infrastructure-v2018.pdf

[2] https://www.metaltoad.com/blog/silos-tech
