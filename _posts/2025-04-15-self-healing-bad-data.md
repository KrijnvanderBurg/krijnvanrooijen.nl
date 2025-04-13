---
title: "Self-Healing Data Pipelines: Automatically Handle and Reprocess Bad Data"
date: 2025-04-13
excerpt: "Learn how to automate the handling, reprocessing, and testing of bad data in PySpark pipelines, ensuring robust data workflows."
tags:
- Data Engineering
- Pyspark
- Databricks
image: /assets/graphics/2025-04-15-self-healing-bad-data/thumbnail-self-healing-bad-data.jpg
pin: false
---
In production environments, bad data is inevitable. Schema mismatches, missing values, and inconsistent formatting can cause failures in data pipelines. However, rather than halting the entire pipeline when these issues arise, the goal is to handle bad data effectively while ensuring the flow of valid data continues uninterrupted. By effectively managing bad data, pipelines continue to run smoothly, minimizing downtime and bottlenecks.

This article will walk through how to build a self-healing PySpark pipeline that:
- Isolates and stores bad records for future processing or debugging.

- Automates the reprocessing of bad data when the issues that caused the failures have been resolved.

- Retains a historical record of bad data for testing purposes, ensuring that data can be processed correctly in future pipeline versions.

# **1. Separating Bad Data**
The first step in handling unprocessable records is to isolate them so the pipeline can continue processing valid data without disruption. Whether due to schema mismatches or data quality issues, bad records should be separated from valid data, allowing the pipeline to run uninterrupted while the problematic records are set aside for later investigation or reprocessing.

**Example: Writing Bad Data to a Separate Location**
1. The pipeline reads the data and applies a schema to ensure that the incoming data matches the expected structure.

2. Any records that don't match the schema are captured in a special column, `_corrupt_record`.

3. Bad data is filtered by checking if the column `_corrupt_record` contains data. If the column is null,the record matches the schema.

4. The remaining valid data is separated from the corrupted records using the `exceptAll()` function, which ensures no overlap between datasets.

5. Finally, valid records are written to a destination table, while bad records are written to a separate table.

```python
# Read the CSV with permissive mode to allow corrupt records into _corrupt_record
df = spark.read.option("mode", "PERMISSIVE").schema(schema).csv("/mnt/bronze/data_source/csv_file.csv")

# Isolate the bad data into a separate dataframe
bad_df = df.filter(df["_corrupt_record"].isNotNull())

# Use exceptAll to avoid duplicates when adding new bad records
df = df.exceptAll(bad_df)

# Write valid and bad data to separate tables
df.write.format("parquet").save("/mnt/silver/<table_name>")
bad_df.write.format("parquet").save("/mnt/bronze/<table_name>_bad")
```

❓ What happens when bad data never becomes valid?

Bad data remains in the bad data table. Alerts can be set to monitor volume growth, and metadata can tag common failure reasons for better visibility.

# 2. Automating Bad Data Reprocessing
With each pipeline run, not only should new incoming data be processed, but bad data should also be re-evaluated to check if it can now be processed. This ensures that records which previously failed are automatically reprocessed when their underlying issues have been resolved. This approach eliminates the need to create a new pipeline to handle bad data, and updates to the pipeline logic will automatically be applied to previously failed records on the next run.

After bad data has been successfully reprocessed, it is typically removed from the bad data table; this is not included in the example code.

**Example: Reading from good and bad from multiple sources**
1. The pipeline reads both new data and previously failed records.

2. The union() function merges these datasets, and the same filtering and separation logic is applied to identify bad data.

3. Valid data is written to the silver layer, and bad data is overwritten in the bronze layer.

```python
df = spark.read.option("mode", "PERMISSIVE").schema(schema).csv("/mnt/bronze/data_source/csv_file.csv")
bad_df = spark.read.format("delta").load("/mnt/bronze/<table_name>_bad")  # Process _corrupt_record column for consistency
df = df.union(bad_df)  # Merge both incoming data and previously failed data

bad_df = df.filter(df["_corrupt_record"].isNotNull())
df = df.exceptAll(bad_df)

df.write.format("parquet").mode("append").save("/mnt/silver/<table_name>")           # Append valid data to the silver layer
bad_df.write.format("parquet").mode("overwrite").save("/mnt/bronze/<table_name>_bad") # Overwrite bad data in the bronze layer
```

❓ Why automate bad data reprocessing?

Automating the reprocessing of bad data streamlines the pipeline, removing the need for manual intervention or separate pipelines for reprocessing. Bad records are handled automatically as soon as the necessary fixes are made.

# 3. Storing Bad Records for Testing

Retaining a history of bad records is essential for testing, debugging, and validating pipeline changes. If changes are made to the pipeline that should address previously encountered issues, it’s important to ensure that those issues are revisited and validated.

Once problematic records are successfully processed, they are typically removed from the bad data table. This can lead to the loss of valuable information for testing. To prevent this, bad records should be stored in a separate archive for later use.

**Writing bad data also to an archive location**
When writing bad data to its separate bad data queue table, also write that same data to another location for archiving. This additional writing must not overwrite but append. However, deduplication is critical as pipelines can fail to reprocess the bad data multiple times, re-adding the same bad data to the archive.

**Using Change Data Feed (CDF) to Track Bad Data History**

Change Data Feed (CDF) is a feature in Databricks that tracks changes in Delta tables, including inserts, updates, and deletions. Enabling CDF ensures that all changes to the bad data table are captured, even when records are deleted or successfully processed. This allows the history of bad data to be retained for future testing and debugging.

**Example: Storing Bad Records for Testing with CDF**

1. CDF is enabled on the bad data table to track all inserts, updates, and deletions.

2. When bad records are processed and removed from the bad data table, CDF ensures that a history of these records is preserved in the logs.

3. The inserted records are written to a separate test table, preserving a history of all bad data for future testing, debugging, and CI/CD validation.

```sql
ALTER TABLE source_table SET TBLPROPERTIES (delta.enableChangeDataFeed = true)
```

```python
# Read only inserted rows from the change data feed
inserts_df = (
    spark.read
    .format("delta")
    .option("readChangeFeed", "true")
    .option("startingVersion", 0)  # Start from the beginning to capture all inserts or track version yourself
    .table("/mnt/bronze/<table_name>_bad")
    .filter("_change_type = 'insert'")  # Filter for newly inserted bad records
)

inserts_df.write.format('delta').mode("overwrite").save("/mnt/bronze/<table_name>_test")
```

# Conclusion
Handling bad data is a fundamental aspect of building resilient data pipelines. By isolating bad data, automating their reprocessing, and retaining them for future testing, the integrity of the pipeline is preserved. This approach ensures that data quality issues do not cause pipeline disruptions while enabling continuous improvements.

Additionally, enabling features like CDF for tracking the history of bad data provides a robust framework for debugging, testing, and validating pipeline changes. With this self-healing approach, pipelines are better equipped to handle bad data without unnecessary disruptions, allowing for greater flexibility and reliability.

## Key Takeaways:
1. **Isolate and Manage Bad Data**: Keep bad records separate from valid data to prevent pipeline failures.

2. **Automate Bad Data Reprocessing**: Automatically reprocess bad data when underlying issues are fixed, removing the need for manual intervention.

3. **Track Bad Data History**: Retain a historical record of bad data using features like CDF to ensure records can be tested and debugged in the future.

By implementing these practices, data pipelines become more resilient, capable of handling errors efficiently, and can evolve without being derailed by bad data.