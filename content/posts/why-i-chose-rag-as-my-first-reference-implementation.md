---
title: "Why I Chose RAG as My First Reference Implementation"
date: 2026-08-02
draft: false
tags: ["RAG", "architecture", "engineering", "architecture through engineering", "series", "AI"]
---

# Why I Chose RAG as My First Reference Implementation

> *Part 1 of the Architecture Through Engineering series.*

**The project:** *[Placeholder — one real sentence describing what the system actually does. Example shape: "a Q&A assistant over our internal engineering documentation," not "a RAG application." Replace before publishing.]*

---

## The Obvious Assumption

The easy explanation would be that RAG is trendy right now.

That would also be the wrong explanation.

This series isn't about keeping up with the latest AI pattern. It's about learning architecture by building something real, one engineering decision at a time. RAG isn't the subject of that. It's simply the first vehicle.

So the real question isn't "why RAG." It's "what does a good first project for this series actually need to look like, and does RAG happen to qualify." That also means this won't be a RAG tutorial, a comparison of vector databases and embedding models, or a chase after whichever retrieval technique is newest this month. Where any of those things show up later, it will be because a specific architectural decision required them, not because they were the point.

---

## Why a Reference Implementation at All

Most architecture writing uses examples that only exist for one paragraph.

A snippet here. A diagram there. A different toy system for every point being made.

That's convenient for the writer, but it costs the reader something important: continuity. You never get to see a decision made in chapter one come back to bite, or pay off, in chapter five.

A reference implementation fixes that. One system, built in the open, returned to across the whole series. Every post can build on the decisions made in the last one, the same way real systems do.

That only works if the system itself is chosen carefully.

---

## What a Reference Implementation Needs

Before picking a project, I wrote down what it actually had to satisfy.

It had to be small enough to build and operate by myself.

It had to force at least one decision that would be genuinely expensive to reverse later, not a trade-off that only exists on paper.

It had to touch more than one architectural concern. Data, integration, infrastructure, and observability, not just one layer in isolation.

It had to be able to evolve credibly. Not stay a weekend prototype, but plausibly grow into something closer to a real, cloud-native platform.

And it had to be timely without depending on being timely. If someone reads this series in three years, the architectural questions should still be worth asking, even if the specific technology has moved on.

Most candidate projects fail at least one of these.

---

## The Tension I Won't Pretend Isn't There

Here's the honest problem with everything above.

The manifesto behind this series argues that architecture isn't planned in advance. It emerges through engineering. Good architecture is discovered, not designed on a whiteboard before anyone writes code.

And yet this entire article is a whiteboard. Criteria written down first, a candidate checked against them second. That's planning, not emergence, and it would be dishonest to pretend otherwise.

I don't think that's a contradiction, but it is a boundary worth being precise about. Choosing *what to build* is an upfront decision by necessity, someone has to start somewhere. What can't be planned in advance is *how the system should be built*, and what it should become once it meets real constraints, real failures, and real usage. This article is about the first kind of decision. Almost everything else in this series will be about the second kind.

There's also a more uncomfortable version of this problem worth naming. Criteria written *after* you already want to build something are very good at justifying that thing. I wrote the list above before I had fully committed to RAG, not after, but I can't prove that to you, and you shouldn't take it on faith. Judge it the same way the manifesto asks you to judge everything else in this series: by what the engineering actually leaves behind, not by what the reasoning claimed in advance.

---

## Why RAG Fits

A Retrieval-Augmented Generation application satisfies the criteria above, and it does so for reasons that have nothing to do with AI being interesting right now.

**Knowledge representation.** Before anything else, you have to decide how knowledge is stored and chunked, and what happens when the source material changes. This is a real, and genuinely uncomfortable, trade-off: smaller chunks give more precise retrieval but fragment context; larger chunks preserve context but dilute relevance and cost more per query. There is no neutral default here, only a choice with consequences that show up much later, closer to the same shape as the ownership problem I described in Part 0, where the cost of a bad boundary decision doesn't appear until something downstream breaks. *[Placeholder — once this decision is actually made and lived with, state which way it went and why, rather than leaving the trade-off unresolved.]*

**Context construction.** Retrieval, ranking, and prompt assembly all sit between the data and the answer. Get this wrong and everything downstream inherits the mistake. This is an integration problem wearing an AI costume: it's the same "who is responsible when the wrong thing gets passed downstream" question as any message contract between two systems.

**Infrastructure evolution.** A vector store that works on a laptop is not the same decision as a vector store that has to work in production, at scale, under a budget.

**Observability.** When a traditional service returns the wrong answer, you can usually trace why. When a RAG system returns a wrong answer that still sounds confident, tracing why is a genuinely hard, still-evolving problem, and it raises the same ownership question as the story in Part 0: when the answer is wrong, whose fault is it, the retrieval, the source data, or the model? Someone still has to own that boundary, even when the failure is fuzzier than a rejected message.

**The local-to-cloud path.** A RAG prototype is trivial to run on a single machine. Turning it into something reliable, observable, and cloud-native is a much longer, much more interesting road, and that road is architecture.

None of those five problems are specific to RAG. They show up in plenty of other systems. RAG just happens to surface all five of them clearly, in a single, buildable project.

That's the actual reason. Not the trend. The fit.

---

## Closing

Choosing RAG was itself the first architectural decision of this series, and like the checklist it came from, it's a decision made in advance, before any consequences of it exist yet.

So here's a claim I'm willing to be held to. If a year from now this project hasn't forced me to reverse at least one decision I was confident about today, I picked the wrong reference implementation, and I'll say so in this series, not quietly move on to a different project.

That's the actual test. Not whether the criteria sounded reasonable in this post, but what the engineering leaves behind.

The next post starts where every real project starts: with the shape of the system before a single line of production code exists, what the first version deliberately won't do yet, and which of those early decisions are cheap to reverse later and which ones aren't.

Not all at once. One engineering decision at a time.
