# 📈 Real-Time Stock Market Analytics Pipeline

An event-driven, serverless AWS pipeline that ingests, processes, stores, and analyzes stock market data in near real-time — with built-in anomaly detection and automated alerts for significant stock movements.

---

## Problem Statement

I have a genuine interest in the stock market and wanted to experiment with AWS to build something that could actually benefit me in some way, even if it doesn't replace the real tools out there. I found myself manually checking prices and missing movements I cared about, so I figured why not try and solve that with services on AWS. This project is the result of that, a serverless, event-driven pipeline that streams real-time stock data, detects anomalies, and sends me an alert when something significant happens. It gave me a reason to get hands-on with services like Kinesis, Lambda, and SNS in a context that felt meaningful.

---

## Architecture Diagram

> 📌 *[Insert architecture diagram here — see `/architecture/diagram.png`]*

---

## Architecture Walkthrough

1. **Data Ingestion** — A Python producer script fetches real-time stock data using `yfinance` and streams records into **Amazon Kinesis Data Streams** every 30 seconds.
2. **Stream Processing** — A dedicated **AWS Lambda function (processor)** is triggered by Kinesis shard events. It processes each record and detects anomalies (e.g. price movements beyond a defined threshold).
3. **Raw Data Archiving** — The processor Lambda simultaneously writes raw stock records to **Amazon S3** in JSON format, partitioned by date, for long-term storage and historical analysis.
4. **Schema Cataloging** — **AWS Glue Data Catalog** automatically crawls and catalogs the S3 data, structuring it so Athena can query it using standard SQL without manual schema definition.
5. **Historical Querying** — **Amazon Athena** queries the cataloged S3 data lake directly, with query results stored back in a separate S3 bucket.
6. **Low-Latency Storage** — Processed and enriched stock records are written to **Amazon DynamoDB** for fast, low-latency querying by downstream applications or dashboards.
7. **Trend Evaluation** — A second dedicated **AWS Lambda function (trend evaluator)** is triggered by DynamoDB Streams. It evaluates stock trends against defined thresholds and determines whether an alert is warranted.
8. **Alerting** — When the trend evaluator detects a significant movement, it publishes a message to **Amazon SNS**, which delivers an Email/SMS alert to subscribed users in near real-time.

---

## AWS Services Used

| Service | Purpose | Why This Service |
| --- | --- | --- |
| Amazon Kinesis Data Streams | Real-time data ingestion | Managed streaming service built for high-throughput, ordered event ingestion |
| AWS Lambda (processor) | Stream processing & anomaly detection | Serverless — no idle compute cost; scales automatically with stream shards |
| AWS Lambda (trend evaluator) | Trend evaluation & alert triggering | Decouples alerting logic from processing logic; single responsibility per function |
| Amazon DynamoDB | Low-latency storage of processed data | Single-digit millisecond reads; schema-flexible for varying stock record structures |
| Amazon S3 | Raw data archiving & query results storage | Durable, cheap object storage; native integration with Athena for querying |
| AWS Glue Data Catalog | Schema cataloging for S3 data | Automatically structures raw S3 data so Athena can query it without manual schema setup |
| Amazon Athena | Historical SQL querying | Serverless query engine — pay per query, no infrastructure to manage |
| Amazon SNS | Real-time alerting (Email/SMS) | Fully managed pub/sub; decouples alert delivery from trend evaluation logic |
| AWS IAM | Access control & security | Least-privilege roles scoped per Lambda function; no hardcoded credentials |
| Amazon CloudWatch | Monitoring & logging | Native Lambda integration; tracks errors, invocation counts, and latency |


## Well-Architected Framework Alignment

**Reliability**
- Kinesis Data Streams retains records for 24 hours by default, ensuring no data loss if Lambda is temporarily throttled or unavailable.
- Lambda Dead Letter Queue (DLQ) configured to capture failed processing events for manual review.

**Performance Efficiency**
- DynamoDB provides single-digit millisecond read latency for processed stock records regardless of dataset size.
- Athena enables on-demand historical analysis without a persistent database or ETL pipeline.

