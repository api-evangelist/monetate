# Monetate

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
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

Monetate is an enterprise experience-optimization and personalization platform for ecommerce and digital
businesses — A/B and multivariate testing, automated 1:1 personalization, product and content
recommendations, personalized search, dynamic bundles, social proof and product badging — operating as part
of Kibo Commerce.

- Website: https://monetate.com/
- Developer hub: https://developer.monetate.com/
- Knowledge base: https://docs.monetate.com/docs
- Status: https://monetate.statuspage.io

## Published APIs

| API | Base | Spec |
|---|---|---|
| Engine API | `https://engine.monetate.net/api/engine/v1` | Swagger 2.0, 1 operation (`POST /decide/{retailerShortname}`) |
| Data API | `https://api.monetate.net/api/data/v1/{retailerShortname}/production` | OpenAPI 3.0.1, 12 operations |
| Metadata API | `https://api.monetate.net/api/metadata/v1/{retailerShortname}/production` | OpenAPI 3.0.1, 10 operations |
| Auth API | `https://api.monetate.net/api/auth/v0` | OpenAPI 3.0.1, 1 operation |

All four specifications are real, provider-authored documents harvested from the Archbee document store
behind developer.monetate.com. Monetate does not serve them from a stable public URL of its own — the
developer hub is a client-rendered Next.js site whose `/openapi.json` returns an HTML "Not found" shell —
so `openapi/_original/` records the exact upload each was taken from.

## What this profile records

`apis.yml` is the index. Alongside the specs, this repo holds the authentication model (a two-scheme JWT →
bearer-token exchange), API conventions, the derived error catalog, the entity/relationship data model,
lifecycle and versioning, the dated changelog, published packages, well-known probes, the MCP server
Monetate publishes on its marketing site, domain-security probes, trust-center findings, and four packaged
Agent Skills grounded in real operations.

Notable honest findings, recorded rather than smoothed over: no published rate limits or rate-limit
headers, no idempotency support on any write, no deprecation policy or Sunset headers, no
`/.well-known/security.txt` or vulnerability-disclosure channel, a trust center with zero named
certifications, quote-only pricing with no published plans, and no `operationId` on 23 of the 24 published
operations. See each artifact for the evidence.
