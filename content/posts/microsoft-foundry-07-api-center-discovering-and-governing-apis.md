---
title: "Microsoft Foundry and API Center: Discovering and Governing APIs"
description: "How a Foundry application can discover the right enterprise APIs inside a governed catalogue."
date: 2026-11-24
draft: true
categories:
  - Foundry
series:
  - Microsoft Foundry
tags: ["microsoft foundry", "architecture", "AI", "API Center", "governance"]
---

# Microsoft Foundry and API Center: Discovering and Governing APIs

> *Part 7 of the Microsoft Foundry series.*

Placeholder. Replace each section before setting `draft: false`. Planned publish: Tuesday, 24 November 2026.

## Opening

**How can an AI application discover the right enterprise APIs without creating another uncontrolled integration surface?**

- The practical problem or question
- Why it matters to an engineer or architect
- What will be built, tested, or investigated
- What the reader will learn

## Context

Discovery and consumption of enterprise APIs inside a governed API ecosystem.

- Relevant Microsoft Foundry and API Center capabilities
- The discovery scenario
- Important assumptions
- What is already known from documentation

## Architecture

- Agent, catalogue, and API owner
- Discovery versus invocation
- Ownership and lifecycle
- What governance is allowed to refuse

## Build

- Introduce API Center
- Register or discover APIs
- Connect the catalogue to the AI scenario
- Decisions about what "discoverable" means

## Experiment

Test discovery against a governed catalogue, including an API the agent should not treat as available.

**Expected**

**Observed**

**Conclusion**

## Trade-offs

- What a catalogue gives the agent
- What it does not guarantee at runtime
- Lifecycle and ownership complexity
- Assumptions about catalogue freshness
- Alternatives, including a fixed integration list

## When NOT to Use It

Where the set of APIs is small, stable, and already owned by the application.

## Practical Takeaway

- What the reader should remember
- What an architect should require before an API is discoverable by an agent
- What the discovery test showed

## Next Article

> In the next article, we move from **governed discovery** to **failure**, and examine **what happens when the agent, model, tool, API, or another dependency fails**.
