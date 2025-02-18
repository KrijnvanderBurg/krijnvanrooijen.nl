---
title: "Idempotency: the missing data engineering principle"
date: 2025-03-04
excerpt: ""
tags:
- Data Engineering
- Data Platform
image: /assets/graphics/2099-01-01-Idempotency-the-missing-data-engineering-principle/thumbnail-....png
pin: false
---


lets workout a backbone of example with how its done wrong, and how it can be fixed to make idempotant.

lets say we have an ingestion process from an external restful HTTP api, fetching json data. Its a daily ingestion, fetching data from that day. If we ingest directly into bronze layer, a delta table, there are a few things going wrong, first of all there is a transformation of the source dat ainto the platform, what if there is an issue in the json to delta transformation? then this process cannot be redone, the failed ingestions will remain failed, it will have to be re-ingested, if its copied as-is, then the data in the raw layer are always accurate because nothing was cahnged.

But the problem continues. What if there is a mistake in this directly into bronze ingestion process. There are no easy underlying files a person can just remove and redo, its in a delta format, good luck finding the data through the delta log, its not workable for a person. What if data was incorrectly ingested and columns were mixed up, the values of column X ended up in Y and vice versa. Now you have to make some query or change to the table to fix this, but you are fixing a live table where you have to be careful which rows to fix the column for, what if it only applied to a small amount of records. You cant remove them because then you lost the data, but you cant just run a query against the whole table because only part rows are affected. This all makes it just increasingly difficult, its not unworkable, its just unneccessarily complex. This is because processes are not idempotant.

So what does that mean? If the processes were idempotant it would be easy to remove all the incorrect files, which if using a raw layer would mean deleting the entire folder of the day that was incorrectly ingested. And without even making a change, everything else would go correctly. If you setup the pipeline with a widget (when using spark) that accepts the date it has to query the http restful api, then you can automate it by looking at the current date and hardcode a start date for this ingestion process, look through all partitioned folders and find missing dates, which if things go correctly would only be the todays date, but if you notice a mistake in the data, you can just go into the raw layer and delete this day and just wait for the process to start itself on its natural schedule or trigger. A self healing pipeline. All I have to do is notice a mistake, with data quality checks that run regularly (not further covered in here) and if a mistake is made, just damn delete the data (if its still retrievable from the original source). But if you use a raw layer anyway without any transformations, then youre not even dependant on the original source. Just delete in the layer that went wrong like bronze and silver, and restart the process or waits its natural trigger and have it self healing.

The goal additionally is to have this self healing propogate through multiple layers. Either batching or streaming, for both it can remain the same process. If you notice something went wrong from bronze to silver. Just delete silver and gold records, or even teh whole table if you dont care about additional compute processing costs (and checkpoint if streaming). If your data pipeline is self healing and idempotant then silver will automatically restor eitself from bronze, and the same for gold from silver.  ALl you have to do is remove the wrong data and nothing else.



Your Data Platform is a House of Cards Without Idempotency

The Problem: When Data Pipelines Don’t Heal

Let’s talk about failure—because in data engineering, things will break. A botched schema update, an API returning bad data, or a simple human mistake. And when that happens, how easy is it for your data platform to recover?

For many teams, the answer is: it’s a nightmare.

Take a simple example. You’re ingesting JSON data daily from an external RESTful HTTP API into your Bronze layer. The pipeline fetches the latest day’s data and writes it directly into a Delta table. Seems straightforward, right? Until you realize:

Transformation errors ruin recoverability. If something breaks in the JSON-to-Delta transformation, there’s no way to retry. Since you didn’t store the raw input, you have to hope the source API retains history (which it probably doesn’t).

Data corruption is buried under Delta logs. A schema mismatch swaps column X and Y. But Delta logs abstract the underlying files, so you can’t just remove the bad batch. You’re now reverse-engineering logs, writing careful SQL updates, and praying you don’t break things further.

Ingestion failures require manual intervention. If the pipeline appends data rather than fully replacing it, a failed run doesn’t auto-recover. Running it again either duplicates records or partially fixes the issue, leaving a mess.

At this point, your recovery process is manual, fragile, and slow. Instead of simply re-running the pipeline, you’re writing one-off fixes, tracking ingestion gaps manually, and dealing with duplicates.

This isn’t just operational pain. It’s a design flaw.

