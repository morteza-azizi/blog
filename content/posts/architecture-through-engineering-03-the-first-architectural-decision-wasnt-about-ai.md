---
title: "The First Architectural Decision Wasn't About AI"
date: 2026-08-26
draft: false
series:
  - Architecture Through Engineering

tags: ["architecture", "engineering", "architecture through engineering"]
---

# The First Architectural Decision Wasn't About AI

> *Part 3 of the Architecture Through Engineering series.*

## The First Decision

The project was supposed to be about RAG.

Documents.  
Chunks.  
Embeddings.  
Retrieval.  
LLMs.

But before any of that, there was a more basic question:

**What depends on what?**

If the API knows about the database, the application knows about HTTP, and the domain knows about infrastructure, technical decisions start leaking across the system.

A change in one place becomes a change in many places.

So before getting into RAG-specific implementation, I made a structural decision:

**The core of the system should not depend on its infrastructure.**

Not which LLM to use.

Not which vector database to choose.

Not even how retrieval should work.

**The first architectural decision was the dependency direction.**

---

## Dependency Direction

The important part isn't the names of the projects.

It is the direction of the dependencies.

This is the same shape as Part 2. The question now is why the arrows point this way.

{{< mermaid >}}
flowchart TB
    API["API"] --> Application["Application"]
    Infrastructure["Infrastructure"] --> Application
    Application["Application"] --> Domain["Domain"]
{{< /mermaid >}}

The dependency direction is deliberately simple:

- API depends on Application
- Application depends on Domain
- Infrastructure depends on Application
- Domain does not depend on Application or Infrastructure.

This means infrastructure can provide implementations without making the application depend on the technology providing them.

The application describes what it needs.

Infrastructure decides how those needs are fulfilled.

That is the boundary I wanted.

---

## What This Buys Us

This gives the system some room to change.

The application can ask for a document repository without knowing whether the implementation uses SQLite or something else.

It can ask for file storage without knowing whether the files live on a local disk or in cloud storage.

The same boundary can later apply to other capabilities:

- embeddings
- vector storage
- chat completion
- document parsing

The goal isn't to make change free.

It is to make some changes **more contained**.

Architecture cannot prevent change.

It can influence how far the consequences of change travel.

---

## What It Costs Us

There is a cost to this structure.

More projects.

More abstractions.

More dependency-injection wiring.

More indirection.

For a small application, that cost is real.

And this is where I don't want to hide behind the words **Clean Architecture**.

A label doesn't justify complexity.

The useful question is:

**Is this structure buying enough flexibility to justify what it costs?**

If the system never evolves, some of this structure may turn out to have been unnecessary.

That's a possibility I'm willing to accept.

---

## Is It Already Paying Off?

The repository gives us some early evidence.

The application layer defines abstractions such as:

- `IDocumentRepository`
- `IDocumentFileStore`
- `IDocumentParser`
- `IEmbeddingGenerator`
- `IVectorStore`
- `IChatCompletionService`

The local infrastructure project provides concrete implementations and infrastructure concerns such as document storage, SQLite persistence, and health checks.

So the boundary isn't just something I drew on a diagram.

There is already code behind it.

But that doesn't prove the architecture is correct.

The system hasn't changed enough yet.

We haven't replaced enough infrastructure.

We haven't discovered all the problems with the abstractions.

We haven't paid all the costs.

So for now, this is simply a **decision under test**.

And that's intentional.

---

## What Comes Next

The dependency direction gives the system a starting point.

Now it needs to do something useful.

Before it can retrieve knowledge, it needs to acquire knowledge.

That means documents.

Uploading them.

Validating them.

Storing them.

And eventually turning them into something the system can retrieve.

So the next architectural question isn't about models or vector databases either.

It is:

**Why start with ingestion?**

That's where the next piece of engineering begins.

**One engineering decision at a time.**
