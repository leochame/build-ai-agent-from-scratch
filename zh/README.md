# Build AI Agent From Scratch 中文版

## 阅读建议

如果你完全没有 LLM 基础，我建议你先从 [llm-fundamentals](./module-1-llm-and-api-basics/01-llm-fundamentals/llm-fundamentals.md) 看起。你至少要先对 GPT 这类模型有一个基本认知。

然后再去看提示词缓存。你需要明白它为什么重要，以及其工作原理。

接下来再去了解 OpenAI 的 Responses API Message Format，以及 AgentLoop。走到这里，才算是真正开始入门 Agent 开发。

当你开始面对不断增长的上下文时，就需要一些管理上下文的手段，这一部分就是 Context Engineering。

当你想做一个 ToC 产品，或者想让 Agent 真的去替代一部分人的工作时，那这个就是 risk control，这一部分就到了 Harness Engineering。

当你想把东西真正做成一个合格的产品时，就需要补上 Eval 系统和可观测体系。

再往后，就是去看一些其他的可能比较重要的功能，比如持续学习、Dream、主动性等等。

多智能体（Agent Team、Agent Swarm等）当然也是一个比较重要的部分。

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
