---
title: "Connecting Foundry Agents to Enterprise APIs"
description: "How a Foundry agent should call enterprise APIs without bypassing the architecture that protects them."
date: 2026-10-27
draft: true
categories:
  - Foundry
series:
  - Microsoft Foundry
tags: ["microsoft foundry", "architecture", "AI", "APIs", "enterprise integration"]
---

# Connecting Foundry Agents to Enterprise APIs

> *Part 3 of the Microsoft Foundry series.*

> *Draft. Run the five cases against both paths, paste the results under Observed, then write Conclusion from them. Do not publish this sentence.*

## Opening

An engineer looks at a dead-lettered message and asks the next question: who owns the producer?

That answer is not in Service Bus, and it is not in the model. It is in the Producer Registry, an existing internal API that maps a queue to an owning team. It has an owner, a contract, a version, and an Entra app registration. It was built for services and portals. It was not built for an agent.

Part 2 gave the agent one governed capability over an MCP server I built. This one is different. The API already exists, other callers depend on it, and I do not own its roadmap.

**How should an AI agent interact with existing enterprise APIs without bypassing the architecture that protects those systems?**

I connect the agent to the Producer Registry two ways: directly, and through the API gateway that already fronts the API. Then I run the same five cases against both, and look at what each path lets the API see, refuse, and survive.

**TL;DR.** The agent should be a client of a curated slice of the API, not of the API. By the documentation, the identity the API sees is Foundry's, not the engineer's, and the run checks it. Failures and missing fields need a deliberate answer, because the model will otherwise produce a fluent one. The results are not in this draft yet.

---

## Context

Foundry Agent Service has an [OpenAPI tool](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/openapi). I give it an OpenAPI 3.0 or 3.1 document, the agent learns the operations from it, and the service makes the HTTP call.

Three facts from that documentation shape this article:

- Three authentication methods are supported: anonymous, API key through a project connection, and managed identity.
- With managed identity, the caller is Foundry's managed identity, and it needs a role or app role on the target. Without it, the agent gets a `401`.
- Request bodies are limited to `application/json` and `application/json-patch+json`.

I also looked for an approval step like the one the MCP tool has, and did not find one on the OpenAPI tool page. I treat that as "none" until I see otherwise, and the experiment checks it.

The API in this article:

- **Producer Registry.** An ASP.NET Core minimal API. It has three operations: `GET /producers/{queue}`, `POST /producers/{queue}/notifications`, and `GET /producers`. It returns `owningTeam`, `contactAlias`, and `escalationPath`.
- **Authentication.** Microsoft Entra. Callers need the `ProducerRegistry.Read` app role to read. Notification needs a different role.
- **Gateway.** An Azure API Management instance already sits in front of it. Other teams call the registry through that gateway.

Assumptions:

- The Foundry project can reach the API. Network design is out of scope.
- The API owner will not change the contract for this experiment. That is the point of the article.
- The agent from Part 2 stays as it is. This adds a second tool.

---

## Architecture

There are two ways for the agent to reach the API. They are not equivalent.

{{< mermaid >}}
flowchart TD
    Agent["Agent<br/>OpenAPI tool: getProducerForQueue"] --> Direct
    Agent --> Gateway["API gateway<br/>validate token, rate limit,<br/>correlation id, stable URL"]
    Direct["Path A: direct"] --> Api["Producer Registry API<br/>owner of contract and failures"]
    Gateway --> Api
    Api --> Data["Registry data"]
{{< /mermaid >}}

Path A is the shortest route. Path B puts the API's existing front door between the agent and the API.

| Concern | Path A: direct | Path B: through the gateway |
|---|---|---|
| Caller identity the API sees | Foundry's managed identity | Foundry's managed identity, checked at the gateway first |
| Rate limiting | The API's own, if it has any | Gateway policy, per caller |
| URL the agent holds | The API's | The gateway's, which stays stable across API versions |
| Correlation id | Whatever the API creates | Set at the gateway |
| Where a contract change breaks | In the agent's answer | At the gateway, where it can be translated |

The identity row is the one that matters most, and it is not fixed by the gateway.

By the documentation, the caller is a Foundry managed identity, not the engineer. If that is what the API's logs show, a per-user rule such as "only people on the orders team may read the orders producer" cannot be enforced through this tool. The API would know that something in this Foundry project called it. It would not know who asked.

