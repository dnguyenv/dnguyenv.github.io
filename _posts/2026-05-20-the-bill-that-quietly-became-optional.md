---
title: "Frontier inference is free now. Sort of."
date: 2026-05-20
author: "Duy Nguyen"
tags: ["AI", "Developer Tools", "NVIDIA", "Inference"]
description: "What NIM actually is, the model identifiers that genuinely work today, the constraints worth understanding before you route real traffic through it, and a small companion tool for readers who want to try it without writing code."
categories: ["AI"]
draft: false
---

***What's in the catalog (and what isn't)***

Exploring a new frontier model used to mean paying for it, usually through the same OpenAI or Anthropic billing relationship that already eats an uncomfortable share of a team's monthly spend. NVIDIA's NIM catalog at [build.nvidia.com](https://build.nvidia.com) has been quietly changing that. Hopefully it will continue doing so for a long time. 
NIM is 2 things. A packaging format for self-hosting inference on NVIDIA hardware, which matters if you're running your own infrastructure. And a hosted catalog you can call right now through a single OpenAI-compatible endpoint at `integrate.api.nvidia.com/v1`, free tier included.
The free tier runs on request-per-minute ceilings. The cap resets. You can do serious evaluation work within those limits without worrying about a budget draining or a trial clock ticking.
The catalog holds over 100 models: NVIDIA's Nemotron family, the major frontier labs, Meta, Mistral, Qwen, and recent open-weights releases from US providers.
A practical warning before you go hunting for identifiers: a lot of the names circulating in posts about NIM don't exist. I checked `docs.api.nvidia.com` before writing this. The 8 I actually reach for are `nvidia/nemotron-3-super-120b-a12b`, `openai/gpt-oss-120b`, `deepseek-ai/deepseek-v4-pro`, `moonshotai/kimi-k2-thinking`, `minimaxai/minimax-m2.7`, `z-ai/glm5.1`, `qwen/qwen3-coder-480b-a35b-instruct`, and `meta/llama-3.3-70b-instruct`. Others like "DeepSeek V3.2" or "Kimi K2.5" that have been circulating are ... FabRiCatIonS, as far as I can tell. My takes:
- For general agentic work I default to `nemotron-3-super-120b-a12b`. 
- The hybrid Mamba-Transformer architecture handles multi-turn, tool-using sessions pretty well. 
- For multi-file code reasoning, `deepseek-v4-pro` wins on the context window alone: 1 million tokens stops being a marketing figure when you're feeding it an unfamiliar codebase (!). 
- Single-shot code generation goes to `qwen3-coder-480b-a35b-instruct` by sheer scale.
- `kimi-k2-thinking` is my pick when I want explicit step-by-step reasoning and can absorb the latency. 
- `minimax-m2.7` and `glm5.1` sit in my A/B rotation, the 1st stronger on general instruction following, the 2nd on Chinese-language work. 
- `gpt-oss-120b` earns its place when a familiar reasoning profile matters, and the corresponding bill ... doesn't. 
- `llama-3.3-70b-instruct` is the baseline everything else has to beat.
Because the endpoint adheres to the same chat-completions contract every major inference client already supports, your existing tooling mostly works without modification. LangChain, LlamaIndex, Vercel AI SDK, Cursor, Zed: all of them. Change the base URL, change the key, and you're in.

***First call in under 60 seconds***

Getting to a first response takes ~ 60 seconds. Sign in at [build.nvidia.com](https://build.nvidia.com), create a key in the API keys panel, set it as `NVIDIA_API_KEY` in your shell, and run this:
```python
import os
from openai import OpenAI

client = OpenAI(
    base_url="https://integrate.api.nvidia.com/v1",
    api_key=os.environ["NVIDIA_API_KEY"],
)
response = client.chat.completions.create(
    model="nvidia/nemotron-3-super-120b-a12b",
    messages=[{"role": "user", "content": "Explain Lambda architecture in a short paragraph."}],
)
print(response.choices[0].message.content)
```
For the prompt above, against `nvidia/nemotron-3-super-120b-a12b`, the model produces a "short" technical summary opening along the lines of:

> _Lambda architecture is a data-processing pattern designed to handle massive quantities of data by balancing latency and accuracy through three distinct layers. The **batch layer** serves as the system of record, continuously recomputing a master dataset to provide the most accurate, immutable views, albeit with high latency. To compensate for this delay, the **speed layer** processes only the recent, incremental data in real-time to generate low-latency, approximate views. Finally, the **serving layer** merges the outputs from both the batch and speed layers, enabling queries that combine historical accuracy with up-to-the-moment freshness._

The `response` object carries more than just the content string. `usage` gives you prompt and completion token counts. `model` tells you the served identifier, which matters when NVIDIA routes you to a versioned underlying weight. `choices[0].finish_reason` catches truncations. For 1st class exploration the content string is usually enough.

***Before real traffic hits it***

Before you route real traffic through the endpoint, a few things are worth knowing. Rate limits have moved at least once since the platform launched, so check your own dashboard rather than any figure quoted in a blog post (including this one). Latency at the median is fine, but the tail is wider than paid first-party endpoints, and that compounds in agentic workflows chaining many sequential calls while someone waits. NVIDIA's terms also govern what they can do with your prompts and completions, and those aren't necessarily the same commitments your existing provider made in an enterprise contract. In a regulated context, compare the data retention language, not the price per token.

For readers who'd rather skip the Python REPL, I built a companion page at [/try-nim/](/try-nim/). It doesn't call the NIM API. The page instead takes your key, your chosen model, and your prompt, and simply assembles a copy-paste-ready curl or Python command you can run yourself. Your key stays in your browser.

The free tier itself is the least interesting part. Companies give inference away for strategic reasons all the time, and NVIDIA's aren't hard to infer. What matters more is where the cost actually shifted. Evaluation is now close to free, which changes how sustainable the "we haven't tested that model yet" answer actually is. In my experience, most conversations that stall on model adoption are still running on cost intuitions formed when frontier inference was genuinely expensive. Those intuitions outlast the price drops that should have revised them. NIM makes that lag harder to defend.

And life goes on...
