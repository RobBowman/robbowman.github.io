---
title: Migrating from Table Storage to Cosmos DB Table Api
category: Azure
tags:
    - Azure
    - Sql Server
    - Cosmos
---
# Migrating XXX from Table Storage to Cosmos DB Table Api
I recently worked on an Azure Function App that was using Table Storage to persist data. Table storage is an obvious choice for Azure Function Apps since:

+ already have a storage account 
+ very low cost

The storage had worked well during development but once NFRs and app query profile became better understood, I became a little concerned about performance and decided to run some load tests.

# Index Limitation
The main cause for concern was the limited flexibility for indexes in Table Storage.

With Azure Table Storage, indexing to speed data retrieval can only be used against Partition Key, Row Key or a combination. Relative performance is given in the following table:

| Type           | When Used                                     | Performance |
|----------------|-----------------------------------------------|-------------|
| Point Query    | Partition Key and Row Key values are known    | Fastest     |
| Partition Scan | Only partition Key value is known             | Mid         |
| Table Scan     | Neither Partition Key or Row Key values known | Slowest     |

During development we began to uncover scenarios in which it would not be possible to make use of a Point Query, our "Where Clause" would need to be made against a column that was not the Row Key.

# Performance Testing
In order to test, a sample PoC was created:
+ C# in-process Azure Function hosted on app service plan
+ Azure table storage table loaded with 1.5m entities

Query response times tested by http requests to the azure function from Postman – results below:

| Query Type                                              | Response Time |
| ------------------------------------------------------- | ------------- |
| Point Query – partition key + row key + search property | 22ms          |
| Partition Scan – partition key + search property        | 36,000ms      |
| Table Scan – search property only                       | 58,000ms      |

The results show a very dramatic decrease in performance for those queries where it's not possible to make us of a point query.

# Solution
Azure Cosmos DB with Table API is an obvious upgrade from Azure Table Storage because:

+ Supports same C# Software Development Kit (SDK) as that used by Azure Table Storage
+ Automatic and complete indexing on all properties by default, with no index management

The sample PoC application mentioned in the previous section was used to test performance of Cosmos DB. The same 1.5m entities from the table storage test were this time loaded into Cosmos DB. The same queries were run, results shown below:

| Query Type                                              | Response Time |
| ------------------------------------------------------- | ------------- |
| Point Query – partition key + row key + search property | 51ms          |
| Partition Scan – partition key + search property        | 53ms          |
| Table Scan – search property only                       | 52ms          |

Note: tests were run against a cosmos db provisioned in auto-scale set to the lowest (and least expensive) maximum performance of 1,000 request units per second.

# Consistency Level
Cosmos DB supports 5 difference consistency levels as described here. The best fit for our use case was consistency level “Strong”, giving the following behaviour:

>The reads are guaranteed to return the most recent committed version of an item. A client never sees an uncommitted or partial write. Users are always guaranteed to read the latest committed write.

# Quick Win!
The only change that was required when running the function app against the Cosmos DB vs Table Storage was to change the connection string.

# Costs
The cost for table storage can be considered zero, since a storage account is required to enable the Azure function and there's minimal additional cost for the extra use of the table storage.

Cosmos DB has several factors that influence cost, the primary being number of request units per second (RUs) that can be processed.

For our database, the most appropriate option was deemed to be Auto-scale. This allows Cosmos DB to automatically adjust the RUs (to a pre-defined maximum) based on current usage, allowing it to handle variable workloads efficiently.

Throughput scale can be set at individual container (table) level, or a single value to cover all containers within the database. Due to the relatively low demands of the Function App, it’s expected that the lowest auto-scale value of 1,000 RUs would be sufficient to apply at database level.

Throughput capabilities of the database can be easily updated in future.

| Element | Level                                    | Price                 |
| ------- | ---------------------------------------- | --------------------- |
| Compute | 1,000 RUs Auto-scale 50% avg utilisation | £48.13 p/m per region |
| Storage | 4gb (1gb per ISDB table)                 | £0.84 p/m per region  |

# Conclusion
+ Migrating an app from Azure Table Storage to Azure Cosmos DB Table Api is as simple as changing the connnection string
+ Ability to index multiple columns (default is to have all columns indexed) can give massive performance improvement
+ Additonal costs easily justified for our use case 