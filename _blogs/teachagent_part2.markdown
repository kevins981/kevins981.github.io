---
layout: post
title:  "Teach Your Agents What You Know (Part 2)"
# subtitle: "A New Interface for Human-Agent Knowledge Transfer"
date:   2025-12-21
---

[Part 1: Motivation and high level idea of teach-student knowledge transfer.](https://kevins981.github.io/blogs/teachagent_part1.html)

TLDR: final accuracy improvement

use teacher-student system (open source prototype [Socratic](https://github.com/kevins981/Socratic)) to optimize a specific vertical agent: airline agent from tau-bench.

<!-- explain briefly what is tau-bench and airline agent -->
## Background: Airline Agent and Tau-bench
standard benchmark used by almost all major LLM to evaluate agentic capabilities (cite).
Two sentence summary from original paper: "an agent interacts with database API tools and an LM-simulated user to complete tasks. The benchmark tests an agent’s ability to collate and convey all required information from/to users through multiple interactions, and solve complex issues on the fly while ensuring it follows guidelines laid out in a domain-specific policy document."

{:refdef: style="text-align: center;"}
![](/assets/images/blogs/teachagent_part2_fig1.png){: width="600" } 
{: refdef}

evaluation criteria: whether the expected functions are called and final database state. in above example, agent must call `cancel_reservation` with specific argument `JK9O19`.

## Goal: Optimizing the Airline Agent
improve the success rate of this airline agent.
do this by improving context given to agent: expert knowledge.

split into train and test sets (20 and 30)
show initial accuracy on train and test set (footnote: this is modified, so acc not comparable to orig taubench. source code here)

{:refdef: style="text-align: center;"}
![](/assets/images/blogs/teachagent_part2_baseline.svg){: width="300" } 
{: refdef}

explain cycle of evaluate, analyze, optimize. show diagram
* evaluate: run agent against test cases (train set only)
* analyze: take the failed tests, figure out what went wrong: what is the root cause? e.g. agent fail to follow airline policy? made inefficient tool calls?
* optimize: based on analysis, improve the agent by e.g. providing better context, better tools

{:refdef: style="text-align: center;"}
![](/assets/images/blogs/teachagent_part2_fig2.png){: width="450" } 
{: refdef}

## Using Socratic in Practice
Typically, both Analyze and Optimize stages needed to be done manually: a human domain expert analyzes failure traces, extract failure root causes, and manually revise agent context prompt hoping to improve agent performance.

Using Socratic, I created *two* knowledge bases to build two specialized agents: one for Analyze stage, one for Optimize stage.


<!-- how socratic is used for evaluate phase -->
## Finding Failure Root Causes: Triage Agent
evaluate: goal is to build KB that enables automatic analysis of failure traces and pinpoint root cause of failure.



So the knowledge base we want to produce is one that would enable an agent to accurate pinpoint failure root cause
This task is not trivial to perform correctly.
understanding the trace format. different for each workload, each with own nuance. e.g. how is correctness of trajectory determined? is it by tool calls or final answer? what if the agent calls more than the set of expected tool calls? what are common failure patterns that we are interested in? 

use teaching via examples to build such a knowledge base. everytime agent encounters new failure trace, I (human teacher) discusses this with agent, and KB is updated.

[The final knowledge base for triaging agent](https://gist.github.com/kevins981/899c5947ac99cd24f0b1e77b5cb89219).



An example: 
<details markdown="1">
<summary markdown="span">Example airline agent failure trace</summary>
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




To accurate identify the failure root cause is a non-trivial task. This is what GPT 5.2 (without context) generated when given the trace:
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
This diagnosis is in fact not bad. The model correctly identified that the airline called the correct tool (`book_reservation`) but used the wrong argument. 
(I am actually somewhat surprised that GPT 5.2 was able to understand the trace format without any prior context.)
However, this is only the point of failure at the **surface level**. To meaningfully optimize the airline agent, we need to take a step further and understand *why* the agent failed.

This is where expert context helps. Socratic Triage agent (GPT 5.2) diagnosis:
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
With proper context, the Socratic agent was able to give a more useful diagnosis, identifying that the agent incorrectly assumed it could reuse an outbound flight for the return segment. 
This insight can be directly translated to an optimization for the airline agent (e.g. improve the system instructions to clarify that outbound flight cannot be reused for return flight).


## Optimizing Airline Agent's Context 
<!-- how socratic is used for optimize phase -->
optimize: goal is build KB for the airline domain. e.g. how to handle edge cases, decision procedures.
I use insights from analysis stage to drive KB updates. e.g. launch a teaching session in Socratic saying "lets study when agent should transfer to human more carefully"
- show agent instruction before and after

[Original airline agent instructions](https://github.com/sierra-research/tau-bench/blob/main/tau_bench/envs/airline/wiki.md). \\
[Optimized knowledge base](https://gist.github.com/kevins981/3ff5af01eb11ba0f742ccd15d995dd5a).


## Results 
<!-- summarize performance improvements -->
Below shows the airline agent success rate after agent optimizations. 
* Success rates are avereaged across 3 trials. 
* GPT-5 mini uses no reasoning. GPT 5.2 uses medium reasoning effort. 

Train and test sets **improved by 10% and 17%**. Not bad! We made GPT-5 mini with no reasoning perform better than GPT 5.2 with medium reasoning effort! 

(Note: the task success rates reported here are not directly comparable with those reported by LLM vendors. I have made numerous fixes to improve the quality of the ground truth grading, which is a [known issue](https://openai.com/index/introducing-gpt-5-2/). All evaluations use the same ground truth grading. See the [repository](https://github.com/kevins981/tau-bench_socratic) for all changes made.)

{:refdef: style="text-align: center;"}
![](/assets/images/blogs/teachagent_part2_optimized.svg){: width="450" } 
{: refdef}

## Workload Specific Findings
### 1. Step-by-step procedures/instructions tend to work better than unstructured rules. 
E.g. The original Airline modification policy:

```markdown
## Modify flight
- The agent must first obtain the user id and the reservation id.
- Change flights: Basic economy flights cannot be modified. Other reservations can be modified without changing the origin, destination, and trip type. Some flight segments can be kept, but their prices will not be updated based on the current price. The API does not check these for the agent, so the agent must make sure the rules apply before calling the API!
- Change cabin: all reservations, including basic economy, can change cabin without changing the flights. Cabin changes require the user to pay for the difference between their current cabin and the new cabin class. Cabin class must be the same across all the flights in the same reservation; changing cabin for just one flight segment is not possible.
- Change baggage and insurance: The user can add but not remove checked bags. The user cannot add insurance after initial booking.
- Change passengers: The user can modify passengers but cannot modify the number of passengers. This is something that even a human agent cannot assist with.
- Payment: If the flights are changed, the user needs to provide one gift card or credit card for payment or refund method. The agent should ask for the payment or refund method instead.
```

Optimized modificaiton policy:
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

If you step into the shoe of the agent and read both versions of instructions, you would probably find the second one be be easier to execute. 
Logically, the two versions are equivalent: they produce the same output given the same input.
But there is a subtle difference.

In programming language terms, the first version is *declarative*: it describes the policy, but not *how* to follow the policy. 
On the other hand, the second version is *imperative*: it tells the agent exactly how determine modification eligibility step-by-step, similiar to a program. 
The latter requires lower mental burden to follow correctly: just follow the steps!

### 2. Agent struggled with when to transfer to a human agent. 
This results in the agent terminating the conversation prematurely, without first confirming with user. 

E.g. User requests to cancel basic economy flight, which is not allowed by airline policy. Agent directly transfers to human agent, without first communicating with the user.

Turns out this is due to an ambiguous instruction: `"You should transfer the user to a human agent if and only if the request cannot be handled within the scope of your actions."` It is unclear exactly WHEN the agent should perform the transfer. 

Updated instructions: 
```
When a request is not allowed by policy, the assistant should first Inform the user that the requested action cannot be performed. 
Only transfer to a human agent if the user still needs an outcome that the assistant cannot provide.

Its important to not prematurely transfer to human agent without first informing the user. 
This is because the user may have alternative requests once they are inform.
```

This effectively reduced premature transfer failures.

### 3. There lacks a tool to find information of a specific flight.
The airline environment provides tools to retrieve details of a specific user or a reservation. But no such tool exists for a flight. This caused the agent to struggle with tasks that require specific flight info e.g. check delay status of flight.
The solution was simple: add such a tool `get_flight_details`. Having the right tool improves both agent reliablity and efficiency.

Please see the [Socratic tau-bench repository](https://github.com/kevins981/tau-bench_socratic) for all optimizations made.


## Summary of Resources
[tau-bench repository used for experiments](https://github.com/kevins981/tau-bench_socratic) (main branch is optimized version, baseline is original.) \\
[Knowledge base for triage agent](https://gist.github.com/kevins981/899c5947ac99cd24f0b1e77b5cb89219) \\
[Knowledge base for optimized airline agent](https://gist.github.com/kevins981/3ff5af01eb11ba0f742ccd15d995dd5a)
