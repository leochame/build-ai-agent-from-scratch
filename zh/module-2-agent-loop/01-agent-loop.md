# 05. AgentLoop

上一章我们已经把一次模型调用拆开了：

```text
messages / input items -> model -> output items
```

也就是说，模型的输入不再只是一整段 prompt，而是一组有来源、有角色、有结构的上下文。模型的输出也不再只是一段文本，而可能包含 assistant message、文本内容、tool call、reasoning item 等不同类型的 output item。

而 AgentLoop 本质上是把“构造输入、调用模型、处理输出”这个过程放进一个循环里，并且允许模型通过工具和外部世界发生交互。

如果上一章讲的是“一次模型调用长什么样”，那这一章讲的就是：

一次又一次模型调用如何组成一个 Agent。

## 从 Chat Loop 再往前走一步

上一章最后我们写过一个最小聊天程序。

它的结构是一问一答：

```text
user message
-> model
-> assistant message
-> 等待下一个 user message
```

这就是 Chat Loop。

Chat Loop 的核心是“对话继续”。用户说一句，模型回一句，然后程序等待用户下一次输入。

但 AgentLoop 的核心不是“对话继续”，而是“任务继续”。

用户给 Agent 一个目标之后，Agent 不一定马上回答。它可能需要先读取文件、搜索资料、运行命令、调用 API、观察结果，然后再决定下一步。

上一章已经讲过：模型可以把 tool call 作为 output 交给应用，应用执行工具，再把工具结果放回下一轮输入。AgentLoop 关注的不是这些字段具体长什么样，而是应用如何把这个过程组织成一个持续推进任务的循环。

## AgentLoop 的核心循环

一个 AgentLoop 大概会反复做这些事：

```text
接收目标
-> 构造上下文
-> 调用模型
-> 解析输出
-> 执行工具
-> 记录 observation
-> 更新状态
-> 判断继续或停止
```

这里的重点是：每一轮模型调用之后，应用都要决定下一步。

如果模型输出的是最终回答，应用可以把结果返回给用户。

如果模型输出的是工具请求，应用要执行工具，把结果整理成 observation，再进入下一轮模型调用。

如果工具失败、权限不足、目标不清楚，应用可能要停下来问用户，而不是继续自动跑下去。

所以 AgentLoop 不是模型自己在循环。

它是应用层在维护一个任务循环：模型负责判断下一步可能需要什么，应用负责执行动作、维护状态、控制风险，并决定循环是否继续。

## 一个最小 AgentLoop 长什么样

还是用上一章的 `list_files` 例子，但这一章不再展开 API 字段。

从 AgentLoop 视角看，它更像一条时间线：

```text
用户：帮我看看当前目录下有哪些文件。

模型：我需要知道当前目录内容，因此请求调用 list_files。

应用：执行 list_files。

observation：当前目录下有 README.md、package.json、src。

模型：当前目录下有 README.md、package.json 和 src。
```

如果任务到这里已经满足，循环就停止。

但模型也可能根据 observation 继续行动：

```text
用户：帮我判断这个项目是什么类型。

模型：我需要先查看当前目录。
应用：执行 list_files。
observation：当前目录下有 README.md、package.json、src。

模型：我看到了 package.json，需要读取它来判断项目类型。
应用：执行 read_file("package.json")。
observation：package.json 里包含 scripts、dependencies 和 devDependencies。

模型：这是一个 Node.js 项目，并且可能使用 npm 管理依赖。
```

从用户视角看，这可能只是一轮请求。

从 AgentLoop 视角看，里面已经发生了多次“模型判断下一步 -> 应用执行动作 -> 观察结果回到上下文”的循环。

这也是 Agent 和普通聊天机器人的关键区别：普通聊天机器人主要生成回复，Agent 需要在外部世界里持续观察和行动。

## Observation 不只是工具原始输出

很多时候，工具结果不应该原封不动地全部塞回上下文。

比如你让 Agent 运行测试，测试输出可能有几千行。如果全部放回去，会浪费 token，也会把真正重要的错误埋掉。

但如果只告诉模型 “测试失败” ，模型又没有足够信息继续修复。

所以工具结果进入下一轮之前，经常需要做一层整理。这个整理后的内容，我们可以把它理解成 observation。

一个好的 observation 至少应该包含：

- 工具是否执行成功；
- 退出码或错误类型；
- 关键输出；
- 输出是否被截断；
- 下一步可能需要继续查看哪些信息。

比如：

```text
命令 npm test 执行失败，exit code = 1。
关键错误：
  expected "hello" to equal "helo"
输出已截断，只保留最后 80 行。
```

这比几千行日志更适合作为下一轮模型输入。

Observation 的很多相关操作，我们会留在上下文工程的章节中讲解。

## AgentLoop 里的状态

上一章讲多轮对话时，我们说过一句话：对话状态不在模型身体里，而在你的应用里。到了 AgentLoop，这句话依然成立，只是状态变多了。

一个 Agent 可能需要维护：

- 对话历史；
- 已经调用过的工具；
- 工具返回的结果；
- 当前任务目标；
- 当前计划；
- 已完成和未完成的步骤；
- 用户确认过的权限；
- 文件修改记录；
- 测试结果；
- 停止原因。

