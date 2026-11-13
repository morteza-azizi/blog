---
title: "Integrating Microsoft Foundry with Azure Service Bus and Event-Driven Systems"
description: "How Foundry workloads should take part in an event-driven architecture with Azure Service Bus."
date: 2026-12-08
draft: true
categories:
  - Foundry
series:
  - Microsoft Foundry
tags: ["microsoft foundry", "architecture", "AI", "service bus", "messaging"]
---

# Integrating Microsoft Foundry with Azure Service Bus and Event-Driven Systems

> *Part 9 of the Microsoft Foundry series.*

> *Draft. Run the four cases against both workers, paste the message counts, response ids, and published advice under Observed, then write Conclusion from them. Do not publish this sentence.*

## Opening

Every article in this series has started with a failed Service Bus message and ended with a chat window. That is backwards.

Failed messages are not reported by someone typing into a chat. A monitor sees the dead-letter queue grow and raises an event. The thing that should happen next is a workflow, not a conversation.

I have spent years designing around Service Bus: at-least-once delivery, locks that expire, duplicates, poison messages. None of that stops applying because the consumer calls a model. It gets sharper.

**How should AI workloads participate in an event-driven enterprise architecture?**

This article puts the assistant behind a queue. A `TriageRequested` message arrives. A worker asks the agent for advice. A `TriageAdvice` message is published. A person acts on it. The agent never takes the action, and it never touches the bus directly.

The question is what the messaging boundary has to absorb that a synchronous call did not: a duplicate request, a redelivery while the first run is still going, and two deliveries racing for the same request. I test each against a naive worker and an idempotent one.

**TL;DR.** A model call is slow, non-deterministic, and expensive, which are the three properties that make at-least-once delivery dangerous. The fix is not in the agent. It is in the worker: claim the request atomically, remember the first response, and let a later delivery pick it up. Lock renewal reduces how often that matters. It does not remove it. The results are not in this draft yet.

---

## Context

Nothing about Foundry changes what Service Bus guarantees. The pieces I rely on are standard:

- **Delivery is at-least-once.** A message can be delivered more than once. Peek-lock gives a consumer a lock, and a lock that expires makes the message available again.
- **Duplicate detection** discards a message sent twice with the same `MessageId` inside a configurable window. It is set when the queue or topic is created, and it acts on the send side only.
- **`MaxDeliveryCount`** moves a message to the dead-letter queue when delivery attempts run out. Abandoning a message counts as a delivery attempt.
- **Lock duration** is an entity setting, with a default of one minute and a maximum of five.
- **Lock renewal.** `ServiceBusProcessor` renews a lock automatically for up to `MaxAutoLockRenewalDuration`, which defaults to five minutes.

That last point matters for the experiment design. With renewal on, a 50-second agent run holds its 30-second lock, and the message is not redelivered. A test that expects redelivery in that configuration tests nothing. So the experiment controls renewal explicitly, and does not rely on a short lock alone.

Renewal is also not a guarantee. A lock is still lost when the process crashes, when it is paused, when the connection drops, or when a renewal fails. Those are the cases the worker has to survive, and the one I use for the realistic run is a killed process.

From Foundry, the piece that matters is background mode. A response created with `background` is polled until it reaches a terminal status, and I can look up a response by its id. That makes the unit of work something I can store and resume.

Assumptions:

- The assistant stays read-only. It advises. The `TriageAdvice` message is information, not a command.
- A request is one failed message: queue name and sequence number. Requests are independent, so there is no ordering requirement.
- The agent's answer takes from a few seconds to about a minute.
- The worker has a small claim store. I use Azure Table Storage, keyed by the request's `MessageId`.
- How a failed tool call surfaces in a response is not settled. Part 8 captures it. Here I treat a failed response as a classified outcome and do not decide its details.

---

## Architecture

