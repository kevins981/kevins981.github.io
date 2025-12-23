---
layout: post
title:  "Teach Your Agents What You Know (Part 1)"
# subtitle: "A New Interface for Human-Agent Knowledge Transfer"
date:   2025-12-20 12:00:00 -0500
---

[Part 2: using Socratic to improve the success rate of tau-bench airline agent by 10-17%.](https://kevins981.github.io/blogs/teachagent_part2.html)

<!-- This summary should go straight to the point: go to the "what", skip the "why". -->
**TL;DR**: Vertical AI agents often struggle because domain knowledge is tacit and difficult to capture via static system prompts or by retrieving from raw documents. This post proposes treating agents as students: human domain experts teach the agent through iterative, interactive chats while the agent distills domain rules, definitions, and heuristics into a continuously improving knowledge base. 
I've implemented this workflow in an open source prototype [Socratic](https://github.com/kevins981/Socratic).

{% include toc.html toc_levels="2..2" %}


## 3-min Video Demo
<center style="margin: 30px 0;">
<iframe width="650" height="420" src="//www.youtube.com/embed/XbFG7U0fpSU" frameborder="0" allowfullscreen="allowfullscreen">&nbsp;</iframe>
</center>




## The Context Bottleneck
It’s widely known that the key to building effective agents is giving them the [right](https://x.com/karpathy/status/1937902205765607626?lang=en) [context](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents), especially for vertical agents: agents that specialize in a specific domain. 
To reliably perform highly specialized tasks, human experts use intuitions, best practices, and heuristics built over accumulated experience.
<!-- 
Historically, expert knowledge lived primarily in people’s heads, which meant only humans could reliably apply it to create value. That assumption is starting to break. Agents today can already complete some [meaningful](https://arxiv.org/pdf/2510.04374) [real-world](https://arxiv.org/abs/2411.02305) [work](https://spider2-sql.github.io/).
Organizations already contain enormous amounts of valuable human knowledge, such as product and system design, operational playbooks, strategy, legal and medical workflows, and more. Much of the excitement around agents is really excitement about unlocking that corpus. -->
However, transferring this expert knowledge from humans to agents is not easy. This knowledge is often fragmented and unstructured. More importantly, domain expertise requires a deep understanding of principles, heuristics, and edge cases beyond memorizing facts. So **how can we transfer such knowledge from humans to agents?**


{:refdef: style="text-align: center;"}
![](/assets/images/blogs/teachagent_part1_fig1.png){: width="450" } 
{: refdef}

## Existing Approaches
<!-- Existing approaches to human–agent knowledge transfer and what they miss. -->
Two common approaches for transferring knowledge are prompt engineering and information retrieval.

### 1. Prompt Engineering (aka. Write the System Prompt by Hand)

In this approach, a domain expert crafts a detailed system prompt that encodes the relevant policies and procedures. A canonical example is an airline customer support agent whose system prompt includes airline rules for cancellations and refunds (e.g., [tau-bench](https://arxiv.org/abs/2406.12045)).

This can work, but it has two costs. First, it requires an expert at *both* the domain and LLM prompt engineering. Typically this person is difficult to find, since the domain can be unrelated to LLMs (e.g., legal, health care, construction).
<!-- In that case, a domain expert must collaborate with another LLM expert (e.g. a forward deployed engineer) to perform the knowledge transfer. -->
Second, writing system prompts demands that experts enumerate their knowledge up front. But human expertise is often tacit and difficult to articulate (["We can know more than we can tell"](https://en.wikipedia.org/wiki/Polanyi%27s_paradox)).
<!-- If you ask me to write a document that includes all the knowledge related to my current work, I would probably do a poor job and imss important details. -->
We don't naturally operate by listing all our rules and heuristics; we apply them in response to specific situations.

### 2. Runtime Information Retrieval (e.g. RAG / File-based Agentic Search)

In this approach, the agent retrieves from a given set of documents and reasons over the information it finds. Practically, this looks like: a human provides a set of potentially relevant artifacts (design docs, code, meeting notes, emails). At runtime, the user asks the agent a question. The agent searches over the artifact corpus, retrieves relevant parts, and synthesizes an answer.

This is a good fit for "facts lookup" tasks, where the answer can be found directly somewhere in the source documents. It's less reliable when the domain knowledge is complex, because the hardest part is not *finding* information, but correctly *understanding and applying* domain knowledge.

An analogy: it’s like onboarding a junior engineer by handing them a folder of documents and saying "good luck." The most critical part of knowledge transfer, synthesizing this "raw" information into usable knowledge, is pushed entirely onto the learner. 
This is the "document dump" approach: dump your docs and hope the agent works.


{:refdef: style="text-align: center;"}
![](/assets/images/blogs/teachagent_part1_fig2.png){: width="450" } 
{: refdef}

## Transferring Knowledge Through _Teaching_
<!-- Introduce the proposed solution at the conceptual level. -->
Humans transfer knowledge to each other all the time, commonly through _teaching_: "the practice implemented by a teacher aimed at transmitting skills to a learner" ([Wikipedia](https://en.wikipedia.org/wiki/Teaching)).
Could this be a viable way to transfer knowledge to agents?
The human expert acts as the *teacher* and the agent as the *student*. 

I’ll focus on two properties that make teaching effective:
- **Interactive:** For effective learning, both the teacher and student should contribute to the process. The teacher teaches new concepts to the student (e.g. lectures, exams). The student digests the materials asks clarifying questions to enhance their understanding (e.g. in-class questions, office hours).
- **Iterative:** Teaching happens across multiple sessions. The student's knowledge evolves over time.

These two properties motivate the design of a 'student agent' system that facilitates this teaching process, which I'll describe next.


## A Teacher-Student System Prototype 
<!-- Talk about what was built. Keep it concrete; avoid implementation details. -->
To test this idea, I prototyped a system called [Socratic (open source, Apache license)](https://github.com/kevins981/Socratic).
A video demo is shown at the beginning of this blog.

The inputs to the system are source documents. In practice, these can be anything that contain domain knowledge: design docs, code, meeting notes, daily logs, past chat transcripts, etc. These are things that you previously would have put into a RAG/document dump.
Teaching happens through _chat sessions_ between the human user and the agent student.
The agent maintains and updates a knowledge base (in plain text) based on information received from the human teacher.

The output of this process is the knowledge base, capturing the distilled rules, definitions, and strategies that emerged during the teaching process.
Practically, it can be exported as `AGENTS.md`, uploaded into a chat UI, or stored alongside a codebase, etc.


{:refdef: style="text-align: center;"}
![](/assets/images/blogs/teachagent_part1_fig3.png){: width="450" } 
{: refdef}

As a concrete example, I used Socratic to build a knowledge base about Socratic itself: the problem it tries to solve, the design decisions I made, and how those decisions evolved over time. The source documents include the Socratic source code, my design logs, and brainstorming conversations with ChatGPT. The resulting knowledge bases are available [here](https://github.com/kevins981/Socratic/tree/main/docs).

Following our insight that effective teaching is interactive, Socratic implements two methods to initiate knowledge transfer:
1. **Teacher-initiated teaching.** The human user instructs the agent on a topic to learn. E.g., "Let's look at how Socratic stores the knowledge base." The agent studies the relevant documents and proposes updates to the knowledge base. The human reviews, corrects, and approves these updates. 
2. **Student-initiated learning.** The agent studies the existing knowledge base and source documents to identify inconsistencies, gaps, or ambiguities and generates questions for the human. This process starts _without_ a specific human instruction. E.g., Agent: "The current knowledge base mentions that we assume X, but a source document seems to assume Y. Please clarify which one is correct."

Socratic is naturally iterative. The student agent updates the knowledge base over each chat session. As the knowledge base evolves, the student agent's understanding of the domain evolves as well. 

## Vision: Teaching as the New Training
The key question is: how much can we improve an agent's practical competence through repeated teaching?
Ideally, the more we teach an agent, the "better" the knowledge base becomes, and the more reliably the agent can act within its domain.

<!-- TODO: add picture comparing gradient descent vs. teaching sessions. -->

This framing also highlights an important question: what does it mean to be a *good teacher* for an agent? If teaching becomes a core workflow, then "agent education" might become a real skill. E.g. deciding what to teach first, picking the right examples, probing for misconceptions.

{:refdef: style="text-align: center;"}
![](/assets/images/blogs/teachagent_part1_fig4.png){: width="450" } 
{: refdef}

Of course, while this vision is exciting, it is unclear how much performance we can squeeze out of existing agents through teaching.
[In Part 2 of this blog series](https://kevins981.github.io/blogs/teachagent_part2.html), I will share a concrete use case of Socratic, optimizing an airline customer service agent, and evaluate its effectiveness. 



<!-- 
## Related efforts
I see this experiment as adjacent to a few active areas:

- **Skills and tool-use scaffolding:** systems that build competence by giving agents structured procedures and interfaces.
- **Agentic memory systems:** approaches that store and retrieve agent experiences and distilled knowledge over time.
- **Continual learning (non-weight update):** workflows that improve behavior by updating prompts, knowledge stores, and constraints rather than model parameters. -->

