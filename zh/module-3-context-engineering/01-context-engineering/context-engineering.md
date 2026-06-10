# Context Engineer

所以如果要用一句话概括 Context Engineer，我会把它理解成：

如何在有限的 token 预算里，让模型在当前时刻看到最值得看到的信息。


## 1. 上下文管理到底在管理什么

先把一个边界讲清楚。

上下文不是记忆，上下文是模型在当前这一轮调用里真正能看到的内容。

比如：

- system prompt
- 用户当前输入
- 对话历史
- 工具调用结果
- 检索回来的资料
- 当前任务状态
- 临时生成的计划

这些内容一起，才构成这一次调用的 context。memory 更像是一个信息仓库。它里面可以存很多东西，但只有当系统把某些记忆取出来，放进本轮调用里，它才会变成 context。

所以从工程角度看：

- memory 是潜在可用信息
- context 是当前实际注入信息

Context Engineer 管理的，就是从所有潜在信息里，挑出当前最值得给模型看的那一部分。

## 2. 我会怎么拆这个问题

如果从上下文真正形成的过程来看，我会把 Context Engineer 大致分成两部分：

- pre-handle
- post-handle

这个划分不一定是标准术语，但它在工程上是很有用的。

`pre-handle` 关注的是：在模型调用之前，系统怎样决定“该拿什么信息进来”。

`post-handle` 关注的是：在模型调用之后，系统怎样处理已经产生的上下文和结果，避免上下文不断膨胀、污染和失控。



## 3. Pre-handle

目前来看，pre-handle 的核心思想其实就是 `Progressive Disclosure`。

也就是不要一开始把所有信息都摊给模型，而是按需要逐步暴露。

### 3.1 Skill

`skill` 很多人会觉得它有营销成分，但如果把概念拆开来看，它真正重要的地方不在名字，而在它体现出来的思路：把一组只在特定场景下才需要的知识、流程、约束和操作习惯，延迟到真正需要的时候再注入。 这本质上就是 Progressive Disclosure。

一个 skill 里通常会包含这些东西：

- 某类任务的操作说明
- 对应任务的最佳实践
- 需要注意的边界条件
- 常见错误和规避方式
- 可用工具说明

它的价值在于它把原本不应该始终驻留在主 prompt 里的内容，从默认上下文里拆了出去。

至于召回的方式，其实很多样：

- 用本地文件组织 skill
- 用检索系统按需召回 skill
- 用 function calling 触发 skill 加载
- 用目录搜索、关键词匹配、grep 等方式选择相关说明

所以 skill 不一定是一种单独技术，更像是一种上下文组织策略。

### 3.2 Tool Search

我们可以把「工具空间」组织成一个分层的动作空间（hierarchical action space）。

根据当前任务或当前状态，按需加载工具，而不是一次性把所有工具都挂在模型前面。但这样做有几个副作用：KV Cache 不能复用，而且「曾经使用过、后来被卸载的工具」这些历史调用仍然留在上下文中，会导致幻觉问题，模型有时还是会尝试去调用那些已经不可用的工具。

在这个前提下，可以把工具层次大致分成三个抽象层级：函数调用层、沙箱工具层、以及脚本 / 包 & API 层。

- **L1：函数调用（Function Calling）**
    - 标准化、Schema 安全。
    - 只要函数列表有改动，缓存几乎就要重建。
    - 函数太多会造成上下文混乱。

- **L2：沙箱工具（Sandbox Utilities）**
    - 每个会话在一个完整的 VM 沙箱中运行。
    - 模型可以直接调用 shell 工具（CLI）。
    - 非常容易扩展新能力，不用改动模型的系统提示或函数列表。
    - 对于输出很大的任务，可以直接写入文件，不挤占上下文。

- **L3：脚本 / 包 & APIs（Packages & APIs）**
    - Manus 会写 Python 脚本来调用预授权的 API。
    - 特别适合数据量大、需要多步调用的复杂流程。
    - 可以在运行时内存中处理大体量数据，只把最后关键信息返回给模型。
    - 保持模型上下文干净，把上下文留给「推理」，而不是「搬运数据」。
    - 第 3 层的用法和 CodeAct 风格的工具调用很相似。

