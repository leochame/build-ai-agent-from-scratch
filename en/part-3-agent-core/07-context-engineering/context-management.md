# Context Management

Context management is one of the most important parts of agent engineering.

At first, people often focus on model strength, number of tools, or prompt wording. But once a system becomes more complex, the real bottleneck is often not whether the model is powerful enough, but whether the model is seeing the right context.

In one sentence, I would define context management like this:

how to make the model see the most useful information at the current moment within a limited token budget.

## 1. What Is Actually Being Managed

Context is not the same thing as memory.

Context is what the model can actually see in the current call, such as:

- the system prompt
- the current user input
- conversation history
- tool outputs
- retrieved materials
- current task state
- temporary plans

Memory is more like a warehouse of possible information. It only becomes context when the system decides to inject part of it into the current call.

So from a systems point of view:

- memory is potentially available information
- context is the information currently exposed

## 2. A Useful Split: Pre-Handle and Post-Handle

I find it useful to split context management into two phases:

- pre-handle
- post-handle

Pre-handle is about deciding what should enter the model before a call.

Post-handle is about deciding what to keep, compress, or unload after a call.

## 3. Pre-Handle

The core idea behind pre-handle is progressive disclosure.

Do not expose everything up front. Expose information gradually as the task demands it.

This matters because agents do not think after receiving the complete world state. They think under incomplete information and decide what else they need next.

### 3.1 Skills

A skill is essentially a delayed context bundle.

It may include:

- operating instructions for a task type
- best practices
- boundary conditions
- common mistakes
- relevant tool guidance

The real value is not the file itself. The value is moving task-specific knowledge out of the always-on prompt and loading it only when needed.

### 3.2 Tool Search

As the toolset grows, showing every tool all the time becomes expensive and noisy.

Tool search is the step that filters candidate tools before exposing them to the model.

That usually helps by:

- reducing prompt length
- lowering misuse risk
- improving selection stability
- making the tool system easier to scale

### 3.3 Code Act

For coding agents, code understanding is also a context-selection problem.

The system has to answer questions like:

- which files matter most right now
- which entry points should be read first
- which symbols are worth expanding
- whether the right order is search, then read, then edit

If this process is weak, the agent either sees too little and misunderstands the codebase, or sees too much and loses focus.

## 4. Post-Handle

If pre-handle decides how context enters, post-handle decides how context is cleaned up.

Without it, long-running agent systems usually suffer from:

- endlessly growing history
- low-value content occupying the window
- repeated tool output injection
- stale plans polluting new work
- rising token cost

### 4.1 Context Compression

Compression should not be thought of as shortening for its own sake.

The goal is to preserve the information that still matters for future reasoning, such as:

- the current objective
- key decisions already made
- unresolved issues
- essential facts from tools
- the next plan

### 4.2 Context Unloading

Some information is important for one phase but should not remain in the main prompt forever.

Examples include:

- logs from completed subtasks
- large raw tool outputs
- irrelevant retrieval results
- details already extracted into state

That information does not have to be deleted. It can be moved out of the main context and brought back only when needed.

### 4.3 State Extraction

Another important post-handle step is extracting structured state from interaction history.

For example:

- current task stage
- completed steps
- failure reasons
- user preferences
- pending confirmations

When these facts remain buried inside natural language, the system has to rediscover them again and again. Extracting them into structured state makes future context assembly much lighter.

## 5. A Simple Way to Think About It

You can reduce the whole topic to three questions:

1. What information deserves to enter the current context?
2. In what structure should that information be injected?
3. After the step ends, what should be kept, compressed, or unloaded?

If those three questions are not handled well, an agent may still appear to work, but it will stay fragile.

## 6. Why This Matters So Much

Many seemingly different agent failures eventually reduce to context management problems:

- the model suddenly feels worse
- tool choice becomes unstable
- long tasks become chaotic
- token costs keep rising
- the system gets harder to extend

That is why context management is much bigger than simple history truncation or a generic summary step.