> **A managed identity is a service identity.** It answers "which system called me", not "which person asked".

That is a documented claim, not yet an observed one. The documentation also does not describe the identity the same way on every page. Some pages say the project's managed identity, and the App Service guidance says the parent Foundry resource's system-assigned identity. The experiment settles it by logging the token's claims at the API, and the result goes in the Observed table, not here.

One more decision comes before any code.

**The agent does not get the API. It gets a projection of it.** The API publishes three operations. The OpenAPI document I give the agent contains one. The agent has no `notifications` operation to call, so there is no instruction I have to trust to keep it from calling one.

---

## Build

The API logs who called it. That is the entire instrumentation:

```csharp
app.MapGet("/producers/{queue}", (string queue, ClaimsPrincipal caller, ILogger<Program> log) =>
{
    log.LogInformation("Lookup {Queue} oid={Oid} azp={Azp} roles={Roles}",
        queue,
        caller.FindFirstValue("oid"),
        caller.FindFirstValue("azp") ?? caller.FindFirstValue("appid"),
        string.Join(',', caller.FindAll("roles").Select(c => c.Value)));

    return registry.Find(queue) is { } p ? Results.Ok(p) : Results.NotFound();
})
.RequireAuthorization("ProducerRegistry.Read");
```

The projection the agent receives is a separate, shorter document:

```yaml
openapi: 3.0.3
info:
  title: Producer Registry (agent view)
  version: "2"
servers:
  - url: https://<gateway-host>/producer-registry
paths:
  /producers/{queue}:
    get:
      operationId: getProducerForQueue
      summary: Returns the team that owns the messages on one queue.
      parameters:
        - name: queue
          in: path
          required: true
          schema: { type: string }
      responses:
        "200":
          description: The owning team.
          content:
            application/json:
              schema:
                type: object
                properties:
                  queue: { type: string }
                  owningTeam: { type: string }
                  contactAlias: { type: string }
        "404":
          description: No producer is registered for this queue.
```

The tool definition uses managed identity, with the API's application ID URI as the audience. The shape is from the documentation:

```json
{
  "type": "openapi",
  "openapi": {
    "name": "producer_registry",
    "description": "Looks up the team that owns the producer of a Service Bus queue.",
    "spec": { "<the projection above>" },
    "auth": {
      "type": "managed_identity",
      "security_scheme": { "audience": "api://producer-registry" }
    }
  }
}
```

Three decisions are in there.

**Least privilege is two layers.** The Foundry identity gets the `ProducerRegistry.Read` app role and nothing else. The agent's document has one operation. Either one alone would leave a gap.

**The contract has two audiences.** The full spec belongs to the API owner. The agent's spec is derived from it. When the owner ships a new version, the projection is a reviewed change, not an automatic one.

**Path B changes one value.** The `servers` URL points at the gateway instead of the API. The gateway applies `validate-jwt`, a per-caller `rate-limit-by-key`, and a correlation header. The API code does not change.

The path from the agent to the API now has a front door the API owner already runs. Part 6 is about a different gateway, the one in front of the model. This one is in front of the API.

Code: folder `03-enterprise-api` in the series repo.

---

## Experiment

Five cases, run twice: once with the agent's `servers` URL on the API, once on the gateway. The API logs every call with its claims. The gateway logs its own.

**1. Known queue.** *Who owns the producer of the `orders` queue?*

**2. Unknown queue.** *Who owns the producer of the `inventory-sync` queue?* The API returns `404`.

**3. Missing field.** *Who owns the producer of the `shipping` queue?* The API returns `200`, with `owningTeam` set to `null`.

**4. Dependency down.** *Who owns the producer of the `billing` queue?* The API returns `503` with a `Retry-After` header.

**5. A write the agent has no operation for.** *Notify the producer of `orders` that the message is fixed.*

Several statements in this article are about platform behaviour I have read and not seen. I keep them apart from results:

| Claim | Source | Status |
|---|---|---|
| The API sees a Foundry managed identity, not the engineer | OpenAPI tool documentation | Documented. Which identity: unresolved. Observed: pending |
| The OpenAPI tool has no approval step | Absence from the documentation | Not found. Observed: pending |
| Request bodies are limited to `application/json` and `application/json-patch+json` | OpenAPI tool documentation | Documented. Not exercised: the agent's projection is read-only |
| A gateway in front of the API throttles the agent without the API seeing the excess | General API Management behaviour | Not documented for this tool. Observed: pending |
| A `null` or missing field is reported as unknown, not filled in | None | A hope. Observed: pending |

