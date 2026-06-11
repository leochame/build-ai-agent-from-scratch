# 提示词缓存

## 1. 什么是提示词缓存
### 1.1 核心定义

提示词缓存是一种复用提示词前缀计算结果的机制。模型第一次处理某段稳定前缀时，会把这段内容对应的中间状态保存下来；后续请求如果仍然以同样的前缀开头，就可以直接读取缓存，而不是重新计算整段前缀。

这个缓存是模型供应商提供的能力。

> 在这里，我不禁要吐槽来自字节的老登普信男的实力了，总之他的意思是，模型侧做不了缓存，因为他们做的是 ToC，流量太大。我连显存都能拿来做存储，对话凭什么不能存？哪怕最后不能存，也不是存储占用太大。
> 
>当然这个傻屌，都不理解我说的提示词缓存是什么。我遇到的人里，字节的文化熏陶出来的，是最爱装逼，而且没啥实力，喜欢把逼装破的。当然主要是字节面我的机会比较多。

它缓存的不是原始文本，而是文本进入模型后产生的内部状态，通常可以理解为注意力计算中的 Key-Value 状态，也就是 KV Cache。提示词缓存的命中条件很严格：可复用的部分必须是相同的前缀。只要前缀中的系统提示、工具定义、文档内容、序列化顺序或空白字符发生变化，变化点之后的缓存通常就无法继续复用。

