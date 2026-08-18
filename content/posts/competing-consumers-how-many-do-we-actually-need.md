---
title: "How Many Consumers Do We Actually Need?"
date: 2026-08-18
draft: false
tags: ["azure", "service bus", "messaging", "competing consumers", "architecture", "distributed systems"]
---

# How Many Consumers Do We Actually Need?

> Exploring the Competing Consumers pattern with Azure Service Bus

## The problem

Imagine a queue of notifications waiting to be processed.

A producer keeps adding messages, while one consumer processes them one at a time.

```text
Producer
   |
   v
┌───────────────┐
│ notifications │
└───────┬───────┘
        |
        v
    Consumer
```

If messages arrive faster than processing them, the queue starts to grow.

One obvious question is:

> **Can we simply add more consumers?**

For example:

```text
                 ┌── Consumer 1
                 │
Producer → Queue ├── Consumer 2
                 │
                 └── Consumer 3
```

This is the **Competing Consumers** pattern.

It sounds simple. And the basic idea is simple. But once we start running it, a few interesting questions appear:

- How are messages actually distributed?
- Does adding consumers always make things faster?
- What happens if one consumer is slower?
- What happens if a consumer disappears while processing a message?
- What happens to ordering?
- When does adding more consumers stop helping?

Those are the questions I want to explore.

---

## The pattern is not new

Let's get this out of the way first.

Competing Consumers is not a new idea.

It is described in [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/patterns/messaging/CompetingConsumers.html), and Microsoft has documented the pattern in the [Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/patterns/competing-consumers).

Both are excellent references for understanding the pattern itself.

So I'm not trying to explain something new here.

**Instead, I want to take the concept and make it tangible.**

I'll build a small .NET implementation using **Azure Service Bus**, deploy the infrastructure with **Bicep**, run multiple consumers, and see how the pattern actually behaves.

> **Build it, run it, change it, and see what actually happens.**

Reading about a distributed-systems pattern and watching it behave are two different things.

---

## A deliberately small implementation

I wanted the implementation to be boring.

No messaging framework.

No benchmark platform.

No metrics infrastructure.

No extra services.

Just:

```text
Producer
   |
   v
Azure Service Bus Queue
   |
   +── Consumer
   +── Consumer
   +── Consumer
```

There is only **one Consumer application**.

The interesting part is that I can start the same application multiple times.

Each process connects to the same `notifications` queue.

So I don't need:

```text
ConsumerA
ConsumerB
ConsumerC
```

I simply run:

```text
Consumer
Consumer
Consumer
```

The code stays the same. The only thing that changes is how many instances are running against the same queue.

---

## The Producer

The producer does very little.

It connects to Azure Service Bus, creates a sender, and sends a number of notification messages to the queue.

The message itself is intentionally simple:

```text
notification-1
notification-2
notification-3
...
```

There is no business domain hiding the idea.

The goal is to make the message flow obvious.

---

## The Consumer

The consumer is equally small.

It creates a `ServiceBusProcessor` for the queue and registers a message handler.

Conceptually:

```text
Receive message
      ↓
Process message
      ↓
Complete message
```

For the experiment with the downstream API, processing becomes:

```text
Receive message
      ↓
Call downstream API
      ↓
Complete message
```

The consumer also prints its process ID so we can see which instance is handling each message.

For example:

```text
consumer-1234 processed notification-1
consumer-5678 processed notification-2
```

This becomes particularly useful when we start multiple instances of the same application.

The consumer itself doesn't need to know whether it is the first, second, or tenth instance.

Every instance simply connects to the same queue.

---

## Now add another consumer

Start the same Consumer application again.

We now have:

```text
                 ┌── Consumer 1
                 │
Producer → Queue ├── Consumer 2
                 │
                 └──
```

Send another batch of messages.

Now something interesting happens.

The two consumers are reading from the **same queue**.

A message is not processed by both consumers simply because both are listening.

Instead, they compete for available messages.

You might see:

```text
Consumer 1 → notification-1
Consumer 2 → notification-2
Consumer 1 → notification-3
Consumer 2 → notification-4
```

But don't expect a perfect alternation.

That's actually one of the things we want to observe.

The important observation is not:

> "Consumer 1 gets odd numbers and Consumer 2 gets even numbers."

It doesn't work like that.

The important observation is:

> **Multiple instances of the same consumer can share the work from a single queue.**

And that is the core idea behind Competing Consumers.

---

## Why a queue?

This distinction matters.

We're using:

```text
Producer → Queue → Consumers
```

not:

```text
Producer → Topic → Subscriptions
```