---

**Q：Manus 是如何管理工具发现，以及在 shell 命令和沙箱代码之间进行混合执行的？**

**A：** Manus 采用了一种**混合架构**来执行任务。对于标准化的操作，它会在系统提示中维护一份精简的「预装 shell 工具列表」，并直接调用这些命令，通过 `--help` 来理解命令的参数格式和用法。


### 3.3 Code Act

在 agent 里，`Code Act` 也是很典型的 pre-handle 操作。

如果一个任务涉及多步骤操作，而且这些步骤的执行路径比较明确，比如先读取某个 excel，再做清洗、分组、统计、排序，最后输出一个摘要表。那么，这个多步骤操作就不一定要拆成多次工具调用，每一步都把中间结果塞回上下文。更好的方式是让模型生成一段代码，把这一串操作放到沙箱里一次性执行，只把最终结果、异常信息或少量关键中间状态返回给模型。

这就是 CodeAct 的核心价值：把「可程序化的操作」,转化为在代码沙箱里生成并运行的脚本。进行循环、过滤、计算、文件读写、批量请求、数据转换等确定性强的操作。这样，代理可以在运行时内存里处理大批量数据，只把真正相关的最终结果返回给模型。

它不太适合完全开放式、强依赖人类判断、每一步都需要重新推理方向的任务，这类任务仍然更适合让模型在上下文中逐步推理。




## 4. Post-handle

Agent 一旦进入多轮交互、频繁工具调用、长任务执行之后，上下文会天然膨胀。很容易出现上下文腐烂的问题。

### 4.1. 上下文腐烂阈值

进行 post-handle，通常不是等到 context window 被彻底塞满，而是在系统接近一个「上下文腐烂阈值」时就提前介入。这里的“上下文腐烂”并不是指窗口容量的物理上限，而是一种更早出现的性能边际递减现象。

基座模型宣称有 1M 甚至更大的上下文窗口，这只说明“理论上能塞进去多少 token”，不代表你的 agent 在那个长度上还能稳定工作。在很多真实系统里，128K 到 200K 往往已经是很关键的退化临界点。

也就是说，上下文虽然还没撞上硬性的 token limit，但模型已经开始出现功能性退化。典型表现包括：

- 逻辑重复，开始反复搜索、反复读取、反复规划；
- 推理延迟增加，每一步都变慢；
- 指令遵循能力下降，容易忽略前面已经明确的约束；
- 工具调用逻辑变乱，开始调用不合适的工具，或者在错误的时机调用工具；
- 上下文里堆满了历史步骤和高密度观测结果，但模型越来越抓不住“现在最该做什么”。

以 Manus、Codex、Deep Research 这一类 agent 为例，频繁的 `web_search`、文件读取、命令执行、页面抓取，都会产生海量 tool result。这些 tool result 往往还是高密度 token 负载：日志长、细节多、局部有用、整体冗余。

如果不做干预，这些陈旧的、高密度的负载会迅速稀释模型真正可用的“认知带宽”。结果就是：窗口还没满，但模型已经开始变笨。

---

但这个上下文腐烂阈值的数字不能直接照搬，架构师必须通过真实任务去测：

- 当上下文增长到什么区间时，重复行为开始明显上升；
- 当 observation 积累到什么规模时，工具调用开始变乱；
- 当 summary 频率降低时，任务成功率会在哪个点开始下降；
- 当压缩触发得太晚时，平均完成时间和失败率会怎么变化。


### 4.2. Offload Context

不要把每一次工具调用的完整结果都塞回到消息列表里。相反，应该把这些结果写进 File System 里的 file，需要的时候再按需检索回来。

