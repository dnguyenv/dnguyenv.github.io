---
title: "On Eager Loading and Architectural Indirection in Agent Design"
date: 2025-12-05
author: "Duy Nguyen"
theme: architect
tags: ["Agent", "AI", "MCP"]
description: "Agentic design patterns"
categories: ["AI Architecture", "Leadership"]
draft: false
---

One thing that's apparent is our industry is in a fervent rush to scale AI agents, a rush that sometimes leads us to invent complex new abstractions before we've fully interrogated our base assumptions. [A recent article from Anthropic on code execution within their Model Context Protocol (MCP)](https://www.anthropic.com/engineering/code-execution-with-mcp) is a fascinating case in point.

The authors describe a real and pressing challenge: as an agent is given access to hundreds or thousands of tools, the sheer volume of their definitions (their schemas, parameters, and descriptions) overloads the LLM's context window. This is a problem every team building agent at scale has encountered. Their proposed solution is to give the agent a single "code execution" tool, which provides a virtual filesystem. The agent can then ls this filesystem to discover tools, cat a file to read a tool's documentation, and finally write and execute code (e.t., TypeScript) to call it.

This is a clever piece of engineering. And yet, my feeling is that this solution—a sandboxed code interpreter with a virtual filesystem—is a profoundly complex workaround for a problem that is largely self-inflicted. The challenge isn't tool-use itself, but a naive architectural choice that MCP's initial design seems to encourage: Eager Loading.

### The Self-Inflicted Wound

The problem, as stated, is that thousands of tool definitions won't fit in a prompt. But this assumes that we must put all possible tool definitions into the prompt at the beginning of an interaction.

This is an Eager Loading pattern. It's the simplest thing that could possibly work, and it's a perfectly reasonable starting point for a system with a dozen, or perhaps even fifty, tools. You simply concatenate all the function signatures and descriptions, place them in the system prompt, and trust the LLM to find the right one.

This pattern, however, fails predictably at scale. It's the agent-architecture equivalent of a monolithic application that loads every single one of its plugins and libraries into memory on startup. No one would design a system like that today, yet we've fallen into the trap of doing it with our agents. The token bloat, latency, and cost issues are not inherent properties of LLM tool-use; they are symptoms of this specific, brittle architectural choice.

We are, in effect, complaining that our car is slow, having chosen to fill its trunk with a thousand tools from our garage "just in case" we might need one. The solution isn't to invent a new way to tow the garage behind the car, but to pack a smaller toolbox.

### On-Demand Tool Discovery Pattern

If the core problem is "how does the model find the right tool out of thousands at the right time," then this is a classic Information Retrieval and Service Discovery problem. We have decades of experience solving this.

A more reasonable and scalable architecture would be built on a foundation of Lazy Loading, or what I'll call On-Demand Tool Discovery. It would look something like this:

1. **Metadata Indexing and Semantic Retrieval (Tool RAG)**

   Instead of putting full tool definitions in the prompt, we treat them as documents in a corpus. We create a vector index of their names, descriptions, and perhaps usage examples. When an agent needs to act, its first step is to perform a semantic search over this index based on the user's intent. This is a Retrieval-Augmented Generation (RAG) pattern, but for tools, not for external knowledge.

   The model's context would then be populated with only the top k (e.g., 3-5) most relevant tool definitions, which it can then use to plan its next step.

2. **Hierarchical Discovery and Routing**

   A flat list of 2,000 tools is a poor design. A better approach is to use Hierarchical Toolsets. We could, for example, have a simple "router" model (or even just a tool-use call) that first classifies the user's intent.

   > User: "What's the status of my last order and is it raining in Raleigh?"
   >
   > Router: "This query requires the *ecommerce_toolbox* and the *weather_toolbox*."

   The system then loads only the definitions for those two toolsets (perhaps 15 tools in total) into the context for the main agent to use. This is a classic separation of concerns, dramatically reducing the search space for the primary model.

3. **Caching**

   Agent interactions are stateful conversations. There is no reason to re-discover and re-load the `get_weather` tool definition on every single turn of a conversation about the weather. Once a tool has been semantically discovered and loaded into the context, its definition should be cached and remain available for subsequent turns, either in the context window itself or in a separate "scratchpad" that the model can reference.

Many production-grade agent systems are already using piecemeal versions of these techniques because they are the only way to scale. My contention is that these patterns—retrieval, routing, and caching—should be first-class citizens of any agent protocol. They should be part of the MCP specification, not a set of desperate workarounds that each implementer must re-discover.

### Shifting Complexity, Not Solving It

This brings us back to the proposed filesystem solution. It's clever, but it doesn't truly solve the underlying problem; it just shifts the complexity.

The filesystem abstraction is, in itself, a form of on-demand discovery. The LLM must `ls` to find tools and `cat` to read their definitions. But it's an incredibly clumsy and inefficient metaphor for what is fundamentally an API discovery task.

1. **It's a Discovery Problem, Not a File-Browsing Problem**

   We are forcing the LLM to use bash-like commands to perform a task that a structured query or semantic search could accomplish in a single step. Why force the model to guess at filenames or browse a directory when we could just give it the top 3 most relevant tools from an index?

2. **It Introduces New Layers of Indirection**

   The proposed solution still requires the host to solve the discovery problem! The article states the host populates this filesystem "just-in-time" based on the user's query. This implies the host is already doing some form of semantic search or intent classification to figure out which tool definitions to place in the virtual `bin/` directory.

   This creates a Rube Goldberg-esque architecture:

   1. User makes a request.
   2. The Host performs a semantic search to find 10 relevant tools.
   3. The Host hides these 10 results from the LLM.
   4. Instead, the Host builds a virtual filesystem containing 10 files that represent those tools.
   5. The Host then asks the LLM to `ls` the directory to "discover" the 10 files it just put there.

   This adds layers of indirection and state management for no discernible architectural benefit. We are using a discovery mechanism to build a different discovery mechanism for the LLM to use.

3. **It Creates Latency and Fragility**

   This design multiplies the number of turns required to accomplish a simple task. What was one `tool_use` call now becomes:

   - Turn 1: User Request
   - Turn 2: LLM -> `ls /servers`
   - Turn 3: Host -> google-drive, salesforce
   - Turn 4: LLM -> `cat /servers/google-drive/getDocument.ts`
   - Turn 5: Host -> (file content)
   - Turn 6: LLM -> (Writes the code to call the tool)

   This is dramatically slower and more brittle. It conflates tool discovery (finding what to call) with code execution (finding how to call it). The code execution part is an interesting pattern for complex data manipulation (as the article rightly points out), but using it as the discovery mechanism seems deeply flawed.

### Architecture First

The fundamental problem is not "too many tools." The fundamental problem is a brittle Eager Loading architecture that fails at scale.

The filesystem-as-tool-browser abstraction doesn't fix this root cause. It's a complex, high-latency detour that introduces new operational burdens (managing a virtual filesystem per session) and still requires the host to solve the core discovery problem under the hood.

My feeling is that a robust specification like MCP should not be sidestepping such a fundamental architectural problem. Instead of inventing novel-but-clumsy abstractions, we should be formally incorporating proven software engineering patterns into the protocol itself.

What I'd like to see in future iterations of MCP is a formal specification for On-Demand Tool Discovery. Define a standard for how an agent can query a tool index, retrieve relevant definitions, and manage a cached context of "active" tools.

Let's stop trying to fit thousands of tools into a single prompt. Let's build an architecture that intelligently retrieves the one tool we need, right when we need it. That is a far more durable, scalable, and efficient solution.

And life goes on ...