With a queue, the consumers are competing for the same work.

If I need multiple independent subscribers to receive their own copy of a message, that's a different messaging problem.

This is one reason I like building the example instead of only looking at the pattern diagram: the topology makes the intended behaviour much easier to understand.

---

## Now let's experiment

At this point, the basic implementation works.

Instead of adding more theory, I wanted to change one thing at a time and see what actually happens.

---

## Experiment 1 — One consumer

Before adding anything else, start with one consumer.

```text
Producer
   ↓
Queue
   ↓
Consumer
```

Send twenty messages.

The consumer processes them one by one.

Nothing surprising yet.

All twenty messages were processed by the single consumer.

That gave us the baseline for everything that followed.

---

## Experiment 2 — Two consumers

Next, I started the same Consumer application twice.

Both processes connected to the same queue.

In one run, the distribution looked almost perfectly alternating:

```text
Consumer 1 → 1
Consumer 2 → 2
Consumer 1 → 3
Consumer 2 → 4
...
```

It was tempting to think:

> *"Okay, it is round-robin."*

But that conclusion would be too strong.

The important thing is that we should not assume a particular distribution pattern between consumers.

We might see messages appear to alternate between consumers in one run, but that does not mean the system guarantees an odd/even or round-robin distribution.

The important observation is:

> **The consumers compete for available work.**

The application should depend on the fact that multiple consumers can process messages from the same queue, not on exactly how those messages are distributed between them.

---

## Experiment 3 — Make one consumer slow

Next, I changed the processing speed.

One consumer used approximately:

```text
200 ms
```

while the other used:

```text
2000 ms
```

The workload stayed the same.

The result was interesting.

The faster consumer processed substantially more messages than the slower consumer.

In our run:

```text
Fast consumer → 17 messages
Slow consumer → 3 messages
```

This is a different kind of unevenness from Experiment 2.

Experiment 2 warned us not to assume round-robin just because one run looked neat.

Here, processing speed clearly influenced the distribution.

The faster consumer kept taking work while the slower one was still busy.

The slow consumer did not stop the fast consumer from continuing to process available work.

This is a useful mental model for competing consumers.

It isn't:

```text
20 messages
÷
2 consumers
=
10 messages each
```

Instead, consumers compete for work as they become available to process it.

A faster consumer can therefore end up processing more messages than a slower one.

---

## Experiment 4 — Stop a consumer

This was probably the most interesting experiment.

I started two consumers.

One of them was processing messages with a longer delay.

Then I stopped that consumer while it was processing a message.

Something important happened.

After starting another consumer, a message that had already been received by the first consumer appeared again.

For example:

```text
Consumer 1
    ↓
notification-2
    ↓
Consumer disappears
    ↓
notification-2
    ↓
Consumer 2
```

The same message was delivered again.

This demonstrates an important distinction:

> **Receiving a message is not the same thing as completing the work.**

Our consumer only completes the Service Bus message after the processing has succeeded.

If the consumer disappears before that happens, the message can become available for another delivery.

And suddenly we have a very real distributed-systems problem:

> **The same message can be processed more than once.**

This isn't something we needed to invent with a complicated failure simulation.

We simply stopped the process.

And we saw it happen.

---

## A small but important consequence

This experiment changes how I think about the consumer.

It is tempting to imagine:

```text
Receive message
      ↓
Message is now done
```

But that's not the right model.

A better model is:

```text
Receive
   ↓
Do the work
   ↓
Successfully complete the message
```

There is a period between receiving and completing the message where something can go wrong.

That means downstream operations need to be designed with failure and possible duplicate delivery in mind.

And that leads naturally to another pattern:

**Idempotency.**

We'll come back to that.

---

## A pattern gives you a mechanism, not an architecture

This is probably the biggest lesson I take from this exercise.

And it became hard to ignore once a consumer disappeared mid-processing.

Competing Consumers gives us a mechanism for processing independent asynchronous work concurrently.

It does **not** tell us:

- how many consumers we should run
- how much concurrency is appropriate
- whether ordering matters
- how to handle duplicate processing
- how to protect a downstream dependency
- what happens when a consumer fails

Those are architectural decisions.

The next experiments make that clearer.

---

## Experiment 5 — What about ordering?

Next, I wanted to look at ordering.

I sent messages in a clear sequence:

```text
1
2
3
4
5
6
7
8
9
10
...
```

and ran multiple consumers with the same processing delay.

The output looked very neat.

The consumers shared the messages in a symmetric way.

For example:

