---
title: "RAG in Microsoft Foundry: Architecture and Trade-offs"
description: "What retrieval architecture should look like when a Foundry application needs enterprise knowledge."
date: 2026-11-03
draft: true
categories:
  - Foundry
series:
  - Microsoft Foundry
tags: ["microsoft foundry", "architecture", "AI", "RAG"]
---

# RAG in Microsoft Foundry: Architecture and Trade-offs

> *Part 4 of the Microsoft Foundry series.*

> *Draft. Run the four cases, paste the retrieved chunks and replies under Observed, then write Conclusion from them. Do not publish this sentence.*

## Opening

The agent can now read a dead-letter message and look up its producer. It still cannot answer the question an engineer asks next: what do we do about this?

That answer is in a runbook. We have twelve of them in markdown, written over several years by people who have since moved teams. Two of them disagree about whether it is safe to bulk-resubmit a dead-letter queue.

**When an AI application needs enterprise knowledge, what should the retrieval architecture actually look like?**

This article builds a small retrieval pipeline over those runbooks and attaches it to the agent. The interesting parts are not the embedding step. They are what gets into the index, what the agent does when the index has no answer, and what happens when the index has two.

I run four cases. One question the corpus answers. One it does not. One where two runbooks disagree. One where retrieval is the wrong source altogether.

**TL;DR.** Retrieval is a pipeline with three owners: what goes into the index, what comes back, and what the agent does with it. The most useful decision is made at ingestion, where a superseded runbook can be kept out. The results are not in this draft yet.

---

## Context

I wrote about the hand-built version of this in [Architecture Through Engineering](/posts/architecture-through-engineering-04-rag-starts-with-ingestion-not-chat/). Ingestion, chunking, and retrieval were separate code that I owned. [Chunking](/posts/architecture-through-engineering-06-chunking-is-an-architectural-decision/) was the cut I could not call a parameter.

This article is the managed counterpart. Foundry gives me several retrieval options, and the choice is the architecture:

| Option | What it does | What I give up |
|---|---|---|
| File search tool | Upload files. The service parses, chunks, embeds, and runs hybrid search. | Control of chunking and of what is retrievable. A per-agent store. |
| Azure AI Search tool | The agent queries one existing Azure AI Search index. | Federation across sources. I build ingestion and citations. |
| Foundry IQ knowledge base | A reusable knowledge base over several sources, with query planning, reranking, citations, and permission-aware retrieval. Built on Azure AI Search. | Simplicity. Parts of it are still in preview. |

The sources are the [file search](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/file-search) and [Foundry IQ](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/what-is-foundry-iq) pages.

I chose the Azure AI Search tool for this article. Not because it is the best of the three. Because the decision I want to test is made at ingestion, and I want ingestion to be mine. File search would hide it. Foundry IQ is the likely direction for a multi-source, multi-agent system, and I come back to it in the trade-offs.

Assumptions:

- The corpus is twelve markdown runbooks, about 40 pages in total. Each has front matter with `status` (`current` or `superseded`), `version`, and `owner`.
- Two runbooks cover the same procedure: `dlq-resubmit-v1.md` and `dlq-resubmit-v2.md`. Version 1 says to resubmit the whole queue from the portal. Version 2, written after an incident, says to check idempotency first and never bulk-resubmit. Version 1 is marked `superseded`.
- One runbook says: "A queue's `MaxDeliveryCount` defaults to 10." The real `orders` queue is set to 5.
- The corpus has no information about recovery objectives for any queue.

Those four facts are the experiment. I wrote the corpus to contain them.

---

## Architecture

Retrieval is three jobs. They fail in different places, and a mistake in the first one cannot be repaired by the third.

