---
title: "The First Architectural Decision Wasn't About AI"
date: 2026-08-09
draft: true
tags: ["architecture", "engineering", "architecture through engineering"]
---

# The First Architectural Decision Wasn't About AI

> *Part 3 of the Architecture Through Engineering series.*

## Central Question

What depends on what?

## Why This Article Exists

Before anything about embeddings, retrieval, or chat, the first real decision in this project was structural: how the layers relate to each other and which direction dependencies point. That decision has nothing to do with AI, and everything to do with whether the rest of the series is even possible to write.

## The Story

- Layering the project before writing any RAG-specific logic.
- Why infrastructure was pushed outside the core instead of the other way around.
- What Clean Architecture actually bought here, versus what it cost.
- Challenging the decision instead of assuming it was automatically correct.

## Key Ideas

- Dependency direction is an architectural decision, not a folder-naming exercise.
- "Clean Architecture" is a label, not a justification — it has to earn its place like anything else.
- The AI part of this project depends on this decision; this decision does not depend on AI.

## Connection to the Series

Builds directly on the system shape from Part 2. Sets up Part 4, which walks through the first real feature (ingestion) built on top of this layering.

## Open Questions

- Is the layering paying for itself yet, or is it still speculative?
- What would make this decision expensive to reverse later?

## Article Body

[TODO]