```text
Consumer 1 → 1, 3, 5, 7, ...
Consumer 2 → 2, 4, 6, 8, ...
```

At first glance, it looked like we had demonstrated ordering.

But there was an important problem with that conclusion.

Our logging showed when the consumer started processing the message.

It did **not** give us a precise measurement of when processing completed.

So I don't want to make a stronger claim than the experiment supports.

What we can say is:

> **The messages were distributed across consumers while retaining the sequence we observed in that particular run.**

But that is not the same as saying:

> "The application is guaranteed to process messages in order."

Once work is being processed concurrently, ordering becomes something we need to reason about explicitly.

And that is exactly the kind of distinction I wanted this exercise to expose.

**An observation is not automatically a guarantee.**

---

## Experiment 6 — Add a downstream API

Finally, I wanted to make the example a little closer to a real integration scenario.

I added a tiny downstream HTTP API.

The topology became:

```text
                  ┌── Consumer 1 ──┐
                  │                │
Producer → Queue ├── Consumer 2 ──┼──→ Downstream API
                  │                │
                  └── Consumer 3 ──┘
```

The API is deliberately simple.

It accepts the notification and can introduce an artificial processing delay.

There is no database.

No business logic.

No API gateway.

It exists for one reason:

> **To give us a controllable downstream dependency.**

And this changes the question.

Before, we were asking:

> "Can more consumers process more messages?"

Now we can ask:

> **"What happens when more consumers all call the same downstream dependency?"**

I have not yet run a meaningful saturation or concurrency experiment against that API.

So I don't want to pretend we already have evidence about where the bottleneck lands.

What we do have is a setup for the next experiment:

> **How much concurrency can the downstream system actually absorb?**

That is where adding more consumers stops being only a messaging question and becomes a system question.

Adding consumers doesn't only increase the amount of work happening inside the consumer layer.

It can also increase the number of concurrent calls made to the system behind it:

```text
More consumers
      ↓
More concurrent processing
      ↓
More concurrent downstream calls
      ↓
More pressure on the dependency
```

Whether that pressure becomes the limiting factor is exactly what still needs to be measured.

---

## What did we actually learn?

When I started this, the question seemed very simple:

> **Can I add more consumers when one consumer isn't enough?**

The answer is yes.

But running the example exposed several things that aren't obvious from the basic pattern diagram.

### Work isn't necessarily divided evenly

Two consumers don't mean:

```text
50%
50%
```

An almost perfectly alternating run can look like round-robin, but that is not a guarantee.

And when one consumer is slower, the split can become much more uneven — in our run, 17 to 3.

The application should therefore depend on the **ability to share work**, not on a particular distribution pattern.

### A slow consumer doesn't necessarily stop everyone else

When one consumer took much longer to process a message, the faster consumer continued processing other available work.

That's one of the main reasons the pattern is useful.

### A failed consumer can lead to another delivery

We stopped a consumer while it was processing a message.

The message appeared again later.

So duplicate processing isn't some exotic failure mode.

It's something the application needs to be prepared for.

### Parallel processing changes the ordering conversation

Once multiple consumers are processing work concurrently, we have to think carefully about what "ordering" actually means.

Receiving messages in a particular order is not automatically the same as completing business operations in that order.

### More consumers can put more pressure downstream

Adding consumers increases concurrency.

That can improve throughput.

But it can also put more pressure on the dependency that the consumers call.

So the question isn't simply:

> "How many consumers can I run?"

It becomes:

> **"How much concurrency can the entire system safely handle?"**

---

## So how many consumers do we actually need?

There isn't a number I can give you from the pattern alone.

The right number depends on the workload, processing time, concurrency, downstream capacity, and failure behaviour.

Competing Consumers tells us that work can be shared.

It does not tell us how much sharing is safe.

That is why a pattern gives you a mechanism, not an architecture.

---

## One question leads to another

We started with a simple problem:

> **One consumer isn't enough.**

Competing Consumers gives us a way to distribute the work across multiple consumer instances.

Then our experiments raised another question:

> **What happens when the same message is processed twice?**

That's where the next pattern becomes interesting:

**Idempotency.**

---

## References

- [Enterprise Integration Patterns — Competing Consumers](https://www.enterpriseintegrationpatterns.com/patterns/messaging/CompetingConsumers.html)
- [Microsoft Azure Architecture Center — Competing Consumers](https://learn.microsoft.com/en-us/azure/architecture/patterns/competing-consumers)

**Implementation:** [azure-integration-architecture/messaging/competing-consumers](https://github.com/morteza-azizi/azure-integration-architecture/tree/main/messaging/competing-consumers)
