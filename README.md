# Amazon DynamoDB (dynamodb)

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

A fully managed NoSQL database service that provides fast and predictable performance with seamless scalability.

**APIs.json:** [https://aws.amazon.com/dynamodb/](https://aws.amazon.com/dynamodb/)

## Tags

- AWS
- Cloud
- Database
- Document Store
- Key-Value
- Managed Service
- NoSQL
- Serverless

## Timestamps

- **Created:** 2024
- **Modified:** 2026-05-19

## APIs

### Amazon DynamoDB API

RESTful API for interacting with DynamoDB tables and items.

- **Human URL:** [https://aws.amazon.com/dynamodb/](https://aws.amazon.com/dynamodb/)
- **Base URL:** `https://dynamodb.{region}.amazonaws.com`

#### Tags

- Database
- Items
- Managed Service
- NoSQL
- Queries
- Tables

#### Properties

- [Documentation](https://docs.aws.amazon.com/dynamodb/)
- [OpenAPI](openapi/dynamodb-openapi.yml) — [OpenAPI Specification](https://spec.openapis.org/oas/latest.html)
- [Postman Collection](collections/dynamodb.postman_collection.json) — [Postman Collection 2.1](https://schema.getpostman.com/json/collection/v2.1.0/collection.json)
- [Open Collection](collections/dynamodb.opencollection.json) — [Open Collection 1.0](https://schema.opencollection.com/opencollection/v1.0.0.json)
- [API Reference](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/Welcome.html)
- [Getting Started](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GettingStartedDynamoDB.html)
- [SDK](https://aws.amazon.com/tools/)
- [Pricing](https://aws.amazon.com/dynamodb/pricing/)
- [Console](https://console.aws.amazon.com/dynamodb/)
- [Best Practices](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html)
- [F A Q](https://aws.amazon.com/dynamodb/faqs/)
- [Status Page](https://status.aws.amazon.com/)
- [Security](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/security.html)
- [Features](https://aws.amazon.com/dynamodb/features/)
- [JSON Schema](json-schema/dynamodb-item-schema.json) — [JSON Schema](https://json-schema.org/specification)
- [JSON-LD](json-ld/dynamodb-context.jsonld) — [JSON-LD](https://www.w3.org/TR/json-ld11/)

### Amazon DynamoDB Streams API

API for capturing and processing change data from DynamoDB tables in near real-time, providing time-ordered sequences of item-level modifications.

- **Human URL:** [https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html)
- **Base URL:** `https://streams.dynamodb.{region}.amazonaws.com`

#### Tags

- Change Data Capture
- Event-Driven
- Real-Time
- Streams

#### Properties

- [Documentation](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html)
- [AsyncAPI](asyncapi/dynamodb-streams-asyncapi.yml) — [AsyncAPI Specification](https://www.asyncapi.com/docs/reference/specification/latest)
- [API Reference](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_Operations_Amazon_DynamoDB_Streams.html)
- [Postman Collection](collections/dynamodb.postman_collection.json) — [Postman Collection 2.1](https://schema.getpostman.com/json/collection/v2.1.0/collection.json)
- [Open Collection](collections/dynamodb.opencollection.json) — [Open Collection 1.0](https://schema.opencollection.com/opencollection/v1.0.0.json)

### Amazon DynamoDB Accelerator (DAX) API

API for managing DynamoDB Accelerator (DAX) clusters, an in-memory caching service that delivers microsecond response times for DynamoDB read workloads.

- **Human URL:** [https://aws.amazon.com/dynamodbaccelerator/](https://aws.amazon.com/dynamodbaccelerator/)
- **Base URL:** `https://dax.{region}.amazonaws.com`

#### Tags

- Accelerator
- Caching
- In-Memory
- Performance

#### Properties

- [Documentation](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DAX.html)
- [OpenAPI](https://api.apis.guru/v2/specs/amazonaws.com/dax/2017-04-19/openapi.yaml) — [OpenAPI Specification](https://spec.openapis.org/oas/latest.html)
- [API Reference](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_Types_Amazon_DynamoDB_Accelerator__DAX_.html)
- [Postman Collection](collections/dynamodb.postman_collection.json) — [Postman Collection 2.1](https://schema.getpostman.com/json/collection/v2.1.0/collection.json)
- [Open Collection](collections/dynamodb.opencollection.json) — [Open Collection 1.0](https://schema.opencollection.com/opencollection/v1.0.0.json)

## Common Properties

- [Blog](https://aws.amazon.com/blogs/database/category/database/amazon-dynamodb/)
- [Support](https://aws.amazon.com/premiumsupport/)
- [Terms of Service](https://aws.amazon.com/service-terms/)
- [Privacy Policy](https://aws.amazon.com/privacy/)
- [Resources](https://aws.amazon.com/dynamodb/resources/)
- [Spectral Rules](rules/dynamodb-spectral-rules.yml)
- [Vocabulary](vocabulary/dynamodb-vocabulary.yaml)
- [Features](undefined)
- [Use Cases](undefined)
- [Integrations](undefined)
- [Integrations](https://aws.amazon.com/marketplace)

## Maintainers

**Email:** aws-dynamodb@amazon.com
**URL:** https://aws.amazon.com
**Email:** kin@apievangelist.com
**URL:** https://apievangelist.com
