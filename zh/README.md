# Build AI Agent From Scratch 中文版

这里是这个仓库的中文内容。

它不是一套严格线性的教程，更像是一条持续整理中的 Agent 自学路径。建议先按下面的顺序读完主线内容，建立对 LLM、API、Agent Loop、Context Engineering 和 Evaluation 的基本框架；剩下的主题可以按兴趣随意挑。

## 阅读建议

如果你完全没有 LLM 基础，我建议你先从 [llm-fundamentals](./module-1-llm-and-api-basics/01-llm-fundamentals/llm-fundamentals.md) 看起。你至少要先对 GPT 这类模型有一个基本认知。

然后再去看提示词缓存。你需要明白它为什么重要，以及工作原理。

接下来再去了解 OpenAI 的 Responses API Message Format，以及 AgentLoop。走到这里，才算是真正开始入门 Agent 开发。

当你开始面对不断增长的上下文时，就需要一些管理上下文的手段，这一部分就是 Context Engineering。

当你想做一个 ToC 产品，或者想让 Agent 真的去替代一部分人的工作时，最重要的事情就是把风险控制住，这一部分就到了 Harness Engineering。

当你想把东西真正做成一个合格的产品时，就需要补上 Eval 体系和可观测体系。

现在相关的 Benchmark 已经足够多了，完全够 AI Agent 开发工程师持续优化和量化自己的产品。可观测体系这边，最基础的事情就是把关键节点的日志记录好。

当然，所谓合格的产品，本质上是能持续解决大多数用户的问题，所以你还得不断去看真实用户到底是怎么用的。更优秀的产品，往往是连长尾问题也能处理掉。

再往后，就是去看一些其他的功能特点，比如持续学习、Dream、Task、主动性这些功能。

这里我暂时没有展开讲多智能体（Agent Team、Agent Swarm、DeepResearch、A2A 等），因为我自己对多智能体这件事也还没有形成足够清晰的认知。

但多智能体为什么有用，Gemini、Anthropic 和 OpenAI 基本都持有一个相近的看法：多智能体系统之所以有效，很大程度上是因为它们能够帮助系统消耗足够多的令牌去解决问题。

## 推荐阅读顺序

1. [02. LLM Fundamentals](module-1-llm-and-api-basics/01-llm-fundamentals/llm-fundamentals.md)
2. [03. Prompt Cache](module-1-llm-and-api-basics/02-prompt-cache/prompt-cache.md)
3. [04. From Prompt To Responses API](module-1-llm-and-api-basics/03-prompt-to-responses-api/from-prompt-to-responses-api.md)
4. [05. AgentLoop](./module-2-agent-loop/01-agent-loop/agent-loop.md)
5. [07. Context Engineering](./module-3-context-engineering/01-context-engineering/context-engineering.md)
6. [08. Evaluation](./module-5-eval-benchmark/01-evaluation-system/evaluation-system)
7. 剩下的随意

## 内容目录

### Module 0 前言

- [00. 前言](./module-0-preface/before-we-begin.md)

### Module 1 LLM 与 API 基础

- [01. What Is AI Agent](./module-1-llm-and-api-basics/01-what-is-ai-agent/what-is-an-ai-agent.md)
- [02. LLM Fundamentals](module-1-llm-and-api-basics/01-llm-fundamentals/llm-fundamentals.md)
- [03. Prompt Cache](module-1-llm-and-api-basics/02-prompt-cache/prompt-cache.md)
- [04. From Prompt To Responses API](module-1-llm-and-api-basics/03-prompt-to-responses-api/from-prompt-to-responses-api.md)

### Module 2 AgentLoop

- [05. AgentLoop](./module-2-agent-loop/01-agent-loop/agent-loop.md)

### Module 3 Context Engineering

- [07. Context Engineering](./module-3-context-engineering/01-context-engineering/context-engineering.md)

### Module 4 Harness Engineering

- [Harness Engineering](./module-4-harness-engineering/README.md)

### Module 5 Eval 与 Benchmark

- [08. Evaluation](./module-5-eval-benchmark/01-evaluation-system/evaluation-system)

### Module 6 Agent Features

- [Dream](./module-6-agent-features/01-dream/dream.md)
- [Task And Proactivity](./module-6-agent-features/02-proactivity/task.md)
- [Agent Team](./module-6-agent-features/03-multi-agents/agent-team/agent-team.md)
- [Multi-Agent](./module-6-agent-features/03-multi-agents/multi-agent.md)
- [Deep Research](./module-6-agent-features/04-deep-research/deep-research.md)

### Module 7 复杂系统与个人思考

- [我对 Agent 的思考](./module-7-complex-systems-and-thoughts/01-thoughts-on-agents/thoughts-on-agents.md)
- [如何开始 AI Coding](./module-7-complex-systems-and-thoughts/如何开始%20AI%20Coding.md)