**Cost Optimization**
- Fully serverless architecture means zero compute cost during off-hours (e.g. outside trading hours).
- Athena charges per query scanned — S3 data is partitioned by date to minimize scan size and cost.

**Security**
- Each Lambda function uses a dedicated IAM role with only the permissions it requires (least privilege).
- No API keys or credentials are hardcoded — all access is handled via IAM roles and environment variables.

**Operational Excellence**
- CloudWatch Logs automatically capture Lambda invocation logs and processing errors.
- SNS alerting provides immediate visibility into anomalies without manual monitoring.
---

## Key Design Decisions & Trade-offs

**Kinesis over SQS for ingestion**
Kinesis was chosen because it preserves the order of stock records per shard, which matters for accurate trend analysis. SQS standard queues do not guarantee ordering. Trade-off: Kinesis has a higher baseline cost and requires shard capacity planning at scale.

**DynamoDB over RDS for processed data**
Stock records have a consistent, document-like structure and are queried primarily by ticker symbol and timestamp — a simple key-value access pattern. DynamoDB handles this with low latency and no schema management overhead. Trade-off: no support for complex joins or ad-hoc relational queries.

**Athena over Redshift for historical analysis**
Given the project's scope, Athena's serverless, pay-per-query model is far more cost-effective than maintaining a Redshift cluster. Trade-off: Athena query latency (seconds) is higher than Redshift (sub-second), but acceptable for non-real-time historical analysis.

**Near Real-Time (30s delay) vs. Fully Real-Time**
The pipeline operates with an intentional 30-second ingestion interval. True tick-by-tick streaming would require higher Kinesis shard counts and more complex Lambda concurrency management, significantly increasing cost. For the analytics use case modeled here, 30-second granularity is sufficient.

---

## Screenshots

> 📌 *[Add screenshots to `/screenshots/` and embed here]*

Suggested screenshots to include:
- Kinesis Data Streams console showing incoming records
- Lambda invocation logs in CloudWatch
- DynamoDB table with processed stock records
- Athena query results against S3 data
- SNS alert email/SMS received

---

## Lessons Learned & What I'd Improve

**What I'd add next:**
- **SQS buffer between Kinesis and Lambda** — At higher throughput, adding SQS would provide an additional buffer and allow more granular retry logic per message rather than per batch.
- **AWS Step Functions** — For more complex multi-stage processing (e.g. enrich → validate → store → alert), Step Functions would provide better orchestration and error handling across the two Lambda functions.
- **Kinesis Data Firehose** — Replace the raw S3 write logic inside the processor Lambda with Firehose for automatic batching, compression, and partitioning — reducing Lambda complexity and S3 costs.
- **QuickSight Dashboard** — Connect to Athena to build a visual dashboard for stock trend analysis without any additional infrastructure.
- **Unit tests for both Lambda functions** — Add pytest-based tests with mocked Kinesis and DynamoDB Stream events to validate processing and alerting logic independently before deployment.

**What I learned:**
- Separating the processing Lambda from the alerting Lambda follows the single responsibility principle and makes each function easier to test, debug, and scale independently.
- Kinesis shard capacity directly affects Lambda concurrency — one Lambda instance per shard. Understanding this relationship is critical for scaling decisions.
- AWS Glue Data Catalog is what makes Athena queries possible against raw S3 data — without it, you'd need to define schemas manually or use a dedicated ETL pipeline.
- DynamoDB's on-demand capacity mode is ideal for unpredictable workloads like stock ingestion, but reserved capacity becomes more cost-effective at consistent high throughput.
- Athena query performance is heavily dependent on S3 data partitioning strategy — poorly partitioned data leads to full table scans and higher costs.

---

## Repo Structure

```
stock-analytics-pipeline/
├── README.md
├── architecture/
│   └── diagram.png
├── src/
│	├── processor.py 			   # yfinance data producer → Kinesis
│   ├── producer.py  			   # Lambda 1 — stream processing & anomaly        
│   └── trend_evaluator.py         # Lambda 2 — trend evaluation & SNS alerting
├── docs/
│   └── decisions.md         # Architecture decision log
└── screenshots/
```

---
