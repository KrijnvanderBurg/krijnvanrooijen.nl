---
title: "[DRAFT] Self-Healing Pipelines: Handling, reprocesing and testing Bad Data"
date: 2025-04-13
excerpt: ""
tags:
- Data Engineering
- Data Platform
image: /assets/graphics/2025-04-15-self-healing-bad-data/thumbnail-self-healing-bad-data.png
pin: false

In this article, we'll walk through how to build a PySpark pipeline that:
- Sends bad records to a bad queue
- Reprocesses that queue automatically over time
- Keeps bad records for future testing and debugging
---
Data pipelines often encounter records that can't be processed due to issues like schema mismatches or data quality problems. It's essential to ensure that the pipeline doesn't fail when faced with these records, but continues processing valid data while also handling bad records appropriately. Bad data should not be discarded, as data loss is often worse than dealing with bad data.

This article discusses how to configure a PySpark pipeline to handle bad data, automate the reprocessing of failed records, and keep a history of bad records for future testing and debugging.

# **1. Separating Bad Data**

The first step in dealing with unprocessable records is to isolate them, ensuring that the pipeline processes valid data without disruption. Records that cannot be processed due to schema mismatches or data quality issues should be written to a separate location. This allows the pipeline to keep running now and on next runs while the problematic records are isolated for further investigation or reprocessing.

**Example: Writing to a Separate Location for Bad Data**

1. The pipeline applies a schema when reading the data, which ensures that the data matches the expected structure.

2. Any records that don't match the schema are captured in a special column, `_corrupt_record`.

3. Bad data is filtered by checking if the column `_corrupt_record` contains data, if its null then the schema matches.

4. The remaining valid data is separated from the corrupted records using the exceptAll function to ensure no overlap.

5. Finally, valid records are written to a destination table, while bad records are written to a separate table.

```python
# Read the CSV with permissive mode to allow corrupt records into _corrupt_record
df = spark.read.option("mode", "PERMISSIVE").schema(schema).csv("/mnt/bronze/data_source/csv_file.csv")

# Isolate the bad data into a separate dataframe
bad_df = df.filter(df["_corrupt_record"].isNotNull())

# Use exceptAll to avoid duplicates when adding new bad records
df = df.exceptAll(bad_df)

# Write valid and invalid data to separate tables
df.write.format("parquet").save("/mnt/silver/<table_name>")
bad_df.write.format("parquet").save("/mnt/bronze/<table_name>_bad")
```

❓ What happens when bad data never becomes valid? It remains in the bad data table. Alerts can be set for volume growth, and metadata can be used to tag common failure reasons. Eventually, old data can be archived while ensuring that valid data continues flowing.

# 2. Automating Bad Data Reprocessing

With each pipeline run, not only should new incoming data be processed, but the bad data should also be checked to see if it can now be processed. This ensures that records that previously failed are re-evaluated and reprocessed when possible, preventing them from accumulating indefinitely. While this approach does make the pipeline more complex, it is preferable to creating a new pipeline with the same ETL logic for processing previous bad data. When the pipeline is updated to fix or accomodate the bad data is automatically picked up on the next run rather than requiring some other pipeline to fix it.

Example: Reading from Multiple Sources in One Call
1. Read both new incoming data and previously failed records.

2. The union function merges the two datasets (incoming and previous bad data) as dataframe, and the same filtering and separation logic is applied to extract the bad data.

3. The pipeline writes valid data to the silver layer, appending it, and writes the remaining bad data back to the bad data table in the bronze layer, overwriting the previous contents.

```python
df = spark.read.option("mode", "PERMISSIVE").schema(schema).csv("/mnt/bronze/data_source/csv_file.csv")
bad_df = spark.read.format("delta").load("/mnt/bronze/<table_name>_bad")  # Process _corrupt_record column for consistency
df = df.union(bad_df)  # Merge both incoming data and previously failed data

bad_df = df.filter(df["_corrupt_record"].isNotNull())
df = df.exceptAll(bad_df)

df.write.format("parquet").mode("append").save("/mnt/silver/<table_name>")           # Append valid data to the silver layer
bad_df.write.format("parquet").mode("overwrite").save("/mnt/bronze/<table_name>_bad") # Overwrite bad data in the bronze layer
```

# 3. Storing Bad Records for Testing

Retaining bad data is essential for testing the pipeline, it must be able to process all previously processed data. If for whatever reason the data must be processed anew from the original source then you want to be sure it can process everything it previously could.

A key limitation of relying exclusively on the bad data table is that once problematic records are processed correctly, they are removed from the table, meaning there's no longer a history of bad data for testing purposes. This makes it necessary to store bad records separately for later use in CI/CD pipelines or manual testing.

One approach is to create a test table to serve as a historical archive of all bad records. This ensures that even if the records are later processed successfully, a history of those failures is preserved for debugging and testing. This can be done by adding another write of bad data to the test table but in append mode with deduplication. Or alternatively, using Change Data Feed (CDF) in Databricks can help track changes made to the bad table, ensuring that deleted records are still available in a log for future testing. This 

Enabling Change Data Feed on the Bad Data Table

```sql
ALTER TABLE source_table SET TBLPROPERTIES (delta.enableChangeDataFeed = true)
```

Example: Storing Bad Records for Testing
1. The CDF feature is enabled on the bad data table to track inserts, updates, and deletions.

2. When bad records are processed, they are deleted from the bad data table, but CDF ensures that the history of these records is still available in the logs.

3. The records that were inserted (and later processed) are written to a separate test table (<table_name>_test), preserving a history of all bad data for future testing, debugging, and CI/CD validation.

```python
# Read only inserted rows from the change data feed
inserts_df = (
    spark.read
    .format("delta")
    .option("readChangeFeed", "true")
    .option("startingVersion", 0)  # Start from the beginning to capture all inserts
    .table("/mnt/bronze/<table_name>_bad")
    .filter("_change_type = 'insert'")  # Filter for newly inserted bad records
)

inserts_df.write.format('delta').mode("overwrite").save("/mnt/bronze/<table_name>_test")
```

# Conclusion

Handling bad records is a critical part of building resilient data pipelines. By isolating invalid records, automating its reprocessing, and retaining the invalid records for future testing, the integrity of the pipeline is preserved. This approach ensures that data quality issues do not disrupt pipeline operations and provides an effective way to test and fix errors as the pipeline evolves.

With this setup, pipelines can efficiently process valid data while continuously checking and reprocessing bad data, ensuring the system remains reliable and capable of handling edge cases without unnecessary disruptions.
