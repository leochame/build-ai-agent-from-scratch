# 03. From Prompt To Chat Completion

前面我们已经从模型结构、token、上下文窗口和 Prompt Cache 的角度理解了 LLM。到这里，我们要开始进入 Agent 开发的第一层工程抽象：一次模型调用到底长什么样。

很多人第一次接触 LLM API 时，会把它想象成：

```text
prompt -> model -> answer
```

这当然没有错。你给模型一段文本，模型根据这段文本继续生成文本。

但是真正开始写 Agent 之后，这个理解很快就不够用了。因为 Agent 不是只问一次问题，它要持续接收用户输入、保留历史、遵守开发者规则、调用工具、读取工具结果，然后继续决定下一步。

所以我们需要从单个 `prompt` 走向一种更结构化的输入格式：

```text
messages / input items -> model -> output items
```

这就是本章要讲的事情。

OpenAI 现在推荐新项目使用 Responses API 来做文本生成、工具调用和更复杂的 agentic workflow。历史上大家经常说的 Chat Completions Format，核心思想仍然非常重要：不要只把输入看成一整段字符串，而要把它看成一组带有角色和语义的消息。

本章先不急着写 AgentLoop。我们先把一次模型调用看清楚。

## 从一段 Prompt 开始

最简单的文本生成就是把用户输入交给模型：

```javascript
import OpenAI from "openai";

const client = new OpenAI();

const response = await client.responses.create({
  model: "gpt-5.5",
  input: "用一句话解释什么是 AI Agent。",
});

console.log(response.output_text);
```

这个例子里，`input` 就是我们平时说的 prompt。

如果你只是写一个玩具程序，这已经够了。但在真实应用里，我们通常还需要控制模型的行为，比如：

- 用什么语气回答；
- 哪些事情不能做；
- 输出格式必须是什么；
- 用户输入和系统规则冲突时听谁的；
- 多轮对话里要保留哪些上下文。

这些问题如果都塞进一个字符串里，程序会很快变得混乱。于是 API 提供了更结构化的方式。

## Prompt 不只是用户说的话

在应用里，prompt 通常由两部分组成：

- 开发者给模型的规则；
- 用户当前给模型的输入。

比如你想让模型一直用简洁中文回答，同时用户问了一个具体问题。这时真正送进模型的内容不是只有用户那句话，而更像这样：

```javascript
const response = await client.responses.create({
  model: "gpt-5.5",
  instructions: "你是一个面向初学者的 AI Agent 开发老师。回答要简洁、具体、少用术语。",
  input: "什么是 Chat Completion？",
});
```

这里有一个很重要的边界：

`instructions` 是应用开发者给模型的高层指令，`input` 是用户这次请求的输入。

这两者都属于 prompt 的一部分，但它们的权重和来源不同。把它们分开，是工程化 Prompt 的第一步。

## 从 Prompt 到 Message

如果我们不用 `instructions` 参数，也可以把不同来源的内容显式写成 message：

```javascript
const response = await client.responses.create({
  model: "gpt-5.5",
  input: [
    {
      role: "developer",
      content: "你是一个面向初学者的 AI Agent 开发老师。回答要简洁、具体、少用术语。",
    },
    {
      role: "user",
      content: "什么是 Chat Completion？",
    },
  ],
});

console.log(response.output_text);
```

这就是从 prompt 走向 chat completion 的关键变化：

```text
一整段文本
```

变成：

```text
一组有 role 的 message
```

在这个格式里，常见角色包括：

- `developer`：应用开发者提供的规则、目标和业务逻辑；
- `user`：终端用户提供的问题、请求和输入；
- `assistant`：模型生成的回复。

你可以把 `developer` message 想成函数定义，把 `user` message 想成函数参数。

```text
developer message: 这个系统应该怎么工作
user message: 这一次用户具体想要什么
assistant message: 模型根据前两者生成的结果
```

这种类比不完美，但对刚开始写 Agent 很有用。

## Chat Completion 的真正含义

“Completion” 这个词容易让人误会。它不是说模型真的理解了一个完整任务，然后一次性完成所有事情。更准确地说，模型是在给定上下文后，继续生成接下来最可能、最符合指令的输出。

在传统 completion 里，输入更像一段待续写文本：

```text
请补全下面这段话：
AI Agent 是一种...
```

在 chat completion 里，输入变成了对话历史：

```text
developer: 你是一个 AI Agent 开发老师。
user: 什么是 AgentLoop？
assistant: AgentLoop 是...
user: 那它和普通聊天有什么区别？
```

模型看到的不是“最后一句话”，而是整个被传入的上下文。它根据这些 message 继续生成下一条 `assistant` message。

所以，Chat Completion 的核心不是“聊天界面”，而是“用消息序列组织模型上下文”。

这件事是 Agent 的基础。因为 AgentLoop 本质上也是不断构造上下文、调用模型、处理输出、再把新信息放回上下文。

## Output 也不只是字符串

