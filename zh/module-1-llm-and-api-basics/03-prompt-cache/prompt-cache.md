# 导读
重要的写在前面。

本篇博客是旨在让大家熟知提示词缓存这个概念，尽可能详细的介绍这个概念。~~因为我的实力不足~~ 为了降低门槛，我尽可能减少了 LLM 相关原理的篇幅，把提示词缓存与 LLM 之间的关系放在了最后，请大家选择性阅读。

> 注：另外由于本人并未系统学习过 LLM 相关知识，如有谬误，欢迎指出，如有可能，请您不吝赐教。
>

---

Sharon Li 在亚马逊中国峰会的演讲《提示缓存 (Prompt Caching) 和跨区域推理 (Cross Region Inference)》是我第一次接触到提示词缓存这个概念，当时就留下了非常深刻的影响。

后来又读了 Manus联合创始人兼首席科学家季逸超（Yichao 'Peak' Ji）撰写的《Context Engineering for AI Agents: Lessons from Building Manus》，加上自己又阅读了《LLMs-from-scratch》这本书，感觉似乎打开了新的认知，所以写下了这篇文章。

# 什么是提示词缓存
## 提示词缓存小白 N 连问
1. 提示词缓存是什么？

提示词缓存是一种保存并复用语言模型对文本前缀部分（the prefix）的中间计算结果的技术。是**大语言模型内部**集成的缓存机制。

2. 缓存的内容是什么？

被缓存的**不是原始文本**，而是文本经过模型初步计算后生成的、代表其上下文含义的一组关键数据（即“键值对”，Key-Value Pairs）。这个结果我们称之为“上下文状态”或“KV Cache”。

3. 谁来控制读取和写入？

由模型自身控制缓存的读取逻辑。当下次处理以相同前缀开头的请求时，模型可以直接加载这个已保存的“上下文状态”，跳过对前缀的重复计算，直接从新的文本部分开始处理。

4. 【对前缀的重复计算】指的是什么？

大模型（基于 Transformer 架构）在生成每一个新词时，都需要回顾（“注意力机制”）之前所有的词。如果不做缓存，生成第100个词时，它会重新计算第1到第99个词的内部状态，造成巨大的浪费。

然而，这里值得注意的是：由于缓存的是已处理的上下文，所以这个缓存命中要求的是提示词前缀百分相同 100% ，一旦前缀发生改变，这个上下文就不是处理过的了，提示词缓存就失效了。

<!-- 这是一张图片，ocr 内容为： -->
![图片来源：https://aws.amazon.com/cn/blogs/machine-learning/effectively-use-prompt-caching-on-amazon-bedrock/](https://cdn.nlark.com/yuque/0/2025/png/40540759/1753586450081-d20b65b3-5157-4025-891c-d1cbfa54f982.png)

## 提示词缓存在 LLM 中的运作原理
我们接下来讲解一下 LLM 是怎么**运行**提示词缓存的：

用户可以明确指定需要缓存的提示部分，也就是提示词前缀。当将提示前缀发送给模型的时候，模型会处理输入并将处理后的上下文保存在缓存中。提示词缓存通常在不活跃5-10分钟后失效，但在非高峰时段最长可达1小时（这里是 Open AI 的数据）。

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/40540759/1753428936095-3cdefd4d-ec95-466b-bce1-669b0ccbf0b6.png)

对于包含相同提示前缀的后续请求，会直接加载先前缓存的提示词前缀，每次访问完成后，会更新缓存的 Live Time。

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/40540759/1753428327713-40963a98-f108-4969-a5eb-de5e4c235905.png)

# 提示词缓存带来的收益
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


# 提示词缓存的应用
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
> 2. 当之前的动作和观察结果仍然引用当前上下文中不再定义的工具时，模型会感到困惑。没有 约束解码[15] (constrained decoding)，这通常会导致模式违规或幻觉动作。提示词缓存的原理详解
>
> 为了解决这个问题，同时仍然改进动作选择，Manus使用上下文感知的 状态机[16] (state machine) 来管理工具可用性。它不是删除工具，而是在解码期间掩码令牌logits，以根据当前上下文防止（或强制执行）某些动作的选择。
>

