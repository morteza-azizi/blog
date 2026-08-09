---
title: "Abstractions Before Providers"
date: 2026-08-09
draft: true
tags: ["architecture", "engineering", "architecture through engineering"]
---

# Abstractions Before Providers

> *Part 9 of the Architecture Through Engineering series.*

## Central Question

Why separate application responsibilities from technology providers?

## Why This Article Exists

The application layer talks about embeddings, vector storage, chat completion, parsing, and document storage in terms of interfaces, not products. This article questions whether that was worth it instead of assuming abstraction is automatically good architecture.

## The Story

- Which abstractions exist and what each one is actually shielding the system from.
- The cost side of the ledger: indirection, extra code, complexity that has to be maintained.
- Challenging each abstraction individually — is it earning its place, or is it speculative?
- What it would look like if one of these abstractions turned out to be unnecessary.

## Key Ideas

- An interface is a bet that something behind it will change. Bets can be wrong.
- Abstraction has a carrying cost, not just a payoff.
- This article should name at least one abstraction that might not be worth it yet.

## Connection to the Series

Follows Part 8's local-infrastructure choice — these abstractions are what made deferring the provider decision possible. Sets up Part 10, where the abstractions face their first real test against Azure.

## Open Questions

- Which of these abstractions have actually been swapped or tested against a second implementation?
- Which ones exist only because "that's how you're supposed to do it"?

## Article Body

[TODO]