The boundary between synchronous and asynchronous is the first decision. The engineer is not waiting for the reply. The monitor is not either. So the call can be asynchronous, and the system gains retries, load levelling, and a record for free.

{{< mermaid >}}
flowchart LR
    Monitor["DLQ monitor"] -->|"TriageRequested<br/>MessageId = orders:seq"| Req["Queue<br/>triage-requests<br/>duplicate detection"]
    Req --> Worker["Triage worker"]
    Worker <-->|"atomic claim,<br/>first response id"| Store["Claim store"]
    Worker -->|"background response"| Agent["Foundry agent"]
    Agent -->|"peek only"| Mcp["MCP server"]
    Worker -->|"TriageAdvice<br/>MessageId = request key"| Topic["Topic<br/>triage-advice<br/>duplicate detection"]
    Topic --> Person["Engineer or dashboard"]
{{< /mermaid >}}

The agent has no role on `triage-requests` or on `triage-advice`. The worker does. The MCP server has a receive role on the `orders` dead-letter queue. They are separate identities for separate jobs.

| Component | Holds | Does not hold |
|---|---|---|
| Queue | The request, until it is completed | Whether advice was produced |
| Worker | The decision to retry, abandon, or dead-letter | The wording of the advice |
| Claim store | Who owns a request, and its first response id | The advice itself |
| Agent | The wording | Any role on the bus |
| Topic | The published advice | The authority to act on it |

Three different duplicates exist, and they have different cures:

| Duplicate | Cause | Cure |
|---|---|---|
| Producer sends the same request twice | The monitor retried its own send | Duplicate detection on `triage-requests`, with a stable `MessageId` |
| Broker delivers the same message again | The lock expired, or the worker crashed before completing | An atomic claim and a stored response id |
| Worker publishes the same advice twice | Two deliveries both reached the publish step | A stable `MessageId` on the advice, with duplicate detection on the topic |

Duplicate detection does nothing for the second row. The message was sent once. It was received twice.

> **A model call is not idempotent.** Asking the same question twice produces two different answers. The system has to remember the first, not ask again.

That is the property that makes this article different from the other messaging patterns I have written about. A duplicate in an ordinary consumer repeats an effect. A duplicate here produces two pieces of advice that may contradict each other, both attributed to the same request.

---

## Build

The request's `MessageId` is derived from what identifies the failure: `orders:<sequence number>`. The monitor sets it. Duplicate detection is enabled on `triage-requests` and on `triage-advice` when they are created.

There are two workers. The naive worker is the first thing most people write.

**Naive.** Receive, call the agent, wait, publish, complete. If the lock is lost while it waits, the message is redelivered and a second worker instance starts a second run.

**Idempotent.** Claim the request atomically, record the response id before waiting, and let a redelivery resume the same response.

The claim is the part that has to be exact. A read followed by a write is not a claim: two deliveries can both read "absent" before either writes. The claim must be a single operation the store can reject. In Table Storage, inserting a row whose `PartitionKey` and `RowKey` already exist fails with `409`. Updating with an ETag fails with `412` if the row changed since it was read.

