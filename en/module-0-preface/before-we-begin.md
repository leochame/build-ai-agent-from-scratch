# Preface: This Is Not a Finished Tutorial

If you opened this repository, I want to make one thing clear first:

this is not a polished course with a fixed syllabus and a fully completed structure.

It is closer to a learning path I am walking through myself, a set of evolving notes, and a map that is still being drawn.

I want to keep the things I understood, the places where I got stuck, the mistakes I made, and the moments where my understanding changed.

So this repo is less about "teaching from authority" and more about:

- how I started to understand AI agents
- where I repeatedly got confused
- why I split the content the way I did
- which questions I currently think matter most

If that process also helps someone else, even better.

## Why I Started Writing This Down

When I began learning AI agents, the feeling was familiar:

- too many resources
- too many new terms
- too many frameworks
- too many demos

But even after reading a lot, the picture in my head was still fragmented.

Once I tried to build things myself, I realized many core questions were still fuzzy:

- What exactly makes an agent different from a chatbot?
- What are LLMs actually good at, and where are their limits?
- Are prompt, message, and context really different things?
- Is tool use a model capability, or a systems design problem?
- How do RAG, memory, and reliability fit together?
- Is multi-agent an upgrade in capability, or just an upgrade in complexity?

At some point I realized I did not need one more "build an agent in 10 minutes" article.
I needed a more continuous learning path.

That is why this repository exists.

## What This Repository Is

If I had to define it, it is a mix of:

- structured notes from self-study
- a knowledge map I built for myself
- a project repository that will keep being rewritten as my understanding changes

That means a few things:

First, it will not be complete from day one. Some chapters are long, some are only openings, and some ideas may change later.

Second, it is not trying to be universally comprehensive. I will prioritize the topics I actually struggled with.

Third, it reflects a personal path. It is not the only valid way to learn this field. It is simply the path that feels natural to me right now.

## Why I Avoid Calling It a Tutorial

The word "tutorial" suggests a few things:

- the structure is already finalized
- every chapter has a firm conclusion
- the author already understands the whole space completely

That is not what this repository is.

I am still learning, reorganizing, and refining my own mental model.

Some topics feel clear already. Others still need work.

If I framed this as a tutorial, I would hide the most honest thing about it:

it does not document a finished answer, but a process of getting closer to one.

## How I Currently See the Learning Path

I still want this repository to follow a clear line, so right now I think of the journey in four stages.

### Stage 1: Build the Map

Before chasing frameworks or piling on features, I want to clarify a few fundamentals:

- what an AI agent actually is
- how it differs from a chatbot
- what role the LLM plays in the system
- why context becomes the center of almost every hard problem

This stage is about drawing the map.

### Stage 2: Build the Smallest Working Loop

Once the map is clearer, the next step is to understand the minimum viable agent:

- basic chat
- message structure
- prompt design
- state management
- function calling
- tool integration

Many "advanced" problems eventually trace back to whether these basics were designed well.

### Stage 3: Understand What Actually Determines the Ceiling

The more I learn, the more I feel that many agent problems are not caused by weak models, but by poorly organized context.

That naturally leads into topics like:

- context engineering
- context management
- memory
- RAG
- evaluation

These topics are harder than a simple demo, but they are much closer to real engineering bottlenecks.

### Stage 4: Look at More Complex System Shapes

For example, multi-agent systems.

It is an attractive topic, but I no longer think it should be the starting point.

If the boundaries, context, tools, and execution flow of a single agent are still unclear, adding more agents often just duplicates the confusion.

## My Current View on Frameworks

I do not want this repository to become a framework manual.

That is not because frameworks do not matter. It is because I first want to understand:

- what abstractions actually solve
- how the underlying chain works without a framework
- which capabilities frameworks hide, and which tradeoffs they hide with them

Without that understanding, it is easy to end up in a shallow state:

- you can call APIs
- you can use framework methods
- you can run the demo

but once you need to modify, extend, or debug the system yourself, confidence disappears quickly.

## What You Will Find Here

I do not want this repository to contain conclusions only.

I also want to preserve:

- places where I thought I understood something but did not
- boundaries between concepts that are genuinely easy to confuse
- problems that only appear when moving from demo to real system design
- open questions that are still worth chasing

So the repository may feel less "standard," but more honest to the real learning process.

## How You Can Use This Repository

I suggest using it as notes from a fellow traveler rather than as an authoritative textbook.

You can use it to:

- see how I currently organize the knowledge map
- compare it against where you are stuck
- identify the next topic you should learn
- read forward while still keeping your own judgment

Different interpretations are normal.

This is not a standard-answer repository. It is an unfolding learning path.
