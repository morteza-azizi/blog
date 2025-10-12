---
title: "From Service Bus to Storage Queue: A Testing Journey"
date: 2025-10-12
draft: false
tags: ["azure", "storage queue", "testing", "testcontainers", "azure functions", "integration testing", "local development", "azurite"]
---

# From Service Bus to Storage Queue: A Testing Journey

After successfully implementing testcontainers for Azure Service Bus (which I wrote about in my [previous post](azure-service-bus-testing-without-the-drama.md)), I found myself working on a new solution that needed Azure Storage Queue instead. 

Naturally, I got curious - could I apply the same testing approach? Turns out, yes! But the journey taught me some interesting things about the differences between these two Azure messaging services.

## The Starting Point

I had already cracked the code for Service Bus testing:
- Testcontainers managing the Service Bus emulator
- Clean Azure Functions with triggers
- Reliable integration tests
- No cloud resources needed for development

The pattern was working beautifully. But now I had a new requirement: Azure Storage Queue.

## The Question

Could the same testcontainers approach work for Azure Storage Queue? I was motivated to find out because:

1. **I knew the pain** - Testing with real Azure resources is expensive and slow
2. **I had a pattern** - Service Bus testcontainers was working well
3. **New solution needed it** - Couldn't just skip testing this time
4. **Curiosity** - How different could Storage Queue really be?

## The Investigation

First discovery: **Azure Storage Queue uses Azurite**, not a separate emulator like Service Bus.

Azurite is Microsoft's unified emulator for Azure Storage services - Blob, Queue, and Table storage all in one. This was actually better than I expected!

The testcontainers setup was even cleaner than Service Bus! No config files needed, just start the Azurite container. The container manager handles everything - starting the emulator, exposing the right ports, and providing the connection string.

Even better: Azurite starts faster than the Service Bus emulator. About 5 seconds versus 20 seconds. Small win, but it adds up when you're running tests frequently.

The Azure Function followed the same clean pattern as Service Bus - a simple trigger attribute, automatic deserialization, and just business logic. No try-catch noise, no manual JSON parsing, no infrastructure code cluttering the function.

Beautiful and focused.

## The Pleasant Surprise

While implementing this, I discovered something nice: **Azure Storage Explorer actually works with Azurite!**

This was different from Service Bus where the Service Bus Explorer doesn't work with the emulator. With Storage Queue, I could:

1. Start Azurite locally
2. Open Azure Storage Explorer
3. Connect to the local emulator
4. See the queues, peek messages, inspect everything

This made debugging so much easier.

But I still wanted programmatic inspection for automated testing. The peek operation turned out to be just as clean as Service Bus - fetch messages without consuming them, inspect the properties, verify the content.

The test logs show exactly what's happening at each step, making it easy to debug when something goes wrong.

## Key Differences I Learned

Working with both made the differences clear:

| Feature | Service Bus | Storage Queue |
|---------|-------------|---------------|
| **Emulator** | Service Bus Emulator | Azurite |
| **Ports** | 5672 (AMQP), 5300 (HTTP) | 10000-10002 (HTTP) |
| **Config** | JSON file required | No pre-config needed |
| **Startup Time** | ~20 seconds | ~5 seconds |
| **Explorer Tool** | Limited support | Full Azure Storage Explorer |
| **Message Size** | Up to 256KB | Up to 64KB |
| **Protocol** | AMQP | HTTP REST API |

## The Testing Strategy

I ended up with a layered testing approach for both Service Bus and Storage Queue:

**Integration Tests** - Test queue operations without running the full Azure Functions runtime. Send messages, receive them, verify serialization works.

**Inspection Tests** - Verify queue state without consuming messages. Peek at what's in the queue, check message properties, confirm everything is there.

**Unit Tests** - Test business logic in isolation with mocks. Fast, reliable, no dependencies.

**Manual Testing** - Full end-to-end with `func start` for final verification against real or local emulators.

Each level serves a purpose. Together they give confidence without excessive test time.

## The Results

After implementing both samples:

✅ **Fast feedback** - Tests run in seconds, not minutes  
✅ **No cloud costs** - Everything runs locally  
✅ **Consistent environment** - Same setup everywhere  
✅ **Easy debugging** - Visual tools work (especially for Storage Queue)  
✅ **Comprehensive coverage** - Multiple test levels  
✅ **Production confidence** - Caught issues before deployment  

## What I'd Do Differently

Looking back, I should have:

1. **Started with the shared library** - Would have saved refactoring time
2. **Used immutable builder** from the start - Better design, even for tests
3. **Added the pause-for-exploration** feature earlier - Made debugging much easier

But these are minor compared to the value of having reliable, automated testing.

## The Code

Both complete samples are available:

- [Service Bus Sample](https://github.com/morteza-azizi/azure-solutions-architecture/tree/main/samples/servicebus-emulator-testcontainers)
- [Storage Queue Sample](https://github.com/morteza-azizi/azure-solutions-architecture/tree/main/samples/storagequeue-emulator-testcontainers)

Each includes:
- Complete Azure Function implementation
- Testcontainers integration
- Comprehensive test suite
- Shared business logic library
- Detailed README with instructions

## Final Thoughts

Starting with Service Bus and expanding to Storage Queue taught me that **the pattern is more important than the technology**. 

The curiosity to explore Storage Queue after solving Service Bus led to deeper understanding of Azure messaging options and reliable testing approaches.

---