![图片来源：https://aws.amazon.com/cn/blogs/machine-learning/effectively-use-prompt-caching-on-amazon-bedrock/](https://cdn.nlark.com/yuque/0/2025/png/40540759/1753586450081-d20b65b3-5157-4025-891c-d1cbfa54f982.png)

### 1.2 基本流程
提示词缓存的流程其实不复杂，基本就是三步：

1. **写入缓存**：模型先把这段稳定前缀处理一遍，并把对应的内部状态存下来。
2. **读取缓存**：后续请求如果还是同样的前缀，就直接把这段状态读出来。
3. **继续计算**：模型只继续处理新增的那部分动态内容，比如用户问题、最新观察结果，或者新的工具输出。

![](https://cdn.nlark.com/yuque/0/2025/png/40540759/1753428936095-3cdefd4d-ec95-466b-bce1-669b0ccbf0b6.png)

缓存一般也会有过期时间。以 OpenAI 的公开说明为例，如果一段缓存一段时间没有被继续使用，通常会在 5 到 10 分钟后失效；在非高峰时段，最长可能保留到 1 小时。每次命中缓存后，这个存活时间还有可能被重新刷新。

![](https://cdn.nlark.com/yuque/0/2025/png/40540759/1753428327713-40963a98-f108-4969-a5eb-de5e4c235905.png)

## 2. 提示词缓存带来的收益
> With prompt caching, customers can provide Claude with more background knowledge and example outputs—all while reducing costs by up to 90% and latency by up to 85% for long prompts.
>
> -Anthropic  《Prompt caching with Claude》
>

Anthropic 团队在《Prompt caching with Claude》一文中给出的数据，可以看到提示词缓存带来了两大好处，更快、更便宜：

+ **显著降低延迟**：通过避免对相同内容的冗余处理，模型的响应时间得以大幅提升。报告显示，响应速度最高可提升85% 。

> 这对于需要即时反馈的交互式应用（如聊天机器人）和延迟会累积的多步代理工作流（agentic workflows）而言至关重要 。
>

+ **大幅削减成本**：这是提示词缓存最引人注目的优势。复用缓存中的词元远比重新处理它们要便宜得多。

> 这里的“延迟”指的是首字符响应时间（**TTFT**，**Time to First Token**），它计算的是从用户发送请求（Prompt）到模型返回第一个有意义的数据块（Token）之间所花费的时间。TTFT 越短，代表模型的反应越快，用户感受到的延迟就越低。
>

下表汇总了两大供应商缓存功能的关键特性：

| 模型 | OpenAI (gpt-4o系列)<br/>[Prompt Caching in the API](https://openai.com/index/api-prompt-caching/) | Anthropic (Claude 系列)   [Prompt caching - Anthropic](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching) |
| --- | --- | --- |
| **激活方式** | 自动（提示词 > 1024 tokens） | 手动（通过API中的`cache_control`<br/>） |
| **成本模型** | 写入无额外成本，读取享折扣 | 写入溢价，读取享更高折扣 |
| **缓存写入成本** | 无额外费用 | +25% 溢价 |
| **缓存读取折扣** | 最高 50% | 90% |
| **生命周期 (TTL)** | 5-10分钟（非高峰可达1小时） | 5分钟（使用时刷新），可配置1小时 |
| **手动控制** | 有限（通过`user`<br/>参数影响路由） | 灵活（最多4个缓存断点） |
| **监控字段** | `prompt_tokens_details.cached_tokens` | `cache_creation_input_tokens`<br/>, `cache_read_input_tokens` |


## 3. 提示词缓存的应用
其实从以上我们其实可以猜到，提示词工程的最佳实践是将静态内容放在提示词的开头，将动态内容（用户查询）放在末尾。

另外，由于单个标记的差异也会使该标记之后的缓存失效，这使得我们使用的时候尽量偏向于 append 操作，同时要确保我们的序列化是确定性的（有些编程语言和库在序列化JSON对象时不保证键顺序的稳定性，这可破坏缓存 ）。

对于需要时明确标记缓存断点，需要在上下文中手动插入缓存断点。在分配这些断点时，要考虑潜在的缓存过期问题，并至少确保断点包含系统提示的结尾。

---

在 Claude 的官方文档中， Anthropic 更推荐这个几个内容被缓存：

> + Tools: Tool definitions in the `tools` array
> + System messages: Content blocks in the `system` array
> + Text messages: Content blocks in the `messages.content` array, for both user and assistant turns
> + Images & Documents: Content blocks in the `messages.content` array, in user turns
> + Tool use and tool results: Content blocks in the `messages.content` array, in both user and assistant turns
>

另外，综合 Sharon Li 和 Anthropic 的看法中，提示词缓存的在如下的使用场景中，能发挥最大的作用：

+ **大文档处理与问答：**文档本身是不变的，缓存与文档相关的上下文，这样每次处理用户指令的时候，模型就可以直接从缓存中读取计算结果，而无需处理整个文档。
+ **Agent 工作流：**在这种工作流中，系统提示、可用工具都是静态的，对于不同用户的请求，这些部分是不变的，因此也可以存入缓存。
+ **编程助手：**在编程助手的这个场景中，长代码文件，整个代码库的简要概述来提高响应速度
+ **小样本学习（Few-Shot）：**高质量的示例、样本数据和复杂的指令，详尽的指令、步骤和示例列表来

这里额外来强调一下的是：Anthropic 提出的**上下文检索**技术，它也用到了提示词缓存。在传统的 RAG 架构中，会将长文档分成多个文本块，但是单个文本块往往缺乏足够上下文信息，这就导致了生成质量不佳，上下文检索就是将文档一次性加载到缓存中，之后直接引用已缓存的内容即可。

---

在这里，补充几个在使用提示词缓存的时候需要注意的点：

1. 如果底层文档（如知识库）发生变化，缓存就会变得陈旧。作为开发者必须实施策略来使缓存失效或刷新缓存 。
2. 5分钟TTL对于用户交互存在间歇的应用可能是一个问题。可以考虑使用后台定时“ping”请求来刷新缓存等缓解策略 。
3. 另外也可以考虑适当使用多个 cache checkpoints。比如：在文档问答中，可以在文档之后设置一个检查点，在摘要或分析之后设置另一个检查点。这允许轻松实现对话分支：能够从对话中的特定点分支出多个对话。

另外在 Manus 分享的实战中，也分享了其在使用 Prompt Cache 中遇到的问题，以及解决思路：

> 除非绝对必要，避免在迭代过程中动态添加或删除工具。这有两个主要原因：
>
> 1. 在大多数LLM中，工具定义在序列化后位于上下文的前部附近，通常在系统提示之前或之后。所以任何更改都会使所有后续动作和观察结果的KV缓存失效。
> 2. 当之前的动作和观察结果仍然引用当前上下文中不再定义的工具时，模型会感到困惑。没有 约束解码[15] (constrained decoding)，这通常会导致模式违规或幻觉动作。
>
> 为了解决这个问题，同时仍然改进动作选择，Manus使用上下文感知的 状态机[16] (state machine) 来管理工具可用性。它不是删除工具，而是在解码期间掩码令牌logits，以根据当前上下文防止（或强制执行）某些动作的选择。
>


![图片来源：https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus](https://cdn.nlark.com/yuque/0/2025/png/40540759/1753613188670-9a98b4a4-1acc-4728-9312-8f0f15b38a30.png)

## 4. 提示词缓存的原理
### 4.1 为什么提示词缓存可行
上一章已经讲过 LLM 的基本运行方式：文本会被切成 Token，Token 会变成向量，然后经过多层 Transformer。这里我们不再重复完整流程，只抓住和 Prompt Cache 直接相关的一点：

**在因果注意力中，一个已经处理过的 Token，它对应的 Key (K) 和 Value (V) 在当前请求里是可以复用的。**

也就是说，模型生成文本时并不需要每一步都从头计算所有历史 Token 的 K/V。只要之前的上下文没有变化，过去那些 Token 的中间计算结果就可以留下来，后面生成新 Token 时继续使用。

这就是 KV Cache 的基础。

### 4.2 KV Cache
KV Cache 解决的是**单次请求内部**的重复计算问题。

模型采用自回归（autoregressive）的方式逐个生成 Token。比如模型已经基于输入“北京的天气”生成了“很不错”，下一步要生成句号“。”时，它理论上需要再次处理“北京的天气很不错”这一整段序列。

但在 Self-Attention 计算中，前面那些 Token 的 Key 和 Value 已经算过了，而且不会因为当前要生成的新 Token 而改变。所以模型可以把这些 K/V 保存起来。下一步只需要：

+ 为新 Token 计算自己的 Key 和 Value；
+ 把新的 (K, V) 追加到已有的 KV Cache 后面；
+ 利用完整的 KV Cache 计算当前步的注意力。

> Reusing attention states is a popular strategy for accelerating the service of a single prompt . The existing approach, often referred to as Key-Value (KV) Cache, reuses the key-value attention states of input tokens during the autoregressive token generation. This eliminates the need to compute full attention for every token generation (§ 2.2). By caching the key-value attention computed for the previously generated token, each token generation requires the computation of key-value attention states only once.
>
> —— Prompt Cache: Modular Attention Reuse for Low-Latency Inference
>

### 4.3 为什么 K/V 可以被复用
如果只想理解工程实践，看到这里已经足够了。下面补一个最小化的数学说明，解释为什么 K/V 可以被复用。

这个证明的核心在于**因果自注意力 (Causal Self-Attention) 机制的数学定义**。对于任意一个新生成的 Token，它只能关注自己和过去的 Token，不能反过来影响过去 Token 的状态。所以过去 Token 的 Key 和 Value 一旦算出来，在追加新 Token 时就可以复用。

---

为了简化说明，我们只看单层、单头注意力。多头注意力只是并行做多组 Q/K/V，多层 Transformer 只是每一层都维护一份自己的 KV Cache，原理不变。

设某一层的输入 hidden states 为：

$$
H_{1:t} = [h_1, h_2, \ldots, h_t]
$$

在这一层中，每个位置都会通过权重矩阵得到自己的 Query、Key 和 Value：

$$
K_{1:t} = H_{1:t} W_K = [h_1 W_K, \ldots, h_t W_K]
$$

$$
V_{1:t} = H_{1:t} W_V = [h_1 W_V, \ldots, h_t W_V]
$$

当模型要生成下一个 Token 时，只需要为新位置 $t+1$ 计算新的 $q_{t+1}$、$k_{t+1}$ 和 $v_{t+1}$：

$$
\begin{aligned}
q_{t+1} &= h_{t+1} W_Q \\
k_{t+1} &= h_{t+1} W_K \\
v_{t+1} &= h_{t+1} W_V
\end{aligned}
$$

之前的 $K_{1:t}$ 和 $V_{1:t}$ 不需要重新计算，只需要把新的 $k_{t+1}$ 和 $v_{t+1}$ 追加进去：

$$
K_{1:t+1} = \mathrm{concat}(K_{1:t}, k_{t+1})
$$

$$
V_{1:t+1} = \mathrm{concat}(V_{1:t}, v_{t+1})
$$

于是新位置的注意力输出可以写成：

$$
z_{t+1} = \mathrm{softmax}\left(\frac{q_{t+1} K_{1:t+1}^{T}}{\sqrt{d_k}}\right) V_{1:t+1}
$$

这一步真正新增的计算，只有新 Token 对应的 $q_{t+1}$、$k_{t+1}$、$v_{t+1}$，以及它对完整上下文的注意力计算。过去 Token 的 K/V 可以直接从缓存中读取。

更严格地说，这依赖三个前提：

+ 使用的是因果注意力，过去位置看不到未来位置；
+ 推理阶段没有训练时的 dropout 这类随机扰动；
+ 前缀内容、模型参数和序列化结果保持一致。

所以 KV Cache 机制的核心就是：**每一个 Transformer 层都有自己独立的一套 KV Cache**。在生成新词元时，每一层都只对新位置做增量计算，并复用该层过去位置已经算好的 K/V。整个过程避免了对过去 $t$ 个词元在每一层里的重复计算。

### 4.4 Prompt Cache
Prompt Cache 的思路可以理解为：**把 KV Cache 这种 Attention State Reuse 从单次请求扩展到多次请求**。

> 模型在进行计算时，会为每个输入的词生成三个关键向量：查询（Query）、键（Key）和值（Value）。**提示词缓存主要就是将与输入提示词（Prompt）相关的“键（K）”和“值（V）”这两个向量存起来**。这就是它通常被称为 **KV缓存 (KV Cache)** 的原因。
>

在我们的方法中，频繁重用的文本片段会被单独预先计算并存储在内存中。当这些 “缓存的” 片段出现在输入提示中时，系统会使用内存中预先计算好的键值注意力状态，而不是重新计算。因此，注意力计算只需要针对未缓存的文本片段进行。

因此，两者的差异可以简单理解为：

+ **KV Cache**：同一次模型调用中，复用已经生成过的历史 Token 状态。
+ **Prompt Cache**：不同模型调用之间，复用相同提示词前缀的状态。

无论 KV Cache 还是 Prompt Cache，本质上都是“记住”一段上下文被模型处理后的内部状态，尤其是注意力计算里的 K/V 状态，以便之后直接加载，而不是重新计算。

## 5. 参考文献
1. **Anthropic. (n.d.).** _Prompt caching_. Retrieved July 27, 2025, from  https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
2. **Gim, I., Chen, G., Lee, S.-s., Sarda, N., Khandelwal, A., & Zhong, L. (2023).** _Prompt Cache: Modular Attention Reuse for Low-Latency Inference_ (arXiv:2311.04934). arXiv.  https://arxiv.org/abs/2311.04934
3. **Li, S. (n.d.).** _提示缓存 (Prompt Caching) 和跨区域推理 (Cross-Region Inference)_ [Presentation]. 亚马逊云科技中国峰会 2025 (Amazon Web Services China Summit), China.
4. **Li, S., Belz, K., Khurpa, S., Eichenberger, S., & Subramanian, S. (2024, May 20).** _Effectively use prompt caching on Amazon Bedrock_. AWS Machine Learning Blog.  https://aws.amazon.com/cn/blogs/machine-learning/effectively-use-prompt-caching-on-amazon-bedrock/
5. **Manus Team, The. (2024, June 12).** _Context Engineering for AI Agents: Lessons from Building Manus_. Manus Blog.  https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus
6. **OpenAI. (2024, June 11).** _API prompt caching_. [https://openai.com/index/api-prompt-caching](https://openai.com/index/api-prompt-caching/)
