---
layout: post
title:  "More reliable CRM"
date:   2025-12-29
---


**TL;DR**: 

{% include toc.html toc_levels="2..2" %}


## Background: Salesforce CRMArena 

Customer Relationship Management (CRM) is a process that organizations use to manage, analyze, and improve their interactions with customers (wikipedia).

LLM agents is promising to automate CRM tasks. goal: automates tasks, learns from data, and proactively manages customer interactions. 

salesforce, the biggest crm provider, release a benchmark, [CRMArena](https://arxiv.org/pdf/2411.02305), to evaluate AI agents on realistic tasks grounded on professional work environments.

This benchmark is designed by CRM expert and gives agents access to a realistic Salesforce environment. 

9 types of tasks. we focus on the 5 categories that focus on tool calling (instead of natural langauge understanding, since that is more related to the raw language capability of LLM): Handle Time Understanding, Transfer Count Understanding, Top Issue Identification, Monthly Trend Analysis, Best Region Identification


{:refdef: style="text-align: center;"}
![](/assets/images/blogs/crm_fig1.png){: width="650" } 
{: refdef}

from paper: "systems that can reliably complete tasks showcase direct business value in a popular work environment"

Goal: improve the reliablity of the CRM agent. 

footnote: benchmark already restricts output length. we add restriction on single tool output to 10k characters (bytes). consistent with e.g. codex setup, to avoid too long output


## Agent Failure analysis
in order to improve reliablity, must first understand how they fail right now. (see Evaluate -> Analyze -> Optimize cycle in previous blog post).

Methodology: gpt 5.2 (reasoning = none). split into train and test. 
<!-- add plot on accuracy -->
20*3 trials, total 15 incorrect tasks. 


### Analyzings failures
now we have 15 failures traces. but figuring out the root cause of each failure is a non-trivial task that requires specialized knowledge of the CRM tasks. e.g. how to read the trace file? what is the expected/correct path for each task? common failure patterns etc. 

Example trace:

similar to previous blog post, I built a triage agent specialized at analyzing failure root cause of CRMArena agents. 
to do this, used an open source tool I built, [Socratic](https://github.com/kevins981/Socratic):

<!-- Insert socratic demo here -->

give link to final knowledge base. 

### Failure analysis result
using the triage agent, found that 9 out of 15 failures are due to tool output too long and truncated. 
agent carries on with incomplete info, resulting in wrong final answer. 

e.g. Which states have the quickest case closures in the past 6 months? 
- show trace of such failed trajectory.

in particular, the get_cases() output is large in this task since number of cases is large during this time period. this causes the agent to fail to retrieve all case infos.  
show code snippet of get_case() output

show diagnosis from socratic agent: 

## Optimizing Token Efficiency
taking a closer look at the output of get_cases(). seems like standard json output. 

key insight: agent is natural language processor. use this flexibility.

initial: 77 tokens
{"OwnerId": "005Ws000001xZcHIAU", "CreatedDate": "2021-11-26T09:30:00.000+0000", "ClosedDate": "2021-11-26T16:14:52.000+0000", "AccountId": "001Ws00003LjwjlIAB"}

show tokenizer. 

do the obvious stuff first
1. remove quotation marks 
73
{OwnerId: 005Ws000001xZcHIAU, CreatedDate: 2021-11-26T09:30:00.000+0000, ClosedDate: 2021-11-26T16:14:52.000+0000, AccountId: 001Ws00003LjwjlIAB}

<!-- TODO: add callout block for insights -->

2. remove headers since order is fixed. provide header at start (similar to csv). 
61
{005Ws000001xZcHIAU, 2021-11-26T09:30:00.000+0000, 2021-11-26T16:14:52.000+0000, 001Ws00003LjwjlIAB}

3. remove millisec + timezone if 0 (with fallback)
51
observation: vast majority is 0, and not needed by CRM tasks
{005Ws000001xZcHIAU, 2021-11-26T09:30:00, 2021-11-26T16:14:52, 001Ws00003LjwjlIAB}

4. base delta compression on IDs 
43
{1xZcHIAU, 2021-11-26T09:30:00, 2021-11-26T16:14:52, LjwjlIAB}
with fallback

5. base delta on time
observation: often have same date (addressed on the same day)
36
{1xZcHIAU, 2021-11-26T09:30:00, 16:14:52, LjwjlIAB}

6. token efficient datetime format
original is ISO format. but no reason it HAS to be. 
27
{1xZcHIAU, 20211126-093000, 161452, LjwjlIAB}

Summary Table
Stage	Tokens
Baseline JSON	79
Remove quotes	73
Compress IDs	66
Remove ms / TZ	56
Reformat timestamps	44
Delta encode dates	40
Remove headers	27


from information perspective, these are all lossless. although that doesnt mean its lossless to agents. but from experiments seems lke its good


## abstraction for agents
need to specialized interface for agents.
virtual/physical memory parallel. 
used to have program interfaces: SQL, JSON. e.g. doesnt matter whether its token efficient. just need to be structured and a format agreed on. receiving program process them reliably.

agents are different. token matters. data format directly impacts reliablity (as shown above)

## Discussions and future works

"Just increase output token limit". not a scalable solution: will have longer outputs

Effect of token efficienct tool output is limited. more data... 
- one potential solution: divide and conquer. this instruction is already in the optimized system prompt. 

Other optimizations: only focused on failures due to truncation. already has good perf improvement. harder benchmarks? e.g. crmarena pro

CRM tasks omitted