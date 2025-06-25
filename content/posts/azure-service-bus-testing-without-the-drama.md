---
title: "Azure Service Bus Testing Without the Drama"
date: 2025-01-27
draft: false
tags: ["azure", "service bus", "testing", "testcontainers", "azure functions", "integration testing", "local development"]
---

# Azure Service Bus Testing Without the Drama

Testing Azure Functions with Service Bus usually involves a lot of drama. You know the drill - setting up cloud resources, tests that work sometimes but not others, and that sinking feeling when something breaks in production but you can't reproduce it locally.

And don't get me started on shared environments. You need extra dev environments because if you share them, you get race conditions. Even worse, if you accidentally test against the QA environment that already has Azure Functions hooked to it and the QA team is actively working - well, let's just say you'll be getting some very unhappy messages.

I wanted to find a better way. Here's how I figured out a testing approach that actually works without all the headaches.

The Azure Function I'm testing is straightforward - it processes orders from a Service Bus queue:

```csharp
[Function("OrderProcessingFunction")]
public async Task Run(
    [ServiceBusTrigger("%QueueName%", Connection = "ServiceBusConnection")] Order order)
{
    await orderProcessingService.ProcessOrderAsync(order);
}
```

## 1. Why Not Just Use the Local Emulator?

You might wonder, "Why not just install the Service Bus emulator locally?" Well, that works for local development, but it doesn't solve the bigger picture:

- **Pipeline Issues** - Your CI/CD pipeline needs to test too, and installing emulators on build agents is a pain
- **Team Consistency** - Everyone needs the same setup, and "works on my machine" isn't good enough
- **Isolation** - Local emulators can have state issues between test runs

## 2. Testcontainers to the Rescue

Testcontainers solves all these problems by running the Service Bus emulator in a Docker container that starts fresh for each test run.

**Benefits:**
- **Consistent** - Same environment everywhere (local, CI/CD, teammate's machine)
- **Isolated** - Each test gets a clean emulator instance
- **Automated** - No manual setup or cleanup needed

Here's how the testing flow works:

{{< mermaid >}}
graph TD
    A["Test Starts"] --> B["Start Testcontainer"]
    B --> C["Service Bus Emulator<br/>in Docker Container"]
    C --> D["Create Test Queue<br/>(order-processing-queue)"]
    D --> E["Send Test Message<br/>(Order object)"]
    E --> F["Test Business Logic<br/>(OrderProcessingFunction)"]
    F --> G["Verify Processing<br/>(Mock assertions)"]
    G --> H["Cleanup Container"]
    H --> I["Test Complete"]
    
    C -.-> J["Fresh Instance<br/>Every Test Run"]
    C -.-> K["No Cloud Resources<br/>Required"]
    C -.-> L["Same Environment<br/>Everywhere"]
    
    style C fill:#e1f5fe
    style J fill:#f3e5f5
    style K fill:#f3e5f5
    style L fill:#f3e5f5
{{< /mermaid >}}

**Setup** is straightforward with a `Config.json` file that defines your queues:

```json
{
  "UserConfig": {
    "NamespaceConfig": [
      {
        "Name": "test-servicebus",
        "Entities": {
          "Queues": [
            {
              "Name": "order-processing-queue"
            }
          ]
        }
      }
    ]
  }
}
```

And a container manager class:

```csharp
public class ServiceBusTestContainer : IAsyncDisposable
{
    private ServiceBusContainer? _serviceBusContainer;
    
    public string ConnectionString => _serviceBusContainer?.GetConnectionString() ?? 
        throw new InvalidOperationException("Container not started");
    
    public async Task StartAsync()
    {
        _serviceBusContainer = new ServiceBusBuilder()
            .WithImage("mcr.microsoft.com/azure-messaging/servicebus-emulator:latest")
            .WithAcceptLicenseAgreement(true)
            .WithPortBinding(5672, true)
            .WithResourceMapping("Config.json", "/ServiceBus_Emulator/ConfigFiles/")
            .Build();
        
        await _serviceBusContainer.StartAsync();
        
        // Wait for emulator to be fully ready
        await Task.Delay(TimeSpan.FromSeconds(20));
    }

    public async ValueTask DisposeAsync()
    {
        if (_serviceBusContainer != null)
        {
            await _serviceBusContainer.DisposeAsync();
        }
    }
}
```

## 3. Testing the Flow

Here's where it gets interesting. Instead of trying to run the entire Azure Functions runtime in tests (which is complicated), I test the flow in a simpler way:

1. **Start the container** with Service Bus emulator
2. **Send a message** to the queue
3. **Let the function process it** (or test the business logic directly)
4. **Assert the results**

```csharp
[Fact]
public async Task AzureFunction_EndToEnd_ShouldProcessMessage()
{
    // Start the Service Bus Emulator
    var containerManager = new ServiceBusTestContainer();
    await containerManager.StartAsync();
    
    var connectionString = containerManager.ConnectionString;
    
    // Arrange - Create test order
    var testOrder = OrderBuilder.Create()
        .WithCustomer("Test Customer")
        .AddLaptop()
        .Build();
    
    // Test 1: Verify Service Bus connectivity
    await SendOrderToQueueAsync(connectionString, testOrder);
    var receivedOrder = await ReceiveOrderFromQueueAsync(connectionString);
    
    Assert.NotNull(receivedOrder);
    Assert.Equal(testOrder.Id, receivedOrder.Id);
    Assert.Equal(testOrder.CustomerName, receivedOrder.CustomerName);
    
    // Test 2: Verify the function logic with mocked service
    var mockService = new Mock<IOrderProcessingService>();
    var orderProcessingFunction = new OrderProcessingFunction(mockService.Object);
    
    await orderProcessingFunction.Run(testOrder);
    
    // Assert the service was called correctly
    mockService.Verify(x => x.ProcessOrderAsync(
        It.Is<Order>(o => o.Id == testOrder.Id)), Times.Once);
}
```

## What I Learned

This journey taught me a few things:

1. **Don't overcomplicate the runtime** - Testing business logic separately is often cleaner than trying to test the entire Azure Functions pipeline
2. **GUI tools aren't always available** - Learning to inspect things programmatically is more reliable anyway
3. **Containers make testing predictable** - No more "works on my machine" issues
4. **Clean architecture pays off** - Separating concerns makes everything easier to test

The final result? Tests that actually work, can run anywhere, and don't require any cloud resources. No drama, just reliable testing.

## The Code

If you want to see the complete implementation, I've put it all in a GitHub repo: [azure-solutions-architecture](https://github.com/morteza-azizi/azure-solutions-architecture/tree/main/samples/servicebus-emulator-testcontainers)

It includes:
- Working Azure Function with proper dependency injection
- Integration tests using Service Bus emulator
- Programmatic message inspection (since Service Bus Explorer doesn't work)
- Clean architecture with separated concerns

The README has all the details for getting it running. It's not rocket science, just a straightforward approach that works reliably.

---