---
title: "Designing Reliable AI Agents: Failure Modes and Resilience"
description: "What happens when a Foundry agent, its model, a tool, an API, or another dependency fails."
date: 2026-12-01
draft: true
categories:
  - Foundry
series:
  - Microsoft Foundry
tags: ["microsoft foundry", "architecture", "AI", "reliability"]
---

# Designing Reliable AI Agents: Failure Modes and Resilience

> *Part 8 of the Microsoft Foundry series.*

> *Draft. Run the six faults, save each raw response as a fixture, paste what the agent said, what the application logged, and how many calls the dependency received under Observed, replace the `OpenApiOutcome` stub from the captured payload, then write Conclusion from them. Do not publish this sentence.*

## Opening

Seven parts in, I have only watched the assistant when everything works.

Everything does not work. The model deployment returns `429`. The MCP server from Part 2 is restarting. The Producer Registry is slow. Service Bus returns an error for an entity that exists. In an ordinary service each of these is a status code and a log line. In an agent, each can become a sentence.

In Part 1 the defect was an agent that wrote as if an action had happened. A tool failure is the same defect from a different source: the agent was supposed to look something up, could not, and still has to write a reply.

**What happens when an AI agent, model, tool, API, or external dependency fails?**

This article injects six faults into the assistant, one at a time, and records three things for each: what the agent said, what the application knew, and how many times the failing dependency was called for one user request.

**TL;DR.** The model cannot be the component that decides a dependency failed. The application has to see the failure in the structured response, classify it, and decide. How much of the failure the platform puts in that structure is partly documented, partly reported, and partly unknown, and this article says which. Retries are the part that gets worse by accident: every layer that retries multiplies the others. The results are not in this draft yet.

---

## Context

Foundry's responses have a lifecycle. A response created in `background` mode is polled until it reaches a terminal status. The [REST reference](https://ai.azure.com/api-reference/responses/get-response/) lists them: `completed`, `failed`, `cancelled`, `incomplete`, and the in-flight `queued` and `in_progress`.

The Agent Service [limits and quotas page](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/limits-quotas-regions) is specific about `429`: use exponential backoff with jitter, and for sustained load consider provisioned throughput. Some `429` codes are not retryable by waiting, such as an exhausted session quota.

Two facts from earlier parts matter:

- A `429` from the AI Gateway (Part 6) means the project exceeded tokens per minute. A `403` means it spent its quota. Those need different handling.
- A tool call is a separate network call to a separate service. The model's output is not evidence that it succeeded.

What the platform documents about a failed tool call is thinner than what the application logic needs. I separate three levels, because the application logic in this article rests on them:

| Level | Status | Source |
|---|---|---|
| A response has a `status` (`failed`, `incomplete`, and others) and an `error` object with a `code` | **Documented** | The Responses REST reference |
| An `mcp_call` output item has an optional `error`, an `output`, and a `status` | **Documented** (field names and types) | The same reference |
| `openapi_call` and `openapi_call_output` are output item types | **Documented** (type names only; I found no field-level description of a failed call) | The same reference |
| A failed MCP tool call ends the whole response with `tool_user_error`, and the model never sees the error | **Reported, not documented.** A community report on Microsoft Q&A describes this on the new Responses API, with a workaround that is itself undocumented. I have not reproduced it | Microsoft Q&A |
| What an OpenAPI tool failure looks like in the output | **Unknown** | Nobody I can cite |

If the community report is right, then a failed MCP call does not appear as an `mcp_call` item with an `error`. It appears as a failed response, and the agent never writes a sentence about it. That would change F3 and F6 completely. It would also change what Part 2's out-of-scope case does. So the application logic below is a hypothesis until the run has produced payloads for it.

A further fact is not from the platform. Azure SDK clients retry transient failures by default. For the Azure.Core-based clients I have used, that is three retries with exponential delay. A call that I wrap in my own retry policy is retried by two layers, and neither knows about the other. I verify the count in the experiment instead of repeating it.

