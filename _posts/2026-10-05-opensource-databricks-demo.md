---
title: "How Far Can You Get with Open Source Databricks?"
date: 2026-10-05
excerpt: "Everyone wants sovereign cloud now, but Databricks only runs on the big three hyperscalers. I built a runnable demo with Unity Catalog OSS, Spark, SeaweedFS and Keycloak to see what you are left with."
tags:
- Databricks
- Unity Catalog
- Spark
- Data Platform
- Sovereignty
image: /assets/graphics/2025-07-20-devops-shared-configuration-architecture/thumbnail.png
pin: false
---

Cloud sovereignty is on every roadmap and everyone wants to move to a European provider like STACKIT or OVH. Why the change of heart doesn't really matter, the requirement is here and it is real.

The problem is Databricks. It only runs on the hyperscalers. STACKIT and OVH have no Databricks offering, so if you want sovereignty you can't have Databricks. What you do have is the handful of Databricks components that are open source: Spark, Delta Lake and Unity Catalog. So how far can you get with just those?

I wanted to find out, so I built it. The repository is a demo of the current state of open source Databricks. It starts with one `docker compose up` and a few shell scripts that walk through the whole flow, from creating users to running an authenticated Spark job. Only Databricks Open Source components.

## The tech stack

Everything runs in Docker Compose:

- **Unity Catalog OSS** (v0.6.0): catalog management and authorization.
- **Unity Catalog UI**: built from source, it is not a released image.
- **Spark 4.1** standalone cluster: master, two workers, history server and a client container to submit jobs from.
- **SeaweedFS**: S3 compatible object storage. This is the piece you swap for whatever object storage your EU cloud offers.
- **Keycloak**: the identity provider, standing in for whatever your company already uses.
- **Delta Lake**: the table format.

All of it can run on a plain VM in any datacenter you choose.

## Unity Catalog OSS is barebones

Let me start with the good part: what it does, it does well. Creating catalogs and schemas works. Granting users access works. OIDC authentication works, I had it running against Keycloak without much trouble. Spark connects through the `unitycatalog-spark` connector and it behaves like you would expect.

But that is everything. The backend is an authorization layer and the UI is a catalog browser. That's it.

- No workspace, no notebooks.
- No overview of running or past Jobs.
- No SQL editor. You can't run a query in the browser.
- Grants can be given only to users, not groups. (On the roadmap for v0.8)
- The UI only supports Google authentication. Not Entra, not your own IdP. (On the roadmap, provider dependant)

If you compare it to Azure or AWS Databricks, it is an entirely different product. You open the UI, you look at your catalogs, and then you close it again because there is nothing else to do. Everything else goes through the CLI or the REST API.

There are some rough edges as well. Unity Catalog can vend storage credentials, but only AWS STS derived ones or static ones with a session token, and SeaweedFS accepts neither. So credential vending didn't work for me. Instead Spark gets temporary credentials straight from SeaweedFS STS. And with authorization enabled, only the metastore admin can register tables at explicit locations. In this demo the data access is therefore governed at the storage layer, not by Unity Catalog. Not ideal.

## Sovereignty comes with a price

If you want sovereignty and you take it seriously, you give something up.

What Unity Catalog gives you is the part you can't live without: a place to manage catalogs, and an authorization layer that decides who reads and writes what. That works. What it doesn't give you is everything that makes Databricks pleasant to work in: seeing what ran, running a quick query, managing people without writing curl commands. That is all gone.

## The demo flow

The scripts are numbered, run them in order after `docker compose up -d`.

**`01-create-keycloak-users.sh`** creates two people in Keycloak via the Admin REST API: `alice` in `lakehouse-admins` and `pietje` in `data-engineers`. The realm, groups and OIDC clients are already imported at startup, so this script only shows how you would automate onboarding.

**`02-create-unity-catalog-users.sh`** registers alice, pietje and a service account `data-eng-pipeline` in Unity Catalog. Being in Keycloak is not enough, a user also has to exist in Unity Catalog itself. Alice also gets `CREATE CATALOG` on the metastore.

**`03-create-catalog.sh`** logs in as alice, exchanges her Keycloak token for a Unity Catalog token and creates the `lakehouse` catalog. She grants pietje and the service account access and creates the SeaweedFS bucket behind it.

**`04-run-spark-job-with-auth.sh`** is the main one. The pipeline authenticates as the service account instead of a person, so nothing breaks when somebody leaves the team. The script:

1. Gets a token from Keycloak with a client-credentials grant.
2. Exchanges it for a Unity Catalog token.
3. Exchanges it at SeaweedFS STS for temporary S3 credentials.
4. Submits the Spark job with all of them passed in as `--conf`.

The job itself, `04-spark-job.py`, contains no auth code. It creates the schema `lakehouse.demo`, writes a Delta table to `s3a://lakehouse/demo/people` and reads it back.

**`05-run-spark-job-without-auth.sh`** is the negative test. It calls Unity Catalog without a token, lists and writes to S3 anonymously, and submits the same Spark job without credentials. Unity Catalog answers 401, SeaweedFS answers 403, the job fails. A demo that only shows what works doesn't prove anything about security, so I wanted to see it fail too.

When everything is running you have the Unity Catalog UI on port 3000, the Spark master on 8082, the history server on 18080 and Keycloak on 8180.

## In summary

A Spark cluster is the same as ever. Master, workers, history server, a compose file. It is not difficult to set up and leaving Databricks changes nothing about that.

Unity Catalog OSS offers just enough. It does the basic catalog management and the authorization, which is the core. What it lacks are the quality of life features that make the full Databricks experience so smooth.

Moving to a private or European cloud with only open source components is possible. But you will sacrifice features, and the development experience will be worse. Whether that is worth it depends on how much sovereignty matters to you.

## Try it yourself

All the scripts, configs and the compose file are in the repository. Clone it and run them in order.

🔗 [GitHub – databricks-oss-demo](https://github.com/KrijnvanderBurg/databricks-oss-demo)
