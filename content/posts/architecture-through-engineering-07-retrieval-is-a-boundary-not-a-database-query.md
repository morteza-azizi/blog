---
title: "Retrieval Is a Boundary, Not a Database Query"
date: 2026-08-09
draft: true
tags: ["architecture", "engineering", "architecture through engineering"]
---

# Retrieval Is a Boundary, Not a Database Query

> *Part 7 of the Architecture Through Engineering series.*

## Central Question

What does retrieval actually mean inside this system?

## Why This Article Exists

It's tempting to describe retrieval as "query the vector store." This article argues retrieval is really an architectural boundary — a contract between the domain and whatever storage technology sits behind it — and treats it accordingly instead of writing a vector database tutorial.

## The Story

- Why retrieval deserves to be modeled as a responsibility, not a query.
- What crosses the boundary in each direction, and what stays behind it.
- How this boundary is meant to keep the rest of the system stable if the storage technology changes.
- What's still unproven about that assumption.

## Key Ideas

- Retrieval is a contract, not a technology.
- The boundary's value is only real if it actually gets exercised by a change later.
- This is not a vector database comparison article.

## Connection to the Series

Follows from Part 6's chunking discussion — chunks are what gets retrieved. Sets up Part 8, which looks at the infrastructure choice made to support this boundary early on.

## Open Questions

- Has this boundary been tested by an actual infrastructure change yet, or only in theory?
- What would break this abstraction if pushed hard enough?

## Article Body

[TODO]