这些状态不一定都要原样塞进 prompt。
但它们必须被应用管理起来。

比如一个 coding agent 在修 bug 时，它可能需要知道：

```text
目标：修复登录接口测试失败
已做：
- 搜索了 login 相关代码
- 读取了 auth.test.ts
- 修改了 validatePassword
- 运行测试失败
当前阻塞：
- 失败原因可能和 password hash mock 有关
下一步：
- 读取 mock 文件
```

这类信息如果组织得好，Agent 会越来越接近目标。
如果组织得差，Agent 就会反复读同一个文件、重复跑同一个错误命令，或者忘记自己刚才改过什么。

所以 AgentLoop 不是一个简单的无限循环。

更准确地说，它每一轮都在更新一组状态：

```text
当前目标是什么？
模型刚刚输出了什么？
有没有工具要执行？
工具结果改变了什么？
现在应该继续、停止，还是问用户？
```

## AgentLoop 什么时候停止

如果 AgentLoop 只是一直循环，那它很快就会变成失控的自动化。所以一个 Agent 必须知道什么时候停。

最简单的停止条件是：模型没有再输出 tool call，而是输出了最终 assistant message。

但真实系统通常还需要更多停止条件：

- 达到最大模型调用次数；
- 达到最大工具调用次数；
- 达到 token、时间或成本预算；
- 工具连续失败；
- 缺少权限，需要用户确认；
- 任务目标已经满足；
- 测试或验证已经通过；
- 信息不足，需要用户补充；
- 有 N 种方案，用户喜欢哪一个；
- 操作风险太高，需要人工接管。

停止不一定是失败。

脚本通常只会按预设流程跑完。
Agent 需要在信息不足、风险变高、目标变模糊时，把控制权交还给人。

## 扩展：常见 AgentLoop 变体

真正开始读 Agent 相关资料时，你会遇到很多名字：ReAct、Plan-Execute、Ralph Loop。

接下来，我们将一一讲解。

### ReAct：推理和行动交错

讲 AgentLoop 时，经常会遇到一个词：ReAct（Reasoning and Acting）。核心思想是：让模型在推理和行动之间交替进行，而不是先完整想完，再一次性执行。

它的形式可以简化成一组交替出现的消息：

```text
Thought: 我需要知道项目里有哪些测试文件。
Action: search_files
Observation: 找到了 auth.test.ts 和 user.test.ts。
Thought: 失败可能在 auth.test.ts，我需要读取它。
Action: read_file
Observation: 返回 auth.test.ts 的内容。
Thought: 我找到了失败断言，现在需要读取实现代码。
Action: read_file
Observation: 返回 auth.ts 的内容。
```

ReAct 重要的不是 `Thought / Action / Observation` 这几个单词本身，而是这个模式：

```text
想一下 -> 做一步 -> 看结果 -> 再想一下
```

### Plan-Execute：先计划，再执行

另一种常见模式叫 Plan-Execute。

它的直觉很简单：先让模型把任务拆成计划，再按计划一步一步执行，每完成一步就更新状态，必要时重新规划。

比如用户说：

```text
帮我给这个项目加一个登录功能。
```

一个直接行动的 Agent 可能马上开始搜索文件、修改代码、运行测试。

Plan-Execute 风格会先让模型生成一个计划：

```text
Plan:
1. 阅读项目结构和路由定义。
2. 确认现有用户模型和认证方式。
3. 添加登录接口。
4. 添加前端登录表单。
5. 补充测试。
6. 运行测试并修复失败。
```

然后 Agent 再进入执行阶段。每完成一步，新的 observation 都会改变计划状态：有的步骤完成了，有的步骤失败了，有的步骤需要被拆得更细。

Plan-Execute 的好处是，它能让长任务更有方向感。

具体实现有两种常见方式。一种是在开始前让用户显式选择 Plan 模式，先形成计划；另一种是把创建和更新计划作为工具的一部分，让模型自己维护 todo list 和任务状态。

### Ralph Loop：更外层的循环

除了 ReAct，还有一个近几年在 coding agent 讨论里经常出现的说法：Ralph Loop，也有人叫 Ralph Wiggum Loop。

按照 [Ralph CLI 的文档](https://ralph-cli.dev/docs/core-concepts/ralph-loop/) 和 [Wiggum CLI 对 Ralph loop 的介绍](https://wiggum.app/blog/what-is-the-ralph-loop/) 来看，它大概是这样：

```text
准备规格或任务列表
-> 启动 coding agent 执行一个任务
-> agent 规划、实现、测试、验证
-> 记录结果和失败原因
-> 如果没完成，带着新状态继续下一轮
-> 如果完成，进入下一项任务或创建 PR
```

可以把它理解成一个更外层的循环：

```text
Ralph Loop
  -> 启动一次 Coding Agent
      -> AgentLoop
          -> ReAct / Plan-Execute / tool call / observation
  -> 检查产物
  -> 决定是否继续下一轮
```

## 参考资料

- [OpenAI: Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop/)
- [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629)
- [Ralph CLI: Ralph loop](https://ralph-cli.dev/docs/core-concepts/ralph-loop/)
- [Wiggum CLI: What Is the Ralph Loop?](https://wiggum.app/blog/what-is-the-ralph-loop/)
