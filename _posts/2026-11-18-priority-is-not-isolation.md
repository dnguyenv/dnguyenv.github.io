---
title: "Priority Is Not Isolation"
date: 2026-11-18
author: "Duy Nguyen"
tags: ["AI", "Inference", "Platform Engineering"]
description: "What llm-d's flow control and a fresh multi-tenant benchmark say about sharing one GPU pool between interactive and batch tenants, and why the lever that protects latency sits upstream of the router."
categories: ["Platform Engineering", "AI Architecture"]
draft: true
---

*Every shared GPU pool eventually gets a nightly batch job, and the batch job always wins unless someone decides otherwise.*

On a plain inference router, a 4,000-request document extraction batch pushed a chat tenant's median time to first token from about 45 ms to [84 seconds](https://llm-d.ai/blog/multi-tenant-multi-priority-model-serving-llm-d). Same pool, model, nothing seemed to be broken. The batch drained in under 5 minutes and was perfectly happy about it. That number comes from a benchmark the llm-d maintainers published today, November 16, 2026, a month after [llm-d v1.0](https://llm-d.ai/blog/llm-d-v1.0-state-of-the-art-inference-ready-for-day-2) made multi-tenant quality of service a first-class router feature. It is the cleanest public measurement I have seen of something most platform teams learn the hard way: putting priorities on traffic is not the same as isolating tenants from each other.

In [Owning Your Inference](/2026/07/owning-your-inference-a-decision-map/) I left the layer above the engine as a mark: "vLLM or SGLang pods behind a router." This post is about what that router has to do when more than one team depends on it.

### The rate limit you already have may count the wrong thing

