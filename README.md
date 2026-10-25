# OTA Data Platform on AWS

This repository documents a reference architecture for ingesting, validating, storing, transforming, governing, and serving connected-vehicle / OTA data on AWS.

> **Status:** Architecture concept / design reference. Service names, data contracts, retention rules, SLAs, RTO/RPO targets, and account boundaries should be validated before implementation.

## Architecture diagram

![OTA data platform architecture](docs/ota-aws-architecture.png)

> Keep the editable diagrams.net source beside the exported image, for example: `diagrams/ota-aws-architecture.drawio`.

## 1. Architecture goals

The platform is designed to:

- accept data from APIs, operational databases, graph data, applications, and vendor file feeds;
- support both event-driven streaming and scheduled/batch ingestion;
- keep an immutable, replayable source copy in Amazon S3;
- validate payloads and quarantine failed records before downstream processing;
- organize data into raw, curated, and serving zones using open lakehouse tables;
- provide SQL, search, BI, ML, and application-facing consumption paths;
- apply security, observability, governance, data quality, and cost controls across all layers.

## 2. End-to-end flow

1. **Source systems produce data.** The diagram includes microservices on Amazon EKS, private APIs, operational/vehicle applications, DynamoDB, Neptune, Aurora-compatible relational data, and vendor files.
2. **Ingestion services capture changes and files.** API-based events are routed through managed streaming services. Database changes can be captured through CDC, while vendor deliveries arrive through AWS Transfer Family into Amazon S3.
3. **The landing layer preserves the source payload.** Original messages/files, source metadata, ingestion timestamps, and CDC attributes are retained so data can be audited and replayed.
4. **A Step Functions workflow coordinates validation.** The workflow checks DMS status, required tables, primary keys, referential integrity, null thresholds, row-count deviation, VIN token format, duplicate-source sequence, checksum, antivirus status, manifest/schema validity, and data-quality rules.
5. **Validation determines routing.** Accepted data is promoted into the raw zone. Rejected data is sent to a quarantine/rejected area with failure context for investigation and controlled replay.
6. **Lakehouse processing creates trusted datasets.** AWS Glue and/or Amazon EMR jobs transform raw data into curated and serving datasets. Apache Iceberg tables provide an open table format for atomic changes, schema evolution, partition evolution, and reliable reprocessing.
7. **Catalog and governance services expose trusted assets.** AWS Glue Data Catalog stores technical metadata; AWS Lake Formation applies fine-grained permissions; Amazon DataZone can publish discoverable business data products.
8. **Consumers access fit-for-purpose outputs.** Amazon Athena supports SQL analysis, Amazon OpenSearch Service supports search-oriented access, BI tools consume governed datasets, and application or ML workloads use serving-layer products.

## 3. Source and ingestion layer

### Application and API sources

- Amazon EKS hosts source microservices.
- Private APIs provide controlled application access without exposing ingestion endpoints publicly.
- Amazon API Gateway and AWS Lambda can validate and normalize API events before publishing them to the streaming layer.

### Streaming and change events

- Amazon MSK provides a durable event backbone.
- Partition events by a stable business key such as `vehicle_id` or a hashed VIN when per-vehicle ordering is required.
- Use a schema registry, producer retries, idempotent consumers, dead-letter handling, and replay procedures.
- Amazon Data Firehose can continuously deliver buffered events to Amazon S3.

### Database and graph sources

- Amazon DynamoDB Streams can expose item-level changes for event-driven processing.
- AWS DMS can capture initial loads and ongoing CDC from supported operational databases.
- Amazon Neptune supplies graph-oriented relationship data where graph extracts are required by the analytical platform.

### Vendor file ingestion

- AWS Transfer Family provides managed SFTP/FTPS/AS2 entry points.
- Each delivery should include, where possible, a manifest containing file name, expected size, checksum, schema/version, record count, source, and business date.
- Land vendor files unchanged before extraction or conversion.

## 4. Landing, validation, and quarantine

The source landing area is immutable. Store:

- the original source payload;
- streaming micro-batches or delivered files;
- database full-load and CDC records;
- graph snapshots and incremental changes;
- vendor batches and partner-specific API payloads;
- event time, ingestion time, source identifier, schema version, and CDC metadata;
- Amazon S3 versioning and SSE-KMS encryption metadata.

### Validation workflow

AWS Step Functions coordinates the landing-to-raw quality gate. The design shows checks for:

- DMS task status and expected tables;
- primary-key presence and uniqueness;
- referential integrity;
- VIN/token format;
- null and completeness thresholds;
- row-count deviation;
- duplicate source sequence;
- checksums and manifests;
- antivirus scanning;
- schema compatibility and domain-specific quality rules.

A recommended state model is:

- `RECEIVED` -> object/event accepted into landing;
- `VALIDATING` -> technical and business checks running;
- `ACCEPTED` -> eligible for raw-zone promotion;
- `REJECTED` -> moved or referenced in quarantine with reason codes;
- `REPLAYED` -> corrected data resubmitted with lineage to the original delivery.

## 5. Lakehouse layers

### Raw zone

The raw zone contains validated but minimally transformed data. It should preserve source fidelity while standardizing technical attributes such as ingestion timestamp, source system, schema version, and batch/run ID.

### Curated zone

The curated zone contains cleaned, deduplicated, conformed, and business-ready entities. Typical processing includes:

- schema enforcement and evolution;
- deduplication and late-arriving-event handling;
- type normalization and canonical identifiers;
- reference-data joins and domain rules;
- PII classification, tokenization, or masking;
- quality metrics and lineage capture.

### Serving zone

The serving zone contains consumer-oriented data products, aggregates, and access patterns optimized for analytics, dashboards, search, APIs, and ML features.

