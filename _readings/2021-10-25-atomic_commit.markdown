---
title: "Two-Phase Commit" 
---

This post is my note while studying the sources. I do not own any of the below materials.

## Problem
We wish to execute atomic commits in a distributed system. For example, if my client's data is 
partitioned across multiple nodes, I may want to perform a transaction that spans multiple nodes.

[1] provide a nice example, where we are trying to transfer money from user A to user B. These two
user data reside in different banks (nodes). We obviously want this transaction to be atomic, that is,
either both banks do it, or neither do it. 

A straw man, or naive approach, is just to tell both banks to do it, as shown below. 

{:refdef: style="text-align: center;"}
![](/assets/images/posts/two_phase_commit/strawman.png){: width="550" }
{: refdef}

This obviously does not work, if ex. A doesn't have enough money, so A aborts but B proceeds.
A crashes before receiving the message, so only B performs the action. The coordinator crashes
just after taking money from A but before giving money to B. 

As an exercise, what if we require both A and B to send back an ACK to coordinator? The coordinator
will only return OK to client if it receives ACKs from both A and B. Note that this is not two-phase commit,
but just something I randomly came up with. I believe the problem with this approach is that we require
the bank operations to be irrevocable. That is, we cannot first give B $20 and tell it to give it back afterwards.
This is because once the data change is made visible, other transactions may start using it. This requirement
is the building block for read commited isolation. 

## Two-phase Commit (2PC)
So what is the solution? The two-phase commit actions start once we are finishe performing the actions of a transactions [2].  
- Once the client is ready to commit, the coordinator first sends *prepare* to both A and B. This prepare message asks whether
each node is ready to commit.
- A node will only reply "yes" if it can definitely commit the transaction if told so by the
coordinator. This includes writting data to disk (meaning that the node is not afraid of a future crash) and checking for
constraints. 
- After the coordinator recieves responses from all nodes (yes or no), it makes a decision on whether to commit or not. The coordinator
will **first write its decision to disk** before sending it to the nodes. This is the *commit point*. 
- Natually, the coordinator decides to commit if all yes. Else abort. 
- After persisting the decision, the coordinator sends its decision to all nodes. 

Two points of no returns
1. when nodes send "yes", it cannot go back anymore. 
2. when the coordinator makes a decision, that decision must happen. This means 2PC is blocking: if a node crashes after
the coordinator made a decision, the system is stalled until that decision is carried out. 


# Questions
1. How can a node promise that it will "definitely" commit? What if someone just unplugs the power cable?
Ah, the node will only return "yes" if it has **already** written the data to disk. 


# Further Readings
- Read committed isolation

# Sources
[1] https://www.cs.princeton.edu/courses/archive/fall16/cos418/docs/L6-2pc.pdf

[2] https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/
