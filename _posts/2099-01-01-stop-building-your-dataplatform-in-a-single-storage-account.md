---
title: "[DRAFT] Stop building your data platform in a single storage account"
date: 2025-02-05
---
# Stop Building Your Data Platform in a Single Storage Account

Every data platform I’ve seen as an external consultant data platform engineer has always been built in a single storage account. Even worse, most of the time it's in a single storage account container. It has always puzzled me why. Not only for security reasons do you want to separate a data platform into multiple containers and storage accounts, but also for availability and reducing costs.

This article is written under the assumption of 4 data layers. A raw (or also called source) layer, which is where all data from the sources is copied to _as is_ with no transformations, as close to the source representation as possible. And followed by a medallion architecture. Bronze silver gold. There are different names for these layers, we use raw/bronze/silver/gold for this article.

## Data Redundancy and Business Continuity

Certain settings, such as redundancy levels, can only be configured at the Storage Account (SA) level, not at the container level. This means that if you want to apply different redundancy strategies to different data layers, you need to separate them into multiple storage accounts.

### Raw Data: most critical

The Raw data layer contains the original, foundational data, and as long as the rebuild processes for subsequent layers are idempotent, everything else can be recreated from the Raw layer. Losing Raw data would be catastrophic because it may be difficult or impossible to rebuild quickly, making it the highest priority for redundancy. For this reason, Geo-Redundant Storage (GRS) may be argued for Raw data to ensure that the data is replicated across regions, protecting it from catastrophic data center failures and ensuring that access to the original data is preserved in any disaster recovery situation.

### Silver and Bronze Data: Lower usage, lower redundancy

Silver and Bronze data are less critical than Raw data. These layers can be rebuilt from the Raw data layer in the event of data loss. Because of this, they don’t require the same level of redundancy. Lower-cost options like Locally Redundant Storage (LRS) are typically sufficient for these layers. LRS is the most cost-effective redundancy option, though it is limited to a single region. While it’s riskier, the trade-off is justified because Silver and Bronze data can be easily rebuilt from Raw data and aren’t directly critical to business continuity.

### Gold Data: Ensure Business Continuity

The Gold data layer, which is actively consumed by end-users and critical for business operations, needs high availability. While it can be rebuilt from Raw data if necessary, doing so during a disruption is not ideal, especially if the disruption is temporary (e.g., power outage). In such cases, Zone-Redundant Storage (ZRS) ensures that Gold data remains available across multiple zones within a region, allowing it to stay accessible even if a single zone fails. This is crucial for business continuity, as downtime or data unavailability at the Gold level can have significant operational and financial impacts.

### Cost-Effective as a Byproduct

By splitting your data into separate storage accounts based on redundancy needs, you gain fine-grained control over costs. The Raw data layer, which requires high redundancy for disaster recovery, and Gold data, which demands high availability, can be allocated to more expensive redundancy options. On the other hand, Silver and Bronze layers, which are less sensitive to immediate availability, can be stored with lower-cost redundancy. This approach ensures that only the most important data is protected with higher-cost redundancy, optimizing your overall storage expenses without sacrificing the integrity of your data architecture.


## Limiting the Blast Radius

When all your data resides in a single storage account and container, an issue with one dataset can easily escalate to others within the same container. Imagine you're running a custom job to compact JSON files—merging smaller files into a larger one. This process is intended to affect only data from a specific source— source_A.

However, due to an unnoticed bug—say, the incorrect file path or merge logic—the job might unintentionally update records from source_B or even affect files across other layers. This means that the issue isn't just limited to source_A; it can spread to other data sources within the same container. When multiple sources share a container, subtle issues like data corruption or unintended updates can spread, affecting more parts of the datalake than intended.

If your sources were isolated into separate containers, the blast radius of an error would be contained within that specific container. If something goes wrong with source_A, it wouldn’t impact source_B or other layers. This partitioning helps reduce the risk of cross-contamination between sources, making the error easier to isolate and fix.

Soft-delete can offer some protection against accidental deletions or corruption. However, soft-delete only works if you actively notice the issue within the retention period. Without proper monitoring or alerting, an unnoticed bug could cause gradual, subtle changes that go undetected until it’s too late.

By isolating data into distinct containers (or storage accounts), you not only limit the potential spread of errors but also improve your ability to detect and address issues before they escalate.


## Simplified Data Lifecycle Management
Different data sets have different lifecycle needs. Some need to be archived, while others should be purged after a certain period. Managing this lifecycle is easier when data is separated into multiple storage accounts.

For example, data in the Bronze layer might only need to be kept for a limited time or archived to cold storage once it’s no longer frequently accessed. By isolating data in separate storage accounts and containers, lifecycle policies can be more easily tailored to the needs of each source. 

## Conclusion

The traditional approach of placing everything in a single storage account might seem simpler at first, but it introduces significant risks in terms of security, cost, and data recovery. By splitting your data into multiple storage accounts, you not only ensure better protection against potential disasters, but you also gain flexibility in terms of redundancy, cost management, and lifecycle management.

To make this transition:

1. **Audit your current data platform**: Determine which data can be isolated into separate accounts based on its value and recovery requirements.
2. **Align redundancy and performance needs**: Choose redundancy strategies (LRS, ZRS, GRS) based on the criticality of each data layer.
3. **Implement proper governance**: Set up the right access control and monitoring systems, ensuring each storage account is secure and well-maintained.

Rethinking the storage account structure isn’t just a technical improvement—it’s a strategic move that can lead to better security, more control, and long-term cost savings.
