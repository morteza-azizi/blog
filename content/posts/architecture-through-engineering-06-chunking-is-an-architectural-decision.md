---
title: "Chunking Is an Architectural Decision"
date: 2026-09-18
draft: true
series:
  - Architecture Through Engineering

tags: ["RAG", "architecture", "engineering", "architecture through engineering"]
---

# Chunking Is an Architectural Decision

> *Part 6 of the Architecture Through Engineering series.*

---

## After Custody

Part 5 left the system with a document it was willing to keep.

Identity.

Metadata.

A status of `Pending`.

That is custody of a file. It is not yet knowledge retrieval can use.

The next cut is the one RAG tutorials treat as a parameter.

**How should source knowledge become retrievable knowledge?**

Not in the abstract.

In this codebase.

---

## The Cut That Looks Small

Chunking is easy to describe as plumbing.

Take the text.

Split it.

Embed the pieces.

Index them.

The diagram in the [architecture notes](https://github.com/morteza-azizi/architecture-through-engineering-rag/blob/main/docs/Architecture.md) already writes it that way:

```text
Upload → Parse Markdown/Text → Chunk → Generate Embeddings → Store in Qdrant
```

That line hides the decision.

A chunk that is too small loses the sentence that made the paragraph mean something.

A chunk that is too large drags in material the question did not ask for, costs more to embed, and makes the score harder to trust.

Overlap is a bet that meaning lives on the boundary.

No overlap is a bet that it does not.

There is no neutral default. "512 tokens with 50 tokens of overlap" is not an architecture. It is a choice that will show up later as retrieval quality, as token spend, and as answers that feel almost right.

I am not going to pretend I already know the right numbers.

I am going to look at what the system has already committed to, before those numbers exist.

---

## What the Domain Already Named

`DocumentChunk` is already in the domain:

```text
Id
DocumentId
Index
Content
```

A chunk knows which document it came from, where it sat in that document, and what text it carries.

It does not know a token count.

It does not know an overlap.

It does not know an embedding.

It does not know a score.

`RetrievedChunk` still wraps a chunk plus a score. That type is not a chunking decision. It is a retrieval decision, waiting for Part 7.

So the cut from Part 5 is still in force.

A document is the source the system accepted.

A chunk is a piece of that source.

A retrieved chunk is a chunk that survived a query.

Chunking is the work that produces the second of those. It has a type to land in. It does not yet have a path that produces one.

---

## The Port That Already Decided Something

The application layer already has a port for this:

```csharp
public interface ITextChunker
{
    IReadOnlyList<string> Chunk(string text);
}
```

No implementation.

No tests.

`DocumentUploadService` does not call it. Neither does anything else. `IDocumentParser` is in the same state: named, unused on the current path.

That is not an accident of incomplete plumbing. It is a shape.

Look at the signature.

The caller passes text.

The caller does not pass a size.

The caller does not pass an overlap.

The caller does not pass a content type.

The caller gets strings back, not `DocumentChunk` values.

Those absences are decisions.

**The strategy lives in the implementation, not in the call site.**

If I later swap a naive splitter for a Markdown-aware one, the port does not have to change. That is the value of putting chunking behind `ITextChunker` in the application layer instead of in a controller, a file store, or a vector database client.

If I later need different strategies per document — tables one way, prose another, code a third — this signature is already too small. The port would have to grow. I have not grown it, because I have not met that problem in this system.

Returning `string` instead of `DocumentChunk` is also a cut.

The chunker splits text.

Something else will have to attach `Id`, `DocumentId`, and `Index`.

That mapping is application work. It is not a vector-store concern, and it is not a domain constructor hiding a splitting algorithm. The domain type is the record of a piece. The port is the algorithm that proposes the pieces.

I could have put size and overlap on the interface and called that flexibility.

It would have been a different kind of incompleteness: a policy surface for numbers I cannot yet defend.

I left the numbers out of the contract.

The contract is still a contract. It says: text in, pieces out, and the rest of the system should not care how the cut was made — until it does.

---

## Precision Against Context

The trade-off does not wait for an implementation.

Smaller chunks make a match more precise. They also throw away surrounding sentences that would have made the match usable.

Larger chunks keep context. They also dilute the match, cost more to embed, and give the model more room to ignore the part that mattered.

Overlap tries to buy both. It also duplicates text, inflates the index, and can retrieve the same idea twice under two scores.

None of those is a bug. They are the bill.

A fixed strategy is cheap to ship and expensive to undo once embeddings exist. Changing chunk size later is not a config tweak. It is a re-index, a new meaning for `Index`, and a silent change in what "relevant" meant last week.

A configurable strategy looks mature. It also asks every caller to become a chunking expert, and it makes test results incomparable across documents.

A content-aware strategy looks intelligent. It also needs a parser that understands more than `.txt` and `.md` extensions, and it needs someone to own the cases where the heuristic is wrong.

The current port is the first of those: **fixed behind the interface.**

Callers cannot tune it. Implementations can replace it. That is the most I am willing to claim before a single chunk has been produced from an uploaded file.

---

## What I Have Not Proven

There are no early experiments here to report.

No chunk-size table.

No retrieval-quality graph.

No overlap study.

If I invented those, this article would be a tutorial with a series badge on it.

What I can point at today is narrower.

I can point at `DocumentChunk` in [the reference implementation](https://github.com/morteza-azizi/architecture-through-engineering-rag). I cannot point at a chunk that came from an uploaded file.

I can point at `ITextChunker`. I cannot point at a class that implements it.

I can point at a pipeline written in `Architecture.md`. I cannot point at a use case that runs parse → chunk → embed → index.

`Pending` still means: accepted, not indexed.

Until something moves a document through `Processing` into `Indexed`, chunking is a named boundary with no runtime.

That is useful to say before Part 7, because retrieval will be tempted to treat chunks as a given. They are not given. They are a cut that has not been made yet.

---

## What This Decision Costs

Treating chunking as architecture instead of a constant has a cost that is easy to feel.

There is still nothing to retrieve.

There is still no grounded answer.

There is a type, a port, and a pipeline that exists mainly as a sentence.

For a weekend chatbot, that is a lot of ceremony in front of `text.Split()`.

I am willing to pay it because the alternative is familiar: pick 512, embed everything, discover in chat that the answers are vague, and then argue about the model.

The model did not choose the cut.

The system did, even if the choice was an imported default.

---

## What It Buys

It buys a place for the decision to live.

If chunking lives in the vector database client, replacing Qdrant later means replacing the cut.

If chunking lives in the API, every future entry point has to relearn it.

If chunking lives in the domain type, `DocumentChunk` stops being a record and becomes an algorithm with fields attached.

The application port is the same kind of placement as the format policy in Part 4. "How do we turn source text into retrievable pieces?" is a system question. Infrastructure can implement it. The domain can hold the result. The application should own the need.

It also buys honesty about timing.

`DocumentChunk` is a place for a decision to land. It is not the decision.

I will have to come back to this article when an implementation exists, and again when retrieval quality starts arguing with it. That is not a failure of the series. It is the point of it.

---

## What Comes Next

A document, in this system, is still not knowledge.

A chunk is how it is supposed to become retrievable.

The type is named. The port is named. Neither has been exercised by an uploaded file.

The next question is not how to split text.

It is what retrieval means once there is something to find — and why that should be a boundary the rest of the system can depend on, not a query a controller happens to run.

**One engineering decision at a time.**
