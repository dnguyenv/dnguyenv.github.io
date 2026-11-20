---
title: "Your Agent Budget Is a Soft Limit"
date: 2026-11-19
author: "Duy Nguyen"
tags: ["AI", "Platform Engineering", "Agentic Systems"]
description: "Most AI gateway spend limits check the budget before a call and charge it after. Here is how far that lets agents overshoot, and what a reserve-then-settle budget changes."
categories: ["Platform Engineering", "AI Architecture"]
draft: false
---

*A spend cap that is checked before the call and charged after it is a counter with a delay.*

Snowflake's documentation for blocking users at a quota limit includes a small worked example. A user with a 100-credit limit reaches 100, spends 30 more before the block lands, and [ends at 130](https://docs.snowflake.com/user-guide/budgets/per-user-quotas-enforcement). The page is candid that "the overshoot is bounded by what they spend in that short window, which is usually small for interactive use.". A person typing into a chat window spends slowly. Fifty sub-agents fanned out by an orchestrator do not.

This week I looked through the limitations of five AI gateway budget features that platform teams are putting in front of coding agents. Four work pretty much the same way, just described differently. One takes a different approach, and that difference highlights an interesting question: what does “block usage” actually mean when the caller is an agent running in a loop rather than a person? In [Priority Is Not Isolation](/2026/11/priority-is-not-isolation/) the lesson was to meter admission upstream so a GPU pool never sees a burst. This is the same lesson applied to money instead.

### What "block usage" actually promises

Here’s how each gateway describes its approach as of November 2026. This comes directly from the vendors’ documentation, and given how quickly these features are evolving, some of the details will inevitably change.

