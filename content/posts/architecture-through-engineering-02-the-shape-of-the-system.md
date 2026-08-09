---
title: "The Shape of the System"
date: 2026-08-09
draft: true
tags: ["architecture", "engineering", "architecture through engineering"]
---

# The Shape of the System

> *Part 2 of the Architecture Through Engineering series.*

## Central Question

What are we actually building?

## Why This Article Exists

Part 1 explained why RAG was chosen as the reference implementation. Before any code decisions get examined, the series needs a shared mental model of the system: what it is responsible for, what sits at its edges, and what it deliberately leaves out.

## The Story

- The temptation to start with a technology choice instead of a system boundary.
- What the system is responsible for, in plain language, before any layer names exist.
- What deliberately does NOT belong inside this system's boundary.
- A first mental model, checked against whatever exists in the repository so far — not invented ahead of it.
- Which parts of that model are guesses that engineering will later confirm or break.

## Key Ideas

- Shape comes before layers.
- Boundaries are a decision, not a diagram.
- A mental model is allowed to be wrong; it just has to be written down so we can tell later.

## Connection to the Series

Follows Part 1's choice of RAG as the vehicle. Sets up Part 3, which looks at the first concrete structural decision (dependency direction between layers) once the system's shape is established here.

## Open Questions

- How much of this "shape" is already implied by code that exists, versus aspiration?
- Where is the line between this system and the things it depends on?

## Article Body

[TODO]