```csharp
public enum ClaimOutcome { Won, Resume, Lost }
public sealed record Claim(ClaimOutcome Outcome, string? ResponseId, ETag? ETag);

public sealed class TriageClaimStore(TableClient table)
{
    const string Partition = "triage";
    static readonly TimeSpan ClaimLease = TimeSpan.FromMinutes(2);

    public async Task<Claim> TryClaimAsync(string key, string owner, CancellationToken ct)
    {
        var row = new TableEntity(Partition, key) { ["Owner"] = owner, ["ClaimedAt"] = DateTimeOffset.UtcNow };
        try
        {
            // Atomic: the service rejects a second insert of the same key.
            var created = await table.AddEntityAsync(row, ct);
            return new Claim(ClaimOutcome.Won, null, created.Headers.ETag);
        }
        catch (RequestFailedException ex) when (ex.Status == 409)
        {
            var existing = (await table.GetEntityAsync<TableEntity>(Partition, key, cancellationToken: ct)).Value;

            if (existing.GetString("ResponseId") is { } responseId)
                return new Claim(ClaimOutcome.Resume, responseId, null);      // a response already exists

            if (existing.GetDateTimeOffset("ClaimedAt") < DateTimeOffset.UtcNow - ClaimLease)
            {
                existing["Owner"] = owner;
                existing["ClaimedAt"] = DateTimeOffset.UtcNow;
                try
                {
                    // Atomic takeover of a dead claim: succeeds only if the row is unchanged since the read.
                    var taken = await table.UpdateEntityAsync(existing, existing.ETag, TableUpdateMode.Replace, ct);
                    return new Claim(ClaimOutcome.Won, null, taken.Headers.ETag);
                }
                catch (RequestFailedException e) when (e.Status == 412) { /* someone else took it */ }
            }

            return new Claim(ClaimOutcome.Lost, null, null);                   // another delivery is mid-start
        }
    }

    public Task SaveResponseIdAsync(string key, string responseId, ETag etag, CancellationToken ct) =>
        // Fails with 412 if this claim was taken over in the meantime.
        table.UpdateEntityAsync(new TableEntity(Partition, key) { ["ResponseId"] = responseId },
                                etag, TableUpdateMode.Merge, ct);
}
```

The processor uses the outcome and nothing else to decide whether to start a response:

```csharp
var processor = client.CreateProcessor("triage-requests", new ServiceBusProcessorOptions
{
    MaxConcurrentCalls = 4,
    AutoCompleteMessages = false,
    MaxAutoLockRenewalDuration = TimeSpan.FromMinutes(5)   // production setting; the experiment overrides it
});

processor.ProcessMessageAsync += async args =>
{
    var key = args.Message.MessageId;
    var ct = args.CancellationToken;

    var claim = await claims.TryClaimAsync(key, instanceId, ct);

    if (claim.Outcome == ClaimOutcome.Lost)
    {
        // Another delivery is starting this request. Never start a second response.
        await args.AbandonMessageAsync(args.Message, cancellationToken: ct);
        return;
    }

    var responseId = claim.ResponseId;
    if (claim.Outcome == ClaimOutcome.Won)
    {
        responseId = await agent.StartBackgroundAsync(args.Message.Body, ct);
        await claims.SaveResponseIdAsync(key, responseId, claim.ETag!.Value, ct);   // before waiting, not after
    }

    var advice = await agent.WaitForTerminalAsync(responseId!, ct);

    if (advice.Failed)
    {
        await FailureHandling.ApplyAsync(args, advice, ct);   // classification from Part 8: abandon or dead-letter
        return;
    }

    await publisher.PublishAdviceAsync(key, advice, ct);      // MessageId = key; the topic drops a repeat
    await args.CompleteMessageAsync(args.Message, ct);        // a lock-lost error here is expected and ignored
};
```

Decisions in that code:

- **`AutoCompleteMessages = false`.** The worker completes the request only after the advice is published. Completing earlier would lose the request if the publish failed.
- **Only a winner starts a response.** `Won` starts. `Resume` polls an existing response. `Lost` abandons.
- **The response id is stored before the wait.** If the process dies in the wait, the next delivery resumes the same response.
- **A dead claim can be taken over.** If the owner crashed between claiming and saving the response id, the lease expires, and a later delivery takes over with an ETag-conditional update.
- **Publishing is made safe by the broker, not by a flag.** Two deliveries that resume the same response may both publish. The advice carries the request key as its `MessageId`, and duplicate detection on the topic drops the second.
- **`MaxConcurrentCalls` is a budget.** It is the number of model calls in flight. It should be set from the token limit that Part 6 put on the project, not from CPU.

Weaknesses that remain, left visible:

