# Amazon DynamoDB (dynamodb)
A fully managed NoSQL database service that provides fast and predictable performance with seamless scalability.

**URL:** [Visit APIs.json URL](https://aws.amazon.com/dynamodb/)

**Run:** [Capabilities Using Naftiko](https://github.com/naftiko/fleet?utm_source=api-evangelist&utm_medium=readme&utm_campaign=company-api-evangelist&utm_content=repo)

## Tags:

 - AWS, Cloud, Database, Document Store, Key-Value, Managed Service, NoSQL, Serverless

## Timestamps

- **Created:** 2024
- **Modified:** 2026-04-18

## APIs

### Amazon DynamoDB API
RESTful API for interacting with DynamoDB tables and items.

**Human URL:** [https://aws.amazon.com/dynamodb/](https://aws.amazon.com/dynamodb/)

#### Tags:

 - Database, Items, Managed Service, NoSQL, Queries, Tables

#### Properties

- [Documentation](https://docs.aws.amazon.com/dynamodb/)
- [OpenAPI](openapi/dynamodb-openapi.yml)
- [APIReference](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/Welcome.html)
- [GettingStarted](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GettingStartedDynamoDB.html)
- [SDK](https://aws.amazon.com/tools/)
- [Pricing](https://aws.amazon.com/dynamodb/pricing/)
- [Console](https://console.aws.amazon.com/dynamodb/)
- [BestPractices](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html)
- [FAQ](https://aws.amazon.com/dynamodb/faqs/)
- [StatusPage](https://status.aws.amazon.com/)
- [Security](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/security.html)

### Amazon DynamoDB Streams API
API for capturing and processing change data from DynamoDB tables in near real-time, providing time-ordered sequences of item-level modifications.

**Human URL:** [https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html)

#### Tags:

 - Change Data Capture, Event-Driven, Real-Time, Streams

#### Properties

- [Documentation](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html)
- [AsyncAPI](asyncapi/dynamodb-streams-asyncapi.yml)
- [APIReference](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_Operations_Amazon_DynamoDB_Streams.html)

### Amazon DynamoDB Accelerator (DAX) API
API for managing DynamoDB Accelerator (DAX) clusters, an in-memory caching service that delivers microsecond response times for DynamoDB read workloads.

**Human URL:** [https://aws.amazon.com/dynamodbaccelerator/](https://aws.amazon.com/dynamodbaccelerator/)

#### Tags:

 - Accelerator, Caching, In-Memory, Performance

#### Properties

- [Documentation](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DAX.html)
- [APIReference](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_Types_Amazon_DynamoDB_Accelerator__DAX_.html)

## Common Properties

- [Blog](https://aws.amazon.com/blogs/database/category/database/amazon-dynamodb/)
- [Support](https://aws.amazon.com/premiumsupport/)
- [TermsOfService](https://aws.amazon.com/service-terms/)
- [PrivacyPolicy](https://aws.amazon.com/privacy/)
- [Resources](https://aws.amazon.com/dynamodb/resources/)

## Features

| Name | Description |
|------|-------------|
| Single-Digit Millisecond Performance | Consistent low-latency reads and writes at any scale with SSD-backed storage. |
| Global Tables | Multi-region, multi-active replication for globally distributed applications. |
| On-Demand Capacity | Automatically scale throughput capacity based on workload without capacity planning. |
| DynamoDB Streams | Capture item-level changes for real-time processing, replication, and event-driven architectures. |
| Point-in-Time Recovery | Continuous backups with restore to any second within the last 35 days. |
| Transactions | ACID transactions across multiple items and tables for complex business logic. |
| PartiQL Support | Execute SQL-compatible queries against DynamoDB tables using PartiQL language. |

## Use Cases

| Name | Description |
|------|-------------|
| Serverless Application Backend | Use as the data layer for serverless architectures with Lambda and API Gateway. |
| Gaming Leaderboards | Store and query player data, session state, and leaderboards with consistent low latency. |
| IoT Data Storage | Ingest and query high-volume time-series data from IoT devices at scale. |
| E-Commerce Shopping Cart | Store shopping cart and session data with high availability and automatic scaling. |

## Integrations

| Name | Description |
|------|-------------|
| AWS Lambda | Trigger Lambda functions from DynamoDB Streams for event-driven processing. |
| Amazon S3 | Export table data to S3 or import from S3 for analytics and data migration. |
| Amazon Kinesis | Stream DynamoDB changes to Kinesis for real-time analytics pipelines. |
| AWS CloudFormation | Provision and manage DynamoDB tables as infrastructure as code. |
| Amazon CloudWatch | Monitor table metrics, set alarms, and track performance with CloudWatch integration. |

## Artifacts

Machine-readable API specifications organized by format.

### OpenAPI

- [DynamoDB OpenAPI](openapi/dynamodb-openapi.yml)

### AsyncAPI

- [DynamoDB Streams AsyncAPI](asyncapi/dynamodb-streams-asyncapi.yml)

### JSON Schema

- [Item Schema](json-schema/dynamodb-item-schema.json)
- [Table Description Schema](json-schema/dynamodb-table-description-schema.json)
- [Query Input Schema](json-schema/dynamodb-query-input-schema.json)

### JSON-LD

- [DynamoDB Context](json-ld/dynamodb-context.jsonld)

## Capabilities

Naftiko capabilities organized as shared per-API definitions composed into customer-facing workflows.

### Shared Per-API Definitions

- [Amazon DynamoDB API](capabilities/shared/dynamodb.yaml) — 16 operations for table, item, query, batch, transaction, and backup management

### Workflow Capabilities

| Workflow | APIs Combined | Tools | Persona |
|----------|--------------|-------|---------|
| [Database Management](capabilities/database-management.yaml) | DynamoDB API | 16 | Backend Developer |

## Vocabulary

- [DynamoDB Vocabulary](vocabulary/dynamodb-vocabulary.yaml)

## Rules

- [DynamoDB Spectral Rules](rules/dynamodb-spectral-rules.yml)

## Maintainers

**FN:** Kin Lane

**Email:** kin@apievangelist.com
