---
title: "RAG Starts With Ingestion, Not Chat"
date: 2026-09-13
draft: false
series:
  - Architecture Through Engineering

tags: ["RAG", "architecture", "engineering", "architecture through engineering"]
---

# RAG Starts With Ingestion, Not Chat

> *Part 4 of the Architecture Through Engineering series.*

---

## The Part Everyone Skips

A user asks a question.

The model answers.

Someone adds a vector database in the middle, and the article moves on.

That picture is not wrong. It is just incomplete in a way that hides the work.

Nothing can be retrieved that was never ingested.

And nothing ingested poorly will suddenly become trustworthy because a chat UI sits in front of it.

So the first useful feature in this system is not chat.

It is the door.

**How does knowledge enter the system?**

---

## Two Flows, One Starting Point

Part 2 already drew this distinction.

There is a query flow:

```text
Question → Retrieve → Generate
```

And there is a knowledge flow:

```text
Document → Ingestion → Storage → Parsing → Chunking → Embedding → Index
```

The first flow is what people demo.

The second is what makes the first one possible.

Ingestion is the first feature that puts Part 3's dependency direction to work. Dependency direction is only a hypothesis until a feature has to live inside it.

That is why I started here. Not because it is glamorous. Because it is the first place the system has to decide what it will accept, what it will refuse, where those decisions live, and who owns the failure when something arrives that should never have been accepted.

---

## What "Ingestion" Means Here

I want to be precise, because the word is doing too much work in most RAG diagrams.

In this system, ingestion is not yet "turn a file into searchable knowledge."

That full path still includes parsing, chunking, embedding, and indexing. Those come later.

What exists today is the front of that path:

A document arrives.

The system decides whether it is allowed in.

If it is, the file is stored, metadata is recorded, and the document is left in a `Pending` state.

That is a smaller definition than the architecture diagram suggests.

It is also an honest one.

In the [reference implementation](https://github.com/morteza-azizi/architecture-through-engineering-rag) I am building, the current ingestion path starts here: it accepts a Markdown or plain-text file, stores it, records its metadata, and leaves the document in `Pending`. The later stages are not part of this path yet.

I could have waited until the whole pipeline existed. That would have made ingestion look more complete than it is. It would also have hidden the decision: treat the door as its own responsibility before the rest of the path exists.

---

## What Happens at the Door

The path is deliberately short.

{{< mermaid >}}
flowchart TB
    Upload["HTTP upload"] --> Validate["Validate"]
    Validate -->|rejected| Refuse["Refuse with a typed failure"]
    Validate -->|accepted| Store["Store the file"]
    Store --> Persist["Persist metadata"]
    Persist -->|success| Pending["Document stays Pending"]
    Persist -->|failure| Compensate["Delete the stored file"]
{{< /mermaid >}}

A request hits `POST /api/documents`.

The API reads the multipart file and hands it to `DocumentUploadService`. The service decides whether the file is allowed in: empty, too large, unsafe name, unsupported format. Only then does anything get written.

The file goes to `IDocumentFileStore`, while its metadata is persisted through `IDocumentRepository`. The document starts in `Pending`.

`Pending` is an admission. The system has accepted custody of a file. It has not yet claimed that the file is knowledge.

If metadata persistence fails after the file is stored, the service deletes the stored file and rethrows. That compensation path is part of the door, not an afterthought.

---

## Where Those Decisions Live

This is where Part 3 stops being a diagram.

The format rules do not live in the controller.

They do not live in the filesystem adapter.

They live in the application layer, in a small policy: only `.txt` and `.md`, no directory paths, no rooted paths, no empty files, no files above the configured size.

That placement is the decision.

If the API owns format rules, every future entry point has to relearn them.

If infrastructure owns them, the rule becomes an accident of the current storage technology.

The application layer is the place that already describes what the system needs. "Which files are we willing to take responsibility for?" is that kind of question.

The API still has a job. It translates HTTP into that use case, and it turns validation failures into `ProblemDetails`.

Infrastructure still has a job. It stores bytes and rows. It also defends itself — the local file store will not write a path that escapes its directory — but that is a storage invariant, not a document policy.

The same file can be rejected twice, for two different reasons, owned by two different layers.

The application is saying: this is not a document we accept.

The file store is saying: this is not a path we will write.

Those are different sentences.

Invalid input is rejected at the boundary. Infrastructure failure is handled as infrastructure failure. Those are different ownership problems.

The infrastructure failure illustrated in the diagram is the case where the file is stored, then metadata persistence fails. The service deletes the stored file and rethrows.

Not because compensation is elegant.

Because a file with no metadata is an orphan, and an orphan is a lie the rest of the system cannot see. If cleanup itself fails, the error is logged. That is not a finished operational story. It is the first honest one.

---

## What the Current Implementation Proves

What I can prove today is narrow.

The path accepts `.txt` and `.md`. It rejects PDF, archives, missing extensions, paths, empty files, and oversized files before any write. Source bytes and metadata live in two stores. Accepted documents stay `Pending`. If metadata persistence fails, the stored file is deleted.

Unit tests cover the policy and the compensation path. Integration tests cover the HTTP shape: Markdown is created, PDF is `400`, empty is `400`, an oversized payload is rejected, a path in the file name does not produce a file on disk.

PDF is deferred because it is not another file extension. It is a parsing problem, an encoding problem, a layout problem, and eventually a chunking problem. Letting a `.pdf` through before those problems have an owner would only move the failure later and call it progress.

What the path does not prove is just as important.

Content validation is still shallow. The policy looks at the extension, not the content. The filename determines the format decision. The content type is stored, not believed. `IDocumentParser` exists in the application layer and is not used on this path.

There is no document lifecycle beyond the initial upload, and no reprocessing path.

Security and tenancy are out of scope.

Those are not accidental omissions. If I add PDF next week without deciding who parses it, or parse during upload before I know what a document is, ingestion will look more capable and become less trustworthy.

That is why this article stops at the door.

---

## What This Decision Costs

Starting with ingestion instead of chat has a cost that is easy to feel.

There is nothing impressive to show. No grounded answer. No retrieval demo.

A `Pending` document is a correct result and a disappointing screenshot.

The structure also costs something: a format policy, typed exceptions, an exception handler, compensation around two writes, and tests that exist mainly to prove that rejected files never land.

For a weekend chatbot, that is a lot of ceremony in front of "save a file."

I am willing to pay it because the alternative is familiar: let chat become the place where data-quality problems are discovered, discussed, and vaguely blamed on the model.

---

## What It Buys

It buys a refusal that belongs to the system, not to the model. When a PDF is rejected, the API can say so. The model is not asked to be honest on behalf of a boundary it does not own.

That is why the ceremony is worth it. The chat experience can only be as trustworthy as the ingestion path behind it.

---

## What Comes Next

The system can now take custody of a file.

That forces a question I have been using the word "document" to avoid.

What is that thing, once it is inside?

A file on disk is not the same as a domain document.

A row of metadata is not the same as knowledge.

`Pending` is a status, not a model.

The next article looks at the document itself: what has to be true about it, what state it carries, and where a document ends and a chunk begins.

Ingestion only gets a file through the door.

The domain still has to decide what it became.

**One engineering decision at a time.**