| Gateway | When the budget is checked | When the cost is charged | Requests in flight at the limit | Budget store unavailable |
|---|---|---|---|---|
| [Cloudflare AI Gateway](https://developers.cloudflare.com/ai-gateway/features/spend-limits/index.md) | Before dispatch, all rules at once | After completion; "eventually consistent" | A concurrent burst "can briefly exceed the limit" | Not documented |
| [Databricks Unity Gateway](https://docs.databricks.com/aws/en/ai-gateway/budgets) | Near real-time estimate | Estimated, reconciled later against billing | "Not interrupted" | Not documented |
| [Snowflake Cortex AI Gateway](https://docs.snowflake.com/user-guide/budgets/per-user-quotas-enforcement) | Pipeline evaluates "within minutes" | Metering pipeline | "Run to completion and are billed" | Not documented |
| [Redpanda Agentic Data Plane](https://docs.redpanda.com/agentic-data-plane/control/budgets.md) | On each request | One spend event per call | Not documented | "Lets the request through" |
| [Telnyx AI Gateway](https://developers.telnyx.com/docs/inference/ai-gateway/controls.md) | Reserves an upper bound before dispatch | Actual cost replaces the reservation | Each request needs its own reserved headroom | `503 enforcement_unavailable` while policy propagates; docs say fail closed |

The vendors aren’t hiding this. Databricks explicitly says its thresholds are “enforced approximately” and aren’t an absolute cap on the final bill. Cloudflare calls its tracking a “best-effort estimation.” And honestly, that makes sense. You don’t know the full cost of an LLM request until it finishes, so the gateway has to check the budget before the call and record the real cost afterward. The dangerous part is what we do with that behavior: seeing “block usage at $100” in a console and translating it into “agents can’t spend more than $100.”

### Where the overshoot comes from

Three things push spend past a check-then-record limit, and agents amplify all three.

Start with work in flight. Every request admitted before the crossing is recorded will finish and be billed. If C requests can be in flight at once and the most expensive costs c_max, that alone is up to C × c_max over the limit. A person has one request in flight; an orchestrator with 50 sub-agents has 50.

Then there is enforcement lag. Between spend crossing the limit and the block taking effect, new requests keep getting admitted at whatever rate the callers produce them. If the gateway evaluates in seconds, this term is small. If it evaluates within minutes, it dominates. Snowflake adds that configuration changes, including a lowered limit, take [about 5 to 10 minutes](https://docs.snowflake.com/user-guide/budgets/per-user-quotas-enforcement) to propagate.

The last is the shape of the budget itself. A user who starts spending near a cycle boundary, Snowflake notes, "can consume up to one full limit before the reset and a fresh limit after it." Redpanda makes the scoping version of the point: a [$100 monthly per-principal limit with 10 callers allows up to $1,000](https://docs.redpanda.com/agentic-data-plane/control/budgets.md). And retries count, since SDK retries layered on gateway retries are separate requests the budget sees separately.

Put the first two together and you get a rough bound worth writing on the whiteboard:

```text
overshoot ≈ C × c_max  +  r × L

C      requests in flight at the crossing
c_max  worst-case cost of one request
r      spend rate of the callers ($/min)
L      enforcement lag (min)
```

A worked example, with illustrative numbers rather than a measurement. Price tokens at $5 per million input and $15 per million output, the [flat reference rate Telnyx uses](https://developers.telnyx.com/docs/inference/ai-gateway/controls.md) for enforcement. Give 50 sub-agents calls of up to 60,000 input tokens and 16,000 output tokens, so c_max is about $0.54. The in-flight term is 50 × $0.54, or $27. Now suppose each agent issues a call every 15 seconds at an average of $0.27. That is 200 calls a minute, or $54 a minute. With a 3-minute lag, the rate term is $162. A $100 daily budget can close the day near $289, and nothing in the console looked misconfigured. With a lag of a few seconds, the rate term drops to a few dollars and the in-flight term is what you are left with.

The formula is crude, but it tells you which knob matters for your gateway: concurrency if enforcement is near real-time, lag if it is not.

### Reserve, then settle

The alternative is old and simple: hold the worst-case cost before dispatch, then replace the hold with the actual cost when the response comes back. Telnyx documents a production version of it. Before a request is dispatched, the gateway [reserves "a conservative upper bound on its cost"](https://developers.telnyx.com/docs/inference/ai-gateway/controls.md) from the input size and the resolved output allowance. If any scope lacks that much headroom, the request is denied with `403 budget_exceeded`, and the reservation "is not shrunk to fit." Concurrent requests each reserve their own bound, so they compete for headroom before money moves rather than after. Solo.io published a [reference implementation of the same pattern](https://www.solo.io/blog/building-real-time-ai-cost-controls-with-agentgateway) in June 2026 on agentgateway's External Processing hook, which estimates cost from a pricing table and atomically reserves budget capacity before forwarding.

![An agent sends a request with max_tokens to the gateway. The gateway reserves the worst-case cost in the budget store, which grants a hold or denies. Only then does the gateway dispatch to the provider, and when the response and usage come back it settles the actual cost in the budget store before returning the response.](/assets/images/your-agent-budget-is-a-soft-limit-reserve-settle.svg)

The part I find most instructive is what Telnyx does when the story does not end cleanly. If a stream is interrupted before usage is reported, the reservation is retained as exposure and the spend event reports `cost: null`: "Unknown usage is not zero usage, and a timeout is not a refund." Gateway retries and fallbacks reserve budget for every planned attempt up front and release the unused part at settlement.

If your gateway does not do this and agent traffic needs it, the core is small. Redis runs a Lua script atomically, so check and hold happen as one step:

```lua
-- reserve.lua  KEYS[1]=budget  ARGV: limit, hold, hold_id, ttl (amounts in micro-USD)
local spent = tonumber(redis.call('HGET', KEYS[1], 'spent') or '0')
local held  = tonumber(redis.call('HGET', KEYS[1], 'held')  or '0')
local hold  = tonumber(ARGV[2])
if spent + held + hold > tonumber(ARGV[1]) then
  return 0                                  -- deny before any provider call
end
redis.call('HINCRBY', KEYS[1], 'held', hold)
redis.call('SET', 'hold:' .. ARGV[3], hold, 'EX', ARGV[4])
return 1
```

Settlement is the mirror image: subtract the hold from `held`, add the actual cost to `spent`, delete the hold record. Two details matter more than the script. A request usually falls under several budgets at once (agent, team, provider total), and all of them have to be reserved in the same atomic step, which in Redis Cluster means hash-tagging the keys into one slot. And a hold that expires without a settlement should be charged, not quietly released, for the same reason Telnyx gives.

### What reservation costs

Reservation trades overshoot for false blocks, and the exchange rate is set by `max_tokens`. Telnyx reserves a model's full context window as output when the request leaves the limit unset. At the same $15 per million, a 128,000-token window is a $1.92 output hold on every call, plus input. Fifty agents that never set `max_tokens` would hold about $111 between them, so a $100 budget would stop admitting new calls at around 45 concurrent agents while actual spend sat at a fraction of that. Reservation only works if the platform makes explicit output limits the default, and sizes them from observed distributions rather than model maximums.

It also puts a strongly consistent store on the hot path of every LLM call, and that store will have bad days. Here the vendors disagree, and it is worth noticing that they disagree. Redpanda lets requests through when it cannot read budgets or spend. Telnyx tells clients to fail closed while enforcement is unavailable. Neither is wrong. My feeling is that the right answer depends on who is calling: fail open with a loud alert for interactive humans, whose spend rate is low and whose work is visible, and fail closed for autonomous or scheduled agents, which are exactly the callers the budget exists for.

Every number in this pipeline is also an estimate. Databricks may retroactively adjust budget estimates and names `system.billing.usage` as the source of truth; Telnyx enforces at a reference rate that is explicitly "not your invoice." Someone should reconcile budget and invoice monthly. And a budget can be silently inert: Redpanda warns that a budget naming a provider that does not exist is accepted and "never caps anything." Test that a limit blocks, in staging, the way you would test a circuit breaker.

### The older instinct

Payment networks solved this long ago. A fuel pump authorizes a hold before it knows how much fuel you will take, then captures the actual amount. Nobody in that world mistakes the running total on a statement for a spending limit. The authorization is the limit, and the capture is the bill.

The other lesson worth borrowing comes from Databricks, which wrote up [how it governs its own engineers' coding-agent spend](https://www.databricks.com/blog/how-databricks-manages-its-own-coding-agent-spend-unity-ai-gateway-budgets) through its gateway. They separated runaway protection from monthly allocation: a small daily limit that an engineer can raise with one acknowledgment, and a larger monthly limit that needs a manager. The daily limit is the one doing the safety work, because it shrinks the window in which the overshoot terms above can accumulate. Their line on why the acknowledgment step works is the best argument for a human in the loop I have read this year: "An unattended cron job cannot click a Slack button."

If I were putting budgets in front of an agent fleet this quarter, I would read my gateway's limitations section first and write down which column of the table it falls in. Then compute the overshoot bound with real concurrency and real lag, and put that number, not the configured limit, in the budget conversation with finance. Require explicit `max_tokens` on agent traffic. Add a daily runaway limit under every monthly one. If the agents run unattended and the overshoot number is uncomfortable, reserve before dispatch and decide, deliberately, what happens when the budget store is down.

A limit you check is a hope. A limit you hold is a limit.

And life goes on...
