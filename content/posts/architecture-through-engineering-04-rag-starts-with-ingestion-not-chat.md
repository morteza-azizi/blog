---
title: "RAG Starts With Ingestion, Not Chat"
date: 2026-08-09
draft: true
tags: ["architecture", "engineering", "architecture through engineering"]
---

# RAG Starts With Ingestion, Not Chat

> *Part 4 of the Architecture Through Engineering series.*

## Central Question

How does knowledge enter the system?

## Why This Article Exists

Most RAG content jumps straight to chat. But nothing can be retrieved that wasn't ingested correctly first. This article treats ingestion as its own architectural responsibility, not a preliminary step on the way to the "interesting" part.

## The Story

- Why ingestion, not chat, is the real starting point of a RAG system.
- What happens to a document from arrival to being usable knowledge.
- Where validation and format decisions live, and why that placement matters.
- What failure looks like during ingestion, and who is responsible for it.

## Key Ideas

- Ingestion is a first-class architectural concern, not plumbing.
- Garbage in the door is a boundary problem, not a chat problem.
- The chat experience can only be as trustworthy as the ingestion path behind it.

## Connection to the Series

Follows Part 3's layering decision — ingestion is the first feature that has to live inside it. Sets up Part 5, which looks at what a "document" actually means once it's inside the system.

## Open Questions

- What does the current ingestion path actually handle well versus poorly?
- What formats or edge cases has ingestion not been tested against yet?

## Article Body

[TODO]
