# 中文目录

本目录存放中文版本内容。

根据前言，这个仓库现在更适合按一条自学路径来理解，而不是按“教程章节”来理解。新的模块划分如下：

## Module 0 前言

先说明这个仓库的定位：它不是一套标准教程，而是一条持续重构的 Agent 自学路径。

- [00. 前言](./module-0-preface/before-we-begin.md)

## Module 1 LLM 与 API 基础

如果你没有任何 LLM 基础，建议从这里开始。你必须先对 GPT 模型、Transformer、token、上下文窗口这些概念有基本认知，再继续进入 Agent 开发。

这一模块也放入 Prompt Cache 和 Responses API 格式。原因是它们还不是 Agent 的复杂系统能力，而是理解 OpenAI API、上下文成本、工具接口和对话格式的入门地基。

- [01. What Is AI Agent](./module-1-llm-and-api-basics/01-what-is-ai-agent/what-is-an-ai-agent.md)
- [02. LLM Fundamentals](./module-1-llm-and-api-basics/02-llm-fundamentals/llm-fundamentals.md)
- [03. Prompt Cache](./module-1-llm-and-api-basics/03-prompt-cache/prompt-cache.md)
- [04. From Prompt To Responses API](./module-1-llm-and-api-basics/04-prompt-to-responses-api/from-prompt-to-responses-api.md)

## Module 2 AgentLoop

当你理解了 Responses API 格式，以及工具调用也是模型 output 的一种之后，再看 AgentLoop：模型如何一轮一轮读取上下文、决定下一步、接收结果、更新状态，并继续推进任务。

- [05. AgentLoop](./module-2-agent-loop/01-agent-loop.md)

## Module 3 Context Engineering

当 Agent 面对不断增长的上下文时，就需要上下文管理、压缩、记忆、检索、任务状态等一整套手段。这就是 Context Engineering。

- [07. Context Engineering](./module-3-context-engineering/context-engineering.md)
- [Context Management](./module-3-context-engineering/context-engineering)

## Module 4 Harness Engineering

当你想建立一个 ToC 产品，或者做一个能取代部分人类工作的产品时，最重要的事情不是“看起来很聪明”，而是风险可控。

这一模块会关注 Agent 的执行边界、权限控制、人工接管、回滚、可观测性、审计和安全运行环境。

- [Harness Engineering](./module-4-harness-engineering/README.md)（待写）

## Module 5 Eval 与 Benchmark

当你想真正做出一个合格的 Agent 产品，就需要合理的 Eval 体系和足够贴近业务的 Benchmark。它们让你能持续优化，也能量化判断系统到底有没有变好。

- [08. Evaluation](./module-5-eval-benchmark/evaluation.md)

## Module 6 Agent Features

这一模块放 Agent 产品里的重要 feature。Dream、主动性、Task、Agent Team 和 Deep Research 都不应该被理解成“未来方向”，它们是 Agent 能不能真正持续工作的关键能力。

- [Dream](./module-6-agent-features/01-dream/dream.md)
- [Task And Proactivity](./module-6-agent-features/02-task-and-proactivity/task-and-proactivity.md)
- [Agent Team](./module-6-agent-features/03-agent-team/agent-team.md)
- [Deep Research](./module-6-agent-features/04-deep-research/deep-research.md)

## Module 7 复杂系统与个人思考

这一模块暂时放一些还没有纳入主线展开的复杂系统话题和个人思考。

多智能体暂时放在这一模块里，而不是主线起点。因为我现在还没有一个足够清晰的多智能体认知。它为什么有用，至少有一个很重要的原因：Gemini、Anthropic 和 OpenAI 都认同一个方向，多智能体系统有效，主要是因为它们能帮助智能体消耗足够的 token 去解决问题。

- [09. Multi-Agent](./module-7-complex-systems-and-thoughts/09-multi-agent/multi-agent.md)
- [10. 我对 Agent 的思考](./module-7-complex-systems-and-thoughts/10-thoughts-on-agents/thoughts-on-agents.md)
- [AI 时代，来自程序员送给非程序员的一封情书](./module-7-complex-systems-and-thoughts/AI%20时代，来自程序员送给非程序员的一封情书.md)

## 推荐阅读顺序

1. [前言](./module-0-preface/before-we-begin.md)
2. [02. LLM Fundamentals](./module-1-llm-and-api-basics/02-llm-fundamentals/llm-fundamentals.md)
3. [03. Prompt Cache](./module-1-llm-and-api-basics/03-prompt-cache/prompt-cache.md)
4. [04. From Prompt To Responses API](./module-1-llm-and-api-basics/04-prompt-to-responses-api/from-prompt-to-responses-api.md)
5. [05. AgentLoop](./module-2-agent-loop/01-agent-loop.md)
6. [07. Context Engineering](./module-3-context-engineering/context-engineering.md)
7. [Harness Engineering](./module-4-harness-engineering/README.md)（待写）
8. [08. Evaluation](./module-5-eval-benchmark/evaluation.md)
9. [Dream](./module-6-agent-features/01-dream/dream.md)
10. [Task And Proactivity](./module-6-agent-features/02-task-and-proactivity/task-and-proactivity.md)
11. [Agent Team](./module-6-agent-features/03-agent-team/agent-team.md)
12. [Deep Research](./module-6-agent-features/04-deep-research/deep-research.md)
13. [09. Multi-Agent](./module-7-complex-systems-and-thoughts/09-multi-agent/multi-agent.md)
