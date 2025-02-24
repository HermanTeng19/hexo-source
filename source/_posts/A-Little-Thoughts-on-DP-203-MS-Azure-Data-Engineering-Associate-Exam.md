---
title: 'A Little Thoughts on DP-203: MS Azure Data Engineering Associate Exam'
date: 2022-06-02 22:11:57
Categories: [Data, BI]
tags: [Microsoft Azure, Cloud, Certificated, Data engineering]
toc: true
cover: /img/dp-203_certs.png
thumbnail: /img/microsoft-certified-azure-data-engineer-associate.png
---

The evolution of cloud computing over the past years has been becoming the most impacted event in both technology and business world. More and more companies from almost all industries such as financial, manufacture, education, Hi-Tech have migrated or are deploying their computing platform onto cloud, cloud provides the capabilities to be able to integrate company's all resources into a focal place, which breaks the isolations and obstacles on all kinds of business operations and implementations, one of the most important digital assets among those resources is data. In that circumstance, data engineer with cloud computing background and experience becomes highly demanded in workplace and job market, and a shortcut to crack into this career path is getting a certification in the subject.
Each cloud provider like Amazon, Microsoft and Google offer a certification specialized to their data engineering services, a very first question is which one should we choose to start with, my answer is Microsoft Azure. Here are some reasons why I draw that conclusion

