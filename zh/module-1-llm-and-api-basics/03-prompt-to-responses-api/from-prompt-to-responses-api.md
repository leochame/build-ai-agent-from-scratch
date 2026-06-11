# 04. From Prompt To Responses API

前面我们已经从模型结构、token、上下文窗口和 Prompt Cache 的角度理解了 LLM。到这里，我们要开始 Agent 开发的第一步：一次模型调用到底长什么样。

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


> OpenAI 现在推荐新项目使用 Responses API 来做文本生成、工具调用和更复杂的 agentic workflow。历史上的 Completions API 已经属于 legacy 接口，并在 OpenAI 平台中逐步让位给更新的 API 形态。你可能还会在老教程里看到 Chat Completions Format，它的核心思想仍然非常重要：不要只把输入看成一整段字符串，而要把它看成一组带有角色和语义的消息。

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
  input: "什么是 Responses API？",
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
      content: "什么是 Responses API？",
    },
  ],
});

console.log(response.output_text);
```

这就是从 prompt 走向 Responses API 输入结构的关键变化：

```text
一整段文本 -> 一组有 role 的 message
```


在这个格式里，常见角色包括：

- `developer`：应用开发者提供的规则、目标和业务逻辑；
- `user`：终端用户提供的问题、请求和输入；
- `assistant`：模型生成的回复。

你可以把 `developer message` 想成函数定义，把 `user message` 想成函数参数。

```text
developer message: 这个系统应该怎么工作
user message: 这一次用户具体想要什么
assistant message: 模型根据前两者生成的结果
```

这种类比不完美，但对刚开始写 Agent 很有用。


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

这也是为什么我们后面讲 AgentLoop 时，会不断强调“解析模型输出”，而不是只说“拿到回答”。

## Tool Call 也是一种 Output

Function Calling 听起来像一个独立功能，但从 Responses API 的角度看，它更适合被理解成一种模型输出。

模型并不会真的在自己的大脑里执行函数。它做的是：在当前上下文里判断“我需要外部世界的信息或动作”，然后输出一个 tool call。应用拿到这个 tool call 之后，负责真正执行对应工具，再把工具结果放回下一轮模型输入。

这里要注意一个格式边界：`tools` 不是一条 message。它不是放在 `input` 数组里的 `developer`、`user` 或 `assistant` 消息，而是作为这次模型调用请求里的 `tools` 参数传给模型。

比如用户问：

```text
帮我看看当前目录下有哪些文件。
```

第一轮请求可以简化成：

```json
{
  "input": [
    {
      "role": "developer",
      "content": "你是一个可以使用工具完成任务的 Agent。需要外部信息时，先调用工具，不要编造结果。"
    },
    {
      "role": "user",
      "content": "帮我看看当前目录下有哪些文件。"
    }
  ],
  "tools": [
    {
      "type": "function",
      "name": "list_files",
      "description": "列出目录下的文件。",
      "parameters": {
        "type": "object",
        "properties": {
          "path": {
            "type": "string",
            "description": "要读取的目录路径。"
          }
        },
        "required": ["path"]
      }
    }
  ]
}
```

模型本身不能直接读取你的文件系统。但因为这次请求里有 `tools`，模型知道它可以通过 `list_files` 请求应用帮它读取目录。

这时模型可能不会直接回答，而是在 `output` 里返回一个 tool call：

```json
[
  {
    "type": "function_call",
    "call_id": "call_123",
    "name": "list_files",
    "arguments": "{\"path\":\".\"}"
  }
]
```

这里的 tool call 仍然只是模型 output 的一部分。它的意思不是“模型已经读取了文件”，而是：

```text
请应用帮我调用 list_files，参数是 path = "."
```

应用真正执行 `list_files` 之后，得到结果：

```json
["README.md", "package.json", "src"]
```

第二轮请求时，应用要把上一轮的 tool call 和这次的工具结果放回 `input`：

```json
{
  "input": [
    {
      "role": "developer",
      "content": "你是一个可以使用工具完成任务的 Agent。需要外部信息时，先调用工具，不要编造结果。"
    },
    {
      "role": "user",
      "content": "帮我看看当前目录下有哪些文件。"
    },
    {
      "type": "function_call",
      "call_id": "call_123",
      "name": "list_files",
      "arguments": "{\"path\":\".\"}"
    },
    {
      "type": "function_call_output",
      "call_id": "call_123",
      "output": "[\"README.md\",\"package.json\",\"src\"]"
    }
  ],
  "tools": [
    {
      "type": "function",
      "name": "list_files",
      "description": "列出目录下的文件。",
      "parameters": {
        "type": "object",
        "properties": {
          "path": {
            "type": "string"
          }
        },
        "required": ["path"]
      }
    }
  ]
}
```

这里的 `call_id` 用来把 `function_call_output` 和前面的 `function_call` 对应起来。真正进入对话上下文、让模型知道“刚才工具执行完了”的，是 `function_call_output`。

现在模型已经能看到工具结果，于是可以生成最终回复：

```json
[
  {
    "role": "assistant",
    "content": "当前目录下有 README.md、package.json 和 src。"
  }
]
```

所以，Function Calling 的关键不是“模型会调用函数”，而是：

- 模型可以用结构化 output 表达工具调用意图；
- `tools` 是模型调用请求的一部分，不是 message；
- 应用负责执行工具；
- 工具结果要以 `function_call_output` 回到下一轮 `input`；
- 模型基于工具结果继续生成文本或继续请求工具。

这正是 AgentLoop 的雏形。

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


对话状态不在模型身体里，而在我们的应用里，我们的应用负责决定：

- 哪些历史要保留；
- 哪些历史要丢弃；
- 哪些历史要总结；
- 哪些规则每次都要重新放进去；
- 哪些工具结果要进入下一轮上下文。

这个也是上下文工程要展开讲的内容。

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


## 参考资料

- [OpenAI API: Text generation](https://developers.openai.com/api/docs/guides/text)
- [OpenAI: Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop/)
