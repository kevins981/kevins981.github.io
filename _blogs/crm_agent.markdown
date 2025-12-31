---
layout: post
title:  "Twoards More Reliable CRM Agent"
date:   2025-12-29
---


**TL;DR**: 

{% include toc.html toc_levels="2..2" %}


## Background: The Challenge of Reliable CRM Agents

Customer Relationship Management (CRM) systems are the operational backbone for how organizations manage and analyze customer interactions. The promise of Large Language Model (LLM) agents to automate CRM tasks is significant: an ideal agent would learn from data, automate routine work, and proactively manage customer relationships to improve satisfaction and efficiency.

Salesforce introduced [CRMArena](https://arxiv.org/pdf/2411.02305), a benchmark that evaluates how well agents can perform professional CRM tasks. It provides agents with access to a realistic Salesforce platform and a set of tasks designed by CRM experts.

CRMArena specifies nine task categories. For this post, we narrow our focus to five categories that heavily rely on an agent's ability to use tools, as opposed to pure natural language understanding:
- Handle Time Understanding
- Transfer Count Understanding
- Top Issue Identification
- Monthly Trend Analysis
- Best Region Identification

All nine task categories and example tasks are show below (figure from CRMArena paper):

{:refdef: style="text-align: center;"}
![](/assets/images/blogs/crm_fig1.png){: width="650" } 
{: refdef}

Achieving high performance on this benchmark is important. As the CRMArena paper notes, "systems that can reliably complete tasks showcase direct business value in a popular work environment." So the central goal of this post is to **improve the reliability of such a CRM agent.**



## Agent Failure analysis
To improve reliability, we first need to understand how and why agents fail. This requires a systematic analysis of their failure modes. I followed an "Evaluate -> Analyze -> Optimize" cycle, starting with a baseline measurement of an agent's performance.

I used a GPT-5.2 (reasoning effort = None) agent and ran it on a set of CRMArena tasks, which were split into a training set for analysis and a held-out test set for final evaluation. Out of 60 trials on the training set (20 tasks run 3 times each), the agent failed on 15, giving us a baseline success rate of 75% and a concrete set of failures to diagnose.

{:refdef: style="text-align: center;"}
![](/assets/images/blogs/crm_baseline.svg){: width="300" } 
{: refdef}

The resulting knowledge base for triage agent is [here](https://gist.github.com/kevins981/09c03232f6b8ee8070ea3c2358bf961c).


### Analyzing failures

We now have 15 failure traces to examine. Diagnosing the root cause of each failure requires specialized knowledge of CRM tasks: how to interpret trace files, what the correct tool-call sequence should be, and which failure patterns appear frequently.

To systematically analyze these failures, I built a triage agent specialized in diagnosing CRMArena agent errors. This triage agent was constructed using [Socratic](https://github.com/kevins981/Socratic), an open-source tool I developed for iterative knowledge-base refinement. The triage agent encodes domain knowledge about CRMArena tasks and common failure modes, enabling it to automatically identify root causes from execution traces.

<!-- Insert socratic demo video here -->

<!-- Insert link to final triage agent knowledge base here -->

### Failure analysis result

The triage agent identified a dominant failure mode: 9 out of 15 failures stem from tool output truncation[^1]. When a tool returns a result that exceeds the output length limit, the agent receives only a partial response. The agent then proceeds with incomplete information, leading to incorrect final answers.

Consider the task: "Which states have the quickest case closures in the past 6 months?" The agent must retrieve all cases from a six-month window, compute average closure time by state, and identify the minimum.

<details markdown="1">
<summary markdown="span"><u>Example CRM agent failure trajectory</u></summary>
<script src="https://gist.github.com/kevins981/75e3d7283830d3148695d26efe9a45af.js"></script>
</div>
</details>
&nbsp;

At line 97 of the trace, the `get_cases()` tool returns a large result because this specific six-month period contains a large number of cases (198 cases in this specific example). The output is truncated, so the agent computes regional averages from an incomplete case set.

The triage agent provides an accurate diagnosis:
```markdown
**Surface-level failure**
- The run submitted `respond(\"NJ\")`, but `gt_answer` is `OH`.
- The `get_cases` tool output is explicitly truncated, 
  so downstream aggregation is based on an incomplete case set.

**Most likely root cause (please confirm)**
- Root cause is **tool output truncation causing incorrect regional averages/min selection**: 
  the agent computed `calculate_region_average_closure_times` from a partial list of cases, 
  so the “fastest” state came out as `NJ` instead of the true `OH`.
- This matches KB pattern CF-001 (“Tool output truncation leads to incomplete aggregates”).
``` 


## Optimizing Token Efficiency

The `get_cases()` output uses standard JSON formatting. Each case object contains four fields with quoted keys and values, following conventional API design. There seems to be no problem. 

However, agents fundamentally differ from traditional programs: they are natural language processors. This creates an opportunity. Unlike structured parsers that require rigid formats, agents can interpret flexible representations. Stepping into shoe of the agent (cite anthropic blog/video), we can ask: what information in the JSON output is redundant and can be removed?

Since each case follows a fixed structure, optimizations applied to a single case compound across the entire output. We start with the original object for a single case (77 tokens):

`{"OwnerId": "005Ws000001xZcHIAU", "CreatedDate": "2021-11-26T09:30:00.000+0000", "ClosedDate": "2021-11-26T16:14:52.000+0000", "AccountId": "001Ws00003LjwjlIAB"}`

The below figure shows a visualization of how this case object is tokenized by GPT models (Source: https://platform.openai.com/tokenizer). In other words, these tokens are what the underlying LLM model sees.
The token breakdown can gives us a good understanding of how to improve the token efficiency.

{:refdef: style="text-align: center;"}
![](/assets/images/blogs/crm_token1.png){: width="600" } 
{: refdef}

**Step 1: Remove quotation marks.** JSON requires quotes for syntactic parsing, but agents do not. Removing quotes reduces token count to 74 without information loss:
`{OwnerId: 005Ws000001xZcHIAU, CreatedDate: 2021-11-26T09:30:00.000+0000, ClosedDate: 2021-11-26T16:14:52.000+0000, AccountId: 001Ws00003LjwjlIAB}`

**Step 2: Remove field names from individual entries.** When field order is stable, repeating keys is redundant. We declare the format once at the top (similar to CSV headers), then provide only values. This reduces per-case cost to 61 tokens:
`{005Ws000001xZcHIAU, 2021-11-26T09:30:00.000+0000, 2021-11-26T16:14:52.000+0000, 001Ws00003LjwjlIAB}`

{:refdef: style="text-align: center;"}
![](/assets/images/blogs/crm_token2.png){: width="600" } 
{: refdef}

**Step 3: Omit milliseconds and timezone when zero.** The timestamps have `.000+0000` suffixes. The default convention can be to omit these, with a fallback to full precision when values differ from zero. Result: 51 tokens: 
`{005Ws000001xZcHIAU, 2021-11-26T09:30:00, 2021-11-26T16:14:52, 001Ws00003LjwjlIAB}`

{:refdef: style="text-align: center;"}
![](/assets/images/blogs/crm_token3.png){: width="600" } 
{: refdef}

**Step 4: Base-delta compression for IDs.** Taking a closer look at the AgentID and OwnerIDs, most IDs share an 11-character prefix within each ID type. For AccountIds, the prefix is `001Ws00003`; for OwnerIds, it is `005Ws000001`. We declare each prefix once, then show only the 7-character suffix per case. This yields 43 tokens per case: `{xZcHIAU, 2021-11-26T09:30:00, 2021-11-26T16:14:52, LjwjlIAB}`

{:refdef: style="text-align: center;"}
![](/assets/images/blogs/crm_token4.png){: width="600" } 
{: refdef}

This technique borrows from base-delta compression in CPU cache design: when values share a common prefix, storing a base plus deltas is more efficient than storing full values. The optimal prefix length is empirical and depends on the ID distribution in the specific CRM database.

**Step 5: Base-delta compression for timestamps.** Cases opened and closed on the same day share year-month-day components. When `ClosedDate` matches `CreatedDate` in date, we show only the time portion for `ClosedDate`. 

Result: 36 tokens.

`{xZcHIAU, 2021-11-26T09:30:00, 16:14:52, LjwjlIAB}`

{:refdef: style="text-align: center;"}
![](/assets/images/blogs/crm_token5.png){: width="600" } 
{: refdef}

**Step 6: Compact timestamp format.** ISO 8601 uses `YYYY-MM-DDTHH:MM:SS`, which tokenizes inefficiently due to hyphens and colons. Replacing with `YYYYMMDD-HHMMSS` reduces tokens to 27:

`{xZcHIAU, 20211126-093000, 161452, LjwjlIAB}`

{:refdef: style="text-align: center;"}
![](/assets/images/blogs/crm_token6.png){: width="600" } 
{: refdef}

We achieve a 2.9× token reduction per case:

| Optimization | Tokens | Improvement |
|---|---|---|
| Baseline JSON | 79 | 1× |
| Remove quotes | 74 | 1.07× |
| Remove field names | 61 | 1.29× |
| Omit default ms/TZ | 51 | 1.55× |
| Base-delta IDs | 43 | 1.84× |
| Base-delta timestamps | 36 | 2.19× |
| Compact timestamp format | 27 | 2.93× |

These optimizations are information-preserving: no data is lost. Whether they are lossless from the agent's perspective—whether the agent can interpret the compressed format as reliably as JSON—requires empirical validation. 

Note: the 2.9x reduction is the best case scenario. Several optimizations are oppotunistic. E.g. Base-delta timestamps do not apply if createdDate and closedDate don't share the same day. 

## Improved Results
{:refdef: style="text-align: center;"}
![](/assets/images/blogs/crm_optimized.svg){: width="450" } 
{: refdef}

These improvements come from 

## Abstractions for Agents
Agents benefit from interfaces designed for their constraints. Traditional program-oriented interfaces (e.g., SQL, JSON) optimize for rigid parsing and stable schemas; token efficiency is largely irrelevant because a compiler or parser consumes bytes, not tokens. In contrast, agents operate over tokens, so representation directly affects reliability—as our CRM results show, verbose formats increase truncation risk and degrade task accuracy.

Conceptually, we can distinguish two interface families:
- Program-oriented: syntax-first, byte-centric, assumes a deterministic parser.
- Agent-oriented: token-aware, semantics-first, resilient to partial outputs.

What should an agent-oriented interface provide?
- Token-aware representation: minimize token load while preserving information. Declare headers once, use base–delta for repeated prefixes, and compact timestamps to avoid unnecessary delimiters.
- Declarative structure with flexible surface form: state the schema up front, then allow compact rows that reuse the declaration. The surface form can vary as long as semantics remain clear.
- Progressive disclosure: paginate or chunk large results and let the agent request refinement (e.g., "next 200 cases" or "only suffixes for IDs"). This reduces truncation-induced errors.
- Robustness to partial results: include counts, continuation markers, or checksums so the agent can detect truncation and request the missing portion.
- Cost-bounded queries: expose size estimates and budget hints so the agent can plan divide-and-conquer strategies before issuing expensive calls.

This is not a new idea in systems. Operating systems introduced virtual memory so processes reason about stable pages rather than raw physical layout; the abstraction improves scalability and predictability. Similarly, an agent-first interface would hide gratuitous verbosity and expose predictable, token-bounded views of data.

Operationally, the benefits are testable: fewer truncation failures, lower tokens-per-answer at equal accuracy, and higher success rates on benchmarks like CRMArena. The CRM case study illustrates the potential; designing general-purpose agent-oriented interfaces is a worthwhile next step.

## Discussions and future works

"Just increase output token limit". not a scalable solution: will have longer outputs

Effect of token efficienct tool output is limited. more data... 
- one potential solution: divide and conquer. this instruction is already in the optimized system prompt. 

Other optimizations: only focused on failures due to truncation. already has good perf improvement. harder benchmarks? e.g. crmarena pro

CRM tasks omitted: task types, also not all tasks evaluated. some are harder than others (e.g. more data required). train/test sets randomly selected. 


## Footnotes
[^1]: benchmark already restricts output length. we add restriction on single tool output to 10k characters (bytes). consistent with e.g. codex setup, to avoid too long output. 