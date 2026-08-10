---
title: "The Shape of the System"
date: 2026-08-09
draft: false
tags: ["architecture", "engineering", "architecture through engineering"]
---

# The Shape of the System

> *Part 2 of the Architecture Through Engineering series.*

The first two articles in this series were about why I chose RAG and why I wanted to build a reference implementation in the first place.

Now it's time to build something.

Not the full system.

Not yet.

Before choosing a vector database, an embedding provider, or an LLM, I need to answer a much simpler question:

**What is the shape of the system I'm actually trying to build?**

Because architecture starts becoming real when responsibilities have somewhere to live.

---

## The Obvious Picture

When people hear RAG, the system often gets reduced to a simple flow.

A user asks a question.

The system retrieves some relevant information.

That information is sent to an LLM.

The LLM generates an answer.

Something like:

{{< mermaid >}}
flowchart LR
    User --> Question
    Question --> Retrieve
    Retrieve --> LLM
    LLM --> Answer
{{< /mermaid >}}

At a conceptual level, that's correct.

But it hides most of the engineering.

Where did the knowledge come from?

Who decided what gets stored?

What happens when a document changes?

How does the system know which information is relevant?

Where does that information live?

And perhaps the most important question:

**Which part of the system owns each of those responsibilities?**

That is where the interesting architecture begins.

---

## What the System Actually Has to Do

The diagram above describes only what happens when a user asks a question.

But that's only half of the system.

Before anyone can retrieve anything, the system has to get knowledge into it.

Documents need to be uploaded.

They need to be validated.

Their contents need to be extracted and stored.

The information needs to be broken into pieces that can later be retrieved.

Those pieces need to become retrievable knowledge.

Only then does the question-and-answer part of RAG really begin.

So the system is not just:

```text
Question
↓
Retrieve
↓
Generate
```

It is closer to:

{{< mermaid >}}
flowchart TB
    subgraph Knowledge["Knowledge Flow"]
        Document --> Ingestion
        Ingestion --> Storage
        Storage --> Parsing
        Parsing --> Chunking
        Chunking --> Embedding
        Embedding --> Index
    end

    subgraph Query["Query Flow"]
        Question --> Retrieval
        Retrieval --> Context
        Context --> LLM
        LLM --> Answer
    end

    Index --> Retrieval
{{< /mermaid >}}

And each step introduces a different responsibility.

That's important because responsibilities eventually become boundaries.

And boundaries eventually become architecture.

---

## The First Shape

Once I understood that there were really two flows, I could start asking a more useful question:

**What responsibilities should the system have, and where should they live?**

I didn't start by choosing projects, frameworks, or cloud services.

I started by separating responsibilities.

At a high level, the first version of the system is taking shape around four areas:

{{< mermaid >}}
flowchart TB
    API["API"] --> Application["Application"]
    Infrastructure["Infrastructure"] --> Application
    Application["Application"] --> Domain["Domain"]
{{< /mermaid >}}

The API is responsible for communicating with the outside world.

The Application layer coordinates what the system needs to do.

The Domain represents the concepts the system actually cares about.

Infrastructure deals with the things the application should not have to know how to implement itself.

That's a deliberately simple shape.

And at this point, that is exactly what I wanted.

I didn't need to decide which vector database to use.

I didn't need to decide which embedding provider to use.

I didn't need to decide where the system would eventually run in the cloud.

I needed to establish **where responsibilities belonged before those technologies started influencing the design.**

That distinction turned out to be more important than any technology choice I could have made at this stage.

This shape is already taking form in the [reference implementation](https://github.com/morteza-azizi/architecture-through-engineering-rag) I'm building for this series, and the code is available in the open.

---

## What the Shape Deliberately Doesn't Say

The diagram above can easily be mistaken for a finished architecture.

It isn't.

At this stage, the shape says **where responsibilities live**.

It does not say which database stores the data.

It does not say which model generates embeddings.

It does not say which LLM answers questions.

It does not say whether the system runs locally, in Azure, or somewhere else.

Those are implementation decisions.

And I deliberately don't want those decisions driving the shape of the system.

The architecture should give those technologies somewhere to live, rather than allowing the technologies to decide the architecture for us.

That's an important distinction.

The first version doesn't need to solve every future problem.

It needs to give us enough structure to start building — and enough room to discover what we got wrong.

---

## What We Are Not Building Yet

There is another architectural decision hidden in this first shape.

Knowing what **not** to build.

It would be easy to add more to the system immediately.

Multiple AI providers.

Distributed infrastructure.

Agentic workflows.

Multi-tenancy.

Complex authorization.

A production-scale vector platform.

All of those may become relevant later.

But adding them now would make the system more impressive without necessarily making it more useful.

For the first version, I want to keep the scope deliberately narrow.

Build the core flow.

Understand the boundaries.

Run it.

Test it.

Then let the engineering tell us what needs to change.

Because every capability we add creates another decision to maintain.

And at this stage, **complexity is not evidence of architectural maturity.**

Knowing what to leave out is part of the architecture too.

---

## The Shape Is a Starting Point, Not the Architecture

At this point, it would be tempting to call this architecture.

I don't think it is.

It is the **starting shape** of the system.

The boundaries are hypotheses.

The responsibilities are hypotheses.

Even some of the decisions that feel obvious today may turn out to be wrong once the system starts dealing with real documents, real retrieval, and real failures.

That's intentional.

The purpose of this first shape isn't to predict the final architecture.

It's to give the engineering somewhere to start.

As the system evolves, some boundaries will hold.

Some will move.

Some decisions will become more expensive to change.

And some will simply turn out to have been wrong.

That's where Architecture Through Engineering really begins.

The interesting part isn't drawing the first diagram.

It's discovering what happens to that diagram once the code has to live with it.

---

## What Comes Next

The shape is now clear enough to start building.

Not because we have solved the architecture.

We haven't.

We have simply established where the first responsibilities live, what we are deliberately leaving out, and which decisions we are willing to let the engineering challenge.

The next step is to look more closely at those boundaries.

Why should the API know about the application, but not the other way around?

Why should the domain remain independent of infrastructure?

Why introduce abstractions before choosing the technologies behind them?

Those are not theoretical questions anymore.

They are decisions already taking shape in the codebase.

And now it's time to examine them.

**One engineering decision at a time.**
