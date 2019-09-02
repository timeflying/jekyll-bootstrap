---
layout: post
title: "OLAP with Google Big Query"
description: ""
category: 
tags: []
---
{% include JB/setup %}

### Why

#### Cost
We do our OLAP task very infrequently, so we don't want to the service charged by the amount of time we use the service.
Big Query is charged by the acomunt of data you scanned during query, plus
the cost for storing data in Big Query, which is neglectable, only $1 for 1 TB of data for 1 month, so we choose it 
as our OLAP tool.

#### No maintainance effort

Compared to setting up a Spark Cluster, we just need import data into Big Query, then you can start query.
No need to worry about maintaining a cluster.

#### Flexibility

It can do almost every operations as SQL.

#### Performance

It can aggregate on TBs of data in less than 1 minute.
 

### Code for importing MySQL table to Big Query
https://gist.github.com/tilumi/28cbbf1b38147b2193dfb576dd9db7a1
##### Usage
1. Install gsutil: https://cloud.google.com/storage/docs/gsutil_install
2. Install Cloud SDK & login: https://cloud.google.com/sdk/docs/
3. Create a directory as a workspace
4. copy the script mysql_table_to_big_query.sh to the folder & make it as executable.
5. `sudo  ./mysql_table_to_big_query.sh [BUCKET_NAME] [DATASET] [DATABASE] [TABLE_NAME] [DB_USERNAME] [DB_PASSWORD] [PARTITION_FIELD] [CLUSTERING_FIELDS]`

##### Parameters:

* BUCKET_NAME: The Google Cloud Storage bucket to upload to
* DATASET: The Big Query database you want to put the table into
* DATABASE: The MySQL Database
* TABLE_NAME: The MySQL Table
* DB_USERNAME: The MySQL Username
* DB_PASSWORD: The MySQL Password
* PARTITION_FIELD: The Timestamp or DateTime column in MySQL Table you want to partition the table, which can reduce scanned data size during Query & improve performance
* CLUSTERING_FIELDS: What fields should the table in Big Query be sorted by, which can improve performance.
    
### Reference

Query Language: https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax
Partition Tables: https://cloud.google.com/bigquery/docs/partitioned-tables
Clustered Tables: https://cloud.google.com/bigquery/docs/clustered-tables

