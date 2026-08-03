---
title: "Why I Chose RAG as My First Reference Implementation"
date: 2026-08-02
draft: false
tags: ["RAG", "architecture", "engineering", "architecture through engineering", "series", "AI"]
---

# Why I Chose RAG as My First Reference Implementation

> *Part 1 of the Architecture Through Engineering series.*

---

## The Obvious Question

A## The Obvious Question

After publishing the manifesto, one question appeared almost immediately.

Why RAG?

It's a fair question.

The internet already has thousands of RAG tutorials, examples, and reference implementations. If the goal of this series is architecture, I could have chosen almost anything.

A microservice platform.

An event-driven system.

An e-commerce application.

So why start here?

Most of my career has been spent building integration-heavy systems.

APIs.

Messaging platforms.

Event-driven architectures.

Systems where information moves across boundaries and ownership matters as much as technology.

When I started exploring RAG, something felt strangely familiar.

The technologies were different, but the questions were not.

The more I looked at it, the less it felt like an AI problem.

It felt like an integration problem wearing an AI costume.

---

## Why a Reference Implementation at All

Most architecture writing uses examples that only exist for one paragraph.

A snippet here. A diagram there. A different toy system for every point being made.

That's convenient for the writer, but it costs the reader something important: continuity.

Real systems don't reset between decisions.

Neither should the examples used to explain them.

You never get to see a decision made in chapter one come back to bite, or pay off, in chapter five.

A reference implementation fixes that. One system, built in the open, returned to across the whole series. Every post can build on the decisions made in the last one, the same way real systems do.

That only works if the system itself is chosen carefully.

---

## What a Reference Implementation Needs

Before picking a project, I had to answer a simpler question.

What would make a project worth following for an entire series?

It had to be small enough to build and operate by myself.

It had to force at least one decision that would be genuinely expensive to reverse later, not a trade-off that only exists on paper.

It had to force decisions across multiple architectural concerns. Data, integration, infrastructure, and observability, not just one layer in isolation.

It had to be able to evolve credibly. Not stay a weekend prototype, but plausibly grow into something closer to a real, cloud-native platform.

And it had to be timely without depending on being timely. If someone reads this series in three years, the architectural questions should still be worth asking, even if the specific technology has moved on.

Most candidate projects fail at least one of these tests.

Some fail several.

---

## The Tension I Won't Pretend Isn't There

Here's the honest problem with everything above.

The manifesto behind this series argues that architecture isn't planned in advance. It emerges through engineering. Good architecture is discovered, not designed on a whiteboard before anyone writes code.

And yet this entire article is a whiteboard.

Criteria written down first.

A candidate checked against them second.

That's planning, not emergence, and it would be dishonest to pretend otherwise.

I don't think that's a contradiction, but it is a boundary worth being precise about.

Choosing *what to build* is an upfront decision by necessity. Someone has to start somewhere.

What can't be planned in advance is *how the system should be built*, and what it should become once it meets real constraints, real failures, and real usage.

This article is about the first kind of decision.

Almost everything else in this series will be about the second.

There's also a more uncomfortable version of this problem worth naming.

Criteria written *after* you already want to build something are very good at justifying that thing.

I wrote the list above before I had fully committed to RAG, not after, but I can't prove that to you, and you shouldn't take it on faith.

Judge it the same way the manifesto asks you to judge everything else in this series:

By what the engineering actually leaves behind.

Not by what the reasoning claimed in advance.

---

## Why RAG Fits

So does RAG actually pass the test?

I think it does.

Not because it's fashionable.

Not because it's AI.

Because it surfaces a surprisingly rich set of architectural problems in a single, buildable system.

What surprised me was that the challenges weren't fundamentally different from the ones I've seen in integration and distributed systems.

The technologies were different.

The architectural questions were not.

Ownership.

Boundaries.

Observability.

Evolution.

Trade-offs.

RAG simply puts all of them in one place.

**Knowledge representation.** Before anything else, you have to decide how knowledge is stored and chunked, and what happens when the source material changes. This is a real, and genuinely uncomfortable, trade-off: smaller chunks give more precise retrieval but fragment context; larger chunks preserve context but dilute relevance and cost more per query. There is no neutral default here, only a choice with consequences that often appear much later, when changing direction becomes expensive.

**Context construction.** Retrieval, ranking, and prompt assembly all sit between the data and the answer. Get this wrong and everything downstream inherits the mistake. This is an integration problem wearing an AI costume. It's the same "who is responsible when the wrong thing gets passed downstream" question that appears in message contracts, APIs, and distributed systems.

**Infrastructure evolution.** A vector store that works on a laptop is not the same decision as a vector store that has to work in production, at scale, under a budget.

The prototype and the platform are not the same thing.

The gap between them is where architecture starts to matter.

**Observability.** When a traditional service returns the wrong answer, you can usually trace why. When a RAG system returns a wrong answer that still sounds confident, tracing why is a genuinely hard, still-evolving problem. It raises the same ownership question as the story in Part 0: when the answer is wrong, whose fault is it—the retrieval, the source data, or the model? Someone still has to own that boundary, even when the failure is fuzzier than a rejected message.

**The local-to-cloud path.** A RAG prototype is trivial to run on a single machine. Turning it into something reliable, observable, secure, and cloud-native is a much longer, much more interesting road.

And that road is architecture.

None of those problems are unique to RAG.

They appear in integration platforms.

Distributed systems.

Data platforms.

Cloud-native applications.

RAG simply surfaces all of them in a single, buildable project.

That's the actual reason I chose it. Not the trend. Not the hype. The fit.

---

## Closing

Choosing RAG was itself the first architectural decision of this series.

And like the checklist it came from, it's a decision made before any of its consequences exist.

So here's a claim I'm willing to be held to.

If a year from now this project hasn't forced me to reverse at least one decision I was confident about today, then I picked the wrong reference implementation.

And I'll say so in this series, not quietly move on to something else.

That's the real test.

Not whether the reasoning in this article sounds convincing.

Not whether the criteria seem reasonable on paper.

But whether the engineering eventually proves them right.

The next post starts where every real project starts.

With the shape of the system before a single line of production code exists.

What the first version deliberately won't do.

Which decisions are cheap to reverse.

Which ones aren't.

Not all at once. One engineering decision at a time.
