---
title: "Lamport Clock" 
---

This post is my note while studying the sources. I do not own any of the below materials.

# Background

Lamport clock is a type of logical clock, which counts the number of events occured instead of physical time. 
The goal of a logical clock is to determine the order of events. Why? This is because distributed systems require 
happened-before relationships in order to functional correctly. Why? 

## Happens-before

A happens-before relationship is a **partial order**, meaning that it is possible that for two events a and b, 
neither a happened before b or b happened before a. In this case, we call a and b concurrent. This is of course not
possible in physical time, but we are in logical time here. 


More formally...

An **event** is something that happens at one node. Ex. sending/receiving a message, a local execution step. 

Event a happens before event b iff (a -> b):
1. a and b occured in the same node and a executed before b; OR (note the OR here)
2. event a is the action of sending message m, and event b is the receipt of that message m (can't receive a message before its sent);
3. there exists an event c such that a -> c, c -> b (transitivity)

Partial order means that it is possible that neither a -> b and b -> a. Then, a \|\| b.

An example set of events are shown below: 


{:refdef: style="text-align: center;"}
![](/assets/images/posts/two_phase_commit/strawman.png){: width="550" }
{: refdef}


# Questions


# Further Readings

# Sources
[1] 
