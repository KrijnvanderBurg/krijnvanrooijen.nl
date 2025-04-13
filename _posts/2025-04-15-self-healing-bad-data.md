---
title: "[DRAFT] Self-Healing Pipelines: Handling, reprocesing and testing Bad Data"
date: 2025-04-13
excerpt: ""
tags:
- Data Engineering
- Data Platform
image: /assets/graphics/2025-04-15-self-healing-bad-data/thumbnail-self-healing-bad-data.png
pin: false
---
When working with data pipelines, one common challenge is dealing with records that can't be processed due to issues like schema mismatches or data quality problems. In these cases, it’s important to ensure that the pipeline doesn't break but continues to process valid records while also not discarding the bad data as data loss is worse than malformed data.

In this article, we'll cover how to set up a Pyspark pipeline to handle bad data, including automating the (retrying or reprocessing? which word to use) of the bad data and and keeping track of bad records for future testing and debugging.

All code provided is for short example purposes and might miss certain steps to make it function properly.

# 1: Separate the bad data

The first step is to define how records that fail processing (due to schema issues or other problems) will be written to a bad queue. This ensures that these records are isolated, allowing the pipeline to continue without interruptions. The bad table is essentially a holding area for records that need further investigation or reprocessing, usually the pipeline must be updated to handle the bad data to process it still.

Example: Writing to the bad Queue

```python
# Read the CSV with permissive mode to allow corrupt records and put them in _corrupt_record
df = spark.read.option("mode", "PERMISSIVE").schema(schema).csv("/mnt/bronze/data_source/csv_file.csv")

# Add all corrupt records to a separate dataframe
bad_df = df.filter(df["_corrupt_record"].isNotNull())

# Use exceptAll to ensure no overlap between valid and invalid data
# exceptAll guarantees no duplicates even when adding more bad data checks
df = df.exceptAll(bad_df)

# Write valid and invalid data to separate tables
df.write.format("parquet").save("/mnt/silver/<table_name>")
bad_df.write.format("parquet").save("/mnt/bronze/<table_name>_bad")
```

In the code above, any records with incorrect schema are writting to a separate bad table in the same layer, while valid records

# 2: Automating bad data reprocessing

Each time the pipeline runs, it should not only process new incoming data but also check the bad queue to see if any previously failed records can now be processed. This ensures that records stuck in the bad queue are continuously re-evaluated and integrated into the main pipeline when possible. By integrating it in the existing pipeline for incoming data we also prevent creating multiple processes to run and maintain, its combined in a single robust pipeline.

The key here is to read both the incoming data and the bad queue in the same read operation. This eliminates the need for separate data frames and simplifies the process.

Example: Reading from Multiple Locations in One Call

```python
df = spark.read.option("mode", "PERMISSIVE").schema(schema).csv("/mnt/bronze/data_source/csv_file.csv")
bad_df = spark.read.format("delta").load("/mnt/bronze/<table_name>_bad") # you still need to process the read _corrupt_record column into a dataframe thats exactly like the read incoming csv.
df = df.union(bad_df) # read both incoming data and previous runs bad data

bad_df = df.filter(df["_corrupt_record"].isNotNull())
df = df.exceptAll(bad_df)

df.write.format("parquet").mode("append").save("/mnt/silver/<table_name>")            # mode append to next layer silver
bad_df.write.format("parquet").mode("overwrite").save("/mnt/bronze/<table_name>_bad") # mode overwrite bad in bronze
```

In this example, the pipeline reads data from both the incoming data directory and the bad queue in a single operation. Records that pass validation (age is not null) are processed normally, while invalid records are left in the bad queue to be handled later.

The bad table serves an essential function by isolating records that cannot be processed, but it's crucial to ensure that the pipeline doesn’t get stuck with unprocessable data. On every run, the pipeline should attempt to process any records in the bad queue that can now be valid.

This approach guarantees that every time the pipeline runs, it checks the bad queue along with the new incoming data. If any bad records can now be processed, they are reintegrated into the pipeline, ensuring that previously failed records don’t pile up unnecessarily.

# 3: Keeping all bad records for Testing

To improve testing and debugging, it’s essential to retain records that couldn't be processed, so you can simulate reprocessing them during manual testing or CI/CD processes. One way would be to, as a test, process all of the data but this could take long or be costly in terms of compute or if records were updated or altered from bad to good then there is no bad data anymore. So we want to keep all the bad data to test against, perhaps in cicd, when the pipeline is updated to make sure it can still handle edge cases by replaying problematic records.

A good way to store these records for later is by creating a `<table_name>_test` table, which serves as a historical archive of all failed or malformed records. You can later use this table to simulate how the pipeline would handle bads if the schema or data quality issues are resolved.

There are two ways of doing this. When encountering bad records in the pipeline they can be written to both a bad table and a test table, the bad table would be overwritten but the test table would be appending with deduplication. However, depending on the size and number of bad data this test table could become quite large, reading it all, dedplicating and writing deduplicated back could be a considerable part of the pipeline execution time while it only serves for testing purposes.

Or in databricks we can leverage Change Data Feed, CDF, which when enabled maintains a log of all inserts, updates and deletes on a table. This way if we reprocess and thus delete bad records theyre still in the logs and can be retrieved and stored in another table for maintaining history.

First enable change data feed on the table storing bad data as a queue

```sql
ALTER TABLE source_table SET TBLPROPERTIES (delta.enableChangeDataFeed = true)
```

Example: Storing bad Records for Testing.

```python
# Read only inserted rows from the change data feed
inserts_df = (
    spark.read
    .format("delta")
    .option("readChangeFeed", "true")
    .option("startingVersion", 0)  # starting version is crucial, 0 is always the entire feed from the beginning. Either thats okay or you have to track the version yourself somehow.
    .table("/mnt/bronze/<table_name>_bad")
    .filter("_change_type = 'insert'") <- improve this filter with a column and such, not all string
)

inserts_df.write.format('delta').mode("overwrite").save("/mnt/bronze/<table_name>_test")
```

By appending records from the bad queue into a unit_test table, you maintain a centralized collection of all previously problematic data, making it easier to debug and simulate how your pipeline would behave with these records when reprocessed.


# Conclusion

Handling bad records is an essential part of building resilient data pipelines. By leveraging PySpark's capabilities, we can automate the process of isolating invalid records in an bad queue, periodically re-checking them for validity, and storing them for future testing. This approach helps maintain the integrity of your pipeline, ensures that data quality issues don't cause unnecessary disruptions, and provides a streamlined way to test and fix bads in the pipeline.

With a setup like this, you can build robust data pipelines that handle edge cases effectively, automate bad management, and continuously process valid records, improving your overall workflow efficiency.
