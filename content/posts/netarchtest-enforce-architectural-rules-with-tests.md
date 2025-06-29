---
title: "NetArchTest for .NET: Enforce Architectural Rules with Tests"
date: 2025-06-28
draft: false
tags: ["netarchtest", "architecture", "clean architecture", "testing", "dotnet", "architecture-as-code", "governance"]
---

# NetArchTest for .NET: Enforce Architectural Rules with Tests

You know that feeling when you're working on a project and someone on the team accidentally adds a dependency that breaks your carefully planned architecture? Controllers start calling repositories directly, domain logic creeps into the API layer, and before you know it, your Clean Architecture isn't so clean anymore.

I've been there. You spend time in code reviews catching these violations, but it's reactive and inconsistent. You document the rules in README files, but let's be honest - who reads those when they're in the middle of implementing a feature?

There had to be a better way to enforce architectural decisions. That's when I discovered **Architecture as Code** - the practice of defining and enforcing architectural rules through executable code rather than just documentation.

## What is Architecture as Code?

Architecture as Code treats architectural decisions as **living, testable policies**. Instead of static diagrams and documentation that get outdated, you write executable rules that validate your architecture automatically.

The core idea is simple: if architectural rules are important enough to document, they're important enough to test.

But there are many ways to implement Architecture as Code - fitness functions, custom analyzers, static analysis tools. So why did I choose to focus on **NetArchTest**?

## Why NetArchTest?

After exploring different approaches, NetArchTest stood out for several reasons:

1. **Familiar Testing Framework** - Uses xUnit/NUnit, so it fits naturally into existing test suites
2. **Clean Architecture Focus** - Specifically designed for .NET layered architectures
3. **Immediate Feedback** - Fails fast in CI/CD when violations occur
4. **Easy to Start** - No complex setup, just NuGet packages and test methods

Here's the architecture I wanted to enforce:

```
┌─────────────────────────────────────────┐
│                   API                   │  ← Controllers, Endpoints
├─────────────────────────────────────────┤
│              Infrastructure             │  ← Business Services
├─────────────────────────────────────────┤
│                  Data                   │  ← Repositories, Persistence
├─────────────────────────────────────────┤
│                 Domain                  │  ← Entities, Business Rules
└─────────────────────────────────────────┘
```

## The Implementation

I built a comprehensive sample demonstrating Clean Architecture with a Book management API. The key insight: instead of hoping developers follow architectural rules, write tests that automatically enforce them.

### 1. Writing Architecture Tests

Here's where NetArchTest shines - you write normal unit tests that validate architectural boundaries:

```csharp
[Fact]
public void Domain_Should_Not_Depend_On_Infrastructure()
{
    var result = Types.InAssembly(typeof(Book).Assembly)
        .That().ResideInNamespace("NetArchTestSample.Domain")
        .ShouldNot().HaveDependencyOn("NetArchTestSample.Infrastructure")
        .GetResult();
    
    Assert.True(result.IsSuccessful);
}
```

### 2. Custom Rules for Your Domain

Beyond basic dependency rules, you can create custom validation for specific architectural decisions - like ensuring domain entities maintain proper encapsulation or that service interfaces are async-only.

### 3. Fast Feedback

When someone breaks a rule, the test fails immediately with a clear message showing exactly which classes violated the architecture. No more catching violations in code reviews or production.

## What I Learned

This journey taught me several important lessons:

1. **Prevention is Better Than Detection** - Catching violations in tests is much better than in code reviews
2. **Tests Document Intent** - The architecture tests serve as executable documentation
3. **Fast Feedback Matters** - Developers get immediate feedback when they violate rules
4. **Custom Rules Add Value** - Generic dependency rules are just the start; domain-specific rules provide real governance

The result? Architecture tests that automatically validate:
- Layer dependency constraints
- Naming conventions  
- Encapsulation rules
- Class design constraints
- Async method requirements

## Beyond the Basics

Once you have basic dependency rules working, you can extend the approach:

- **Performance constraints** - Test cyclomatic complexity limits
- **Security rules** - Ensure sensitive data handling patterns
- **Team standards** - Enforce coding conventions automatically

## The Code

You can see the complete implementation in my architecture playbook: [NetArchTest Sample](https://github.com/morteza-azizi/architecture-playbook/tree/main/architecture-as-code/netarchtest-sample)

The sample demonstrates how to move from "hoping developers follow the rules" to **automatically enforcing them**. No more architectural drift, no more surprise violations in production.

Architecture as Code isn't just about NetArchTest - it's about making architectural decisions executable, testable, and enforceable. But NetArchTest provides an excellent starting point for .NET developers wanting to implement these practices.