Most organizations put an API gateway in front of a model pool and give each team a requests-per-second quota. The [Gateway API Inference Extension flow control guide](https://gateway-api-inference-extension.sigs.k8s.io/guides/flow-control/) explains why that might be problematic: in LLM serving, cost is driven by token counts and the open-ended decode loop, so one long-context request can consume more KV cache than hundreds of short chats. An RPS quota lets the heavy requests through and fills the engine anyway.

The second problem is where the queue lives. Without anything smarter in front, excess load piles up inside each vLLM replica's local queue. The engine batches by arrival and sequence length, not by business priority, so a real-time request waits behind an offline job. Worse, once a request is in a replica's queue the router cannot move it, even if a better replica frees up a moment later. This is called scheduling regret. I would call it the same mistake as deep buffers in a network device: the queue sits in the one place where nobody can reorder it.

### What flow control actually does

Flow control in the llm-d router (the Endpoint Picker, or EPP) moves that queue to a place that can reason about it. Each request gets a flow key made of a fairness ID, which is the tenant, and a priority taken from an `InferenceObjective` resource. In v1.0 the canonical headers are [`x-llm-d-inference-fairness-id` and `x-llm-d-inference-objective`](https://llm-d.ai/docs/api-reference/epp-http-headers), with the older `x-gateway-*` names accepted as deprecated aliases since router v0.9.0. Every flow key gets its own in-memory queue. Dispatch picks the [highest-priority band with work, then a tenant within it](https://llm-d.ai/docs/architecture/core/router/epp/flow-control) by a fairness policy such as round robin, then a request within that tenant's queue.

Before anything leaves, a saturation detector decides whether the pool has room. If it does, the request goes to the best-scoring replica at that moment, which is the late binding that keeps prefix-cache affinity useful under load. If it does not, the dispatch loop halts and the request waits in the router, where it can still be outranked. Configuration is short:

```yaml
featureGates:
  - flowControl
plugins:
  - type: round-robin-fairness-policy
  - type: concurrency-detector
    parameters:
      maxConcurrency: 64        # per replica, set at the measured knee
saturationDetector:
  pluginRef: concurrency-detector
flowControl:
  defaultRequestTTL: "60s"
  priorityBands:
    - priority: 100             # interactive
      maxRequests: "500"
      fairnessPolicyRef: round-robin-fairness-policy
    - priority: -10             # batch, sheddable
      maxRequests: "2000"
      fairnessPolicyRef: round-robin-fairness-policy
```

It is still opt-in: the `flowControl` feature gate guards it, and according to the v1.0 release notes the defaults mimic first-come, first-served, so nothing changes until you write a policy.

### Three front doors, one pool

The benchmark is small and honest about it: two A100 40 GB nodes, one vLLM v0.25.0 replica of Qwen3-8B on each, saturation measured at 128 concurrent requests across the pair. A chat tenant sends 4 requests per second straight to the router, and two minutes in, a batch tenant releases 4,000 requests through one of three front doors.

| Batch front door | Chat TTFT p50 | Chat TTFT p99 | Batch 429s | Batch drain |
|---|---|---|---|---|
| Direct, no flow control, 10x saturation | 84,372 ms | 92,704 ms | 0 | 287 s |
| Direct, flow control, 10x saturation | 235 ms | 3,000 ms | 18,846 | 272 s |
| Queued, async dispatcher, 128 workers | 322 ms | 4,303 ms | 0 | 379 s |
| Queued, async dispatcher, 64 workers | 69 ms | 298 ms | 0 | 374 s |

Figures from the [llm-d multi-tenant benchmark](https://llm-d.ai/blog/multi-tenant-multi-priority-model-serving-llm-d), November 2026. Idle chat TTFT was 44 to 47 ms at the median in every run.

Read the second row carefully, because it is where most teams would stop. Flow control does its job: chat latency holds at about 5x idle whether the batch arrives at 1x, 3x, or 10x saturation. It does not get chat close to idle, because once the pool hits the detector's cap even priority-100 requests wait for a slot. And the pain moves to the batch tenant. At 10x, landing 4,000 requests took 22,684 attempts, 18,846 of them rejected when the batch band filled, and 162 requests never landed within the client's 25 retries. Priority protected the chat tenant from the batch. Nothing protected the batch tenant from its own retry storm.

### Meter below the knee

The fourth row is the interesting one. The [llm-d Async](https://github.com/llm-d/llm-d-async) component puts a durable per-tenant queue and a gated dispatcher in front of the router. Quota gates classify traffic as reserved or overflow, six strict lanes order it, and the dispatcher stamps the same objective and fairness headers so the router continues the policy.

![The chat tenant calls the router directly. The batch tenant publishes once to a durable queue, and a dispatcher with a fixed worker cap meters it into the router, which gates dispatch into a vLLM pool that keeps only a small local buffer. Saturation metrics flow back to the dispatcher as backpressure.](/assets/images/priority-is-not-isolation-admission-path.svg)

At full width, 128 workers, the queued path did no better for chat than flow control alone, and it drained a third slower. It did remove every 429 and every retry; the batch producer published once and disconnected. The reason is telemetry lag: Prometheus scraped the router every 10 seconds, so 128 workers overshot the cap before the saturation gate noticed, then the gate closed late and reopened all at once and the pool oscillated. Cutting the dispatcher to 64 workers, half of saturation, dropped chat to 69 ms median and 298 ms p99, the best tail in the study, at the cost of draining in 374 seconds instead of about 285 with the pool near 80% of peak output. Ninety-six workers matched that median but posted a 1.5 s p99 and the slowest drain in the study, 528 seconds: the pool brushed the threshold often enough that the gate kept bouncing work back to the broker, and backoff left the pool idle for 42% of the drain.

So the knob that actually isolated the tenants was not a priority value. It was a worker count, chosen upstream, comfortably below the point where the detector would have to intervene. Isolation came from never presenting the router with a burst in the first place. My feeling is that this generalizes well beyond llm-d: any closed-loop gate that reacts to scraped metrics needs the open-loop side metered below its threshold, or the loop itself becomes the source of jitter.

The detector choice sets the floor too. Rerunning with the default `utilization-detector`, which reacts to engine queue depth and KV-cache pressure, held chat at roughly 14x idle instead of 5x, because by the time it trips the chat requests are already behind a queue. The docs describe the trade plainly: the [utilization detector](https://llm-d.ai/docs/architecture/core/router/epp/flow-control) sees real memory pressure but lags, while the concurrency detector reacts instantly but is blind to KV-cache filling. On a homogeneous pool with predictable prompts, a calibrated concurrency cap looks like the better default. On a mixed pool with long-context outliers, I would not trust it without a KV-cache backstop.

### Where it breaks

Several things here are easy to get wrong in production.

The flow control queues are in memory. If the router restarts, [queued requests are lost](https://llm-d.ai/docs/architecture/core/router/epp/flow-control): a graceful shutdown evicts them with a 500, a crash drops connections. v1.0 adds a hot standby for the router, but as far as I can tell from the docs that keeps routing alive, not the queued requests. Anything that must survive belongs in the durable tier, not in router memory.

The headers are a trust boundary. A tenant that can set its own `x-llm-d-inference-objective` can promote itself to premium. The docs say to strip these at the edge and inject them from an identity-aware gateway, and the async post adds that the fairness ID should be an opaque tenant ID, not an email, because it ends up in access logs.

Band depth is a sizing decision with visible consequences. In the same study, a 500-deep batch band turned a 10x burst into tens of thousands of 429s, while a 2,000-deep band turned the same burst into queueing time and every batch request landed on its first attempt. Clients should also read `x-llm-d-request-dropped-reason`: `rejected-*` means no GPU work was done and a retry is cheap, `evicted` means cycles were spent and backoff should account for it.

Finally, this is one model, one pool, one prompt mix, published by the project itself, and the router image tracked `main`. The shape of the result matches what the docs predict, but the specific multipliers are a starting hypothesis for your own load test, not a number to put in an SLO. Flow control also cannot create capacity. It only decides where TTFT accrues and for whom.

### The older instinct

None of this is new as an idea. Networking went through the same lesson with bufferbloat: queues belong where something can schedule them, and the sender has to be paced below the bottleneck, not at it. What is new is that inference finally has the pieces to do it with Kubernetes-native resources rather than a custom scheduler in front of every model.

If I were setting up a shared pool this quarter, I would do it in this order. Measure the pool's saturation concurrency with a batch-only ramp. Turn on flow control with a calibrated detector and sized bands, and inject tenant and priority headers at the trusted edge. Then move every non-interactive tenant behind a durable queue and meter it at roughly half of saturation, and accept the slower drain as the price of isolation. Watch `inference_extension_flow_control_request_queue_duration_seconds` per tenant, because it is the honest record of who waited.

Priorities tell the router who matters more. Isolation is deciding, ahead of time, who never gets to show up all at once.

And life goes on...