### Apache Iceberg conventions

- Use Iceberg tables for atomic commits and reliable concurrent processing.
- Partition using query patterns and data volume, not only source-folder structure.
- Schedule compaction to control small files.
- Maintain snapshot-retention and orphan-file cleanup policies.
- Test schema and partition evolution before promotion.
- Keep jobs idempotent so a failed batch can be rerun safely.

## 6. Processing and orchestration

- **AWS Step Functions:** coordinates validation, promotion, retries, timeout handling, and failure routing.
- **Amazon EventBridge:** starts workflows from events and schedules periodic processing or maintenance.
- **AWS Glue:** performs serverless ETL, crawling, catalog integration, data-quality checks, and table maintenance.
- **Amazon EMR:** handles large-scale or specialized Spark processing where cluster-level control is useful.
- **AWS Lambda:** supports lightweight event handling and control-plane activities, rather than long-running bulk transformations.

Every job should publish a run ID, input/output counts, rejected-record count, start/end time, watermark, source version, target snapshot, and failure reason.

## 7. Consumption layer

- **Amazon Athena:** interactive SQL over governed lakehouse tables.
- **Amazon OpenSearch Service:** indexed search and operational exploration where low-latency text or document retrieval is needed.
- **BI and reporting:** dashboards query curated or serving products rather than landing data.
- **Applications and analytics:** domain APIs, notebooks, and downstream workloads consume versioned data products with explicit contracts.

Avoid giving consumers direct access to landing prefixes. Access should flow through cataloged tables, views, governed shares, or purpose-built APIs.

## 8. Security, observability, and FinOps

The diagram shows a shared control plane containing:

- **IAM / IAM Identity Center:** least-privilege identities, roles, and federation;
- **AWS KMS:** key ownership, encryption, rotation, and separation by environment or data class;
- **AWS CloudTrail and AWS Config:** audit history and configuration compliance;
- **Amazon CloudWatch and AWS X-Ray:** logs, metrics, traces, dashboards, and alarms;
- **Amazon GuardDuty and AWS Security Hub:** threat detection and security posture management;
- **AWS Budgets and AWS Cost Explorer:** tagging, allocation, anomaly review, and cost controls.

Recommended safeguards include private networking, VPC endpoints, encryption in transit and at rest, bucket policies, block-public-access controls, secrets rotation, short-lived credentials, and separate development, test, and production accounts.

## 9. Data governance and catalog

- **AWS Lake Formation:** fine-grained database, table, column, row, and location permissions.
- **AWS Glue Data Catalog:** schemas, table definitions, partitions, and technical metadata.
- **Amazon DataZone:** business catalog, ownership, discoverability, subscriptions, and data-product publishing.
- **Amazon Macie:** discovery of PII and sensitive objects in Amazon S3.
- **AWS Organizations:** multi-account guardrails and service control policies.
- **AWS Audit Manager:** evidence collection and compliance reporting.

Each published product should define an owner, domain, description, classification, SLA, retention period, quality thresholds, approved consumers, and escalation path.

## 10. Platform operations

The platform-operations layer includes:

- a control tower and account-vending model;
- AWS CloudFormation or AWS CDK for infrastructure as code;
- AWS CodePipeline for CI/CD and controlled promotion;
- AWS Secrets Manager for credential and secret rotation;
- backup, cross-Region disaster recovery, and recovery tests;
- a service catalog of approved reusable platform patterns.

Operational readiness should cover idempotency, checkpoints, back-pressure, retries with limits, dead-letter queues, replay, freshness and completeness SLAs, lineage, alarms, runbooks, and tested failure scenarios.

## 11. Reliability and recovery

Define measurable targets for every critical data product:

- freshness SLA and maximum acceptable data loss;
- RTO and RPO;
- retry and replay boundaries;
- checkpoint and watermark behavior;
- reconciliation between source, landing, raw, curated, and serving counts;
- backup retention and restore testing;
- multi-Region recovery scope, if required;
- ownership and incident escalation.

A replay must be deterministic: the same input and transformation version should either reproduce the same result or create a traceable new version without uncontrolled duplicates.

## 12. Suggested repository structure

```text
ota-data-platform/
├── README.md
├── docs/
│   └── ota-aws-architecture.png
├── diagrams/
│   └── ota-aws-architecture.drawio
├── infrastructure/
│   ├── cdk/
│   └── cloudformation/
├── pipelines/
│   ├── ingestion/
│   ├── validation/
│   └── transformation/
├── contracts/
│   ├── schemas/
│   └── data-quality/
└── runbooks/
    ├── replay.md
    ├── incident-response.md
    └── disaster-recovery.md
```

## 13. Naming and documentation conventions

- Use lowercase, stable file names without spaces.
- Keep the editable `.drawio` source under version control.
- Export a PNG at a readable resolution whenever the diagram changes.
- Update this README in the same pull request as an architecture change.
- Do not commit credentials, tokens, private endpoints, account IDs, internal hostnames, customer data, or production samples.

## 14. Assumptions and items to validate

The diagram is detailed, but implementation teams should confirm:

- exact source-to-ingestion mappings;
- whether each database uses snapshots, CDC, or both;
- Kafka topic names, partition keys, retention, and ordering requirements;
- the authoritative VIN/tokenization approach;
- table format and catalog ownership;
- raw/curated/serving retention policies;
- expected data volumes and freshness requirements;
- cross-account and cross-Region boundaries;
- BI, search, application, and ML access patterns;
- disaster-recovery tier and recovery targets.

## Disclaimer

This is a reference design and not an AWS-issued architecture. Security, compliance, service quotas, availability, and cost must be evaluated for the target workload and organization.
