# Microsoft Foundry — Publishing Plan

Working plan for the ten-article series. The article briefs stay in
`microsoft-foundry-10-article-architecture-series.md`. This file is the
schedule, the tags, and the checklist.

Goal: all ten articles published by **31 December 2026**.

Target finish: **Tuesday 15 December 2026**.

## Who this series is for

The author is an architect, a consultant, and an Azure expert. That is the standing position: someone who already designs and delivers systems, and who is now going deep on Foundry in public.

This series has one job. It is the body of work for a Microsoft MVP award in Foundry.

The articles stay at architect level. They show a decision, a boundary, an experiment, and a GitHub result. They do not teach Azure from the beginning, and they do not recast the author as a Foundry beginner. The learning path is the new ground: agents, MCP, retrieval, evaluation, gateways, governance. The judgment is the judgment of someone who already lives in enterprise Azure.

MVP readers should be able to see original work: something built, something tested, something shared. A rephrased documentation page does not count.

## Cadence

Publish on **Tuesday**, every week.

Tuesday is the publish day. The readers are architects and engineers,
and the posts are also shared on LinkedIn. Monday is still inbox and
planning. Friday attention drops. 31 December itself is a Thursday, and
it is New Year's Eve, so it is not a publish day.

Each week has the same shape. The experiment starts the day the previous
article is published. The draft is due Friday. Monday is the edit, and
`draft: false` is set that day. Tuesday morning is publish.

Tuesday 22 December and Tuesday 29 December are not on the plan. They
are the recovery weeks if one article slips. One slip still finishes
before Christmas. Two slips still finish before 31 December.

Do not publish a placeholder.

## Schedule

| # | Article | Publish | Experiment | Draft due | Final edit | Draft |
|---|---|---|---|---|---|---|
| 01 | Agents | Tue 13 Oct 2026 | Mon 5 Oct – Thu 8 Oct | Fri 9 Oct | Mon 12 Oct | [01](content/posts/microsoft-foundry-01-from-first-agent-to-production-architecture.md) |
| 02 | MCP | Tue 20 Oct 2026 | Tue 13 Oct – Thu 15 Oct | Fri 16 Oct | Mon 19 Oct | [02](content/posts/microsoft-foundry-02-building-an-mcp-server.md) |
| 03 | Enterprise APIs | Tue 27 Oct 2026 | Tue 20 Oct – Thu 22 Oct | Fri 23 Oct | Mon 26 Oct | [03](content/posts/microsoft-foundry-03-connecting-agents-to-enterprise-apis.md) |
| 04 | RAG | Tue 3 Nov 2026 | Tue 27 Oct – Thu 29 Oct | Fri 30 Oct | Mon 2 Nov | [04](content/posts/microsoft-foundry-04-rag-architecture-and-trade-offs.md) |
| 05 | Evaluation | Tue 10 Nov 2026 | Tue 3 Nov – Thu 5 Nov | Fri 6 Nov | Mon 9 Nov | [05](content/posts/microsoft-foundry-05-evaluating-ai-applications.md) |
| 06 | AI Gateway | Tue 17 Nov 2026 | Tue 10 Nov – Thu 12 Nov | Fri 13 Nov | Mon 16 Nov | [06](content/posts/microsoft-foundry-06-ai-gateway.md) |
| 07 | API Center | Tue 24 Nov 2026 | Tue 17 Nov – Thu 19 Nov | Fri 20 Nov | Mon 23 Nov | [07](content/posts/microsoft-foundry-07-api-center-discovering-and-governing-apis.md) |
| 08 | Resilience | Tue 1 Dec 2026 | Tue 24 Nov – Thu 26 Nov | Fri 27 Nov | Mon 30 Nov | [08](content/posts/microsoft-foundry-08-failure-modes-and-resilience.md) |
| 09 | Service Bus | Tue 8 Dec 2026 | Tue 1 Dec – Thu 3 Dec | Fri 4 Dec | Mon 7 Dec | [09](content/posts/microsoft-foundry-09-service-bus-and-event-driven-systems.md) |
| 10 | Enterprise architecture | Tue 15 Dec 2026 | Tue 8 Dec – Thu 10 Dec | Fri 11 Dec | Mon 14 Dec | [10](content/posts/microsoft-foundry-10-from-prototype-to-enterprise-architecture.md) |

Article 10 has no new platform. Its experiment is one path through the
system the earlier articles already built. The writing window is for
synthesis, not a new product. Article 01 is the only one with more than
a week of lead time. It starts 5 October.

## Progress

- [ ] 01 Agents — Tue 13 Oct 2026
- [ ] 02 MCP — Tue 20 Oct 2026
- [ ] 03 Enterprise APIs — Tue 27 Oct 2026
- [ ] 04 RAG — Tue 3 Nov 2026
- [ ] 05 Evaluation — Tue 10 Nov 2026
- [ ] 06 AI Gateway — Tue 17 Nov 2026
- [ ] 07 API Center — Tue 24 Nov 2026
- [ ] 08 Resilience — Tue 1 Dec 2026
- [ ] 09 Service Bus — Tue 8 Dec 2026
- [ ] 10 Enterprise architecture — Tue 15 Dec 2026

## How each article is filed

| Field | Value | Role |
|---|---|---|
| Category | `Foundry` | The shelf. Later Foundry writing can live here without joining this ten-part arc. |
| Series | `Microsoft Foundry` | Reading order, series page, and previous/next links. |
| Tags | Below | Technical topics a reader can follow across the whole site. |

Every article carries the same three tags, then only the topics that
article actually investigates:

- `microsoft foundry`
- `architecture`
- `AI`

| Article | Extra tags |
|---|---|
| 01 Agents | `agents`, `azure` |
| 02 MCP | `MCP`, `agents` |
| 03 Enterprise APIs | `APIs`, `enterprise integration` |
| 04 RAG | `RAG` |
| 05 Evaluation | `evaluation` |
| 06 AI Gateway | `AI gateway` |
| 07 API Center | `API Center`, `governance` |
| 08 Resilience | `reliability` |
| 09 Service Bus | `service bus`, `messaging` |
| 10 Enterprise architecture | `distributed systems` |

`azure`, `RAG`, `enterprise integration`, `governance`, `service bus`,
`messaging`, and `distributed systems` already exist on the site. Reuse
those spellings so Hugo does not create a second tag page.

The blank template for a later article is `archetypes/microsoft-foundry.md`.

## Checklist for every article

Copy this list when an article enters its experiment window.

- [ ] Experiment is running, and the result can be told as Expected, Observed, Conclusion
- [ ] Opening states the question, why it matters, what was built, and what the reader learns
- [ ] Context names the Foundry capability, the scenario, and the assumptions
- [ ] Architecture, with a diagram if the boundary is easier to see than to read
- [ ] Build shows the decision and the integration point, not a full tutorial
- [ ] Trade-offs name what failed, what got more complex, and what was considered instead
- [ ] When NOT to Use It is a real case, not a disclaimer
- [ ] Practical takeaway is three concrete points
- [ ] Next-article bridge is one sentence, except article 10, which closes the series
- [ ] Category is `Foundry`, series is `Microsoft Foundry`, tags match the table above
- [ ] Placeholder line at the top is removed
- [ ] `draft: false` on the Monday edit
- [ ] Published Tuesday morning, then shared

## After 15 December 2026

- [ ] Read the ten articles in order and check that each bridge still matches the next article
- [ ] Record the strongest experiments, the strongest insights, and the reusable GitHub assets
- [ ] List unanswered questions, follow-up articles, and talks worth proposing
