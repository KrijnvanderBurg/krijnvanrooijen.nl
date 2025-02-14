---
title: "Rethinking Data Platforms: The Limits of Single Storage Architecture"
date: 2025-02-13
excerpt: "Relying on one storage account may seem simple, but it increases risk and cost. Learn how a multi-layered approach improves efficiency and security."
tags:
- Data Platform
- Architecture
- Cost Cutting
image: /assets/graphics/2025-02-13-stop-building-dataplatform-in-single-resource/thumbnail-medallion-eggs-in-broken-basket.png
pin: false
---

Every data platform I’ve worked on has always been built in a single storage resource. I always wondered, why put all your eggs in a single storage basket? While it may seem simpler - less infrastructure to manage and easier acces - the lack of separation can make your storage a mammoth. Any changes apply to all your data, sacrificing flexibility, cost control, and management.

This article reference architecture is based on Azure with four distinct data layers, each serving a specific purpose. A raw (or also called source) layer, which is where all data from the sources is copied to as is with no transformations, as close to the source representation as possible. And followed by a medallion architecture: Bronze, Silver, and Gold.

## Data Redundancy

Certain settings, such as redundancy levels, can only be configured at the Storage Account (SA) level, not at the container level. Meaning that if you want to apply different redundancy strategies to different data layers, you must separate them into multiple storage accounts.

### Raw Data: Most Critical

The Raw data layer contains the original, foundational data, and as long as the rebuild processes for subsequent layers are idempotent, everything else can be recreated from the Raw layer. **Losing Raw data would be catastrophic** because it may be difficult or impossible to rebuild quickly, making it the highest priority for redundancy. This is why Geo-Redundant Storage (GRS) is a strong choice for Raw data, guaranteeing the original data is preserved in any disaster recovery situation.

### Silver and Bronze Data: Lower Usage, Lower Redundancy

Silver and Bronze data are less critical than Raw data. **These layers can be rebuilt** from the Raw data layer in the event of data loss. Because of this, they don’t require the same level of redundancy. Lower-cost options like Locally Redundant Storage (LRS) are typically sufficient for these layers. This trade-off keeps costs low while ensuring that non-critical data remains recoverable.

### Gold Data: Ensure Business Continuity

The Gold data layer can of course then also be rebuilt, however, this layer is usually actively consumed by end-users and critical for business operations. While it can be rebuilt from Raw data if necessary, doing so is disruption of data availability. **The business disruption could be considerable**, for example in the case of a power outage at the data centre. Zone-Redundant Storage (ZRS) ensures Gold data remains available within a region, protecting against localized data centre failures and minimizing downtime while not overpaying.

## Cost-Effectiveness as a by-product

By splitting your data into separate storage accounts based on redundancy needs, you gain fine-grained control over costs. Instead of applying expensive redundancy to all data, you can selectively optimize spending based on criticality. The Raw data layer, which requires high redundancy for disaster recovery, and Gold data, which demands high availability, can be allocated to more expensive redundancy options. While Silver and Bronze layers can be stored with lower-cost redundancy. This results in significant long-term cost savings without sacrificing reliability.

## Limiting the Blast Radius

Beyond cost and redundancy, organizing storage accounts also helps contain the impact of potential failures. When all your data resides in a single storage account and container, an issue with one dataset can easily escalate to others within the same container. Imagine you're running a custom job to compact JSON files—merging smaller files into a larger one. This process is intended to affect only data from _source A_.

However, due to an unnoticed bug—say, the incorrect file path or merge logic—the job might unintentionally update records from _source B_ or even affect files across other layers. When multiple sources share a container, a single issue can cascade into a widespread data integrity problem.

If your sources were isolated into separate containers, the blast radius of an error would be contained within that specific container. If something goes wrong with _source A_, it wouldn’t impact _source B_ or other layers. This simple architectural decision enhances resilience by making errors easier to isolate.

Soft-delete can offer some protection against accidental deletions or corruption. However, soft-delete has no proper monitoring and alerting and thus only works if you actively notice the issue within the retention period. An unnoticed bug could cause gradual, subtle changes that go undetected until it’s too late.

By isolating data into distinct containers (or storage accounts), you not only limit the potential spread of errors but also improve your ability to detect and address issues before they escalate.

## Simplified Data Lifecycle Management

Different data sets have different lifecycle needs. Some need to be archived, while others should be purged after a certain period. Trying to apply uniform lifecycle policies across a single storage account leads to inefficiencies. Managing this lifecycle is easier when data is separated into multiple containers.

For example, data might only need to be kept for a limited time or archived to cold storage once it’s no longer frequently accessed. By isolating data in separate storage accounts and containers, you can automate retention and archival policies more effectively, reducing unnecessary storage costs.

## Considerations for Microsoft Fabric's OneLake

Microsoft Fabric introduces the concept of OneLake, which promotes a unified storage approach in a singular storage account. If your organization is considering a future migration to Fabric, adopting a multiple-storage account strategy could create obstacles, making the transition more complex. This concern is only relevant if Fabric is on your roadmap—otherwise, optimizing for cost and security with multiple storage accounts remains appealing.

## In Summary

Stop putting all your eggs in a single storage basket. The traditional approach of placing everything in a single storage account might seem simpler at first, but it introduces significant risks in terms of security, cost, and data recovery. Modern data architectures must prioritize resilience, efficiency, and scalability—none of which are served by a single storage account strategy.

By splitting your data into multiple storage accounts, you not only ensure better protection against potential disasters, but you also gain flexibility in terms of redundancy, cost management, and lifecycle management.

To make this transition:

1. **Align redundancy and performance needs**: Choose redundancy strategies (**LRS, ZRS, GRS**) based on the criticality of each data layer.
2. **Audit your current data platform**: Determine which data can be isolated into separate containers based on its value and recovery requirements.
3. **Implement proper governance**: Set up the right access control and monitoring systems, ensuring each storage account is secure and well-maintained.

I argue that a multi-account approach enhances security while also optimizing costs, allowing for more sustained growth.