* Microsoft almost dominates business software and application therefore it has the most potential customers to choose its services to be able to make their business process being migrated transparently and consistently.
* Microsoft Azure is constantly growing these years, market share growth rate is even over Amazon AWS. It is acquiring 1000 customers a day. Moreover, Azure Cloud is used by 95% of Fortune 500 companies.
* Most of the skills are easily transferable to other Cloud Providers such as [AWS](https://docs.microsoft.com/en-us/azure/architecture/aws-professional/services) and [Google Cloud](https://docs.microsoft.com/en-us/azure/architecture/gcp-professional/services)

<!-- more -->

# What about DP-203 exam

Actually, the Azure DP-203 is a very touch certificate exam to pass. It is supposed that candidate would have hand-on experiences on data engineering like data modelling, data warehouse design architecture, solid SQL, ETL data process, big data ect. Microsoft reformed exam and remove all pre-requisite exams, but I still recommend to learn **DP-900: Azure Data Fundamental** before jump to that exam. Through Azure Data fundamental, you can build the comprehensive theoretical foundation on core data concept, relational/non-relational database and analytical data process in terms of all Azure data related services, it's high-level roadmap to show horizontal and scope of data engineering, whereas, **DP-203: Azure Data Engineering Associate** focuses on know-how, how to build data infrastructure by utilizing those data services, now let's see that exam inside out.

**Exam Length**: 120 minutes

**Number of questions**: 40 - 60 (personally 45 for me)

**Full Mark**: 1000

**Passing Mark**: 700

There are three sections in the exam, ***full case study*** (5 questions), ***multiple choice*** (35 questions), and ***mini case study***(5 questions). The sequence might be vary and random, personally, I got full case study first once finish this section, I took multiple choice and mini case study. One thing mention here is full case study is separated from other sections, which means you have to finish that before move to next section, so be sure answer all questions (usually 5), multiple choice and mini case study is combined so you can mark those questions you are not sure as review and answer them later on. The difference between full case study and mini case study is former one gives you a very long business story with different scenarios which cover almost everything from storage to data process, from monitoring to security, but latter one only checks one data services like Databricks from different angles. 

Skill measurement and respective weight:

1. Design and Implement Data Storage (40-45%)
2. Design and Develop Data Processing (25-30%)
3. Design and Implement Data Security (10-15%)
4. Monitoring and Optimize Data Storage and Data Processing (10-15%)

# How to prepare DP-203 exam

## Course and materials are referred

For any standardized examination, going though all knowledge check points in syllabus and practice test are very necessary. I used Microsoft documentation, Udemy course and online practice test on github to study all exam topics

*  [Udemy course](https://www.udemy.com/course/data-engineering-on-microsoft-azure/) by Alan Rodrigues to work through the content. I liked that it showed the tools in action, illustrated the concepts, and contained exam tips and practice tests.
* There is [official content from Microsoft](https://docs.microsoft.com/en-us/learn/certifications/exams/dp-203) and that looks good too. Mixed feeling on that, because it's very boring but comprehensive, it's better to read it through out as the first learning material, it helps you build the whole Azure data engineering knowledge framework.
* [grabcert/DP203](https://github.com/grabcert/DP203) is the github repository to store 8 practice tests for free.
* [Practice Tests | DP-203: Azure Data Engineer 2022](https://www.udemy.com/course/microsoft-azure-data-engineer-exam-practice-tests/) provides 4 up to date practice tests, it is very good material even not free.
* Sign up free Azure account to get USD $200 credit to get hands-on practice.

## What are the things you should know to accomplish the exam

The scope of the exam is so broad that it’s hard to really know everything. You’ll even get occasional wildcard questions that bring in Azure services that don’t relate to data. When that does happen and you don’t know about those services, you can typically guess your way through it so long as you’re clear enough about the data services.

### Data storage in Azure cloud

* Data format - data is either *structured*, *unstructured* or *semi-structured*. The best way to exchange semi-structured Data is through data serialization languages such as (JSON, YAML..) 

* Azure data storage type

  * **Blob** - very important, all kinds of data especially unstructured
  * File - only know the basic concept, similar with file share on-premise 
  * Queue - only know the basic concept
  * Table - only know the concept, pair of key - value for semi-structured data
  * **SQL database** - very important, structured data
  * **Cosmos DB** - very important, semi-structured data

* Understand how resources are organized within Azure by hierarchical structure - Azure subscription -> resource groups -> resources like storage accounts

   ![azure_resources.png](https://o.130014.xyz/2022/06/03/azure_resources.png)

* Azure Data Lakes Gen2(ADLS) - supper important and it's key data storage service for both batch and streaming processes.

  * ADLS VS Blob: the key differences between Azure Blob storage and Azure Data Lake Storage Gen2

  * Data Lake Storage Gen2 builds on Azure Blob storage capabilities to optimize it specifically for analytics workloads. Similarly to HDFS against Linux file system, HDFS is built upon Linux file system but provides capabilities for big data analytics for HIVE and Impala.

  * If you want to store data *without performing analysis on the data*, set the **Hierarchical Namespace** option to **Disabled** to set up the storage account as an Azure Blob storage account. You can also use blob storage to archive rarely used data or to store website assets such as images and media.

  * If you are performing analytics on the data, set up the storage account as an Azure Data Lake Storage Gen2 account by setting the **Hierarchical Namespace** option to **Enabled**. Because Azure Data Lake Storage Gen2 is integrated into the Azure Storage platform, applications can use either the Blob APIs or the Azure Data Lake Storage Gen2 file system APIs to access data.

  * From data access API prospective, the way we access data through blob is different from ADLS

    | Storage account              | Prefix  | API end point                                                |
    | ---------------------------- | ------- | ------------------------------------------------------------ |
    | Azure Blob Storage           | wasb[s] | <container>@<storage_account>.blob.core.windows.net          |
    | Azure Blob Storage           | http[s] | <storage_account>.blob.core.windows.net/<container>/subfolders |
    | Azure Data Lake Storage Gen2 | abfs[s] | <container>@<storage_account>.dfs.core.windows.net           |
    | Azure Data Lake Storage Gen2 | http[s] | <storage_account>.dfs.core.windows.net/<container>/subfolders |

    you can easily tell, ADSL actually utilizes dfs (distributed file system) to support enterprise data lake for big data processes and analytics, which allow user to create external table directly querying data in raw file such as CSV, JSON and Parquet by leveraging SQL language so called **Polybase** technology which is great feature to be able to interactive with raw data without staging layer.

  * Azure Data Lake Storage Gen2 plays a fundamental role in a wide range of big data architectures. These architectures can involve the creation of:

    * A modern data warehouse
    * Advanced analytics against **big data**
    * A real-time analytical solution

  * There are four stages for processing big data solutions that are common to all architectures:

    * Data Ingestion
    * Data Store
    * Model Prep and Train
    * Model and Serve

  * SQL database is not main topic on DP-203 but DP-900. Because Azure Synapse Dedicated SQL Pool is SQL data warehouse powered by SQL database. One important function for streaming data process is you can add reference input on both **Azure SQL Database** and **Azrue ADLS Gen2 Account**.

  * Cosmos DB is a new part of the Azure data stack. It’s not explicitly part of the syllabus right now but the exam scope is broad and it does creep in. It’s worth knowing a little about it but don’t get sucked into learning all its ins and outs.

  * Data Replication Redundancy. 

    * Locally-redundant storage (LRS): replicates your storage account three times within a single data center in the primary region.

      ![locally-redundant-storage.png](https://o.130014.xyz/2022/06/03/locally-redundant-storage.png)

    * Zone-redundant storage (ZRS): replicates your storage account synchronously across three Azure availability zones in the primary region.

      [![zone-redundant-storage.md.png](https://o.130014.xyz/2022/06/03/zone-redundant-storage.md.png)](https://www.wailian.work/image/Q4Dn3k)

    * Geo-redundant storage (GRS): copies your data synchronously three times within a single physical location in the primary region using LRS. It then copies your data asynchronously to a single physical location in a secondary region that is hundreds of miles away from the primary region.

      [![geo-redundant-storage.md.png](https://o.130014.xyz/2022/06/03/geo-redundant-storage.md.png)](https://www.wailian.work/image/Q4Dp74)

    * Geo-zone-redundant storage (RZRS): it is not a often used.

      [![geo-zone-redundant-storage.md.png](https://o.130014.xyz/2022/06/03/geo-zone-redundant-storage.md.png)](https://www.wailian.work/image/Q4DNrn)

  * Data Access Tiers

    * **Hot tier** - An online tier optimized for storing data that is accessed or modified frequently. The Hot tier has the highest storage costs, but the lowest access costs.
    * **Cool tier** - An online tier optimized for storing data that is infrequently accessed or modified. Data in the Cool tier should be stored for a minimum of 30 days. The Cool tier has lower storage costs and higher access costs compared to the Hot tier.
    * **Archive tier** - An offline tier optimized for storing data that is rarely accessed, and that has flexible latency requirements, on the order of hours. Data in the Archive tier should be stored for a minimum of 180 days.

  * Users and Application services data access: **Azure Active Directory (AAD)**, **Access Key**, **Connection String**, **Key Vault Service**, **Shared Access Signatures**, **Role Base Access Control (RBAC)**, **Access Control List (ACL)**.

    * Azure Active Directory - Identity provider to grant top-level Azure could service like Azure portal, subscription, resource group.
    * Access Key - Your storage account access keys are similar to a root password for your storage account.
    * Connection String - A connection string includes the authorization information required for your application to access data in an Azure Storage account at runtime using Shared Key authorization.
    * Key Vault Service - Encrypt your access key as secret never exposure to public.
    * Shared Access Signatures - It is a URL that grants restricted access right s to Azure Storage resources. Unlike Access Key provides full access of storage resource, SAS is able to grant individual storage account resources.
      * Allowed type: Blob, File, Queue, Table
      * Allowed resource type: Service, Container, Object
      * Allowed permissions: Read, Write, Delete, List, Add, Create, Update, Process
      * Allowed date/time frame: defined access for certain start and expiry date/time
      * Allowed IP addresses
    * Role Base Access Control - It helps you manage who has access to Azure resources, what they can do with those resources, and what areas they have access to for the users in AAD.
    * Access Control Lists - Each file and directory in your storage account has an access control list. When a security principal attempts an operation on a file or directory, An ACL check determines whether that security principal (user, group, service principal, or managed identity) has the correct permission level to perform the operation.

  * Azure storage account supports almost all major programming languages including `Java`. 

### Azure Data Factory (ADF)

* Understand why the development of Data Platform changed the focus of Data Engineers from ETL to ELT approaches.
* Understand how you can perform code-free Data processing within Azure Data Factory for common transformations by leveraging so called **Data Mapping Flow** which is great toolbox if you are familiar with SSIS on-premise.
* Understand the four key components of Azure Data Factory (Datasets, Pipelines, Activities and Linked Services) and how they enable Data Factory to connect to Azure (or on-prem) Data storage / processing services.
* Understand the three integration run types of Azure Data Factory: **Azure integration run type**, **Self host integration run type**, **SSIS integration run type**.
* Trigger/schedule: **scheduled trigger**, **Event trigger**, **Tumbling window trigger**, **Custom trigger**.

### Azure Synapse Analytics

* Understand Azure Synapse Analytics environment (Data Warehousing, Big Data Analytics, Data Integration, Visualization)

  * Data Warehousing - dedicated SQL pool
  * Big Data Analytics - Spark Pool
  * Data Integration - ADF Pipeline
  * Visualization - Power BI

* Synapse SQL Pools: **Serverless Pools**, **Dedicated Pools**

* Serverless SQL Pool

  * Available in Synapse studio -> Data tab -> Workspace -> Lake database

  * It enables you to query data in your ADSL and offer a T-SQL query interface area that accommodates semi-structured and unstructured data queries by **Polybase**.

  * Polybase is a technology that access external data stored in Azure Blob storage or ADSL via T-SQL. 

    * Create external data source

      ```sql
      CREATE DATABASE SCOPED CREDENTIAL [ADLS_credential]
      WITH IDENTITY='SHARED ACCESS SIGNATURE',  
      SECRET = 'sv=2018-03-28&ss=bf&srt=sco&sp=rl&st=2019-10-14T12%3A10%3A25Z&se=2061-12-31T12%3A10%3A00Z&sig=KlSU2ullCscyTS0An0nozEpo4tO5JAgGBvw%2FJX2lguw%3D'
      GO
      
      CREATE EXTERNAL DATA SOURCE population_ds
      WITH
        -- Please note the abfss endpoint when your account has secure transfer enabled
        ( LOCATION = 'abfss://data@newyorktaxidataset.dfs.core.windows.net' ,
          CREDENTIAL = ADLS_credential ,
          TYPE = HADOOP
        ) ;
      ```

    * Create external file format

      ```sql
      CREATE EXTERNAL FILE FORMAT census_file_format
      WITH
      (  
          FORMAT_TYPE = PARQUET,
          DATA_COMPRESSION = 'org.apache.hadoop.io.compress.SnappyCodec'
      );
      ```

    * Create external table

      ```sql
      CREATE EXTERNAL TABLE census_external_table
      (
          decennialTime varchar(20),
          stateName varchar(100),
          countyName varchar(100),
          population int,
          race varchar(50),
          sex    varchar(10),
          minAge int,
          maxAge int
      )  
      WITH (
          LOCATION = '/parquet/',
          DATA_SOURCE = population_ds,  
          FILE_FORMAT = census_file_format
      )
      GO
      
      SELECT TOP 1 * FROM census_external_table
      ```

  * Serverless supports cross-databases data queries 

* Dedicated SQL Pools (SQL Data Warehouse)

  * Available in Synapse studio -> Data tab -> Workspace -> SQL database
  * Elastic pool and autoscaling
  * Data modelling: **Star schema**, **snowflake Schema**
  * SCD (slowly changing dimensions) type 1,**2**,**3**,6
  * Fact tables, dimension tables
  * fundamentals of MPP concepts and distributed architectures (Control node and compute nodes …)
  * select the correct Data Distribution to optimize your query performance (Hash Distribution, Round-Robin Distribution, Replicated Tables Distributions)
  * Dedicated SQL pools indexing options (Clustered Columnsotre index, Clustered Inedex, non-clustered)
  * Table partitioning and partitions switching
  * Best practice on authentication scenarios: AAD, managed identities
  * Security
    * Row-level security: security policy, function
    * Column-level security: grant access
    * Data sensitivity classification
    * Data Encryption - Transparent Data Encryption (TDE)  - at rest

* Apache Spark Pools

  * Process and consume data in **Lake database**.
  * Notebooks from Develop tab as an interactive way to develop ETL jobs or do ad-hoc Exploratory Data Analysis.
  * Spark high-level APIs (ideally Scala, but pyspark, C#,  Spark SQL are okay as well) but doesn't support Java and R.
  * Architecture of a Spark Workload (Driver JVM, Worker nodes running Executor JVMs, Cluster orchestrator YARN / k8s …)

### Azure DataBricks (ADB)

* Workspace and Cluster
* Differences among Apache Spark, Azure Spark Pool, Azure HD Insight, Azure Databricks
* Automated Cluster VS Interactive Cluster
  * Automated primarily for job
  * interactive primarily for analytical work
* High-concurrency VS Standard Cluster 
  * Standard supports autoscaling, Python, Scala, R and Java; run job in single cluster
  * High-concurrency doesn't support Scala; multiple users collaboration and connection
* Best fit to statistical analysis
*  integrations between Azure Databricks and the other Azure Machine Learning and Data Services.
* Databricks File System (DBFS).
* Delta Lake integration
* Data process workflow
  1. Mount the Data Lake Storage onto DBFS
  2. Read the file into a data frame
  3. Perform transformations on the data frame
  4. Specify a temporary folder to stage the data
  5. Write the results to Data Lake Storage

### Real time streaming data processing

* Input data source: **Azure Event Hub**, **ADLS Gen2**, **IoT hub**
* Stream data service: **Azure Streaming Analytics**, **Azure Databricks**
*  Five window types within Azure world (Snapshot Window, Sliding Window, Hopping Window, Tumbling Window and Session Window)
* Copy activity copy behavior
  * Flatten Hierarchy: All files from the source folder are in the first level of the target folder. The target files have autogenerated names.
  * Merge Files: Merges all files from the source folder to one file.
  * Preserve Hierarchy (default): Preserves the file hierarchy in the target folder. The relative path of the source file to the source folder is identical to the relative path of the target file to the target folder.
* The way to reduce the backlogged input events count is increase the **streaming units (SU)** for the streaming job.

### Monitor and optimize data storage and data processing

* Azure log analytics
* Azure monitoring
* Diagnostics logs

At last, I would say it usually takes about 2 month to prepare along with working full time, taking care of kid and household, average 8 -10 hours a week. The first month, go through Microsoft documentation and Video course; the second month, take the practice tests. I hope this helps you all to prepare for your exam.















