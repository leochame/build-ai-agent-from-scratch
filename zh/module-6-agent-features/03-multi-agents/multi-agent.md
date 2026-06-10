

### 3.5. SubAgent

- **在多个 Agent 之间拆分上下文负载**（参考：[Drew 的文章](https://www.dbreunig.com/2025/06/26/how-to-fix-your-context.html)、[Anthropic 多 Agent 系统](https://www.anthropic.com/engineering/built-multi-agent-research-system)）。

但这里也有不少坑（参考：[Cognition](https://cognition.ai/blog/dont-build-multi-agents#a-theory-of-building-long-running-agents)、[Walden Yan](https://x.com/jxnlco/status/1945490018127987092)）：

- Multi Agent System 很容易做出互相冲突的决策（原因见上面同一批参考）。
- 子 Agent 在设计上最好避免直接做高风险决策，而是把重心放在检索、分析、信息收集等工作上（参考：[open-deep-research](https://github.com/langchain-ai/open_deep_research)）。

换句话说：在 Agent 之间隔离上下文，以减少单一上下文的负担；但在决策层面尽量集中控制，避免系统整体陷入混乱。

在多智能体协同中，可以粗略区分两类协作模式：
- 只需要简单委派的任务，更偏向「Communication」；
- 与历史高度相关、依赖长期记忆的子任务，则更强调在 Agent 之间「Share Context」。

---

常见的实现比如说是在 search result 这种 token 量很大的工具输出，一方面需要防止上下文膨胀，另一方面又需要保证信息可访问

那么，这时候可以用一个分层策略，按任务复杂度区分处理方式：

- **复杂任务：** 使用 **sub-agents（或 “agent-as-a-tool”）**，并为其定义固定的输出 Schema。
  这样可以把复杂的工作流封装在子 Agent 内部，只把结构化的、必要的结果返回给主 Agent。

- **简单任务：** 在一开始可以直接返回完整细节，但**在后续阶段进行压缩（compress）**：
  把原始数据卸载到 external state（比如 File System 或 URL 等），在对话上下文中只保留唯一标识符（ID）。

- **信息持久化：** 明确指示模型把中间洞见和关键发现记录到文件中。即使之后对话历史被压缩或裁剪，这些关键信息仍能通过文件被重新加载回来。