{{< mermaid >}}
flowchart LR
    Source["Runbooks<br/>markdown + front matter"] --> Ingest["Ingestion<br/>parse, filter, chunk, embed"]
    Ingest --> Index["Azure AI Search index<br/>runbooks-current"]
    Agent["Agent"] -->|"query"| Index
    Index -->|"top chunks + source"| Agent
    Agent --> Answer["Answer with citation<br/>or no answer"]
    Config["Entity config<br/>MaxDeliveryCount per queue"] -.->|"not retrieved"| Agent
{{< /mermaid >}}

| Job | Owner | Decides |
|---|---|---|
| Ingestion | Me, in code | What is retrievable: which documents, which versions, where the cuts are |
| Retrieval | Azure AI Search | Which chunks match a query, and in what order |
| Generation | The agent | What to say with the chunks it was given, including saying nothing |

The retrieval job has no concept of "this runbook is wrong now". An index returns what is close to the query, and `superseded` is not close or far. It is a fact about the document.

So the first architectural decision sits before any query exists.

> **Lifecycle is an ingestion concern.** If a superseded document must not be an answer, do not rely on a query to avoid it. Do not let it into the index.

The dotted line in the diagram is the second decision. `MaxDeliveryCount` is a number that Service Bus knows. A runbook can describe the concept. It cannot hold the current value for each queue. A fact with an authoritative source should come from that source, through an API or a tool, not from a paragraph that was true when somebody typed it.

---

## Build

The ingestion job reads each runbook, drops anything that is not current, splits by heading, and writes the chunks to the index.

```csharp
foreach (var doc in runbooks)
{
    if (doc.Status != "current")
    {
        log.LogInformation("Skipping {Doc}: status {Status}", doc.Id, doc.Status);
        continue;
    }

    foreach (var (section, text) in MarkdownSections.Split(doc.Body))
    {
        var chunk = new RunbookChunk
        {
            Id = $"{doc.Id}:{section.Slug}",
            DocId = doc.Id,
            Title = doc.Title,
            Section = section.Heading,
            Version = doc.Version,
            Status = doc.Status,
            Content = text,
            ContentVector = await embedder.EmbedAsync(text)
        };
        await index.UploadAsync(chunk);
    }
}
```

The index has `Id`, `DocId`, `Title`, `Section`, `Version`, `Status`, `Content`, and `ContentVector`. `Title`, `Section`, and `Version` are there for the citation. `Status` is there as a safety net, and as a way for a test to see what the filter did.

I split by heading, not by token count. A runbook section is a unit a person can follow: a cause, a check, an action. A fixed window cuts through those. That is a choice, not a default, and I did not tune it.

The agent gets the Azure AI Search tool pointed at the `runbooks-current` index. Its instructions gain three lines:

```text
Answer questions about handling failed messages only from the retrieved runbook sections.
Cite the runbook title and section for every step you give.
If the retrieved sections do not answer the question, say that the runbooks do not cover it.
Do not state a queue's configuration from a runbook. Say it must be looked up.
```

For the experiment I need a contrast. A second index, `runbooks-all`, is built with the status check removed. It contains both resubmit runbooks.

Code: folder `04-rag` in the series repo.

---

## Experiment

Four cases. Each response also records the chunks returned by the search: their ids, their scores, and their order. I want to see what was retrieved, not only what was said.

**1. Answerable.** *The delivery count on a message reached the maximum and it was dead-lettered. What do I check first?*

**2. Unanswerable.** *What is the recovery point objective for the `payments` queue?*

**3. Conflicting versions.** *Can I resubmit everything in the `orders` dead-letter queue?* Run against `runbooks-all`, then against `runbooks-current`.

**4. The wrong source.** *How many delivery attempts does `orders` allow before dead-lettering?*

Some of what this experiment depends on is platform behaviour I have not seen. I keep it apart from results:

| Claim | Source | Status |
|---|---|---|
| The Azure AI Search tool queries one existing index | Foundry documentation | Documented |
| The run output exposes the retrieved chunks, with ids and scores | None | Unknown. If it does not, I run the same query directly against the index and record what the index returns. That is a record of the search, not of what the agent was shown |
| A query-time filter on `Status` can be applied through the tool | None | Unknown. If it cannot, ingestion is the only place lifecycle can be enforced, which is the point of case 3, and I say so |
| Foundry IQ would handle this better | Foundry documentation describes the capabilities | Not tested |