Assumptions:

- The assistant reads. It does not act. There is no tool in this system whose retry creates a side effect. That is deliberate, and it is the reason these experiments are safe to run repeatedly.
- One user waits for one answer. That sets a time budget.
- The faults are injected by me. A real incident will be messier.

---

## Architecture

The path from the engineer to an answer has more places to fail than the diagram of any single article showed.

{{< mermaid >}}
flowchart TD
    Engineer["Engineer"] --> App["Application<br/>timeout, retry budget, classification"]
    App -->|"F1"| Model["Model deployment"]
    Model --> Agent["Agent"]
    Agent -->|"F2, F3"| Mcp["MCP server"]
    Mcp -->|"F6"| Bus["Service Bus"]
    Agent -->|"F4"| Api["Producer Registry"]
    Agent -->|"F5"| Search["Runbook index"]
    Agent --> Reply["Reply"]
    Reply --> App
    App --> Log["Logs, response id,<br/>correlation id"]
{{< /mermaid >}}

Six faults, one per arrow:

| Fault | Where | Injected by |
|---|---|---|
| F1 | Model returns `429` | Gateway token limit set very low |
| F2 | MCP server is slow | Server sleeps 60 seconds before responding |
| F3 | MCP server errors | Server returns an error for every call |
| F4 | Producer Registry returns `503` with `Retry-After` | Fault switch on the API |
| F5 | Runbook index returns nothing | Query for a term that is in no runbook |
| F6 | Service Bus peek throws | Revoke the server's role on the namespace |

The responsibilities differ for each:

| Layer | Handles |
|---|---|
| Dependency | Telling the truth about its failure, with a status code and a retry hint |
| Agent | Saying that a lookup failed, in words |
| Application | Detecting that it failed, in structure; deciding to retry, stop, or show a partial answer |
| Person | Deciding whether to try again |

The agent's sentence is a courtesy. The application's decision is the control.

> **A tool failure that exists only in prose is a failure the application cannot see.**

That is the central design rule. If the only evidence that the MCP call failed is a paragraph saying so, the only thing that can react is another model reading the paragraph. The structured response is where evidence should live, and the application should read it.

How much evidence the platform puts there, for each kind of tool, is what the run has to establish. Two outcomes are possible for a failed tool call: the agent writes about it, or the platform fails the response before the agent can. The application needs to handle both, and it must never treat a shape it does not recognise as success.

The second rule is about retries. Retries recover transient failures, and they multiply load during an outage. They need a budget: a maximum number of attempts for the whole request, across every layer, not per layer.

---

## Build

The application layer does four things. It sets a deadline, disables retries that it does not control, classifies what comes back, and logs the join between the user's request and Foundry's response.

**A deadline.** One user is waiting. The whole request has 45 seconds. A tool that takes 60 seconds fails the request, rather than the user's patience.

```csharp
using var cts = CancellationTokenSource.CreateLinkedTokenSource(requestAborted);
cts.CancelAfter(TimeSpan.FromSeconds(45));
```

**A single retry owner.** The SDK's default retry is turned off, and the application's policy is the only one. Two retry policies behind one another are the same as one that nobody can read.

**Classification.** The application decides what to do from the status, never from the reply text:

```csharp
enum Next { Retry, Stop, ShowPartial }

static Next Classify(int status) => status switch
{
    429                             => Next.Retry,       // honour Retry-After
    408 or 500 or 502 or 503 or 504 => Next.Retry,       // transient, within the budget
    403                             => Next.Stop,        // gateway quota is spent; waiting will not help
    400                             => Next.Stop,        // the request is wrong; so will be the next one
    _                               => Next.Stop
};
```

**Reading the response, not the reply.** The application inspects the structure at three levels, in order, and has a fourth outcome for anything it cannot place:

```csharp
enum FailureKind { None, ResponseFailed, ToolCallFailed, Unclassified }

static (FailureKind Kind, string Detail) ReadFailure(JsonElement response)
{
    // Level 1, documented: the response itself.
    var status = response.GetProperty("status").GetString();
    if (status is "failed" or "incomplete")
        return (FailureKind.ResponseFailed, ErrorCode(response));          // error.code, for example tool_user_error

    foreach (var item in response.GetProperty("output").EnumerateArray())
    {
        var type = item.GetProperty("type").GetString();

        // Level 2, documented fields: an MCP call with an error.
        if (type == "mcp_call" && item.TryGetProperty("error", out var e) && e.ValueKind != JsonValueKind.Null)
            return (FailureKind.ToolCallFailed, e.ToString());

        // Level 3, fields undocumented: an OpenAPI call. Parsed only after a failure has been captured.
        if (type is "openapi_call" or "openapi_call_output" && !OpenApiOutcome.IsKnownGood(item))
            return (FailureKind.Unclassified, type!);                      // not a pass: a person looks
    }

    return (FailureKind.None, "");
}
```

Three rules sit in that code.

- **An unrecognised shape is not success.** `OpenApiOutcome.IsKnownGood` is deliberately strict. It returns true only for payloads that were captured during a successful run. Anything else is `Unclassified`, and the application reports that it could not tell.
- **The detector is tested against captured payloads.** Every fault in the experiment saves the raw response JSON to a fixtures folder. The detector's tests read those files. A shape without a fixture is a shape the detector does not claim to understand.
- **`OpenApiOutcome` is written after F4.** I do not know what a failed OpenAPI call looks like in the output. Writing a parser for it now would be a guess with a unit test attached. The class stays a stub until a captured payload exists.

If any tool failed, or the response failed, the application shows the user what it can: the reply if there is one, and a notice that the lookup failed and the advice is incomplete. It does not hide the reply and does not retry silently. If the response failed outright, there is no reply, and the notice says why, using the error code.

**The join.** The application writes its own correlation id, the response id, and the time into one log line, and sets the correlation id in the response's `metadata`. Whether any dependency can receive it is one of the questions below.

Code: folder `08-resilience` in the series repo.

---

## Experiment

Six faults. Each is run once with the application's policy, with the agent otherwise unchanged. For each fault I count the calls that reached the failing dependency for one user request.

For F1, the user's request is made once, and the count is of calls arriving at the gateway.

**Expected**

*These are expectations. They are not results. Where the platform's behaviour is documented, the row says so. Where it is only reported, or unknown, the row has two outcomes, and the run decides.*

| Fault | Agent says | Application knows | Application does |
|---|---|---|---|
| F1 `429` | Nothing: no response | A `429` with a retry hint | Retries after the hint, within the budget, then stops with an honest message |
| F2 slow MCP | Nothing, or an error | The deadline passed | Fails at 45 seconds. Does not wait for 60 |
| F3 MCP error, outcome A | That it could not read the queue | An `mcp_call` with an error | Shows the reply with a "lookup failed" notice |
| F3 MCP error, outcome B (reported, not documented) | Nothing: the response failed | `status = failed`, an error code such as `tool_user_error` | Shows no reply and a notice with the code. Decides retry from the code, not from the text |
| F4 `503` | Unknown | Unknown until observed. The item types `openapi_call` and `openapi_call_output` exist | `Unclassified` until a payload has been captured. A person looks |
| F5 empty index | That the runbooks do not cover it | Nothing in structure | Nothing. This is not a failure of a dependency |
| F6 peek throws | Same two outcomes as F3 | Same two outcomes as F3 | Same as F3, and the cause is visible in the server log |

F3 and F6 are run twice, once per way the server reports the error: a protocol-level error (`McpException` or an `isError` result), and a normal result that carries the error as text. If the platform treats them differently, that difference is a design input for Part 2's server: it may be better for a refusal to be data than to be an error.

Four things I want from the run that the table cannot predict:

- **The retry count.** One user request, how many calls reach the failing dependency. If the SDK retried three times and my policy retried twice, the dependency saw up to nine.
- **Fabrication.** In F3, F4, and F6, does the reply contain a reason, an owner, or a delivery count that nobody supplied?
- **Visibility.** Which of the six can the application detect from structure alone?
- **The join.** Starting from the response id, can I reach the MCP server's log entry and the API's log entry for the same request?

The failure worth catching is a fluent reply on F3 or F6 that includes a plausible dead-letter reason. It would pass a reader's glance and fail an evaluation case from Part 5. I add F3 and F6 to the set as cases.

**Observed**

Not run yet. Paste each result here, unchanged.

| Fault | Agent said | `status` and `error.code` | Failed output item (type and fields) | Detector result | Calls received |
|---|---|---|---|---|---|
| F1 | | | | | |
| F2 | | | | | |
| F3, protocol error | | | | | |
| F3, normal result | | | | | |
| F4 | | | | | |
| F5 | | | | | |
| F6, protocol error | | | | | |
| F6, normal result | | | | | |

For every row, save the raw response JSON to the fixtures folder and link it. The detector's tests read those files.

Also record: whether a failed MCP call ended the response (outcome B) or reached the agent (outcome A), whether that depends on how the server reports the error, what the OpenAPI tool leaves in the output when it fails, and whether the correlation id reached any dependency.

**Conclusion**

Written from the results, after the run. Not before.

---

## Trade-offs

A retry recovers a transient failure that the user would otherwise see. For a `503` that clears in a second, it is the difference between an answer and an error.

It also hides things and amplifies others. A retry that succeeds on the third attempt is a latency spike nobody reported. A retry during a real outage turns one user's request into several against a service that is already struggling, and a queue of users into more. The retry budget is a statement about whose problem that is.

Fallback has its own cost. A fallback needs a source of truth that is not the failed one. For the producer lookup, that could be a cached value, which can be stale. For the runbook index, it could be the previous snapshot. Every fallback is a second implementation with its own failure modes. I did not build one. Showing a partial answer with a notice is the fallback I could defend.

Assumptions I am making explicit:

- **Reads are idempotent.** Peeking twice is the same as peeking once. That is why retries are safe in this system. A tool that writes would break this, and Part 9 deals with the queue-based version of that problem.
- **Partial success is acceptable.** An answer without the producer is still useful. For other workflows it would be worse than no answer.
- **The user can wait 45 seconds.** If the answer feeds an automated step, the budget is different.

I considered letting the agent retry its own tool calls. It is the least code, and it is also the least visible. The model would be deciding how many times to hit a service from inside a reply I cannot count.

I considered wrapping everything in a circuit breaker. For one assistant used by a few engineers, the value is small and the state is another thing to reason about. It is the right tool when many callers share a failing dependency.

---

## When NOT to Use It

Do not retry when the failure is the answer. A `403` for a spent quota, a `400` for a bad request, and an MCP error saying a queue is not exposed are all deterministic. The second attempt gets the same response, and costs tokens.

Do not retry a call that has a side effect without an idempotency key. The assistant has no such call. A notification or a resubmission added later would.

Do not retry while a person is waiting for a long time on a dependency that is down. Stop, say so, and let them decide.

Do not build a fallback you cannot test. An untested fallback is a failure that happens during the incident.

---

## Practical Takeaway

1. **Read tool failures from structure, and test the detector on captured payloads.** The reply text is for the person. The response record is for the application. A shape the detector has not seen is `Unclassified`, never a pass.
2. **Give each request one retry budget.** Count across the SDK, the application, and the gateway, not within each.
3. **Decide what a partial answer is before the first outage.** If the run shows fabricated detail on a failed lookup, the case belongs in the evaluation set, and the notice belongs in the application.

---

## Next Article

> In the next article, we move from **failure inside the agent path** to **the event-driven enterprise**, and examine **how a Foundry workload should participate through Azure Service Bus**.