- **An orphan response.** A crash between starting a response and saving its id leaves a response nobody polls. The takeover starts a second. One run is wasted. No duplicate advice is published, because the first is never read.
- **Abandon costs delivery attempts.** A `Lost` delivery is abandoned, and abandoning counts against `MaxDeliveryCount`. The count has to leave room for it, or a busy request can be dead-lettered by contention alone.
- **The duplicate-detection window is finite.** A second advice published after the window is not dropped. Subscribers should still treat the request key as theirs to deduplicate on.
- **The claim store is a dependency.** If Table Storage is unavailable, the worker has to stop. It must not process without a claim.

Code: folder `09-service-bus` in the series repo.

---

## Experiment

Each case runs against the naive worker and the idempotent worker. Lock renewal is a variable, not a default. I state it for every run.

**1. Producer duplicate.** The monitor sends the same `TriageRequested`, with the same `MessageId`, twice within the duplicate detection window.

**2a. Lock expires while the agent runs.** Lock duration 30 seconds. Auto-renewal is turned off for both workers (`MaxAutoLockRenewalDuration` of zero). The agent's run is made to take about 50 seconds, so expiry is certain by construction.

**2b. The same, with renewal on.** Same lock, same 50-second run, renewal at five minutes. This asks whether renewal alone prevents the redelivery.

**2c. The worker dies mid-run.** Renewal on. The worker process is killed while the agent is running. The lock cannot be renewed by a dead process. This is the case renewal does not cover.

**3. A claim race.** Twenty concurrent calls to `TryClaimAsync` for one key, with no Service Bus involved. Then the same twenty calls against a read-then-write version of the claim.

For every run I count: messages delivered, responses created, advice messages published, and the delivery count on the request.

Several statements in this article are about behaviour I have read and not seen. I keep them apart from results:

| Claim | Source | Status |
|---|---|---|
| `MaxAutoLockRenewalDuration` defaults to five minutes | Service Bus client documentation | Documented. Observed: pending (case 2b) |
| Duplicate detection acts on the send side, and is set when the entity is created | Service Bus documentation | Documented. Observed: pending (case 1) |
| Abandoning a message counts as a delivery attempt | Service Bus delivery semantics | Documented. Observed: delivery count after contention |
| Inserting an existing key fails with `409`, and an ETag-conditional update fails with `412` | Azure Table Storage documentation | Documented. Observed: pending (case 3) |
| A background response can be looked up and resumed by its id | Responses API reference | Documented. Observed: pending (case 2c) |
| Duplicate detection on the topic drops a second advice with the same `MessageId` | Service Bus documentation | Documented. Observed: published count in 2a and 2c |
| How a failed response is classified | Part 8 | Unresolved until Part 8's fixtures exist |

**Expected**

*These are expectations. They are not results.*

| Case | Worker | Delivered | Responses created | Advice published |
|---|---|---|---|---|
| 1 | Naive | 1 | 1 | 1 |
| 1 | Idempotent | 1 | 1 | 1 |
| 2a | Naive | 2 or more | 2 or more | Possibly more than 1, with different text |
| 2a | Idempotent | 2 or more | 1 | 1 |
| 2b | Naive | 1 | 1 | 1 |
| 2b | Idempotent | 1 | 1 | 1 |
| 2c | Naive | 2 or more | 2 | 1 or 2 |
| 2c | Idempotent | 2 or more | 1 (or 2 if the crash fell before the response id was saved) | 1 |

Case 1 is expected to pass on both. Duplicate detection acts before the worker sees anything. The case exists to show that its protection stops at the sender.

Case 2b is expected to pass on both too. If it does, then renewal alone is enough for a slow call in a healthy process, and the idempotent worker earns its cost only in 2c and in lock-loss cases I can force. If 2b does not pass, the renewal setting is not doing what I assumed.

In case 3, the atomic claim is expected to return `Won` exactly once and `Resume` or `Lost` for the rest. The read-then-write version is expected to return `Won` more than once under enough concurrency. It may not. A race that does not reproduce in one run is not evidence of safety, so I record how many runs I made.

