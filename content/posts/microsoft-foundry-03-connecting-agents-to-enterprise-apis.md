---
title: "Connecting Foundry Agents to Enterprise APIs"
description: "How a Foundry agent should call enterprise APIs without bypassing the architecture that protects them."
date: 2026-10-27
draft: true
categories:
  - Foundry
series:
  - Microsoft Foundry
tags: ["microsoft foundry", "architecture", "AI", "APIs", "enterprise integration"]
---

# Connecting Foundry Agents to Enterprise APIs

> *Part 3 of the Microsoft Foundry series.*

Placeholder. Replace each section before setting `draft: false`. Planned publish: Tuesday, 27 October 2026.

## Opening

**How should an AI agent interact with existing enterprise APIs without bypassing the architecture that protects those systems?**

- The practical problem or question
- Why it matters to an engineer or architect
- What will be built, tested, or investigated
- What the reader will learn

## Context

Move from a controlled MCP experiment toward enterprise systems and APIs.

- Relevant Microsoft Foundry capability
- The enterprise API scenario
- Important assumptions
- What is already known from documentation

## Architecture

- Agent, mediation point, and enterprise API
- Authentication and authorization boundaries
- Who owns the contract, failures, and retries
- Direct access compared with mediated access

## Build

- Introduce the enterprise API scenario
- Connect the agent to the API
- Important configuration at the trust boundary
- Decisions about what the agent may call

## Experiment

Test contract behaviour and failure behaviour.

**Expected**

**Observed**

**Conclusion**

## Trade-offs

- What direct access makes easy
- What mediated access protects
- What becomes harder to operate
- Assumptions about the API contract
- Alternatives considered

## When NOT to Use It

Where the agent should not be a client of the enterprise API at all.

## Practical Takeaway

- What the reader should remember
- What an architect should keep on the enterprise side of the boundary
- What the failure test showed

## Next Article

> In the next article, we move from **deterministic APIs** to **knowledge retrieval**, and examine **what the retrieval architecture should look like when the application needs enterprise knowledge**.
