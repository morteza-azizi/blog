---
title: "What Is a Document?"
date: 2026-09-14
draft: true
series:
  - Architecture Through Engineering

tags: ["RAG", "architecture", "engineering", "architecture through engineering"]
---

# What Is a Document?

> *Part 5 of the Architecture Through Engineering series.*

---

## After the Door

Part 4 ended with a file the system was willing to keep.

A row in SQLite.

A status of `Pending`.

And a word I had been using too loosely: document.

**What does the system actually mean by a document?**

Not in the abstract.

In this codebase.

---

## A File Is Not a Document

The file still lives on disk, behind `IDocumentFileStore`.

The domain does not hold those bytes.

What it holds is this:

```text
Id
FileName
ContentType
SizeBytes
UploadedAt
Status
```

That is the current `Document` type in [the reference implementation](https://github.com/morteza-azizi/architecture-through-engineering-rag). Six fields. No content. No chunks. No behavior.

A file is what arrived.

A document is the system's claim that it has taken custody of that arrival: identity, a name, a recorded type, a size, a time, and a status.

Those are different things. Part 4 already stored them in different places. The model is just catching up to that split.

If the bytes and the metadata were the same concept, losing one would mean losing the other. They are not. That is why an orphaned file was a lie. The document is the part the rest of the system can see.

---

## Status Is the Model That Matters

`Document` is a data bag. I am not going to pretend it is a rich domain object.

The interesting part is `Status`.

```text
Pending
Processing
Indexed
Failed
```

The upload path only writes `Pending`.

`Processing`, `Indexed`, and `Failed` already exist. So does `UpdateStatusAsync`. A repository test can move a document to `Indexed`. No application path does.

That is not accidental incompleteness in the type. It is a lifecycle drawn before the work that would use it.

`Pending` means: we accepted the file. We have not turned it into retrievable knowledge.

`Processing` would mean: someone is doing that work.

`Indexed` would mean: the work finished, and retrieval may see it.

`Failed` would mean: the work did not finish, and the document should not be treated as knowledge.

I do not yet have the code that justifies the last three. I have the words.

A model that names states it cannot reach is already making a promise. The honest version of that promise is: this is the lifecycle I expect, not the lifecycle I operate.

A model with only `Pending` would have been smaller, and more true to the current path. It would also have left the next stages with nowhere to record what happened. I chose the larger enum. That choice is still under test.

---

## Where a Document Ends

`DocumentChunk` already exists too.

```text
Id
DocumentId
Index
Content
```

A chunk knows which document it came from, where it sat in that document, and what text it carries. It does not know the file name. It does not know the status. It does not know a score.

`RetrievedChunk` adds the score. That type is not a document. It is what search returns: a chunk plus a number.

So the cut is already in the domain, even though nothing on the ingestion path creates a chunk yet.

A document is the source the system accepted.

A chunk is a piece of that source, later, if indexing happens.

A retrieved chunk is a chunk that survived a query.

Those are three concepts, not one with extra fields. Putting content on `Document` would have collapsed the first two. Putting a score on `DocumentChunk` would have collapsed the last two.

The types are small because the distinctions are the point.

---

## What I Left Out

There is no owner.

No version.

No checksum.

No hash of the bytes.

No original path.

No encoding.

No page count.

No embedding on the chunk.

No token count.

No overlap.

I could have added any of those and called the model more complete.

It would have been a different kind of incompleteness: a model that looks ready for problems the system has not met.

`Document` does not contain the file. That is deliberate. Content lives with the file store until something later reads it.

`DocumentChunk` does not contain an embedding. That is also deliberate. An embedding is an infrastructure artifact. The chunk is still text with a place in a document.

`RetrievedChunk` wrapping a chunk plus a score is the one extra shape I allowed, because retrieval is not the same operation as indexing. Part 7 will have to live with that.

The risk in this article is not that the model is too thin.

It is that `DocumentChunk`, `RetrievedChunk`, and three unused statuses are already a sketch of Parts 6 and 7, drawn in types before those paths exist.

A model that is too rich too early is its own kind of mistake. I am closer to that mistake than to a missing `Document` type.

---

## What This Model Proves

What I can prove today is still narrow.

I can create a `Document` with `Pending`, persist those six fields, and read them back. I can update a status in the repository. I cannot show a document moving through `Processing` into `Indexed` or `Failed` because that path is not built.

I can point at `DocumentChunk` and `RetrievedChunk` in the domain project. I cannot point at a chunk that came from an uploaded file.

So the model is slightly ahead of the engineering. Part 4's door is real. This article's lifecycle is mostly a drawing in code.

That is useful to say before Part 6, because chunking will be tempted to treat `DocumentChunk` as a finished decision. It is not. It is a place for a decision to land.

---

## What Comes Next

A document, in this system, is not the file and not yet the knowledge.

It is identity, metadata, and a status that currently only means: accepted, not indexed.

Chunks are how that document is supposed to become retrievable. They are already named. They are not yet produced.

The next question is not what a document is.

It is how source material should be broken so retrieval has something to find — and what that choice will cost later, when changing it is no longer cheap.

**One engineering decision at a time.**
