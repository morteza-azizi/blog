---
title: "Building an MCP Server for Microsoft Foundry"
description: "How an MCP server gives a Foundry agent external capabilities without tightly coupled tool integrations."
date: 2026-10-20
draft: true
categories:
  - Foundry
series:
  - Microsoft Foundry
tags: ["microsoft foundry", "architecture", "AI", "MCP", "agents"]
---

# Building an MCP Server for Microsoft Foundry

> *Part 2 of the Microsoft Foundry series.*

Placeholder. Replace each section before setting `draft: false`. Planned publish: Tuesday, 20 October 2026.

## Opening

**How do we connect an agent to capabilities outside the model without turning the application into a collection of tightly coupled tool integrations?**

- The practical problem or question
- Why it matters to an engineer or architect
- What will be built, tested, or investigated
- What the reader will learn

## Context

MCP as a mechanism for giving an AI application structured access to external capabilities.

- Relevant Microsoft Foundry capability
- The scenario
- Important assumptions
- What is already known from documentation

## Architecture

- The MCP server, the agent, and the capability being exposed
- Responsibilities on each side of the boundary
- What the agent is allowed to discover and invoke
- Security and operational boundaries

## Build

- Build a small MCP server
- Expose one useful capability
- Connect it to the Foundry agent
- Decisions made at the integration point

## Experiment

Test tool discovery and invocation.

**Expected**

**Observed**

**Conclusion**

## Trade-offs

- What MCP clarifies
- What it does not remove
- Operational and security cost
- Assumptions about the capability behind the server
- Alternatives, including a direct tool integration

## When NOT to Use It

Where a dedicated MCP server adds a boundary the capability does not need.

## Practical Takeaway

- What the reader should remember
- What an architect should consider before connecting real enterprise systems
- What discovery and invocation showed

## Next Article

> In the next article, we move from **a controlled MCP experiment** to **enterprise APIs**, and examine **how an agent should meet an existing API without bypassing the architecture that protects it**.
