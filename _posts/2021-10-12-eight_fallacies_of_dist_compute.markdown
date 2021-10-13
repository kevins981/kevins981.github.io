---
title:  "Eight Fallacies of Distributed Computing"
---

# Summary
Programming in a local, monolithic environment is very different from developing distributed
applications such as microservices. This aritcle discusses 8 common pitfalls in distributed 
development. 

# Background
## Monolith vs Microservices
Monolith: a geological feature consisting of a single massive stone
This is a monolith:

{:refdef: style="text-align: center;"}
![](/assets/images/posts/eight_dist_compute_fallacies/monolith1.jpg){: width="550" }
{: refdef}

But this is also a monolith. Ok.
{:refdef: style="text-align: center;"}
![](/assets/images/posts/eight_dist_compute_fallacies/monolith2.jpg){: width="550" }
{: refdef}

Back to distributed systems. What is a monolith in this context? 
- Traditional, default model for creating software
- Application is built as one single unit. Typically consists of client-side user interface, server
side application, and database.
- Developers need to change the entire code base for application updates.
- The main disadvantage is scalability.

Microservices
- Split up the application into independent, modular services
- Services communicate through APIs
- Each service can be updated and scaled independently
- Ex. we can have one service for payment processing, one for accessing the backend database

# Details
The main difference between monolith and microservice development is that in a local setting, 
the memory space reside on the same physical machine. This means that we can modify and access
the data immediately and deterministically. This assumption needs to be reconsidered for distributed
developments.

**Fallacy #1: The network is reliable**
- In local setting, functions always deterministically return something (error or not). 
- Distributed apps should account for the potential of network failures.

**\#2: Latency is zero**
- Local memory accesses take ns-us, while remote accesses can take ms. When developing and debugging 
locally, this difference in latency is not reflected. The customer experiences the slower time.
- Access latencies are also not uniform. Long tail latencies can even dominate overall performance.

**\#3: Bandwidth is infinite**
- Need to consider bandwidth limitations and how they affect user experience. Ex. avoid 
blindly sending a very large chunk of data.

**\#4: The network is secure**
- The network is not secure. Microservices exacerbates this since more links are exposed.

**\#5: The network topology does not change**
- The arrangement of nodes change all the time. Nodes are added, removed, reconfigured etc.

**\#6: There is one administrator**
- In a distributed setting, there are usually more than one person who have the power
to perform administrative tasks. If not coordinated correctly, unexpected configurations
can damage the system.

**\#7: Transport cost is zero**
- The larger your application reaches, the more resource cost you need to pay. This cost includes
both compute cost and data transportation cost. Zoom reported that its platform was moving 
217 Petabytes of data per month. This would have costed Zoom 11 million dollars to just transporting
the data.

**\#8: The network is homogeneous**
- Cloud computing providers such as Amazon can most likely achieve homogeneity. However, if 
the end user if ex. on a mobile LTE network, the developer needs to pay attention to the network
variations.

# Comments
- This was a pretty high level article, which is good when I am getting started with a new topic :)
- Would have been nice to show some real code examples that explicitly considers these fallacies.

# Sources
https://sookocheff.com/post/distributed-systems/unpacking-the-eight-fallacies-of-distributed-computing/

https://www.vox.com/culture/22062796/monoliths-utah-california-romania

https://en.wikipedia.org/wiki/Monolith#/media/File:Uluru,_helicopter_view,_cropped.jpg
