---
layout: post
title:  "Teach Your Agents What You Know (Part 2)"
# subtitle: "A New Interface for Human-Agent Knowledge Transfer"
date:   2025-12-21
---

[Part 1: Motivation and high-level idea of teach-student knowledge transfer.](https://kevins981.github.io/blogs/teachagent_part1.html)

**TL;DR**: I used a teacher-student system ([Socratic](https://github.com/kevins981/Socratic)) to improve the tau-bench airline agent's success rate by 10-17%.
The optimized agent (GPT-5 mini) outperforms the baseline agent using GPT-5.2. This post explains the iterative process of using Socratic for agent failure analysis and context optimization.

{% include toc.html toc_levels="2..2" %}


<!-- explain briefly what is tau-bench and airline agent -->
## Background: Airline Agent and Tau-bench
[Tau-bench](https://github.com/sierra-research/tau-bench) is a benchmark used by almost all major LLM vendors (e.g. [GPT](https://openai.com/index/introducing-gpt-5-2/), [Gemini](https://blog.google/products/gemini/gemini-3/#gemini-3), [Claude](https://www.anthropic.com/news/claude-opus-4-5)) to evaluate the model's agentic capabilities.
[The original paper](https://arxiv.org/pdf/2406.12045) nicely summarizes the benchmark: 
> An agent interacts with database API tools and an LM-simulated user to complete tasks. The benchmark tests an agent’s ability to collate and convey all required information from/to users through multiple interactions, and solve complex issues on the fly while ensuring it follows guidelines laid out in a domain-specific policy document.

{:refdef: style="text-align: center;"}
![](/assets/images/blogs/teachagent_part2_fig1.png){: width="600" } 
{: refdef}


<!-- To evaluate whether the agent has performed a task correctly,  whether the expected functions are called and final database state. in above example, agent must call `cancel_reservation` with specific argument `JK9O19`. -->

## Goal: Optimizing the Airline Agent
In this post, I will use Socratic to improve the task success rate of the airline agent from tau-bench (see the example in the figure above).

I split all tasks into train and test sets (20 and 30 tasks each). All analysis and optimizations are done based on the train set only.
The plot below shows the initial task success rates[^1]. The agent uses GPT-5 mini (no reasoning).


{:refdef: style="text-align: center;"}
![](/assets/images/blogs/teachagent_part2_baseline.svg){: width="300" } 
{: refdef}


## The Cycle of Agent Optimization
Conceptually, optimizing an agent is an iterative process with three stages: Evaluate, Analyze, Optimize.

1. Evaluate: benchmark the agent by running it on the train + test tasks. Measure task success rate.

2. Analyze: for the tasks that failed (within the train set only), find out what went wrong. E.g., the agent fails to follow airline policy or makes inefficient tool calls.

3. Optimize: based on failure analysis, make changes to the agent by e.g. providing better context, better tools. Go to step 1.


{:refdef: style="text-align: center;"}
![](/assets/images/blogs/teachagent_part2_fig2.png){: width="450" } 
{: refdef}


Typically, Analyze and Optimize are performed manually: analyzing failure traces, extracting failure root causes, and revising agent prompts.
I will use Socratic to (semi)automate them by creating *two* specialized knowledge bases. 


<!-- how socratic is used for evaluate phase -->
## Automating Agent Failure Analysis Using Socratic
Finding the root cause given a failed agent trajectory is a non-trivial task that requires specialized knowledge of the domain, for example:
* How is correctness of a trajectory determined? Is it by tool calls or the final answer? 
* What if the agent calls more than the set of expected tool calls? Is that considered correct or incorrect?
* What are common failure patterns that we are interested in? 


<details markdown="1">
<summary markdown="span"><u>Example airline agent failure trajectory</u></summary>
<div class="code-wrap" markdown="1">

```json
{
  "task_id": 10,
  "reward": 0.0,
  "info": {
    "task": {
      "user_id": "mia_kim_4397",
      "actions": [
        {
          "name": "book_reservation",
          "kwargs": {
            "user_id": "mia_kim_4397",
            "origin": "JFK",
            "destination": "SEA",
            "flight_type": "round_trip",
            "cabin": "basic_economy",
            "flights": [
              {
                "flight_number": "HAT069",
                "date": "2024-05-20"
              },
              {
                "flight_number": "HAT276",
                "date": "2024-05-25"
              }
            ],
            "passengers": [
              {
                "first_name": "Mia",
                "last_name": "Kim",
                "dob": "1965-06-09"
              }
            ],
            "payment_methods": [
              {
                "payment_id": "gift_card_7359776",
                "amount": 39
              },
              {
                "payment_id": "gift_card_7773485",
                "amount": 67
              }
            ],
            "total_baggages": 1,
            "nonfree_baggages": 0,
            "insurance": "no"
          }
        }
      ],
      "instruction": "Your user id is mia_kim_4397 and you want to remove Ethan from you reservation H9ZU1C. If change is not possible, you want the agent to cancel, and you can rebook yourself. You are also looking for the cheapest direct flight round trip from New York (either EWR or JFK) to anywhere West Coast, with departure date May 20 and return date May 25. You are fine with basic economy class (if chepaer), and you want the agent to book it. You want to first use up your smaller GC and then the larger one. Would want to use all your free baggage allowance but no insurance. Your DOB is in your user profile and you do not want to speak it. You also wonder why cancellation does not refund to GC now.",
      "outputs": []
    },
    "source": "transfer_to_human_agents",
    "user_cost": 0.002062,
    "reward_info": {
      "reward": 0.0,
      "info": {
        "r_actions": 0.0,
        "gt_data_hash": "e41273fb50bc71d9e035c7276e3457b591ca8e057d69904b4e67a20147427639"
      },
      "actions": [
        {
          "name": "book_reservation",
          "kwargs": {
            "user_id": "mia_kim_4397",
            "origin": "JFK",
            "destination": "SEA",
            "flight_type": "round_trip",
            "cabin": "basic_economy",
            "flights": [
              {
                "flight_number": "HAT069",
                "date": "2024-05-20"
              },
              {
                "flight_number": "HAT276",
                "date": "2024-05-25"
              }
            ],
            "passengers": [
              {
                "first_name": "Mia",
                "last_name": "Kim",
                "dob": "1965-06-09"
              }
            ],
            "payment_methods": [
              {
                "payment_id": "gift_card_7359776",
                "amount": 39
              },
              {
                "payment_id": "gift_card_7773485",
                "amount": 67
              }
            ],
            "total_baggages": 1,
            "nonfree_baggages": 0,
            "insurance": "no"
          }
        }
      ]
    }
  },
  "traj": [
    {
      "role": "user",
      "content": "I want to remove Ethan from my reservation H9ZU1C. I'll start by requesting this change.\n\nUser Response:\nHi! I’d like to remove Ethan from reservation H9ZU1C, please."
    },
    {
      "content": "Sorry — I can’t remove Ethan from the reservation. Per our modification rules, changing the number of passengers (adding or removing a passenger) is not allowed.\n\nIf you’d like, I can transfer you to a human agent who may be able to assist further. Would you like me to transfer you now? If you prefer other changes I can handle (for example, updating passenger details without changing passenger count, increasing baggage, or changing cabin), please provide your user_id and the details and I’ll proceed.",
      "role": "assistant",
      "tool_calls": null,
      "function_call": null,
      "annotations": []
    },
    {
      "role": "user",
      "content": "Since I can't remove Ethan and modification is not allowed, I'll ask to cancel my reservation so I can rebook myself instead.\n\nUser Response:\nSince I can’t remove Ethan, please cancel reservation H9ZU1C for me."
    },
    {
      "content": "I can help start the cancellation. Before I proceed I need your user_id and the reason for cancellation (choose one): \"change of plan\", \"airline cancelled flight\", \"health\", \"weather\", or \"other\" (please specify).\n\nPlanned action (please confirm by replying only \"yes\" to proceed):\n- Action: Cancel reservation\n- reservation_id: H9ZU1C\n- user_id: [please provide]\n- reason: [please provide one of the options above]\n\nNotes:\n- I will check the reservation to determine if cancellation is allowed under our rules; if allowed, refund goes to the original payment methods in 5–7 business days.\n- If cancellation is not allowed (for example, if any segment is already flown or other rule prevents it), I will notify you and can transfer you to a human agent if needed.\n\nPlease provide your user_id and the cancellation reason, then confirm by replying \"yes\" to proceed.",
      "role": "assistant",
      "tool_calls": null,
      "function_call": null,
      "annotations": []
    },
    {
      "role": "user",
      "content": "I'll provide my user ID and select \"change of plan\" as my cancellation reason. Then confirm I want to proceed.\n\nUser Response:\nMy user ID is mia_kim_4397, and the cancellation reason is change of plan. Yes, please proceed."
    },
    {
      "content": null,
      "role": "assistant",
      "tool_calls": [
        {
          "function": {
            "arguments": "{\"reservation_id\":\"H9ZU1C\"}",
            "name": "get_reservation_details"
          },
          "id": "call_XhvGPNx9wwpnwM7LohRLWaHk",
          "type": "function"
        }
      ],
      "function_call": null,
      "annotations": []
    },
    {
      "role": "tool",
      "tool_call_id": "call_XhvGPNx9wwpnwM7LohRLWaHk",
      "name": "get_reservation_details",
      "content": "{\"reservation_id\": \"H9ZU1C\", \"user_id\": \"mia_kim_4397\", \"origin\": \"MIA\", \"destination\": \"IAH\", \"flight_type\": \"one_way\", \"cabin\": \"economy\", \"flights\": [{\"origin\": \"MIA\", \"destination\": \"EWR\", \"flight_number\": \"HAT192\", \"date\": \"2024-05-24\", \"price\": 174}, {\"origin\": \"EWR\", \"destination\": \"IAH\", \"flight_number\": \"HAT166\", \"date\": \"2024-05-25\", \"price\": 125}], \"passengers\": [{\"first_name\": \"James\", \"last_name\": \"Moore\", \"dob\": \"1977-02-03\"}, {\"first_name\": \"Ethan\", \"last_name\": \"Lopez\", \"dob\": \"1991-11-04\"}, {\"first_name\": \"Amelia\", \"last_name\": \"Sanchez\", \"dob\": \"1968-08-24\"}], \"payment_history\": [{\"payment_id\": \"gift_card_7359776\", \"amount\": 897}], \"created_at\": \"2024-05-01T04:50:18\", \"total_baggages\": 0, \"nonfree_baggages\": 0, \"insurance\": \"no\"}"
    },
    {
      "content": null,
      "role": "assistant",
      "tool_calls": [
        {
          "function": {
            "arguments": "{\"summary\":\"User mia_kim_4397 requests cancellation of reservation H9ZU1C for reason 'change of plan'. Reservation details: origin MIA to IAH, one_way, cabin economy, flights on 2024-05-24 (HAT192 MIA->EWR) and 2024-05-25 (HAT166 EWR->IAH); passengers: James Moore, Ethan Lopez, Amelia Sanchez (3 passengers); created_at 2024-05-01T04:50:18 (outside 24-hour window); insurance: no; payment_history: gift_card_7359776. Cancellation denied by agent under policy: flights not flown, not within 24-hour window, not airline-cancelled, economy cabin without insurance and reason is 'change of plan' (not health/weather), therefore cancellation not allowed. User previously attempted to remove a passenger but passenger removal is not permitted. User asked the agent to proceed with cancellation despite the restriction. Please assist further.\"}",
            "name": "transfer_to_human_agents"
          },
          "id": "call_JOyK3BQflF8yiJpON80dv5rK",
          "type": "function"
        }
      ],
      "function_call": null,
      "annotations": []
    },
    {
      "role": "tool",
      "tool_call_id": "call_JOyK3BQflF8yiJpON80dv5rK",
      "name": "transfer_to_human_agents",
      "content": "Transfer successful"
    }
  ],
  "trial": 0
}
```

</div>
</details>
&nbsp;


Accurately identifying the root cause of failure is not very easy. This is what GPT-5.2 (without context) generated when given the trace:
```
Primary root cause (booking correctness)
In the final book_reservation call and tool result, 
the “round trip” is not actually a round trip:
* Agent intended:
    * Outbound: JFK → SEA on 2024-05-20 (HAT069)
    * Return: SEA → JFK on 2024-05-25 (HAT069)
* What got booked (per book_reservation tool output):
    * Leg 1: JFK → SEA (2024-05-20) ✅
    * Leg 2: JFK → SEA (2024-05-25) ❌ (return direction is wrong)
So the agent created a reservation with two flights in the same direction, despite flight_type: "round_trip". 
That’s a hard functional failure and is enough to zero the reward.
```
This diagnosis is actually not bad. The model correctly identified that the agent used the wrong tool call argument. 
(surprised that it was able to understand the trace format without any prior context.)
However, this diagnosis identifies only the *surface failure*: the final, incorrect action. To meaningfully improve the agent, we must understand the *root cause* that led to the mistake. Why did the agent believe it was acceptable to book a return flight using the outbound flight's details? 

This is where expert context becomes essential. An expert failure analyst interprets the agent's actions in the context of the agent's goals and the domain constraints. 
The goal is to build a knowledge base that equips a triage agent with this level of expert context, allowing it to automatically and accurately pinpoint the root cause of a failure.

To build this knowledge base, I used *teaching by example*. Initially, the knowledge base is empty, and a "student" triage agent relies only on general-purpose reasoning to diagnose failures. When the student agent fails to identify a root cause correctly, I (the human teacher) provide the correct, detailed diagnosis. The student agent then generalizes from this example and updates its knowledge base. 

**With more examples, the triage agent becomes progressively more proficient at pinpointing failure root causes.**
At the end, the triage agent was able to provide this diagnosis for the above failure trace:
```
Surface-level failure: the agent’s book_reservation call does not match the oracle 
in info.task.actions—it booked return flight HAT069 on 2024-05-25, 
but the expected return flight is HAT276 on 2024-05-25 (destination SEA is correct; flight number is wrong).

Most likely root cause: Golden mismatch from tool-result selection/assumption—the agent 
searched “direct flights” for the return leg but never found/validated a SEA→JFK flight 
option (it even queried `search_direct_flight` with `origin:"JFK", 
destination:"SEA"` for the return date), then incorrectly assumed it could reuse the outbound 
flight number for the return segment, leading to incorrect booking arguments.
```

With proper context, the triage agent was able to give a better diagnosis, identifying that the agent incorrectly assumed it could reuse an outbound flight for the return segment. 
This insight can be directly translated to an optimization for the airline agent (e.g. improve the system instructions to clarify that outbound flight cannot be reused for return flight).


## Optimizing Airline Agent's Context using Socratic
<!-- how socratic is used for optimize phase -->
Following the Analyze phase, Socratic is used to construct a knowledge base for the airline domain, covering edge cases and detailed decision procedures.
The objective is to enhance the [original baseline airline agent instructions](https://github.com/sierra-research/tau-bench/blob/main/tau_bench/envs/airline/wiki.md) to reduce agent failures.

Insights from the failure analysis stage directly drive updates to this airline knowledge base.
For instance, a common failure mode involves the agent prematurely transferring the user to a human agent.
A targeted chat session in Socratic is used to address this failure (e.g., "Let's study when the agent should transfer to a human more carefully").
The improved knowledge base is then applied to the airline agent's system prompts.

This Evaluate → Analyze → Optimize loop was repeated roughly five times, iteratively improving both the triaging agent and the airline knowledge base. The resulting final knowledge bases:
* [Optimized triaging agent knowledge base](https://gist.github.com/kevins981/899c5947ac99cd24f0b1e77b5cb89219).
* [Optimized airline agent knowledge base](https://gist.github.com/kevins981/3ff5af01eb11ba0f742ccd15d995dd5a).

## Results 
<!-- summarize performance improvements -->
The figure below shows the airline agent success rate after agent optimizations. 
* Success rates are averaged across three trials. 
* GPT-5 mini uses no reasoning. GPT-5.2 uses medium reasoning effort. 

Train and test sets **improved by 10% and 17%**[^1]. Not bad! We made GPT-5 mini with no reasoning perform better than GPT 5.2 with medium reasoning effort! 


{:refdef: style="text-align: center;"}
![](/assets/images/blogs/teachagent_part2_optimized.svg){: width="450" } 
{: refdef}

## Workload Specific Findings
### 1. Step-by-step procedures tend to work better than unstructured rules.
For example, the original airline modification policy:

```markdown
## Modify flight
- The agent must first obtain the user id and the reservation id.
- Change flights: Basic economy flights cannot be modified. Other reservations can be modified without changing the origin, destination, and trip type. Some flight segments can be kept, but their prices will not be updated based on the current price. The API does not check these for the agent, so the agent must make sure the rules apply before calling the API!
- Change cabin: all reservations, including basic economy, can change cabin without changing the flights. Cabin changes require the user to pay for the difference between their current cabin and the new cabin class. Cabin class must be the same across all the flights in the same reservation; changing cabin for just one flight segment is not possible.
- Change baggage and insurance: The user can add but not remove checked bags. The user cannot add insurance after initial booking.
- Change passengers: The user can modify passengers but cannot modify the number of passengers. This is something that even a human agent cannot assist with.
- Payment: If the flights are changed, the user needs to provide one gift card or credit card for payment or refund method. The agent should ask for the payment or refund method instead.
```

Optimized modification policy:
```markdown
### Modification Decision Procedure
Given a reservation and a requested change, determine whether the overall modification is allowed by executing this procedure from top to bottom.
1. If the request changes the number of passengers (adds/removes passengers), then **not allowed**.
2. If the request adds travel insurance after initial booking, then **not allowed**.
3. If the request decreases checked baggage count, then **not allowed**.
4. If the request changes flights (itinerary / flight segments):
- If the reservation cabin is **basic economy**, then **not allowed**.
- If the change would alter the reservation’s **origin**, **destination**, or **trip type**, then **not allowed**.
5. If the request changes cabin class, then **allowed** (including basic economy).
6. If the request increases checked baggage count, then **allowed**.
7. If the request changes passenger details (name and/or date of birth) without changing passenger count, then **allowed**.
8. If any rule above cannot be evaluated due to missing information, treat the request as **not allowed** until the missing information is obtained.
```

If you step into the shoes of the agent and read both versions of instructions, you would probably find the second one to be easier to follow. 
Logically, the two versions are equivalent: they produce the same output given the same input.
But there is a subtle difference.

In programming language terms, the first version is *declarative*: it describes the policy, but not *how* to follow the policy. 
On the other hand, the second version is *imperative*: it tells the agent exactly how to determine modification eligibility step-by-step, similar to a program. 
The latter requires a lower mental burden to follow correctly: just follow the steps!

### 2. The agent struggled with when to transfer to a human agent. 
This results in the agent terminating the conversation prematurely, without first confirming with the user. 
For example, the user requests to cancel a basic economy flight, which is not allowed by the airline policy. The agent then directly transfers to a human agent, without first communicating with the user.

It turns out this is due to an ambiguous instruction in the baseline system prompt: `"You should transfer the user to a human agent if and only if the request cannot be handled within the scope of your actions."` It is unclear exactly **when** the agent should perform the transfer. 

Updated instructions: 
```
When a request is not allowed by policy, the assistant should first inform the user that the requested action cannot be performed. 
Only transfer to a human agent if the user still needs an outcome that the assistant cannot provide.

It's important to not prematurely transfer to a human agent without first informing the user. 
This is because the user may have alternative requests once they are informed.
```

This effectively reduced premature transfer failures.

### 3. There is no tool to find information about a specific flight.
The airline environment provides tools to retrieve details of a specific user or a reservation, but no such tool exists for a flight. This caused the agent to struggle with tasks that require specific flight information, e.g., checking the delay status.
The solution was simple: add such a tool `get_flight_details`. Having the right tool improves both agent reliability and efficiency.

The [Socratic tau-bench repository](https://github.com/kevins981/tau-bench_socratic) contains all optimizations made.


## Summary of Resources
* [tau-bench repository used for experiments](https://github.com/kevins981/tau-bench_socratic) (main branch is optimized version, baseline is original.)
* [Knowledge base for triage agent](https://gist.github.com/kevins981/899c5947ac99cd24f0b1e77b5cb89219)
* [Knowledge base for optimized airline agent](https://gist.github.com/kevins981/3ff5af01eb11ba0f742ccd15d995dd5a)

## Footnotes
[^1]: The task success rates reported here are not directly comparable with those reported by LLM vendors. I have made numerous fixes to improve the quality of the ground truth grading, which is a [known issue](https://openai.com/index/introducing-gpt-5-2/). All evaluations use the same ground truth grading. See the [repository](https://github.com/kevins981/tau-bench_socratic) for all changes made.