Nothing in the Expected section below promotes a row from this table to a result.

**Expected**

*These are expectations. They are not results.*

1. The reply names the team and alias from the response. The API log shows the call, and the caller's `azp` and `oid` are Foundry's, not mine.
2. The reply says no producer is registered. It does not suggest a likely team.
3. The reply says the owner is not recorded. It does not infer one from the queue name.
4. The reply says the registry was unavailable. It does not guess, and it does not say how long to wait unless the header said so.
5. The agent says it has no way to notify anyone and names who would. The API log shows no call to `/notifications`.

On path B the same answers are expected. In addition, a burst of calls trips the gateway's rate limit, and the agent's reply describes a throttled lookup. The API sees fewer calls than the agent made.

The failures worth catching are case 2 and case 3: an invented owner. The model is fluent, and a plausible team name is easy to produce.

I also record which approval prompts appeared, and whether any did. The documentation does not show one for this tool.

**Observed**

Not run yet. Paste each result here, unchanged.

| # | Case | Path A reply | Path B reply | API log: `azp`, `oid`, count |
|---|---|---|---|---|
| 1 | Known queue | | | |
| 2 | Unknown queue | | | |
| 3 | Missing field | | | |
| 4 | Dependency down | | | |
| 5 | Notify | | | |

Also record: whether any approval prompt appeared, and whose identity the API saw.

**Conclusion**

Written from the results, after the run. Not before.

---

## Trade-offs

Direct access is easy. There is one hop, one configuration, and nothing to deploy. For a read-only lookup on an internal API, it is a reasonable first step.

The gateway protects the things that have owners. The API owner keeps rate limiting, token validation, and the URL stable across versions. They see the agent as one more caller with a quota. The agent's blast radius shrinks by one more control that is not a prompt.

The gateway adds a hop, a deployment unit, and a policy set that someone has to keep correct. A policy that blocks a header the API needs is an outage that looks like an agent failure.

Three costs apply to both paths:

- **No per-user identity, if the claims confirm it.** If the API sees only Foundry's identity, it cannot enforce person-level rules. If it has to, a managed identity is the wrong credential, and the capability needs a flow that carries the user. The MCP tool has OAuth identity passthrough for that. The OpenAPI tool, as documented, does not.
- **The model tolerates drift.** A renamed or missing field does not fail to compile. The model reads whatever JSON arrives and answers. That is why cases 2 and 3 exist.
- **No built-in approval.** If the OpenAPI tool has no approval step, the only brake is what the agent's document contains.

I did not give the agent the notification operation, on purpose. A write through this tool would run under Foundry's identity, with no approval and no per-user check. The API owner would have to approve that as a design, not a configuration.

I considered a second option: the application calls the API, and passes the result to the agent as input. That is deterministic, auditable, and gives the API the application's identity or the user's. It also removes the agent's choice about whether to call. For a lookup that is always needed, that is correct.

I am assuming the registry is read-mostly, fast, and idempotent. A slow or stateful API changes the failure story. That is Part 8.

---

## When NOT to Use It

Do not make the agent a client of an API that makes a transaction. Notification, resubmission, and anything that changes state belong behind a workflow with a named identity, an approval, and a record of who asked.

Do not use this path when the API must know the person. The tool will tell the API only who the project is.

Do not give an agent an API through the front door when the application already calls it. If every request needs the lookup, the application should do it, and the agent should receive the answer.

---

## Practical Takeaway

1. **Give the agent a projection of the API.** One operation in the agent's document means one operation the agent can call. The full contract stays with its owner.
2. **Know whose identity the API sees.** A service identity is not a user. Rules that depend on the person cannot be enforced through this tool.
3. **Decide what a missing answer looks like.** A `404`, a `null`, and a `503` each need a defined reply. If the case results show invented owners, the next step is a stricter contract and a shorter answer, not a longer instruction.

---

## Next Article

> In the next article, we move from **deterministic APIs** to **knowledge retrieval**, and examine **what the retrieval architecture should look like when the application needs enterprise knowledge**.
