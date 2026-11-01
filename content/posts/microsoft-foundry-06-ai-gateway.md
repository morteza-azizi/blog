---
title: "AI Gateway for Foundry-Based Applications"
description: "What changes when Foundry access becomes a shared platform capability instead of a direct model call."
date: 2026-11-17
draft: true
categories:
  - Foundry
series:
  - Microsoft Foundry
tags: ["microsoft foundry", "architecture", "AI", "AI gateway"]
---

# AI Gateway for Foundry-Based Applications

> *Part 6 of the Microsoft Foundry series.*

> *Draft. Run the four cases, paste the status codes, headers, and gateway logs under Observed, then write Conclusion from them. Do not publish this sentence.*

## Opening

Until now the assistant has been the only workload in its Foundry project. That is not how the platform will be used.

Another team wants a batch summarizer in the same Foundry resource. It will run overnight and use as many tokens as it can get. My assistant runs when an engineer has a failed message, which is when someone is already waiting. If both draw on the same model capacity, the summarizer decides whether my engineer gets an answer.

A model deployment has a quota. It has no concept of whose request is more important.

**What changes when AI access becomes a shared platform capability rather than a direct application-to-model connection?**

This article puts the Foundry AI Gateway in front of the project. Then I give the assistant's project and the summarizer's project separate token limits, and run four cases: a burst that exceeds a limit, a quota that runs out, the MCP tool from Part 2, and the API tool from Part 3.

**TL;DR.** The gateway enforces limits per project and gives one place to see usage. It does not know who the engineer is, it does not cover every tool, and its `429` is the client's problem to handle. The results are not in this draft yet.

---

## Context

Foundry's AI Gateway is Azure API Management behind the scenes. The [configuration page](https://learn.microsoft.com/en-us/azure/foundry/configuration/enable-ai-api-management-gateway-portal) states what I rely on:

- It is enabled at the Foundry resource level. Projects are added to it. New projects are enabled by default. Existing projects must be added by hand.
- Limits apply per project: tokens per minute, and a token quota.
- An existing API Management instance must be a v2 tier, in the same tenant and subscription. The instance created for you is Basic v2, which the page describes as for development and testing.
- A request that exceeds the tokens-per-minute limit returns `429 Too Many Requests`. A request that exceeds the token quota returns `403 Forbidden`.

The governance page for [MCP tools](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/governance) adds the limits that matter for this series. It is a preview feature. Only new MCP tools created in the Foundry portal are routed through the gateway, and only those that do not use managed OAuth. Existing tools are not mediated automatically. OpenAPI tools and several Foundry-based tools are not supported. The gateway does not log tool traces.

That last list is the experiment. The assistant has an MCP tool from Part 2 and an OpenAPI tool from Part 3.

The underlying APIM policy for token limits is `llm-token-limit`. I do not hand-write it here, because the portal sets these limits per project, but it shows what is being enforced:

```xml
<llm-token-limit counter-key="@(context.Subscription.Id)"
                 tokens-per-minute="2000"
                 estimate-prompt-tokens="true" />
```

Assumptions:

