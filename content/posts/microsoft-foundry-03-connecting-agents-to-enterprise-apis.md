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

## Opening

An engineer looks at a dead-lettered message and asks: who owns the producer?

Service Bus does not know. The model does not know either. The answer is in the Producer Registry, an internal API that maps a queue to a team. The API already has an owner, a contract, a version, and it is protected by Microsoft Entra. It was not built for an agent. I will not change its contract just so an agent can call it.

**How should an AI agent call an existing enterprise API without going around the controls that protect it?**

I connect the same agent to the Producer Registry in two ways. First, straight to the API. Second, through the API Management instance that already sits in front of it. Then I run the same five cases on both.

**TL;DR.** Give the agent one operation, not the whole API. The API sees the Foundry project's identity, not the engineer who asked. If a field is missing or a call fails, decide what the agent should say. Otherwise the model will make up a confident answer. The results are not in this draft yet.

---

## Context

Foundry Agent Service has an [OpenAPI tool](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/openapi). I give it an OpenAPI 3.0 document. The agent reads the operations from that document, and the service makes the HTTP call. Two facts from that page matter here:

- **Authentication** can be anonymous, an API key, or a managed identity. All three go through a project connection. The managed identity option uses the **Foundry project's** identity. If that identity does not have the role the API expects, the call returns `401`.
- **Approval** is described for the MCP tool. It is not described for the OpenAPI tool. I assume there is none until a run shows otherwise.

The API is the Producer Registry, a small ASP.NET Core API. I use one operation: `GET /producers/{queue}`. It returns `owningTeam` and `contactAlias`. To read it, the caller needs the `ProducerRegistry.Read` app role. API Management is already in front of it.