<!-- 这是一张图片，ocr 内容为： -->
![图片来源：https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus](https://cdn.nlark.com/yuque/0/2025/png/40540759/1753613188670-9a98b4a4-1acc-4728-9312-8f0f15b38a30.png)

## LLM 是怎么工作的 ？
要想明白 Prompt Cache 可以工作的底层原理，我们必须要明白 LLM 是怎么工作的。

LLM 的工作流程本质上是将输入的文本序列，通过一系列复杂的数学变换，转换成一个概率分布，然后根据这个概率分布预测出下一个最有可能的词，并不断重复这个过程来生成新的文本。

因为计算机无法直接理解“文本”，所以我们需要将文本转换成计算机可以处理的数字。所以我们首先将原始文本字符串分解成更小的单元，称为“Token”。Token 可以是单词、子词，甚至是单个字符。然后，创建一个从 Token 到唯一整数 ID 的映射表（即“词汇表” ）。再用这个表将 Token 序列转换为整数 ID 序列。

但是由于单纯的整数 ID 是无法表达词与词之间的复杂关系的，我们就需要将这些 ID 转换为包含丰富语义信息的连续向量，这个过程就是“嵌入”。于是，我们得到了嵌入向量序列 $x_1, x_2, \ldots, x_n$。

但是，上下文的深度理解还是得依赖 Transformer。 多个堆叠的 Transformer 模块 (Transformer Block) 组成了 LLM 的大脑，每个模块的核心是自注意力机制 (Self-Attention)。它允许输入序列中的每个 Token 关注序列中的所有其他 Token（包括自身），并计算它们之间的相关性（注意力分数）。

为了计算注意力，每个输入序列的向量通过权重矩阵 $W_q, W_k, W_v$ 会生成三个独立的向量：Query (Q)、Key (K) 和 Value (V)。

+ **Q (查询)**：$q_i = x_i \cdot W_q$，代表当前 Token，它要去“查询”其他 Token 与自己的关系。
+ **K (键)**：$k_j = x_j \cdot W_k$，代表序列中的所有 Token，它们被 Q 查询，用来计算相关性分数。
+ **V (值)**：$v_j = x_j \cdot W_v$，代表序列中所有 Token 的实际内容。

通过计算 Query 向量和所有 Key 向量的点积 ($\omega_{ij} = q_i \cdot k_j$) 得到注意力分数。分数越高，代表相关性越强。

对分数进行 Softmax 归一化。在因果注意力中，模型只能关注到当前位置及之前的位置，因此 $j$ 的取值范围是 1 到 $i$：

$$
\alpha_{ij} = \operatorname{softmax}(\omega_{i1}, \omega_{i2}, \ldots, \omega_{ii})
$$

Softmax 函数的作用是将所有分数 $(\omega_{i1}, \omega_{i2}, \ldots, \omega_{ii})$ 转换成一组总和为 1 的注意力权重 $(\alpha_{i1}, \alpha_{i2}, \ldots, \alpha_{ii})$。

Value 向量的加权和得到上下文向量：

$$
z_i = \sum_{j=1}^{i} \alpha_{ij} v_j
$$

这个上下文向量 $z_i$ 代表了模型在充分理解了上下文之后，对原始位置 $i$ 上的那个 Token 的全新、更深刻的理解。它已经不再是孤立的了，而是融合了所有相关历史信息的“语境化”表示。

> 比如，通过阅读上下文中的“river”和“water”，模型现在明白了这里的“bank”指的是“河岸”。$z^{(i)}$ 就是这个包含了“河岸”这个明确含义的、被上下文信息“重新着色”后的新向量。
>

$z^{(i)}$ 只是单个 Transformer 模块中注意力子层的输出，它还需要经过一系列步骤才能最终生成一个新词元。

> 一个 GPT 模型包含很多层 Transformer 模块（例如12层）。第 1 个模块的输出，会成为第 2 个模块的输入，然后重复上述的注意力和前馈网络处理。向量的信息在每一层都会被进一步提炼和抽象。
>

当输入序列中最后一个 Token 的向量通过了所有 Transformer 模块的处理后，我们会得到一个最终的、高度精炼的输出向量。我们称之为 $z_n^{\text{final}}$（假设输入序列长为 $n$）。

这个最终输出向量 $z_n^{\text{final}}$ 会被送入模型末端的一个线性层 (Output Layer)。这个线性层将其映射成一个非常长的 Logits 向量，其维度等于整个词汇表的大小。Logits 向量经过 softmax 函数，转换成覆盖整个词汇表的概率分布。模型从这个概率分布中选择概率最高的那个词，其对应的整数 ID 就是我们最终生成的新词元（其实就是 Token）。

模型将新生成的 Token 拼接到输入末尾，形成一个更长的序列。然后，将这个新序列作为全新的输入，再次完整地执行上述所有步骤，预测再下一个 Token，循环往复。

总结一下，大语言模型采用自回归（autoregressive）的方式进行解码，即逐个生成输出词元 。每生成一个新的词元，它都会被附加到现有输入序列的末尾，形成一个更长的序列，然后这个新序列被重新输入模型以生成下一个词元 。

## KV Cache
从自回归过程中我们可以发现一个巨大的性能瓶颈：**每生成一个新 Token，模型都需要把越来越长的完整序列重新计算一遍**。这个计算的核心就是 Self-Attention，其计算复杂度与序列长度的平方成正比。

在 Self-Attention 计算中，对于一个已经处理过的 Token，它的 **Key (K) 和 Value (V) 向量是固定不变的**。我们完全没必要在每一步都重新计算它们。

在生成新 Token 时，模型不再需要重新计算整个 Prompt 的 K 和 V。它只需：

+ 为**上一个刚生成的 Token**计算其自身的 Key 和 Value。
+ 这对新的 (K, V) 追加到 KV Cache 的末尾。
+ 利用**完整的 KV Cache**（包含 Prompt 和所有已生成 Token 的 K/V）来为当前步计算注意力。

> Reusing attention states is a popular strategy for accelerating the service of a single prompt . The existing approach, often referred to as Key-Value (KV) Cache, reuses the key-value attention states of input tokens during the autoregressive token generation. This eliminates the need to compute full attention for every token generation (§ 2.2). By caching the key-value attention computed for the previously generated token, each token generation requires the computation of key-value attention states only once.
>
> —— Prompt Cache: Modular Attention Reuse for Low-Latency Inference
>

Prompt Cache 的想法是基于 KV Cache 而来的。

KV Cache 是为了实现单个请求加速的。模型生成文本时，它是一个自回归（autoregressive）的过程。比如，模型根据输入 "北京的天气" 生成了 "很不错"，下一步为了生成句号 "。" ，它需要把 "北京的天气很不错" 作为整体输入再计算一次。KV Cache 的作用就是把 "北京的天气很不错" 这部分已经计算出的中间结果（即 Attention 模块中的 Key 和 Value 值）缓存起来 。在生成下一个字 "。" 时，模型可以直接利用这个缓存，只需计算新输入对应的 attention 即可，避免在生成每个新词时都从头计算一遍已经处理过的内容。

> 模型在进行计算时，会为每个输入的词生成三个关键向量：查询（Query）、键（Key）和值（Value）。**提示词缓存主要就是将与输入提示词（Prompt）相关的“键（K）”和“值（V）”这两个向量存起来**。这就是它通常被称为 **KV缓存 (KV Cache)** 的原因。
>

**而 Prompt Cache 是将 Attention State Reuse 从单个请求扩展到多请求**。

在我们的方法中，频繁重用的文本片段会被单独预先计算并存储在内存中。当这些 “缓存的” 片段出现在输入提示中时，系统会使用内存中预先计算好的键值注意力状态，而不是重新计算。因此，注意力计算只需要针对未缓存的文本片段进行。

无论 KV Cache 还是 Prompt Cache 其本质都是“记住”一个提示词前缀被处理后的模型内部状态，特别是注意力模式和隐藏状态向量，以便在后续请求中直接加载而非重新计算 。

## 证明 KV Cache 能够起作用
其实这里，我们就明白了，证明 KV Cache 可以起作用，就证明了 Prompt Cache 有效果。

这个证明的核心在于**因果自注意力 (Causal Self-Attention) 机制的数学定义**。我们将证明，对于任意一个新生成的 Token，其上下文向量的计算，依赖于**过去所有 Token 的 Key 和 Value 向量**，而这些过去的 Key 和 Value 向量一旦生成，其值就是**固定且独立**的，与当前正在计算的 Token 位置无关。

---

我们的目标是证明，在计算第 $i+1$ 个 Token 的上下文向量 $z^{(i+1)}$ 时，可以完全重用在计算 $z^{(i)}$ 时已经得到的 Key 和 Value 向量。

输入序列为：

$$
X_t = [x_1, \ldots, x_t]
$$

其中 $x_t$ 其实就是 $z_{t-1}$。

+ **Q (查询)**：$q_t = x_t \cdot W_q = x_t W_Q$；
+ **K (键)**：$K_{1..t} = X_{1..t} \cdot W_k = [x_1 W_K, \ldots, x_t W_K]$；
+ **V (值)**：$V_{1..t} = X_{1..t} \cdot W_v = [x_1 W_V, \ldots, x_t W_V]$；

我们可以得到新的上下文向量 $z_t$：

$$
z_t = \operatorname{softmax}\left(\frac{Q_t [K_1, K_2, \ldots, K_t]^T}{\sqrt{d_k}}\right) [V_1, V_2, \ldots, V_t]
$$

> 这里省略的多头注意力机制和多 Transformer 的计算流程（因为这两个其实并不产生影响）。
>



我们得到新的输入序列：

$$
X_{t+1} = [x_1, \ldots, x_{t+1}]
$$

这里的 $x_{t+1}$ 其实就是 $z_t$。

+ $q^{(t+1)} = x^{(t+1)} \cdot W_q = q_{t+1} = x_{t+1} W_Q$
+ $K_{1..t+1} = X_{1..t+1} \cdot W_k = [x_1 W_K, \ldots, x_{t+1} W_K] = \operatorname{concat}(K_{1..t}, K_{t+1}) = [K_1, \ldots, K_t, K_{t+1}]$
+ $V_{1..t+1} = X_{1..t+1} \cdot W_v = [x_1 W_V, \ldots, x_{t+1} W_V] = \operatorname{concat}(V_{1..t}, V_{t+1}) = [V_1, \ldots, V_t, V_{t+1}]$

> 其实我们可以看到 KV Cache 方法只是省去了对 $K_{1..t}$ 和 $V_{1..t}$ 的重复计算，直接复用了上一步的结果。
>

我们最终得到了

$$
z_{t+1} = \operatorname{softmax}\left(\frac{Q_{t+1} [K_1, K_2, \ldots, K_{t+1}]^T}{\sqrt{d_k}}\right) [V_1, V_2, \ldots, V_{t+1}]
$$

然后作为下一层 Transformer 的输出，最终得到 $z_{t+1}^{\text{final}}$（$\text{final}$ 代表着的是 Transformer 的层数）

所以 KV Cache 机制的核心就是**每一个 Transformer 模块（层）都有自己独立的一套 KV Cache**。在生成新词元时，每一层都只对新信息（来自前一层的输出向量）进行计算，生成一个新的 $(k, v)$ 对。这个增量计算的结果会作为输入，触发下一层的增量计算，一直传递到最后一层。最终，最后一层也只用 $z_{t+1}^{L-1}$ 计算出最终的 $z_{t+1}^{L}$，然后用它来预测下一个词。整个过程避免了对过去 $t$ 个词元在任何一层进行重复计算。

# 参考文献 (References)
1. **Anthropic. (n.d.).** _Prompt caching_. Retrieved July 27, 2025, from  https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
2. **Gim, I., Chen, G., Lee, S.-s., Sarda, N., Khandelwal, A., & Zhong, L. (2023).** _Prompt Cache: Modular Attention Reuse for Low-Latency Inference_ (arXiv:2311.04934). arXiv.  https://arxiv.org/abs/2311.04934
3. **Li, S. (n.d.).** _提示缓存 (Prompt Caching) 和跨区域推理 (Cross-Region Inference)_ [Presentation]. 亚马逊云科技中国峰会 2025 (Amazon Web Services China Summit), China.
4. **Li, S., Belz, K., Khurpa, S., Eichenberger, S., & Subramanian, S. (2024, May 20).** _Effectively use prompt caching on Amazon Bedrock_. AWS Machine Learning Blog.  https://aws.amazon.com/cn/blogs/machine-learning/effectively-use-prompt-caching-on-amazon-bedrock/
5. **Manus Team, The. (2024, June 12).** _Context Engineering for AI Agents: Lessons from Building Manus_. Manus Blog.  https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus
6. **OpenAI. (2024, June 11).** _API prompt caching_. [https://openai.com/index/api-prompt-caching](https://openai.com/index/api-prompt-caching/)