初学时，我们很容易写出这样的代码：

```javascript
console.log(response.output_text);
```

这很好，SDK 提供 `output_text` 是为了方便你直接拿到模型生成的文本。

但在更完整的 Responses API 里，模型返回的是一个 `output` 数组。这个数组里可能有：

- assistant message；
- 文本内容；
- 工具调用；
- reasoning 相关项目；
- 其他类型的输出项。

因此，在 Agent 开发里不要默认认为结果一定在 `output[0].content[0].text`。那种写法在简单文本生成里可能可行，但一旦引入工具调用、多模态输入或 reasoning model，就会变得脆弱。

更稳妥的理解是：

```text
input items -> model -> output items
```

文本只是 output 的一种。

工具调用也是 output 的一种。

这也是为什么我们后面讲 Function Calling 和 AgentLoop 时，会不断强调“解析模型输出”，而不是只说“拿到回答”。

## 多轮对话不是模型自己记住了

还有一个初学者很容易误解的地方：模型不会天然记住你上一次 API 请求里说过什么。

如果你要让模型理解多轮对话，就必须通过某种方式把历史重新交给它。最直观的方式是维护一个 message 列表：

```javascript
const messages = [
  {
    role: "developer",
    content: "你是一个面向初学者的 AI Agent 开发老师。",
  },
  {
    role: "user",
    content: "什么是 AI Agent？",
  },
  {
    role: "assistant",
    content: "AI Agent 是能围绕目标自主使用模型、工具和上下文的程序。",
  },
  {
    role: "user",
    content: "那它和普通 Chatbot 有什么区别？",
  },
];

const response = await client.responses.create({
  model: "gpt-5.5",
  input: messages,
});
```

这段代码背后的思想非常重要：

对话状态不在模型身体里，而在你的应用里。

你的应用负责决定：

- 哪些历史要保留；
- 哪些历史要丢弃；
- 哪些历史要总结；
- 哪些规则每次都要重新放进去；
- 哪些工具结果要进入下一轮上下文。

这就是后面 Message、Prompt、State 章节要展开的内容。

## 最小聊天程序的形状

一个最小聊天程序大概长这样：

```javascript
import OpenAI from "openai";
import readline from "node:readline/promises";

const client = new OpenAI();

const terminal = readline.createInterface({
  input: process.stdin,
  output: process.stdout,
});

const messages = [
  {
    role: "developer",
    content: "你是一个简洁、耐心的 AI Agent 开发老师。",
  },
];

while (true) {
  const userInput = await terminal.question("user> ");

  if (userInput === "exit") {
    break;
  }

  messages.push({
    role: "user",
    content: userInput,
  });

  const response = await client.responses.create({
    model: "gpt-5.5",
    input: messages,
  });

  const answer = response.output_text;
  console.log(`assistant> ${answer}`);

  messages.push({
    role: "assistant",
    content: answer,
  });
}
```

这段代码已经具备了一个对话系统的基本结构：

```text
读取用户输入
-> 追加 user message
-> 调用模型
-> 读取 assistant 输出
-> 追加 assistant message
-> 等待下一轮用户输入
```

注意，这还不是 AgentLoop。

它只是 Chat Loop。

Chat Loop 只负责对话。AgentLoop 还要在模型输出之后判断：模型是不是要调用工具？工具结果是什么？是否需要继续推理？任务是否完成？是否需要向用户汇报？

## 为什么这一章是 AgentLoop 的前置知识

AgentLoop 看起来比聊天复杂很多，但它的核心仍然建立在这几个概念上：

- message 是上下文的基本单位；
- role 决定了不同消息的来源和优先级；
- 应用负责维护状态，而不是指望模型自动记忆；
- 模型输出可能是文本，也可能是工具调用；
- 每一轮模型调用都要重新组织输入。

如果你理解了从 prompt 到 message，再从 message 到一次模型 response 的过程，AgentLoop 就不神秘了。

它只是把这个过程放进一个循环里：

```text
构造上下文
-> 调用模型
-> 解析输出
-> 执行动作
-> 记录结果
-> 再次构造上下文
```

这也是 Codex 这类代码 Agent 的基本运行方式。它不是一次 prompt 就把软件工程任务全部完成，而是在一次次模型调用、工具执行、状态更新和结果观察之间循环前进。

## 本章小结

从 prompt 到 chat completion，其实是从“写一句话给模型”走向“用结构化消息组织上下文”。

这一章你需要记住四件事：

- prompt 不只是用户输入，也包括开发者规则和上下文；
- message 用 `role` 区分不同来源和权重；
- assistant 的回复也要进入历史，才能形成多轮对话；
- output 不一定只有文本，后面工具调用也会以输出项的形式出现。

下一章会进入 Function Calling / Tool Use / MCP。等你理解模型如何表达工具调用意图之后，再进入真正的 AgentLoop。

## 参考资料

- [OpenAI API: Text generation](https://developers.openai.com/api/docs/guides/text)
- [OpenAI: Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop/)
