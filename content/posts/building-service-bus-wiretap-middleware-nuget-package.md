---
title: "From Idea to NuGet Package: Building a Service Bus WireTap Middleware"
date: 2025-01-12
description: "The journey of building and publishing my first NuGet package - a Service Bus WireTap Middleware for Azure Functions"
tags: ["azure-functions", "service-bus", "middleware", "nuget", "dotnet", "messaging"]
---

# From Idea to NuGet Package: Building a Service Bus WireTap Middleware

## The Problem

It started with a simple need: **"How can I see every message that flows through my Azure Service Bus-triggered Functions, no matter how the function is written?"**

I was working on a project where we needed deep visibility into our message flows for debugging, monitoring, and compliance purposes. The challenge was that Azure Functions with Service Bus triggers can have different parameter types - POCOs, strings, or raw ServiceBusReceivedMessage objects. I wanted a solution that would work regardless of the function signature, without requiring changes to existing business logic.

## The Solution

I decided to implement the **Wire Tap pattern**, a well-established messaging design pattern described in "Enterprise Integration Patterns" by Gregor Hohpe and Bobby Woolf. This pattern allows for intercepting and duplicating messages as they flow through a system without affecting the original message flow.

The solution would be a middleware for Azure Functions (.NET 8+ Isolated Worker) that could:
- Intercept all Service Bus messages automatically
- Log complete message details (body, metadata, application properties)
- Work with any function parameter type
- Integrate seamlessly with existing logging infrastructure

### Understanding the Wire Tap Pattern

Before diving into the implementation, it's important to understand how the Wire Tap pattern works. I've written a detailed article about this pattern: [Understanding the Wire Tap Pattern in Messaging Systems](https://www.mortezaazizi.com/posts/understanding-wire-tap-pattern-in-messaging-systems/).

Here's a visual representation of how the Wire Tap pattern operates:

{{< mermaid >}}
flowchart LR
    A[Sender] --> B[Original Channel]
    B --> C[Receiver]
    B -. Wire Tap .-> D[Tap Channel]
    D --> E[Tap Receiver / Monitoring System]
{{< /mermaid >}}

The beauty of this pattern is that it keeps monitoring and auditing concerns separate from the main message route, resulting in cleaner architecture and better separation of concerns.

## The Implementation Journey

### Choosing the Right Architecture

I decided to build this as middleware using `IFunctionsWorkerMiddleware`, which intercepts messages at the Azure Functions pipeline level. This approach meant:
- No changes needed to existing functions
- Centralized logging logic
- Easy to configure and extend

### Key Technical Decisions

1. **Middleware vs. Attribute Approach**: I chose middleware because it's more flexible and doesn't require decorating every function.

2. **Message Body Handling**: Azure Service Bus treats all payloads as binary, so I had to ensure robust handling for both text and binary messages.

3. **Configuration Options**: I made the logging configurable - users can control log levels, body inclusion, and even add custom logging actions.

4. **Testing Strategy**: Given the challenges of mocking Azure Functions internals, I focused on comprehensive unit tests for the core logic and relied on integration tests for the pipeline.

### The Code Structure

The complete solution is available in the [WireTapMiddleware GitHub repository](https://github.com/morteza-azizi/wire-tap-middleware):

```
WireTapMiddleware/
├── src/ServiceBusWireTap.Middleware.Logging/     # Main library
├── examples/ServiceBusWireTap.Middleware.Logging.Sample/  # Sample app
├── tests/WireTap.ServiceBus.Middleware.Tests/    # Unit tests
└── WireTapMiddleware.sln
```

## The Publishing Process

### Preparing for NuGet

I configured the project with comprehensive NuGet metadata:
- Package ID: `ServiceBusWireTap.Middleware.Logging`
- Version: 1.0.0
- License: MIT

### Installation and Usage

The package is available on [NuGet](https://www.nuget.org/packages/ServiceBusWireTap.Middleware.Logging/):

```bash
dotnet add package ServiceBusWireTap.Middleware.Logging
```

### Basic Usage Example

```csharp
// Program.cs
using ServiceBusWireTap.Middleware.Logging;

var host = new HostBuilder()
    .UseServiceBusWireTap(options =>
    {
        options.LogLevel = LogLevel.Information;
        options.IncludeMessageBody = true;
        options.IncludeApplicationProperties = true;
        options.MaxBodyLength = 2000; // Truncate messages longer than 2000 chars
    })
    .ConfigureFunctionsWorkerDefaults()
    .Build();

// Your functions work unchanged
[Function("ProcessOrder")]
public async Task ProcessOrder([ServiceBusTrigger("orders")] Order order)
{
    // Business logic here - middleware automatically logs the message
}
```

## What's Next

This is just the beginning. The repository is structured to support multiple messaging brokers:

- **RabbitMQ WireTap Middleware** - Coming soon
- **Apache Kafka WireTap Middleware** - Coming soon  
- **Amazon SQS WireTap Middleware** - Coming soon

Each implementation will follow the same patterns and provide consistent logging capabilities.

### Upcoming Features

- **Large Message Chunking**: Break down large message bodies into smaller chunks
- **Selective Property Logging**: More granular control over what gets logged- 
- **Message Correlation**: Enhanced correlation between related messages

## Conclusion

Building and publishing this middleware taught me that the journey from idea to production-ready NuGet package is both challenging and rewarding. The key is to start with a clear problem, design a focused solution, and build it with future extensibility in mind.

The Wire Tap pattern provided a solid theoretical foundation, and implementing it for Azure Functions gave me deep insights into the platform's internals. Most importantly, I now have a tool that solves a real problem and can help other developers achieve better visibility into their messaging systems.

## Resources

- **NuGet Package**: [ServiceBusWireTap.Middleware.Logging](https://www.nuget.org/packages/ServiceBusWireTap.Middleware.Logging/)
- **GitHub Repository**: [wire-tap-middleware](https://github.com/morteza-azizi/wire-tap-middleware)
- **Wire Tap Pattern Article**: [Understanding the Wire Tap Pattern in Messaging Systems](https://www.mortezaazizi.com/posts/understanding-wire-tap-pattern-in-messaging-systems/)
- **Sample Application**: Complete working examples in the GitHub repository

---

*This article chronicles the journey of building a real-world solution to a common problem in Azure Functions development. The middleware is now available as an open-source NuGet package, ready to help developers achieve better visibility into their Service Bus message flows.* 