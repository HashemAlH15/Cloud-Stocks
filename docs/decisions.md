Architecture Decision Log
This document records the key architectural decisions made in the stock market analytics pipeline, the reasoning behind each choice, and the trade-offs considered.

ADR-001 — Amazon Kinesis Data Streams for data ingestion
Decision: Use Kinesis Data Streams as the ingestion layer for real-time stock data from the Python producer.
Reasoning: Stock data needs to be ingested in order — a price movement at 09:31 must be processed before one at 09:32 for trend analysis to be accurate. Kinesis guarantees ordering within a shard, which makes it the right fit here.
Alternative considered: SQS standard queues. Rejected because SQS does not guarantee message ordering. SQS FIFO queues do preserve order but have a lower throughput ceiling (3,000 messages/second vs Kinesis's effectively unlimited throughput with additional shards).
Trade-off: Kinesis has a higher baseline cost than SQS and requires upfront shard capacity planning. For this project's scale, a single shard is sufficient.

ADR-002 — Two separate Lambda functions instead of one
Decision: Split processing and alerting into two dedicated Lambda functions — a processor triggered by Kinesis, and a trend evaluator triggered by DynamoDB Streams.
Reasoning: Each function has a single, clearly defined responsibility. The processor handles data ingestion, anomaly detection, and storage. The trend evaluator handles business logic around alerting. Keeping them separate means each can be developed, tested, debugged, and scaled independently without either function becoming too complex.
Alternative considered: A single Lambda function handling the full pipeline end-to-end. This would have simplified the architecture but created a tightly coupled function that is harder to maintain and harder to test in isolation.
Trade-off: Two functions means two IAM roles, two sets of CloudWatch logs, and slightly more infrastructure to manage. The operational clarity gained outweighs this overhead.

ADR-003 — Amazon DynamoDB for processed data storage
Decision: Store processed and enriched stock records in DynamoDB.
Reasoning: The primary query pattern is simple — look up records by ticker symbol and timestamp. DynamoDB handles this access pattern with single-digit millisecond latency and no schema management overhead, making it well suited for low-latency downstream reads.
Alternative considered: Amazon RDS (PostgreSQL). Rejected because the data structure does not require relational joins or complex queries. RDS would introduce unnecessary infrastructure cost and management overhead for a workload that fits DynamoDB's strengths.
Trade-off: DynamoDB does not support ad-hoc relational queries. If query patterns become more complex in the future, a relational database would be reconsidered.

ADR-004 — Amazon S3 + AWS Glue Data Catalog for raw data archiving
Decision: Write raw stock records to S3 partitioned by date, and use AWS Glue Data Catalog to structure the data for Athena queries.
Reasoning: S3 provides durable, cost-effective long-term storage for raw data. Glue Data Catalog acts as the metadata layer that allows Athena to understand the structure of the S3 data without requiring a dedicated ETL pipeline or a persistent database. Together they form a lightweight, serverless data lake.
Alternative considered: Writing raw data directly to RDS or Redshift. Rejected because both require persistent infrastructure running 24/7, significantly increasing cost for a workload that only needs historical querying occasionally.
Trade-off: Glue crawlers need to be run periodically to keep the catalog up to date as new data arrives. This adds a small operational step compared to a database that updates in real time.

ADR-005 — Amazon Athena for historical querying
Decision: Use Athena to query historical stock data stored in S3.
Reasoning: Athena is serverless and charges per query scanned, making it highly cost-effective for infrequent historical analysis. No cluster to provision, no idle compute cost. It queries S3 directly using standard SQL via the Glue Data Catalog schema.
Alternative considered: Amazon Redshift. Rejected for this project's scope because Redshift requires a persistent cluster with an ongoing hourly cost regardless of usage. Athena's pay-per-query model is far more appropriate for occasional historical lookups.
Trade-off: Athena query latency is in the range of seconds, compared to Redshift's sub-second responses at scale. For non-real-time historical analysis this is acceptable. At higher query volumes or with stricter latency requirements, Redshift would be reconsidered.

ADR-006 — Amazon SNS for alerting
Decision: Use SNS to deliver stock trend alerts via Email and SMS.
Reasoning: SNS fully decouples the alert delivery mechanism from the trend evaluation logic. The trend evaluator Lambda simply publishes a message to an SNS topic — it does not need to know or care how the alert is delivered. SNS handles fan-out to multiple subscribers (Email, SMS, or future channels like Slack) without any changes to the Lambda code.
Alternative considered: Sending emails directly from Lambda using SES. Rejected because it tightly couples delivery logic to the function and makes it harder to add new notification channels later.
Trade-off: SNS adds a small additional service dependency. For a project of this scale this is negligible.

ADR-007 — Near real-time (30-second delay) vs fully real-time
Decision: Ingest stock data on a 30-second polling interval rather than tick-by-tick.
Reasoning: True tick-by-tick streaming would require significantly more Kinesis shards, higher Lambda concurrency, and a more complex producer implementation — all of which increase cost and operational complexity. For the trend detection and anomaly alerting use case modeled here, 30-second granularity is sufficient to capture meaningful price movements.
Trade-off: The pipeline is described as near real-time rather than fully real-time. This is an intentional and documented design constraint, not a limitation. If sub-second latency were required, the producer would need to be redesigned and shard capacity scaled accordingly.