The failure worth catching is on the idempotent worker: two responses for one request outside the orphan window.

I also record what the worker does when the agent fails permanently. I stop the MCP server, run one request, and note which dead-letter reason the request carries, and whether the attempts stay within `MaxDeliveryCount`.

**Observed**

Not run yet. Paste each result here, unchanged.

| Case | Worker | Renewal | Delivered | Responses created | Advice published | Advice texts identical? |
|---|---|---|---|---|---|---|
| 1 | Naive | | | | | |
| 1 | Idempotent | | | | | |
| 2a | Naive | Off | | | | |
| 2a | Idempotent | Off | | | | |
| 2b | Naive | On | | | | |
| 2b | Idempotent | On | | | | |
| 2c | Naive | On | | | | |
| 2c | Idempotent | On | | | | |

| Case 3 | Runs | `Won` per run | `Resume` / `Lost` per run |
|---|---|---|---|
| Atomic claim | | | |
| Read-then-write | | | |

Also record: duplicate detection window, lock duration, `MaxDeliveryCount`, the dead-letter reason on a failed request, and the delivery count after contention.

**Conclusion**

Written from the results, after the run. Not before.

---

## Trade-offs

The queue makes several things safe at once. The request survives a restart. The load on the model is levelled, and `MaxConcurrentCalls` is a throttle I can set against the token budget. A failing agent does not lose requests. It makes them retry, and then makes them visible in the dead-letter queue.

It delays and complicates.

- **The answer is not synchronous.** Nobody gets advice in the same breath as the request. For the monitor, that is fine. For a person watching a screen, it is not.
- **State is needed.** The claim store is a new dependency with its own availability.
- **Idempotency spans three places.** The sender, the worker, and the publisher each need their own protection. Duplicate detection covers two of them, and only inside its window.
- **Contention has a price.** A delivery that loses the claim is abandoned, and it spends a delivery attempt.
- **Poison handling is recursive.** The dead-letter queue of `triage-requests` holds failed triage requests. Something has to triage those. I leave that to a person and say so.
- **Ordering is given up.** I rely on requests being independent. If a later failure on the same queue should be triaged after an earlier one, sessions are needed, and they change the concurrency model.

The message contract is an assumption I am making explicit: a request carries identifiers, not the message body. The body is read by the MCP tool at triage time. That keeps data off the bus and keeps the read inside the boundary from Part 2.

The alternative is a synchronous call from a dashboard button. The engineer clicks, waits 20 seconds, and reads the answer. It is less to build, it needs no store, and it has none of these failure modes. For a handful of failures a day, I would choose it.

I set aside putting the worker's logic inside the agent, with a tool that receives from the queue. That would give the model a receive-side role on the bus. The cost of the idempotent worker is code I can read. The cost of the alternative is a role I cannot take back.

---

## When NOT to Use It

Do not put a bus between a request and an answer when the person asking needs the answer now. The queue will not make the model faster. It will hide how long the engineer has been waiting.

Do not build the claim store for a system that handles a few events a day. A human can notice a double reply.

Do not let an agent consume from the bus directly. The role it needs includes acting on messages. The worker is the component that should hold it.

---

## Practical Takeaway

1. **Claim before you call.** A model call cannot be repeated to get the same answer. The claim must be one operation the store can reject, and the response id is stored before the wait.
2. **Separate the duplicates.** Duplicate detection handles the sender and the publisher. A claim handles redelivery. Lock renewal handles slowness. None of them replaces the others.
3. **Keep the agent off the bus.** The worker holds the roles, the retries, and the dead-letter decisions. The agent holds the wording.

---

## Next Article

> In the next article, we move from **the event-driven boundary** to **the whole architecture**, and examine **what the system looks like when agents, MCP, APIs, RAG, evaluation, the gateway, governance, resilience, and messaging are one design**.
