# RWA Interview: ERC-20 Transfers

## Instructions to Run

1. **AWS Account & Glue Notebook**
   - Ensure you have access to an AWS account with Glue Notebook permissions.
2. **Create Service Role**
   - For proof-of-concept, I used the managed `AWSGlueServiceNotebookRole` with `AmazonS3FullAccess`.
   - **Note:** To enable notebook sessions, the service role must either be prefixed with `AWSGlueServiceNotebookRole`, or include a policy granting `iam:PassRole` permission.
3. **Load & Execute**
   - Load the provided Spark script into the Glue Notebook and execute.  
   - Set `OUTPUT_BASE` to desired output S3 Bucket.
   - The script will read ERC-20 logs and token metadata from S3, perform the necessary transformations and aggregations, and output the computed metrics to S3 as CSV files.

## Tool Selection

I chose **AWS Glue Notebook** with **Spark** for the following reasons:  

- **Volume Scalability:** Distributed computing and scalable hardware enable efficient processing of large datasets.  
- **Solution Scalability:** The notebook provides an environment for rapid Spark prototyping, allowing for easy extension into a production grade solution. 
- **Flexibility and Reusability:** Spark is easily integrated with most cloud providers and modern data platforms.

## Code Improvements

- **Modular metric computation**  
  I'd break out each metric type into its own method, adding parameters for things like frequency (hourly, daily, weekly) and aggregation granularity. This would make the logic reusable and easier to extend later on.

- **Event signature flexibility**  
  Instead of hardcoding the ERC-20 `Transfer` topic hash, I’d add a small utility to compute Keccak-256 hashes from event signatures for easier extension to new event types or token standards.

- **Config-driven schemas and transformations**  
  I’d move schema definitions and column selections into external config files, so the transformation logic stays clean and easy to maintain. This would also make it simpler to adjust to schema changes or new data sources down the line.
  


## Scaling System Architecture for Near-Realtime Metrics

1. **Event / Message Stream** serves as the hub for all live data events, feeding both a real-time processor and a stream-to-lake sink.

2. **Real-time processor** computes and pushes hot aggregates directly into **Redis/Aerospike cache** for low-latency retrieval.

3. **Stream-to-lake sink** writes raw events to the **data lake**, ensuring durability and enabling historical recomputation.

4. **Batch ETL jobs** read from the **data lake**, compute long-range aggregates, and push them into the **columnar warehouse**.

5. **API Layer** routes queries to cache or warehouse layer based on range / granularity.

```text
Event Stream (Kafka / Kinesis)
         │
 ┌───────┴───────────┐
 │                   │
 ▼                   ▼
Hot Cache Updates   Data Lake (Iceberg / Delta Lake)
 (Redis)             │
    │                ▼
    │               Raw Event Data (Data Lake)
    │                │
    │                ▼
    │               Columnar Warehouse (Redshift / BigQuery)
    │                │
    │                ▼
    └─────────► API Layer ─────► Frontend / Charts