- 把 File System 当做 Note（参考：[Drew 的文章](https://www.dbreunig.com/2025/06/26/how-to-fix-your-context.html)、[Anthropic Multi Agent System](https://www.anthropic.com/engineering/built-multi-agent-research-system)）。
- 用 File System（例如 `todo.md`）做计划和进度跟踪（参考：[Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)）。
- 用 File System 读写那些 token 量很大的上下文（参考：[Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)）。
- 用 file 保存长期记忆（参考：Ambient Agents 的[课程](https://academy.langchain.com/courses/ambient-agents/) / [仓库](https://github.com/langchain-ai/agents-from-scratch)）。

---

### 4.3. Reduce Context

除了把数据卸载到外部存储，我们也可以在「留在模型里的上下文」上动手脚，更聪明地压缩它。

为了更好管理这个取舍，可以先统一工具调用的内部表示方式：

每一次工具调用及其结果，都应该同时维护两种版本：**full version** 和 **compact version**。
compact version 会剔除所有那些可以从 File System 或其他 external state 中可靠重建的信息，只保留真正必要的 summary 和 index 信息。

在进行任何总结之前，先把「summarize 之前的 raw content」以 text file 或简单 log 的形式写入 File System，这样一旦需要，就能从 File System 中恢复全部细节。

接下来，为你的 context window 设置一个「pre-rot threshold」。如果对话长度一旦超过这个阈值，就开始执行压缩策略：例如，优先压缩最早的那 50% 工具调用记录，而把最近的调用保持原样。

压缩完成后，再检查一下你实际释放出来的 context space。如果节省得不多，可以进一步压缩：
比如把剩余仍保留 full context 的部分也再 summarize 一遍——但**始终保留最新的一小段上下文为完整形态**，这样模型才能清楚地知道「自己是在哪里停下来的」。

Q：AI Agent 如何在不丢失关键信息的前提下，做高召回率（high-recall）的 summary？

A：不要只用自由文本总结，而要用结构化 output （ Schema）。

> Open Code 中的压缩：
> - Goal： 用户目标
> - Instructions 重要用户指令和计划
> - Discoveries： 对话中的重要发现
> - Accomplished： 已完成/运行中/剩余
> - Relevent Files： 相关文件

---

- 总结 Agent 的消息历史（参考：[Drew 的文章](https://www.dbreunig.com/2025/06/26/how-to-fix-your-context.html)、Claude Code）。
- 裁剪与当前任务无关的消息片段（参考：[同一篇 Drew 的文章](https://www.dbreunig.com/2025/06/26/how-to-fix-your-context.html)）。
- 对工具调用结果进行总结或裁剪（参考：[open-deep-research](https://github.com/langchain-ai/open_deep_research)）。
- 在 Agent 与 Agent 之间进行「交接」时，对上下文进行总结或裁剪（参考：[Cognition](https://cognition.ai/blog/dont-build-multi-agents#a-theory-of-building-long-running-agents)）。

但要非常小心信息损失（参考：[Cognition](https://cognition.ai/blog/dont-build-multi-agents#a-theory-of-building-long-running-agents) 和 [Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)）！目标是在尽可能节省上下文的同时，避免毁掉后续还需要的信息。

---

### 4.4. Retrieve Context

当我们已经把上下文卸载和压缩之后，就需要一套好的策略把「正确的那一小部分」重新带回到模型上下文里：

- **混合检索 + 重排序（re-ranking）**（参考：Windsurf 团队 Varun 的[分享](https://x.com/_mohansolo/status/1899630246862966837?ref=blog.langchain.com)）。
- **构建可以把多个检索结果自动组装成连贯提示的系统**（参考：Cursor 中的 [Preempt](https://lexfridman.com/cursor-team-transcript)）。
- **按工具描述对工具本身做检索**（参考：[Drew 的文章](https://www.dbreunig.com/2025/06/26/how-to-fix-your-context.html)）。


---

### 4.5. Cache Context

除了检索和隔离之外，缓存也可以显著降低成本和延迟：

- 对 Claude Sonnet 来说，缓存中的输入 token 价格最多可以便宜 10 倍。
- 把 Agent 指令和工具描述放在提示词前缀中，并尽可能缓存。
- 把可变上下文和最近观察到的信息放在提示词后缀中，每步更新。

一个好的缓存策略，会让系统中那些「Stable 的部分」始终维持在 Message 的前半部分，而把那些「随每一步变化的部分」放在后半部分，按步更新。
