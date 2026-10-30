# Motadata

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **Not from the company, and here with a question?** You are welcome here — we would rather be the
> front line and point you the right way than have a good report go nowhere. What this repository
> can answer is narrow, though, so it is worth knowing who you are actually looking for:
>
> - **A question about how the API works, an account, billing, or a bug in the service** — that is
>   the company's own support, not us. We profile this API; we do not operate it and cannot see
>   your account.
> - **A bug in an open-source project we only catalog** — file it on that project's own repository.
>   This has happened with a real and correct bug report that reached us instead of the people who
>   could fix it, which helped nobody.
> - **Anything about this listing itself** — the description, the tags, the rating, a missing or
>   wrong artifact — is ours. Open an issue here.
> - **Not sure, or something general about API Evangelist or APIs.io** — open an issue on the
>   [APIs.io Inbox](https://github.com/api-search/inbox) and we will route it.
>
> **This repository contains no software, and we will never ask you to download anything.** There is
> no build, release, installer, or binary here — only text and machine-readable API descriptions, so
> there is nothing here that can be "corrupt" or need "repairing". Any issue, comment, or email
> claiming otherwise and offering a download link is not from us and is hostile. Do not follow the
> link; it is a lure. Report it to GitHub and, if you like, tell us at
> [info@apievangelist.com](mailto:info@apievangelist.com) so we can take it down.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

Motadata (Mindarray Systems) is a global IT software company running two AI-native platforms on a shared deep-learning foundation: ObserveOps, a unified observability and AIOps platform that brings metrics, logs, traces, flows and topology into one backend, and ServiceOps, an ITIL-aligned IT and enterprise service management suite covering service desk, CMDB, asset management, patch management and workflow automation. Both products are deployed in the customer's own environment, so every API host, credential and rate limit belongs to the customer rather than to Motadata. ObserveOps exposes a REST API at /api/v1 across 96 configuration resource types plus a Vert.x EventBus websocket for live streams, and ships an official agent-first CLI that doubles as an MCP server with 22 tools and five packaged Agent Skills. ServiceOps exposes a documented REST API over OAuth 2.0 or an API key across request, problem, change, release, CMDB, asset and service-catalog objects. Motadata publishes no OpenAPI for either product.

**Areas:** AIOps · Monitoring · Observability · ITSM · Service Desk · Network Monitoring · Log Management · IT Operations · Application Performance Monitoring · OpenTelemetry

## Motadata on APIs.io

Everything profiled for Motadata is published on APIs.io rather than in this repository. Start at the provider page — **[apis.io/providers/motadata/](https://apis.io/providers/motadata/)** — or go straight to one of the APIs or properties below.

### APIs (2)

- [Motadata ObserveOps](https://apis.io/apis/motadata/observeops/) — ObserveOps (formerly Motadata AIOps) is a unified observability platform correlating metrics, logs, traces, flows and topology with anomaly detection that…
- [Motadata ServiceOps](https://apis.io/apis/motadata/serviceops/) — ServiceOps is Motadata's AI-powered, ITIL-aligned IT and enterprise service management suite. Its REST API lets third-party systems create, read, update and…

### Published properties

Each property below is a page on APIs.io carrying the machine-readable artifact and what it says.

#### MCP servers (1)

Model Context Protocol servers that expose the API to agents.

- [observeops](https://apis.io/servers/motadata/observeops/)

#### Security (2)

Authentication, trust centre, and vulnerability disclosure.

- [Motadata Authentication](https://apis.io/security/motadata/motadata-authentication/)
- [Motadata Domain Security](https://apis.io/security/motadata/motadata-domain-security/)

#### OAuth scopes (1)

The permissions the API asks for.

- [Motadata Scopes](https://apis.io/scopes/motadata/motadata-scopes/)

#### Plans and pricing (1)

Published plans, tiers and prices.

- [Motadata Plans Pricing](https://apis.io/plans/motadata/motadata-plans-pricing/)

#### Rate limits (1)

Published request limits.

- [Motadata Rate Limits](https://apis.io/rate-limits/motadata/motadata-rate-limits/)

#### FinOps (1)

What the API costs to run, by operation.

- [Motadata Finops](https://apis.io/finops/motadata/motadata-finops/)

## Where to find this API

Links to the company's own pages. This repository publishes no copy of their specifications — the canonical document is theirs and lives at the URLs below.

- [Website](https://www.motadata.com)
- [Documentation](https://docs.motadata.com)
- [Getting started](https://www.motadata.com/getting-started)
- [Sign up](https://www.motadata.com/free-trial)
- [Support](https://support.motadata.com/support-portal)
- [Blog](https://www.motadata.com/blog)
- [Terms of service](https://www.motadata.com/terms-of-service)
- [Privacy policy](https://www.motadata.com/privacy-policy)
- [GitHub](https://github.com/motadata)

## API Evangelist rating

**Kin Score 37.1** — thin · rubric 0.24.0 · scored 2026-10-17

| Facet | Score |
|---|---|
| Discoverability | 71.7 |
| Contract quality | 0.0 |
| Contract governance | 18.2 |
| Operational transparency | 39.5 |
| Developer ergonomics | 64.3 |
| Access clarity | 34.2 |

**Agent readiness 15.7** — agent-aware

| Agent-readiness dimension | Result |
|---|---|
| Spec Presence | — |
| Agentic Access | — |
| Reversibility Documented | — |
| MCP Server | documented (partial credit) |
| Auth Clarity | bearer (partial credit) |
| Idempotency | — |
| Error Semantics | derived (partial credit) |
| OpenAPI Examples | — |
| Rate Limit Signal | documented (partial credit) |
| Event Surface Described | — |
| Agent Skills | yes |
| Well Known Catalog | yes |
| Consent Identity | — |
| Agent Card | — |
| Dry Run Mode | — |
| Delegated Identity | — |
| Protected Resource Metadata | — |
| Dynamic Client Registration | — |
| Agentic Commerce | — |

**Regulatory alignment 34.3** — Horizontal (data, software, accessibility, platform)

The rubric is published at [github.com/api-evangelist/kin-score](https://github.com/api-evangelist/kin-score). The per-check breakdown behind these numbers — which checks were earned, partial or missed — along with score history and how this provider compares to its peers, is available at [apis.io/providers/motadata/](https://apis.io/providers/motadata/).

## How this profile is made

API Evangelist profiles a company's public API surface the way a developer would find it: the website, developer portal and documentation, the specifications it publishes, its public repositories, and its pricing, status and changelog pages. From that it derives the properties above — contracts split per API, workflows, schemas, collections, governance rules, plans and rate limits — and scores them against the published [Kin Score rubric](https://github.com/api-evangelist/kin-score).

The artifacts are published on [APIs.io](https://apis.io), not in this repository. This repository is the public record for Motadata: the place to say something is wrong, missing or out of date, and to ask for a re-score or a removal.

## Open an issue

- **Something about this listing** — a wrong description, a missing API, a stale artifact or a question about the rating: [open an issue on this repository](https://github.com/api-evangelist/motadata/issues/new).
- **A new API, or one that is not in the catalog yet** — [submit it on APIs.io](https://apis.io/add/).
- **Anything else** — [the APIs.io inbox](https://github.com/api-search/inbox/issues/new) or [info@apievangelist.com](mailto:info@apievangelist.com).
