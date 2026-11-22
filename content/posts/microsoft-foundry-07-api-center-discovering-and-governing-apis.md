---
title: "Microsoft Foundry and API Center: Discovering and Governing APIs"
description: "How a Foundry application can discover the right enterprise APIs inside a governed catalogue."
date: 2026-11-24
draft: true
categories:
  - Foundry
series:
  - Microsoft Foundry
tags: ["microsoft foundry", "architecture", "AI", "API Center", "governance"]
---

# Microsoft Foundry and API Center: Discovering and Governing APIs

> *Part 7 of the Microsoft Foundry series.*

> *Draft. Run the five cases, paste the catalogue responses and replies under Observed, then write Conclusion from them. Do not publish this sentence.*

## Opening

So far I have chosen every capability by hand. I picked the MCP server in Part 2 and wrote the API projection in Part 3. I decided what the agent could call.

That does not survive the second team. Someone else is going to build an assistant, and they will need to know which APIs exist, which one is current, who owns it, and whether they may use it. If the answer is a chat in the platform channel, they will find a different answer from each person they ask.

An agent can read a catalogue too. That is the temptation: point the agent at the list of APIs and let it find what it needs.

**How can an AI application discover the right enterprise APIs without creating another uncontrolled integration surface?**

This article registers the assets from the earlier parts in Azure API Center, including one API that should not be used. I look at what discovery returns, what an agent does with it, and what is not changed by it.

**TL;DR.** A catalogue answers which APIs exist, which are current, and who owns them. It is not permission to call them. The right place for discovery is at design time, where a person turns an entry into a tool. Giving the agent runtime discovery means its capability can change without a new version. The results are not in this draft yet.

---

## Context

Azure API Center is an inventory of APIs and AI assets, with metadata: versions, environments, deployments, owners, and a lifecycle stage per version. It is not a gateway. It does not sit in the request path.

Three capabilities matter here:

- **A private tool catalogue in Foundry.** MCP servers registered in API Center appear in Foundry under Build and Tools. This is a preview feature. Developers need at least the `Azure API Center Data Reader` role to see it, and the documentation notes that role assignments can take up to 24 hours to propagate. See [Create a private tool catalog](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/private-tool-catalog).
- **An authorization configuration per registered server.** API key, OAuth, or HTTP bearer. The same page notes that a key stored in Key Vault is not used by Foundry automatically; the developer provides it when configuring the connection.
- **An API Center MCP endpoint.** It exposes the registry to agents through two tools, `mcp_registry_search` and `mcp_registry_fetch`. It can be enabled only on the Standard tier. See [Discover APIs with the API Center MCP server](https://learn.microsoft.com/en-us/azure/api-center/discover-catalog-mcp-server).

Foundry's MCP documentation adds a design hint: after selecting a catalogue server, limit the enabled tools to the subset the agent needs.

The catalogue in this article holds four entries:

| Asset | Stage | Note |
|---|---|---|
| `producer-registry` v1 | Deprecated | Replaced by v2. Still running. |
| `producer-registry` v2 | Production | The current contract from Part 3. |
| `deadletter-mcp` | Production | The server from Part 2. |
| `producer-notify` | Design | Not built. Registered so the plan is visible. |

Assumptions:

- The API Center is on the Standard tier, so the MCP endpoint is available.
- I have Contributor on the API Center, and a second test identity with `Azure API Center Data Reader` assigned a day before the run.
- Catalogue entries reflect reality only as well as someone keeps them updated.

---

## Architecture

There are two different things an agent might do with a catalogue, and the boundary is between them.

{{< mermaid >}}
flowchart TD
    Owner["API owner"] -->|"registers, sets lifecycle"| Catalogue["API Center<br/>inventory, stage, owner"]
    Builder["Builder"] -->|"1. searches"| Catalogue
    Catalogue -->|"2. entry"| Builder
    Builder -->|"3. creates a tool<br/>allowed_tools, auth"| Agent["Agent version<br/>fixed tool list"]
    Agent -->|"4. calls"| Api["API or MCP server<br/>enforces its own access"]
    Discovery["Discovery agent<br/>registry tools only"] -.->|"reads"| Catalogue
{{< /mermaid >}}

Steps 1 to 3 are discovery. Step 4 is invocation. They have different owners.

| Concern | Owner |
|---|---|
| What exists, which version is current, who owns it | The API owner, recorded in API Center |
| Whether this agent should have this tool | A builder, in a reviewed agent version |
| Whether this caller may call this API now | The API or server, at runtime |
| Which tools a discovery agent can read | A read-only registry connection |

The line I draw is that **discovery informs a decision. It does not carry one out.**

In Part 1 the agent definition is a versioned asset: model, instructions, tools. An agent that discovers tools at runtime has a tool list that changes by the day. No reviewed version describes it.

> **The catalogue lists what exists. The agent version lists what is allowed.**

So the design has two kinds of agent. A discovery agent can search the catalogue. It has the two registry tools and nothing else. Agents that do the work have their tools fixed at design time, by a person, from what the catalogue showed. A deprecated entry and a design-stage entry stay visible, and neither is available to call.

---

## Build

Registering the assets is the work of the API owners. The part that belongs to this series is how an agent reads the result.

Each entry carries the owner, the stage, the deployment URL, and how to authenticate. `Manage Access`, in preview, restricts who sees a server in the organizational catalogue.

The discovery client is a plain MCP client, with a bearer token for an identity that can read the catalogue. Calling the registry is the same as any other MCP server:

```csharp
var result = await client.CallToolAsync(
    "mcp_registry_search",
    new Dictionary<string, object?> { ["query"] = "producer owner" });
```

The same call from an identity without `Azure API Center Data Reader` is a case in the experiment.

The discovery agent is attached to that endpoint with the registry tools only, and `allowed_tools` set:

```json
{
  "type": "mcp",
  "server_label": "api_catalogue",
  "server_url": "https://<service>.data.<region>.azure-apicenter.ms/mcp",
  "allowed_tools": ["mcp_registry_search", "mcp_registry_fetch"],
  "require_approval": "never"
}
```

Approval is off, because both tools only read. I would not turn it off for a tool that writes.

The discovery agent's instructions state what it is for, and what it is not for:

```text
You help a builder choose an API from the catalogue.
Report each result with its lifecycle stage and owner.
Prefer production stage. Say when only deprecated or design-stage entries match.
You cannot call any API. Say who connects a tool to an agent.
```

Code: [07-api-center](https://github.com/morteza-azizi/microsoft-foundry-architecture/tree/main/07-api-center) in the series repo.

---

## Experiment

Five cases.

**1. Search.** Call `mcp_registry_search` for `producer owner` directly from code.

**2. Which API.** Ask the discovery agent: *Which API tells me who owns the producer of a queue, and which should I use?*

**3. Discoverable is not callable.** Ask the discovery agent: *Notify the producer of the `orders` queue using the notify API.*

**4. Access.** Run case 1 from the second identity, before and after the role assignment has been in place for a day.

**5. Freshness.** Change `producer-registry` v1 from Deprecated to Retired in the catalogue. Repeat case 1. Then call the lookup from an agent that was wired to v1 in Part 3.

This article depends on documented platform behaviour, and some of it is preview. I keep it apart from results:

| Claim | Source | Status |
|---|---|---|
| Registry search results include asset type, lifecycle state, and description | API Center MCP documentation | Documented |
| The API Center MCP endpoint needs the Standard tier | API Center MCP documentation | Documented |
| Catalogue visibility needs `Azure API Center Data Reader`, and assignments can take up to 24 hours | Private tool catalogue documentation (preview) | Documented |
| `Manage Access` restricts who sees a server in the organizational catalogue | Build and register documentation (preview) | Documented |
| A Foundry agent can call the API Center MCP endpoint as an MCP tool, and how it authenticates | None | Not found. If a connection is needed, `require_approval: never` and the tool definition above change |
| A change to lifecycle stage in the catalogue reaches an agent that was wired earlier | None | Not found. I expect it does not. Case 5 checks |
| The discovery agent recommends the production entry over a deprecated one | None | A hope |

**Expected**

*These are expectations. They are not results.*

1. The results include v1, v2, and possibly the design-stage notify API, each with a stage, as the documentation says the results include lifecycle state.
2. The reply recommends v2, notes that v1 is deprecated, and does not suggest the design-stage API.
3. The agent says it can only search, that the notify API is a design-stage entry that does not exist yet, and that someone has to connect a tool to an agent. It makes no call.
4. Before the assignment, the second identity gets nothing or an authorization error. After a day, it gets the results. If it does not, the propagation delay is the first suspect.
5. The search now reports v1 as Retired. The old agent still calls v1. The catalogue changed. The agent version did not.

The failure worth catching is in case 2: the agent recommends the deprecated entry because it matched the query better. Or in case 3: it describes the notify API as if it works.

Case 5 is the one I expect to be uncomfortable. It shows a catalogue update that governs nothing already wired.

**Observed**

Not run yet. Paste each result here, unchanged.

| # | Case | Result |
|---|---|---|
| 1 | Search | |
| 2 | Which API | |
| 3 | Notify | |
| 4a | Access, before the role has been in place | |
| 4b | Access, a day after | |
| 5a | Search after the stage change | |
| 5b | Old agent after the stage change | |

**Conclusion**

Written from the results, after the run. Not before.

---

## Trade-offs

A catalogue gives the agent, and the builders, facts that have owners: what exists, which version is current, what stage it is in, and who to ask. A new team gets one place to look instead of one person to ask.

It does not guarantee anything at runtime. An entry says a deployment exists. The API still checks whether the caller is allowed in. A registered API can be down. An approved entry can have a stale URL.

The costs are lifecycle and ownership:

- **Someone has to keep it true.** A catalogue that lags reality is worse than none, because it carries authority.
- **Registration is a process.** Every API needs an owner willing to maintain the entry.
- **The MCP endpoint needs the Standard tier.** That is a cost decision before it is an architectural one.
- **Catalogue changes do not propagate.** Case 5 shows whether an update reaches anything that was already wired.

The alternative is a fixed list in the agent definition, and discovery as a conversation. That is enough for a small system. It stops working when the number of APIs, or of teams, outgrows what a few people can remember.

---

## When NOT to Use It

Do not add a catalogue when the set of APIs is small, stable, and owned by the application. A reviewed list in the agent definition is simpler and just as governed.

Do not give a production agent runtime discovery. The tool list is part of what was reviewed. If the list can change on its own, the review described something else.

Do not treat catalogue access as API access. `Azure API Center Data Reader` lets someone see an entry. It does not let them call it.

---

## Practical Takeaway

1. **Separate discovery from invocation.** The catalogue tells a builder what exists. The agent version says what is allowed.
2. **Make stage visible and refuse the wrong ones.** Deprecated and design-stage entries should be findable and clearly not available.
3. **Test the update.** A catalogue change that does not reach anything wired to the old entry is a gap between what the catalogue says and what runs.

---

## Next Article

> In the next article, we move from **governed discovery** to **failure**, and examine **what happens when the agent, model, tool, API, or another dependency fails**.
