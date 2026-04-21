> This note summarizes two context-engineering patterns inspired by Manus: offloading context and compressing context, then connecting them to retrieval, isolation, and caching.

## 1. Offload Context

Do not push the full result of every tool call back into the message list.

Instead, write large outputs into the file system and retrieve them only when needed.

Useful patterns include:

- treating the file system as an external notebook
- using files such as `todo.md` for planning and progress tracking
- storing long tool outputs outside the prompt
- saving long-term memory in durable files

Once tool outputs are offloaded, the tool space can also become hierarchical:

- L1: function calling
- L2: sandbox utilities such as shell commands
- L3: scripts, packages, and APIs

This makes it possible to expose only the right level of action for the current task.

## 2. Compress Context

Beyond offloading, we can also compress the context that remains inside the model window.

Typical methods include:

- summarizing message history
- trimming irrelevant spans
- summarizing tool outputs
- compressing context during agent handoff

The key warning is information loss.

To manage that tradeoff better, it is useful to keep two versions of tool results:

- a full version
- a compact version

The compact version should retain only the summary and indexing information needed to reconstruct or revisit the full result later.

A practical strategy is:

1. save raw content to files first
2. start compression once context length crosses a threshold
3. compress older records earlier than recent ones
4. keep the most recent context in full form whenever possible

## 3. Retrieve Context

Once context has been offloaded or compressed, you need a retrieval strategy to bring back the right fragments.

Examples:

- hybrid retrieval with reranking
- systems that assemble multiple retrieval results into a coherent prompt
- searching tools by their descriptions

## 4. Isolate Context

Another pattern is splitting context across multiple agents instead of forcing everything into one prompt.

This can reduce context pressure, but it also introduces coordination risk:

- agents may make conflicting decisions
- high-risk decisions should usually stay centralized
- sub-agents work best for retrieval, analysis, and bounded information work

In other words, isolate context when useful, but avoid scattering final control too widely.

## 5. Cache Context

Caching can reduce both latency and cost.

A strong cache strategy usually keeps stable content near the front of the prompt, such as:

- agent instructions
- tool descriptions

Then it keeps fast-changing observations near the end, where they can be updated step by step.

## 6. Indexes and Vector Stores

For short-lived sandboxes, building an index during the session may be too slow.

That is why lightweight tools such as `grep` and `glob` are often better in the local code sandbox case.

For large external knowledge bases, vector stores still matter, but inside a short interactive runtime, speed often matters more than prebuilding an index.

