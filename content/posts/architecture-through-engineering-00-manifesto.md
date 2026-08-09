---
title: "Architecture Is Easy Until You Have to Build It"
date: 2026-07-28
draft: false
tags: ["software architecture", "engineering", "distributed systems", "architecture through engineering", "manifesto"]
---

# Architecture Is Easy Until You Have to Build It

> *Part 0 of the Architecture Through Engineering series — the manifesto behind it.*

---

## Introduction

There is no shortage of articles about software architecture.

Open any technical blog, attend a conference, or browse LinkedIn, and you'll find countless diagrams, principles, frameworks, and opinions about what good architecture looks like. Most of them are written the same way: a clean model, a confident conclusion, and very little of the mess that came before it.

I've read many of them.

I've learned from many of them.

But over the years, I've come to realize that architecture isn't something you truly understand by reading about it.

You understand it by building software.

---

## The Beginning

When I started my career, I wasn't thinking about software architecture.

I was thinking about making the code work.

Like many software engineers, my world revolved around implementing features, fixing bugs, learning new technologies, and becoming a better developer every day.

To be honest, I didn't even know what software architecture really meant.

It wasn't something I was looking for.

It found me gradually.

As the systems I worked on became larger and more connected, spanning more integrations, more teams and more moving parts, the questions I faced began to change.

The challenge was no longer simply implementing another feature.

It became understanding how an entire system should evolve.

---

## What Engineering Taught Me

Somewhere along that journey, I noticed something interesting.

The most valuable lessons I learned about architecture never came from a diagram.

They came from engineering.

A deployment that didn't go as planned.

An integration that became difficult to maintain.

A dependency that looked harmless at first but slowly spread throughout the entire solution.

A shortcut that solved today's problem while creating tomorrow's technical debt.

One story stays with me more than the others.

We were responsible for an integration platform that received messages from another system and passed them downstream.

One day, messages started arriving that simply couldn't be processed.

They were missing required data, or they violated the agreed contract.

The obvious response was to retry.

Retrying didn't help.

The message wasn't temporarily unavailable.

It was simply wrong.

The harder problem wasn't technical. It was ownership.

We couldn't correct the data, because we didn't own it.

We couldn't silently drop it, because downstream systems depended on it.

And we couldn't keep retrying forever. The same invalid message would keep generating exceptions, bury real problems in the noise, and delay everything behind it.

Eventually, we stopped treating these messages as technical failures and started treating them as business exceptions.

Invalid messages were routed into a dedicated workflow: the source system was notified, the problem was explained, and a corrected message was resent.

The platform became quieter.

Operations got simpler.

And the responsibility for fixing the data stayed exactly where it belonged — with the team that owned it.

Those experiences taught me something that no certification, book, or conference session could.

Architecture is not a document.

It is the accumulation of thousands of engineering decisions, each leaving a mark on the system that follows.

Some good.

Some bad.

All educational.

---

## A Change in Perspective

As my career continued, I moved from software engineering into cloud architecture, distributed systems, integration architecture, and Azure. Along the way, a lot of that work centered on messaging systems, service integration, and the infrastructure that keeps distributed systems reliable when things fail at 3 a.m.

The technologies changed.

The scale changed.

The responsibilities changed.

But one thing remained surprisingly constant.

The best architectural decisions were rarely made in front of a whiteboard.

They emerged while building, operating, debugging, and improving real systems. I never stopped writing code, and I don't think I want to. Staying close to implementation is what keeps architectural thinking honest — it's much harder to design around a constraint you've never personally hit.

Good architecture wasn't created first and implemented later.

Good architecture evolved through engineering.

---

## Why This Series Exists

Today, our industry is experiencing another major transformation: AI, large language models, retrieval-augmented generation, agentic systems, and a new cloud-native platform seemingly every week.

The possibilities are exciting. But I've seen this pattern before. New technology arrives, the terminology changes, the diagrams get redrawn — and the underlying engineering questions stay exactly the same.

How do we design systems that can evolve?

How do we keep complexity under control?

How do we make good architectural decisions?

How do we build software that remains maintainable years after the first deployment?

Those are not AI questions.

They are engineering questions, and they were true before AI and will still be true after whatever comes next.

Technologies change. Engineering principles survive.

That is the lens I want to apply to this new wave, not the other way around.

---

## Architecture Through Engineering

That realization inspired this series.

**Architecture Through Engineering** is my attempt to explore software architecture the same way I have learned it throughout my career.

By building.

Not by creating perfect diagrams before writing code.

Not by presenting theoretical architectures that never face production.

But by designing, implementing, documenting, reviewing, and continuously improving real systems.

This series is not about perfect architectures, theoretical diagrams, or framework comparisons. It is about learning architecture by building systems, documenting the engineering decisions behind them, understanding the trade-offs, and watching the design evolve as the system does.

Every repository starts small.

Every abstraction has to earn its place.

Every architectural decision should have a reason.

Every important trade-off deserves to be documented.

Some decisions will prove to be right.

Others will prove to be wrong.

Both are valuable.

---

## The Journey Begins

The first project in this series explores a production-oriented Retrieval-Augmented Generation (RAG) application. But RAG itself is not really the subject. It's the vehicle.

Not because RAG is the latest trend.

Not because I want to build another chatbot.

But because it raises fascinating architectural questions, and it happens to be a good, concrete system to build those questions around.

How should knowledge be represented?

How should context be constructed?

How should infrastructure evolve?

How should systems remain observable?

How do we move from a local prototype to a cloud-native platform without rewriting everything?

Those are the questions I want to answer, using RAG as the first case study. The series won't stay there. Whatever system comes next will simply be the next vehicle for the same underlying questions.

Not all at once.

But one engineering decision at a time.

---

## Looking Forward

This series isn't about finding the perfect architecture.

I don't believe such a thing exists.

Every architecture is a collection of trade-offs shaped by business requirements, technical constraints, and experience.

My goal is simply to make those trade-offs visible.

To explain not only **what** I built, but **why** I built it that way.

If these articles encourage another engineer to question an assumption, reconsider a design decision, or think differently about software architecture, then this series has achieved its purpose.

---

## Final Thoughts

After nearly two decades of building software, I've come to believe one simple idea.

Architecture is not what happens before engineering.

Architecture is what engineering leaves behind — in the decisions we're willing to explain, the trade-offs we're willing to admit, and the systems that are still standing, and still understandable, long after the diagram that started them has been forgotten.

That is why this series exists.

Welcome to **Architecture Through Engineering**.