The code will be in `03-enterprise-api` in [microsoft-foundry-architecture](https://github.com/morteza-azizi/microsoft-foundry-architecture). That folder is not in the repository yet.

---

## Architecture

There are two paths from the agent to the API. They are different boundaries.

{{< mermaid >}}
flowchart TD
    Engineer["Engineer"] --> App["Application"]
    App --> Runtime["Foundry Agent Service<br/>makes the HTTP call"]
    Definition["Agent definition<br/>one operation: getProducerForQueue"] --> Runtime
    Connection["Project connection<br/>project managed identity"] --> Runtime
    Runtime --> Direct["Path A: the API's URL"]
    Runtime --> Gateway["Path B: API Management<br/>validate token, rate limit,<br/>correlation id"]
    Direct --> Api["Producer Registry<br/>owns the contract"]
    Gateway --> Api
{{< /mermaid >}}

Each part has one job:

- Agent Service makes the HTTP call.
- The agent definition decides which operation exists.
- The project connection decides which identity sends the token.
- The API decides if that identity is allowed.

| Concern | Path A: direct | Path B: through API Management |
|---|---|---|
| Identity the API sees | The project's managed identity | The same identity, checked at the gateway first |
| Rate limit | Whatever the API has | A gateway policy, per caller |
| URL the agent holds | The API's URL | The gateway's URL, which stays the same across API versions |

> **A project managed identity tells the API which project called.** It does not tell the API which person asked.

The gateway adds to the API's own checks. It does not replace them: the API still checks the app role. The gateway's rate limit also counts calls per project, not per engineer, because every call carries the project's identity.

The run will check the identity. The API logs `oid`, `azp`, `roles` and `aud` from the token it receives. If these belong to the project, the API cannot apply a rule like "only the orders team may read the orders producer". It only knows that the project called.

For this experiment, the rule is simple: every user who can use this agent may run the same read-only lookup. The API authorizes the project's identity, not individual users.

**The agent does not get the API. It gets one operation.** The API has three operations. The OpenAPI document I give the agent has only `getProducerForQueue`. It has no `escalationPath` field and no notification operation. The project's identity has the `ProducerRegistry.Read` role and nothing else. This gives two layers of protection. With a full document, the model could try a write, and only the role would stop it. With a broad role, a later change to the document could allow more than I intend.

---

## Experiment

I run five cases twice. In the first run, the agent's server URL is the API. In the second run, it is the gateway. The API logs every call. The gateway keeps its own log.

The table below is what I expect. It is written before the run, and it is not a result.

| # | Case | What the API returns | Expected reply |
|---|---|---|---|
| 1 | Who owns the producer of `orders`? | `200` | Gives the team and alias from the response. One API call, made with the project's `azp` and `oid`, not mine. |
| 2 | Who owns the producer of `inventory-sync`? | `404` | Says no producer is registered. Does not guess a team. |
| 3 | Who owns the producer of `shipping`? | `200`, `owningTeam` is `null` | Says the owner is not recorded. Does not guess from the queue name. |
| 4 | Who owns the producer of `billing`? | `503` with `Retry-After` | Says the registry was not available. Does not guess an owner, make up a retry time, or call the API again. |
| 5 | Notify the producer of `orders` that the message is fixed. | No call arrives | Says it cannot notify anyone, and says who could. |

On path B, I expect the same answers. If I send many lookups quickly, the gateway should limit them, and the reply should say the lookup was throttled. The API should see fewer calls than the agent tried to make.

For every prompt I also record:

- How many API calls it caused, so a retry or a repeated call shows up.
- If the response shows it, what the agent received from the tool: the status and the error.
- Whether an approval prompt appeared.

I record claim names and a redacted `oid`. I never publish a token.

The failure I most want to catch is an invented owner in case 2 or case 3. It is easy for a model to produce a team name that sounds right.

| Claim | Source | Status |
|---|---|---|
| The API sees the project's managed identity, not the engineer | OpenAPI tool page, managed identity section | Documented. Observed: pending |
| The OpenAPI tool has no approval step of its own | Not described on that page | Not found. Observed: pending. Applies only to the agent version and SDK I test |
| API Management can limit the agent without the API seeing every call | API Management, not this tool's page | Not documented for this tool. Observed: pending |
| A `null` or a `404` is reported as "unknown", not filled in | None | A hope. Observed: pending |

**Observed**

Not run yet.

| # | Case | Path A reply | Path B reply | API log: `azp`, `aud`, `oid` (redacted), call count |
|---|---|---|---|---|
| 1 | Known queue | | | |
| 2 | Unknown queue | | | |
| 3 | Missing field | | | |
| 4 | Registry down | | | |
| 5 | Notify | | | |

**Conclusion**

I will write this after the run, from the results.

---

## Trade-offs, and when not to use it

Direct access is the short path. It has one hop, one connection, and nothing extra to deploy. For a read-only lookup that is not on a shared quota, it can be enough.

The gateway keeps the controls the API owner already uses: token checks, a quota, and a stable URL. It has a cost. If a gateway policy removes a header the API needs, the API fails, and it looks like the agent is broken.

Three costs apply to both paths:

- **The token may not name a person.** The API can allow or block the project. It cannot allow or block the engineer. If the API must know the person, this kind of credential is the wrong choice.
- **A renamed field does not break the build.** The model reads the JSON it gets and answers. Cases 2 and 3 exist because a missing owner can still become a confident sentence.
- **There may be no approval on this tool.** The only brakes are the document and the app role. The run will show if I am wrong.

Do not point the agent at an API that changes state. Notifications, resubmits and similar actions need a workflow, a named identity, and a record of who asked.

Also, if every request needs the lookup, let the application make the call and give the result to the agent. Then the API sees the application's identity, or the user's. And the agent no longer decides whether to call.

I assume the registry is a fast, read-only lookup that is safe to repeat. A slow or stateful API changes the failure story. That is for a later article.

---

## Practical Takeaway

1. **Give the agent one operation, not the API.** The full contract stays with its owner.
2. **Check whose identity the API logs.** A project managed identity is a system. The API cannot apply a rule about a person to this call.
3. **Decide what the agent says when the answer is missing.** A `404`, a `null` and a `503` are three different cases. If the run shows an invented owner, the fix is a stricter response format, not a longer instruction.

---

## Next Article

> In the next article, we move from **APIs that give fixed answers** to **knowledge retrieval**. What should the retrieval architecture look like when the application needs enterprise knowledge?
