---
title: "Designing Reliable AI Agents: Failure Modes and Resilience"
description: "What happens when a Foundry agent, its model, a tool, an API, or another dependency fails."
date: 2026-12-01
draft: true
categories:
  - Foundry
series:
  - Microsoft Foundry
tags: ["microsoft foundry", "architecture", "AI", "reliability"]
---

# Designing Reliable AI Agents: Failure Modes and Resilience

> *Part 8 of the Microsoft Foundry series.*

Placeholder. Replace each section before setting `draft: false`. Planned publish: Tuesday, 1 December 2026.

## Opening

**What happens when an AI agent, model, tool, API, or external dependency fails?**

- The practical problem or question
- Why it matters to an engineer or architect
- What will be built, tested, or investigated
- What the reader will learn

## Context

Move from capability to reliability.

- Relevant Microsoft Foundry capability
- The failure scenario
- Important assumptions
- What is already known from documentation

## Architecture

- Failure points across agent, model, tool, API, and dependency
- Timeouts, retries, and partial failure
- Fallback ownership
- Observability
- Deterministic failure handling compared with probabilistic model behaviour

## Build

- Identify the failure points in the running system
- Implement the handling under test: retry, timeout, or fallback
- Make the failure observable

## Experiment

Test a failed tool or dependency.

**Expected**

**Observed**

**Conclusion**

## Trade-offs

- What retries recover
- What they hide or amplify
- Complexity of fallback
- Assumptions about idempotency and partial success
- Alternatives considered

## When NOT to Use It

Where another retry is the wrong response, and the system should stop.

## Practical Takeaway

- What the reader should remember
- What an architect should decide before the first production failure
- What the induced failure showed

## Next Article

> In the next article, we move from **failure inside the agent path** to **the event-driven enterprise**, and examine **how a Foundry workload should participate through Azure Service Bus**.