The issue? Your data pipelines are not idempotent.

What Does Idempotency Mean in Data Engineering?

Idempotency means that re-running the same process produces the same result—without duplication, without side effects, and without requiring manual cleanup.

If your pipeline were idempotent, fixing issues wouldn’t require SQL gymnastics or rollback scripts. Instead, you could simply delete the bad data and let the pipeline heal itself.

Let’s rebuild this ingestion process the right way.

The Fix: Self-Healing Ingestion with a Raw Layer

Instead of dumping API data straight into Bronze, store the raw JSON as-is first:

/raw/api_source/yyyy/MM/dd/*.json

This instantly solves multiple problems:

✅ No transformation failures—because no transformation happens here.✅ Full recoverability—delete a bad day’s data and just re-run ingestion.✅ No dependency on the API’s history—you control the raw data.

Now, let’s fix the second issue—how do we make Bronze (Delta) self-healing?

How a Self-Healing Pipeline Works

If your data platform is idempotent, fixing an error is as simple as:

Delete the incorrect data at any layer (Bronze, Silver, or Gold).

The pipeline naturally reprocesses it without human intervention.

Here’s how it’s designed:

Instead of blindly ingesting “new” data, the pipeline checks which days are missing and backfills them.

If today’s ingestion fails? Just delete today’s folder in Raw and let the job run again. No manual SQL fixes. No hunting through Delta logs. No hoping you fix only the bad rows.

And this logic cascades downstream:

If you find bad data in Silver, delete Silver and let it rebuild from Bronze.

If Gold is corrupted, delete Gold and let it rebuild from Silver.

Each layer automatically restores itself from the layer below. The only thing you have to do? Delete the bad data. That’s it. The system does the rest.

Expanding the Example: From Raw Data to a Resilient Platform

1. Strengthening the Raw Layer Foundation

Imagine your data pipeline ingesting JSON files from an external RESTful API every day. Instead of immediately transforming and loading the data into your Bronze layer, you first store the raw JSON files:

/raw/api_source/yyyy/MM/dd/*.json

Why This Matters:

Data Preservation: The raw files capture data exactly as received. If a transformation later fails or is found to be flawed, you always have a pristine copy for reprocessing.

Independent Recovery: Even if downstream transformations encounter errors, you can always restart the ingestion process for a particular day without relying on the API to retain historical data.

2. The Transformation Process: Raw to Bronze

With raw data safely stored, you move it into the Bronze layer, where minimal transformations (like schema normalization or minor cleansing) are applied. Crucially, this transformation is designed to be idempotent. For example, consider this simplified process in pseudo-code:

def process_day(date_str):
    raw_path = f"/raw/api_source/{date_str}/*.json"
    bronze_path = f"/bronze/api_source/{date_str}"
    
    # Read the raw JSON data
    raw_data = spark.read.json(raw_path)
    
    # Normalize and cleanse the data
    transformed_data = normalize_data(raw_data)
    
    # Write data to Bronze using overwrite mode to ensure idempotency
    transformed_data.write.mode("overwrite").format("delta").save(bronze_path)

3. Advanced Self-Healing Techniques

Metadata & Checkpointing: Logs track which partitions were processed, helping identify where failures occurred.

Automated Monitoring & Alerting: Anomalies trigger automated workflows that clean and reprocess affected partitions.

Immutable Data Lakes: By treating data as immutable—never altering stored data but appending corrections—you ensure clean and controlled reprocessing.

Final Thoughts: Building a Truly Resilient Data Platform

Designing your data pipelines for idempotency and self-healing ensures they are fault-tolerant, self-recovering, and optimized for growth. If your pipelines can’t recover on their own, you’re building a house of cards. Make it self-healing. Make it idempotent. Or be ready to spend your nights debugging broken data.


    The pain of non-idempotent data pipelines.
    How failure manifests in real-world ingestion pipelines.
    The core concept of idempotency in data engineering.
    Introduction to self-healing mechanisms.


    Best practices for storing raw data before processing.
    Versioning and partitioning strategies to ensure re-runs are clean.
    How to use Delta Lake, Iceberg, or Hudi for reliable ingestion.
    Example implementation with Spark/Delta.


    Using event triggers instead of cron-based scheduling.
    How Kafka or cloud-native event buses can enable automatic reprocessing.
    Designing a system that reacts dynamically to failures.