---
layout: post
title:  "Teach Your Agents What You Know (Part 1)"
# subtitle: "A New Interface for Human-Agent Knowledge Transfer"
date:   2025-12-20 12:00:00 -0500
---

<!-- This summary should go straight to the point: go to the "what", skip the "why". -->
(TODO: this summary needs to be better. 3 sentences would be good)
TL;DR: A new way to transfer specialized domain knowledge from human experts to agents: teach the agent through conversations.

<!-- TODO: insert demo short video here. Keep this overview before diving into details. -->

This post is Part 1. It frames the knowledge-transfer problem, explains why common non-training approaches often fall short, and describes an early prototype I built to test a “teacher–student” workflow. Part 2 and Part 3 will focus on the Socratic teaching loop (including a more structured, question-driven approach) and the design details.


## The “Context” Bottleneck
It’s widely understood that one of the most important parts of building effective agents is giving them the [right](https://x.com/karpathy/status/1937902205765607626?lang=en) [context](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents).
Context becomes even more important for vertical agents: agents that specializes in a specific domain. Human experts use intuitions, best practices, heuristics built over experiences to make good decisions.

Historically, that kind of expert knowledge lived primarily in people’s heads, which meant only humans could reliably apply it to create value. That assumption is starting to break. Agents today can already complete some [meaningful](https://arxiv.org/pdf/2510.04374) [real-world](https://arxiv.org/abs/2411.02305) [work](https://spider2-sql.github.io/).
Organizations already contain enormous amounts of valuable human knowledge, such as product and system design, operational playbooks, strategy, legal and medical workflows, and more. Much of the excitement around agents is really excitement about unlocking that corpus.

The catch is that transferring this intricate expert human knowledge to agents is hard. It’s fragmented and unstructured, it changes daily, and the most important parts are often not "facts" but judgment: edge cases, interpretations, and rules of thumb. So the core question becomes:
**How can we transfer domain expertise from a human to an agent?**


## Existing Approaches
<!-- Existing approaches to human–agent knowledge transfer and what they miss. -->
Two common approaches to transfefr knowledge: prompt engineering and infomation retrieval.

### 1. Prompt Engineering (aka. Write the System Prompt by Hand)

In this approach, a domain expert crafts a detailed system prompt that encodes the relevant policies and procedures. A canonical example is an airline customer support agent whose system prompt includes airline rules for cancellations and refunds (e.g., [tau-bench](https://arxiv.org/abs/2406.12045)).

This can work, but it has two costs. First, it requires an expert at *both* the domain and LLM prompt engineering. Typically this person is difficult to find, since the domain can be unrelated to LLMs (e.g. legal, health care, construction).
In that case, a domain expert must collaborate with another LLM expert (e.g. a forward deployed engineer) to perform the knowledge transfer.

Second, writing system prompts demands that experts enumerate their knowledge upfront. But human expertise is often tacit: knowledges that are difficult to articulate (["We can know more than we can tell"](https://en.wikipedia.org/wiki/Polanyi%27s_paradox)).
If you ask me to write a document that includes all the knowledge related to my current work, I would probably do a poor job and imss important details.
We don't naturally operate by listing all our rules and heuristics; we apply them in response to specific situations.

### 2. Runtime Information Retrieval (e.g. RAG / File-based Agentic Search)

In this approach, the agent retrieves from a given set of documents and reasons over the information it finds. Practically, this looks like: human provides a set of potentially relevant artifacts (design docs, code, meeting notes, emails). At runtime, the user asks the agent a question. Agent searches over the artifact corpus, retrieves relevant parts, and synthesize an answer.

This is a good fit for "look-up" tasks, where the answer can be found directly somewhere in the artifacts. It's less reliable when the domain knowledge is complex, because the hardest part is not *finding* information, but correctly *understanding and applying* the knowledge.

An analogy: it’s like onboarding a junior engineer by handing them a folder of documents and saying "good luck." The most critical part of knowledge transfer,  synthesizing these "raw" information into usable knowledge, is pushed entirely onto the learner. 
I call this the "document dump" approach: dump your docs and hope the agent works.



## Transferring Knowledge Through _Teaching_
<!-- Introduce the proposed solution at the conceptual level. -->
The solution I’m proposing is to **teach the agent**.

Taking a step back and looking at the task of knowledge transfer: A human expert already knows how to do something well. We want an agent to develop a similar capability.

<!-- TODO: add image: human brain → agent brain. -->

Humans perform this transfer process all the time: through teaching. 
Teaching is defined as "the practice implemented by a teacher aimed at transmitting skills to a learner." ([Wikipedia](https://en.wikipedia.org/wiki/Teaching)).
Here, the human expert is the *teacher* and the agent is the *student*. 
A useful mental model is: the agent is like a very strong college student: quick to learn, broadly knowledgeable, but missing the specialized context of your domain. (This analogy is imperfect as LLMs are still fundamentally different from humans, but it’s good enough for the purpose of this.)

Two properties that make teaching effective:
- **Interactive:** For effective learning, both the teacher and student should contribute to the process. The teacher teaches new concepts to the student (e.g. lectures, exams). The student digests the materials asks clarifying questions to enhance their understanding (e.g. in-class questions, office hours). Both are needed for effective knowledge transfer.
- **Iterative:** Teaching happens across mulitple sessions instead of a single-shot. The student's knowledge evolves over time. As the student becomes more competent, teacher adapts the teaching strategy (e.g. intro vs. advanced courses). 

The goal is to build a "student agent" system to faciliate this teaching process, which I describe next.


## A Prototype 
<!-- Talk about what was built. Keep it concrete; avoid implementation details. -->
To test this idea, I built a system called [Socratic (open source, Apache license)](https://github.com/kevins981/Socratic).
A video demo is at the beginning of this blog.

The inputs to the system are raw source documents. In practice, this can be anything that contains domain knowledge: design docs, code, meeting notes, daily logs, past chat transcripts etc. These are things that you previous would have put into a RAG/document dump.
Teaching happens through _chat sessions_.
The student agent maintains and updates a knowledge base (in plain text) based on information received from the human teacher.

The output of this process is the knowledge base, capturing the distilled rules, definitions, and strategies that emerged during the teaching process.
Practically, it can be exported as `AGENTS.md`, uploaded into a chat UI, or stored alongside a codebase etc.

<!-- TODO: insert diagram showing input → teaching loop → KB output. -->

As a concrete example, I used Socratic to build a knowledge base *about Socratic itself*: the problem it tries to solve, the design decisions I made, and how those decisions evolved over time. The source documents include the Socratic source code, my design logs, and brainstorming conversations with ChatGPT. The resulting knowledge bases are available [here](https://github.com/kevins981/Socratic/tree/main/docs).

Following our insight about effective teaching (interactive), Socratic implements two ways to initiate knowledge transfer:
1) **Teacher-initiated teaching.** The human decides what to focus on. The agent studies the relevant documents and proposes structured knowledge. The human then reviews, corrects, and approves. E.g. User: "Let's look at how Socratic stores the knowledge base."
2) **Student-initiated learning ("digest mode").** The agent studies the existing knowledge base and source documents to identify inconsistencies, gaps, or ambiguities and generates questions for the human teacher. E.g. Agent: "The current knowledge base mentions that we assumes X, but a source document seems to assume Y. Please clarify which one is correct".

Socratic is naturally iterative. The student agent updates the knowledge base over multiple chat sessions. As the knowledge base evolves, the student agent's understanding of the doamin evolves as well. 

## Vision: Teaching as the New Training
The key question is: how far can we scale this teaching approach? 
That is, how far can we improve an agent’s practical competence through repeated teaching sessions?
Ideally, the more we teach an agent, the "better" the knowledge base becomes, and the more reliably the agent can act within its domain.

<!-- TODO: add picture comparing gradient descent vs. teaching sessions. -->

This framing also highlights an important question: what does it mean to be a *good teacher* for an agent? If teaching becomes a core workflow, then "agent education" might become a real skill. E.g. deciding what to teach first, picking the right examples, probing for misconceptions.


Of course, while this vision is exciting, its also highly hypothetical. 
It is possible that there is only so much performance we can squeenze out of existing agents throught teaching.
In Part 2 of this blog series, I will share a concrete use case of Socratic: optimizing an airline customer service agent. 
Hopefully this will give us a glimpse of the potential of this approach.



<!-- 
## Related efforts
I see this experiment as adjacent to a few active areas:

- **Skills and tool-use scaffolding:** systems that build competence by giving agents structured procedures and interfaces.
- **Agentic memory systems:** approaches that store and retrieve agent experiences and distilled knowledge over time.
- **Continual learning (non-weight update):** workflows that improve behavior by updating prompts, knowledge stores, and constraints rather than model parameters. -->