**Expected**

*These are expectations. They are not results.*

1. The top chunks come from the `MaxDeliveryCountExceeded` runbook. The answer follows that section and cites it.
2. A nearest-neighbour search usually returns its closest chunks whether or not they answer the question, so I expect the search to return something. The reply says the runbooks do not cover it. It does not give a number.
3. On `runbooks-all`, both versions are likely to be retrieved, and the answer may follow either. On `runbooks-current`, only version 2 is in the index, and the answer says not to bulk-resubmit.
4. The search returns the sentence about the default of 10. The reply either says the value must be looked up, as instructed, or states 10.

The failure worth catching in case 3 is a confident answer that follows the superseded runbook. In case 4 it is "10".

**Observed**

Not run yet. Paste each result here, unchanged.

| # | Case | Index | Chunks returned (id, score) | Reply | Cited |
|---|---|---|---|---|---|
| 1 | Answerable | `runbooks-current` | | | |
| 2 | Unanswerable | `runbooks-current` | | | |
| 3a | Conflicting | `runbooks-all` | | | |
| 3b | Conflicting | `runbooks-current` | | | |
| 4 | Wrong source | `runbooks-current` | | | |

**Conclusion**

Written from the results, after the run. Not before.

---

## Trade-offs

The pipeline makes the agent's knowledge inspectable. I can list what is in the index, what was retrieved for a question, and which source the citation points to. That is the main thing it buys. An answer that cannot be traced to a section is a claim I cannot check.

Relevance and grounding fail in different ways.

- **Relevance fails quietly.** The right chunk is in the index but ranks fifth. The agent answers from the first four.
- **Grounding fails fluently.** The agent is given the right chunk and says something slightly different. A citation is attached to a sentence the source does not contain.
- **Absence is not signalled.** A search for something that does not exist still returns the nearest text. The only protection is an instruction to say "not covered", and that is a sentence, not a control. Part 5 measures whether it holds.

The cost is operational. Ingestion is a job to schedule, monitor, and re-run when the corpus changes. An index that lags its source is a stale answer with a citation. A change to chunking is a re-index and a change in what "relevant" meant yesterday.

I did not choose the other two options. File search would have run in less time, and it is a good fit for a handful of documents attached to one agent. It does not give me the ingestion decision I wanted to test. Foundry IQ is where I would look for a system with several sources, several agents, and per-user permissions on the content. It builds on Azure AI Search, and parts of it are still in preview. I have not tested it here.

I am assuming the corpus is small, one language, and maintained by people who mark what is superseded. If the front matter is not trustworthy, the filter is not either.

---

## When NOT to Use It

Do not retrieve a fact that has an authoritative source. A queue's `MaxDeliveryCount` is a property of an entity. A producer's owner is in the registry from Part 3. A runbook that states a value is a copy, and a copy goes stale.

Do not build a pipeline for a corpus that fits in the prompt. Twelve short runbooks are close to that line. Retrieval is worth its cost when the corpus is large, changing, or access-controlled.

Do not use retrieval when the answer has to be the same every time. A rule such as "dead-letter after five attempts" is a configuration, not knowledge.

---

## Practical Takeaway

1. **Decide what may be retrieved before anyone asks.** Lifecycle, ownership, and version are properties of a document, and ingestion is the only place that can act on them.
2. **Measure retrieval separately from the answer.** Look at the chunks the search returned. A good-sounding reply built on the wrong chunk is the failure that stays hidden.
3. **Route facts to their source.** Use retrieval for explanations and procedure. Use an API for values.

---

## Next Article

> In the next article, we move from **building the RAG flow** to **evaluation**, and examine **how to know whether the application works without judging it from a few successful tests**.
