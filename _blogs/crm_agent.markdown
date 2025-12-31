---
layout: post
title:  "Towards More Reliable CRM Agent"
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

All nine task categories and example tasks are show in the figure below (from CRMArena paper).

{:refdef: style="text-align: center;"}
![](/assets/images/blogs/crm_fig1.png){: width="650" } 
{: refdef}

The agent is given an input task such as "Which agent has the shortest handle time during the past 6 month?". 
The agent is also provided a fixed set of functions designed specifically for the Salesforce CRM environment. A few examples (see [the full paper](https://arxiv.org/pdf/2411.02305) Table 7 for full list):

| Functions | Description |
|---|---|
| `get_cases(start_date, end_date, agent_ids, case_ids, order_item_ids, issue_ids, statuses)`  | Retrieves cases based on various filtering criteria. |
| `calculate_average_handle_time(cases)`  | Calculates the average handle time for each agent based on a list of cases. |
| `calculate_region_average_closure_times(cases)`  | Calculates the average closure times for cases grouped by region (shipping state). |
| `get_agent_handled_cases_by_period(start_date, end_date)`  | Retrieves the number of cases handled by each agent within a specified time period |


Based on the given task, the agent can choose to invoke the available functions for a maxmimum of 20 turns. Correctness is evaluated based on the final answer provided by the agent.



Achieving high performance on this benchmark is important. As the CRMArena paper notes, "systems that can reliably complete tasks showcase direct business value in a popular work environment." So the central goal of this post is to **improve the reliability of such a CRM agent.**



## Agent Evaluation and Failure Analysis
To improve reliability, we first need to understand how and why agents fail. This requires a systematic analysis of their failure modes. I followed an "Evaluate -> Analyze -> Optimize" cycle, starting with a baseline measurement of an agent's performance.

I randomly select a set of 50 CRMArena tasks, which were split into a training set for analysis (20 tasks) and a held-out test set for final evaluation (30 tasks). 
I use a GPT-5.2 (reasoning effort = None) agent and ran it on the training set for 3 trials. the agent failed on 15, giving us a baseline average success rate of 75% and a concrete set of failures to diagnose.

{:refdef: style="text-align: center;"}
![](/assets/images/blogs/crm_baseline.svg){: width="300" } 
{: refdef}

The resulting knowledge base for triage agent is [here](https://gist.github.com/kevins981/09c03232f6b8ee8070ea3c2358bf961c).


### Triaging Failure Root Causes
We now have 15 failure traces to examine. Diagnosing the root cause of each failure requires specialized knowledge of CRM tasks: how to interpret trace files, what the correct tool-call sequence should be, and which failure patterns appear frequently.

To systematically analyze these failures, I built a triage agent specialized in diagnosing CRMArena agent errors. This triage agent was constructed using [Socratic](https://github.com/kevins981/Socratic), an open-source tool I developed for iterative knowledge-base refinement. The triage agent encodes domain knowledge about CRMArena tasks and common failure modes, enabling it to automatically identify root causes from execution traces.

<!-- Insert socratic demo video here -->

### Failure Analysis

The triage agent identified a dominant failure mode: 9 out of 15 failures stem from **tool output truncation**. This pattern reveals a fundamental constraint: CRM tasks often require aggregating data across hundreds or thousands of records—case histories, account logs, support tickets—and when the tool output exceeds available token budget, the agent receives incomplete information and computes incorrect results.

**Why truncation occurs.** Consider the task: "Which states have the quickest case closures in the past 6 months?" To answer this, the agent must:
1. Retrieve all cases opened in the six-month window.
2. Compute average closure time per state.
3. Identify the minimum.

In a realistic CRM system, a six-month period can contain hundreds of cases. In this example, 198 cases exist in the query window. When `get_cases()` returns these records in standard JSON format, the output size exceeds the 10k character limit[^1], triggering truncation.

**Why this is a general problem.** Large result sets are not edge cases—they are the norm for analytical CRM tasks. Monthly trend analysis requires time-series data across accounts; top-issue identification needs aggregation over ticket categories; regional performance comparisons demand multi-dimensional grouping. Any non-trivial query on a production CRM database produces outputs measured in thousands of records and hundreds of kilobytes.

Traditional programs handle this via pagination, streaming, or SQL aggregation—mechanisms that operate outside the LLM context window. Agents face a different constraint: they reason over tool outputs using tokens[^2]. When token costs per record are high, even moderately-sized datasets exceed context limits, forcing the agent to operate on partial data.

<details markdown="1">
<summary markdown="span"><u>Example failure trace</u></summary>
<script src="https://gist.github.com/kevins981/75e3d7283830d3148695d26efe9a45af.js"></script>
</div>
</details>
&nbsp;

In this specific failed task trace, at line 97, `get_cases()` returns 198 cases in JSON format. The output is truncated mid-stream. The agent computes regional averages from the incomplete set, yielding `NJ` as the fastest state instead of the correct answer, `OH`.

The triage agent's diagnosis:
```markdown
**Surface-level failure**
- The run submitted `respond(\"NJ\")`, but `gt_answer` is `OH`.
- The `get_cases` tool output is explicitly truncated, 
  so downstream aggregation is based on an incomplete case set.

**Most likely root cause**
- Root cause is **tool output truncation causing incorrect regional 
  averages/min selection**: the agent computed 
  `calculate_region_average_closure_times` from a partial list of cases, 
  so the "fastest" state came out as `NJ` instead of the true `OH`.
- This matches KB pattern CF-001 ("Tool output truncation leads to 
  incomplete aggregates").
```

**The core problem is token inefficiency, not absolute data size.** The agent can reason over 198 cases if each case is represented compactly. The issue is that standard JSON formatting consumes 77 tokens per case (repeated keys, quoted strings, verbose timestamps). For 198 cases, this totals ~15,000 tokens—well over budget. 

<!-- This reframes the problem: **truncation is not a hard limit to be raised; it is a signal that the data representation is wasteful.** The next section shows how to address this. -->



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

We apply the same idea to several other functions to make them more token efficient. The full set of optimizations can be found in [this repository](https://github.com/kevins981/CRMArena_socratic), which contains the repo used for evaluations in this blog post.

## Improved Results


With the above token efficiency optimizations, the train and test sets saw a 18% and 2% success rate improvements respectively. Overall, the success rate improves from 85.3% to 94.0%. 
All results are average of three trials.

{:refdef: style="text-align: center;"}
![](/assets/images/blogs/crm_optimized.svg){: width="450" } 
{: refdef}

## Rethinking System Interfaces for Agents
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

## Discussions
*"Just increase output token limit"*. One may argue that we can just increase the output token limit from 10k characters. But this is not a scalable solution: it shifts rather than solves the truncation boundary. Larger limits mean longer outputs to parse and reason over, which increases latency and error rates even when truncation is avoided. The fundamental problem—that token cost scales with verbosity—remains.

*Limitations of token-efficient output*. Token compression defers but does not eliminate the scaling problem. If the dataset grows from 200 to 2000 cases, even compact representations will eventually hit token limits. Two strategies mitigate this:
- Divide-and-conquer: the optimized system prompt already includes instructions to partition large queries and aggregate results. Token-efficient output makes each partition more informative, improving the accuracy of partial views.
- Decomposition and delegation: architectures like subagents or tool-generating agents benefit from compact intermediate results, as they must parse, validate, and reason over outputs from delegated calls. Agent-oriented interfaces amplify the benefit of these techniques rather than replace them.

*"We will have LLMs with longer context length"*. Longer context windows address capacity but introduce two persistent costs. First, context dilution: as context grows, retrieval accuracy degrades (the "lost in the middle" phenomenon), which directly harms reliability on multi-step reasoning tasks. Second, cost scaling: token processing cost grows with context size, so achieving high reliability at minimal token cost remains valuable even as windows expand. Token efficiency is an orthogonal optimization.

*Scope of evaluation*. This study focuses on truncation-induced failures, which account for a measurable fraction of errors in CRMArena. Other failure modes (reasoning errors, tool selection mistakes) are not addressed here. Extending the approach to harder benchmarks (e.g., CRMArena Pro) or tasks requiring complex multi-tool orchestration would test the generality of agent-oriented interfaces beyond the CRM domain.


## Footnotes
[^1]: benchmark already restricts output length. we add restriction on single tool output to 10k characters (bytes). consistent with e.g. codex setup, to avoid too long output. 

[^2]: While alternative architectures exist (subagents, code-generating agents), token efficiency benefits all approaches: subagents must parse summaries or validate results, and code-generating agents inspect samples and debug traces. For the analytical aggregation tasks in CRMArena, agents primarily use direct reasoning over outputs. 