- Two projects share one Foundry resource: `triage` (the assistant) and `summarizer` (another team's workload).
- Both call the same model deployment.
- Token limits are set low enough to hit in a short test. They are not production numbers.

---

## Architecture

The gateway moves a decision from each application to a shared place. It does not move all decisions.

{{< mermaid >}}
flowchart TD
    Triage["triage project<br/>assistant"] --> Gateway["AI Gateway<br/>API Management"]
    Summ["summarizer project<br/>batch workload"] --> Gateway
    Gateway -->|"limits per project<br/>logs and metrics"| Model["Model deployment"]
    Triage -.->|"MCP tool created before gateway"| Mcp["MCP server"]
    Triage -.->|"OpenAPI tool"| Api["Producer Registry"]
    Gateway -->|"MCP tool created after gateway"| Mcp
{{< /mermaid >}}

The solid lines go through the gateway. The dotted lines are the ones that, by the documentation, do not.

| Belongs in the gateway | Stays in the application |
|---|---|
| Token limits and quota per project | Which engineer is asking |
| One place to see usage and rejected calls | Whether this person may peek at this queue |
| Policies on new MCP tool traffic | Approval of a tool call |
| Routing across deployments, as an API Management capability rather than a Foundry portal setting | What to do when the answer is `429` |
| | The meaning of an incomplete answer |

The left column is about **capacity and consistency**. The right is about **meaning**. A gateway sees tokens and status codes. It does not see "the engineer asked about the wrong queue".

> **A gateway limits the project, not the person.** Every engineer using the assistant shares the project's limit.

Where the limit is set matters. A project-level limit protects the summarizer from the assistant and the assistant from the summarizer. It does not protect one engineer from another inside the assistant.

---

## Build

Three steps, mostly configuration.

1. Enable the AI Gateway on the Foundry resource. Use an existing v2 APIM instance, or accept the Basic v2 one created for development.
2. Add both projects. Set the `triage` project to a higher token limit and the `summarizer` project to a lower one.
3. Confirm routing. Make a call, then check API Management metrics for the request count and the `ApiManagementGatewayLogs` table in Log Analytics.

Confirmation comes first, because a gateway that is configured but bypassed gives the same answers as no gateway. The documentation lists the symptom: requests bypass the gateway when the project was not enabled before the calls were made.

The test harness uses a raw HTTP call instead of the SDK. An SDK that retries on `429` hides the status code I want to see.

```csharp
using var http = new HttpClient();
http.DefaultRequestHeaders.Authorization = new("Bearer", token);

var calls = Enumerable.Range(0, 40).Select(async i =>
{
    var res = await http.PostAsJsonAsync($"{projectEndpoint}/openai/v1/responses",
        new { model = deployment, input = prompt, max_output_tokens = 200 });

    return new
    {
        Call = i,
        Status = (int)res.StatusCode,
        RetryAfterSeconds = res.Headers.RetryAfter?.Delta?.TotalSeconds,
        Headers = res.Headers.Select(h => h.Key).ToArray()
    };
});

foreach (var r in await Task.WhenAll(calls)) Console.WriteLine(r);
```

I record the header names on every response. Which headers the gateway adds, and whether `Retry-After` is one of them, is a fact I want from the run.

For the tool cases, I create a new MCP tool for the same server from Part 2 after the gateway is in place, and leave the existing one untouched. The documentation says routing is decided when the tool is created.

Code: folder `06-ai-gateway` in the series repo.

---

## Experiment

Four cases. For each one I read API Management's request count before and after.

**1. Burst above the project limit.** Forty parallel calls from `triage` in under a minute, with the limit set so that about half should fit. At the same time, one call per ten seconds from `summarizer`.

**2. Quota exhausted.** Set a small token quota on `triage`. Run calls until it is spent. Then make one more.

**3. MCP tools, before and after.** Run the assistant's MCP call from Part 2 through the old tool and through a newly created one. Compare the tool endpoint shown in Foundry, and the gateway's request count.

**4. The OpenAPI tool.** Run the producer lookup from Part 3 and read the gateway's request count.

Most of this article rests on documented behaviour that I have not yet seen. I keep it apart from results:

| Claim | Source | Status |
|---|---|---|
| Exceeding tokens per minute returns `429`. Exceeding the token quota returns `403` | AI Gateway configuration page | Documented. Observed: pending |
| Limits apply per project | AI Gateway configuration page | Documented. Observed: pending (case 1, the `summarizer` project) |
| Existing projects must be added to the gateway by hand | AI Gateway configuration page | Documented. Not tested here |
| Only new MCP tools, not using managed OAuth, are routed through the gateway | MCP governance page (preview) | Documented. Observed: pending (case 3) |
| OpenAPI tools are not supported by gateway governance | MCP governance page (preview) | Documented. Observed: pending (case 4) |
| The gateway does not log tool traces | MCP governance page (preview) | Documented. Observed: pending |
| The project endpoint is unchanged once the project is enabled | Implied by the verification steps | Not stated. Observed: pending |
| Which headers the gateway adds, including `Retry-After` | None | Unknown. Observed: pending |

Because the governance page describes a preview feature, any row marked documented can be out of date by the time I run it. A result that disagrees with the documentation is recorded as the result, and goes in the Part 10 corrections ledger.

**Expected**

*These are expectations. They are not results.*

1. Roughly the first calls succeed. The rest return `429`. `summarizer`'s calls are unaffected by `triage`'s burst, and the reverse. The gateway log shows the rejected calls.
2. The call after the quota is spent returns `403`, not `429`. A client that retries on `403` will retry until it is told to stop. A client that retries on `429` after the header's delay will recover.
3. The newly created tool shows a gateway URL, and the gateway count rises when it is called. The old tool still shows the direct server URL, and the gateway count does not change.
4. The gateway count does not change for the OpenAPI call. The call succeeds without it.

The failure worth catching is silent: believing the gateway governs the tools because it governs the model.

**Observed**

Not run yet. Paste each result here, unchanged.

| # | Case | Status codes (counts) | Headers seen | Gateway count before / after |
|---|---|---|---|---|
| 1 | Burst | | | |
| 2 | Quota | | | |
| 3a | MCP tool, old | | | |
| 3b | MCP tool, new | | | |
| 4 | OpenAPI tool | | | |

Also record: gateway SKU, whether the `summarizer` project was affected by the `triage` burst, and which log table had the entries.

**Conclusion**

Written from the results, after the run. Not before.

---

## Trade-offs

The gateway makes capacity consistent. A team cannot take the whole deployment, and the limit is a setting somebody can read. Usage and rejected calls are in one place. For multi-team platforms, that is what a direct connection cannot express.

It adds to the path. Every model call has another hop, another service, and another thing that can be down. The gateway must be at least as available as the thing it protects, and I have not measured that.

Complexity grows in specific places:

- **Two kinds of rejection.** `429` means wait. `403` on a quota means stop. A client that treats both as "retry" is wrong in one of them.
- **Partial coverage.** Some traffic goes through and some does not. If an auditor asks "what did the agent call?", the gateway cannot answer for tools that bypass it, and the documentation says it does not log tool traces.
- **A setting made once.** Tool routing is decided at creation. A tool created yesterday is not governed by a policy added today.
- **Limits are per project.** A fair split between two projects is not a fair split between users.

I assume shared policy is wanted, and that someone owns it. A gateway nobody owns is a rate limit nobody remembers setting.

The alternative I considered is the one that needs no gateway: a separate model deployment per project, each with its own quota. It contains the summarizer without an extra hop, and it is the first thing I would try. It costs more deployments to manage and gives no single view of usage. At two projects, it may be the better design. At twenty, the gateway earns its place.

---

## When NOT to Use It

Do not add a gateway when one application talks to one deployment and one team owns both. A quota on the deployment is the same control with less to operate.

Do not put business rules in it. "Only the orders team may peek at the orders queue" is a rule about people and queues. It belongs in the MCP server from Part 2.

Do not treat it as an audit log of the agent. It sees model calls and some tool traffic. It does not see all of what the agent did.

---

## Practical Takeaway

1. **Use the gateway for capacity.** Per-project limits and one view of usage are what it does well. Check its coverage against every tool you attached.
2. **Handle `429` and `403` differently in the client.** Wait and retry on one. Stop on the other. Part 8 puts that into a policy.
3. **Know what it does not cover.** The documentation lists some of the tools the assistant uses as outside it. Case 3 and case 4 check whether the documentation is right.

---

## Next Article

> In the next article, we move from **the gateway as a control point** to **API Center**, and examine **how an AI application discovers enterprise APIs without opening another uncontrolled integration surface**.
