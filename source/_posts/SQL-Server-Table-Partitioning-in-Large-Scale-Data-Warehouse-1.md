---
title: SQL Server Table Partitioning in Large Scale Data Warehouse 1
date: 2020-12-25 21:20:12
categories: [Database, BI]
tags: [SQL Server, partitioning]
toc: True
cover: /img/tablepartitionribbon1.jpg
thumbnail: /img/tablepartitionicon1.jpg
---

This is a series articles to demonstrate table partitioning technology and how to apply it in a large scale data warehouse to improve database performance and even benefit for maintenance operation based on Microsoft SQL Server. This series is composed by three parts

1. The first one is a fundamental introduction on basis knowledge
2. The second part is showcase the entire production workflow to apply table partitioning to large tables
3. Finally, it's going to be advanced topic like partition merge, split, conversion and performance optimization in terms of different business demands

## _Table Partitioning The Basics_

For this part, I will reference Cathrine Wilhelmsen's work, she is  Microsoft Data Platform MVP, BimlHero Certified Expert, international speaker, author, blogger, and chronic volunteer. She loves data and coding, as well as teaching and sharing knowledge - oh, and sci-fi, chocolate, coffee, and cats :)

I learnt a lot and got many ideas from her blogs especially for table partitioning technology, below is her blog address, hope it can help you as well:blush:

**[Cathrine blog](https://www.cathrinewilhelmsen.net/)**

<!-- more -->

This post is part 1 of 2 in the series [Table Partitioning in SQL Server](https://www.cathrinewilhelmsen.net/series/table-partitioning-in-sql-server/)

[![SQL Server Table Partitioning Cheat Sheet](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/SQLServerTablePartitioningCheatSheet-200x172.png)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/SQLServerTablePartitioningCheatSheet.png)

There are many benefits of partitioning large tables. You can speed up loading and archiving of data, you can perform maintenance operations on individual partitions instead of the whole table, and you may be able to improve query performance. However, implementing table partitioning is not a trivial task and you need a good understanding of how it works to implement and use it correctly.

Being a business intelligence and data warehouse developer, not a DBA, it took me a while to understand table partitioning. I had to read a lot, get plenty of hands-on experience and make some mistakes along the way. (*The illustration to the left is my Table Partitioning Cheat Sheet.*) One of my favorite ways to learn something is to figure out how to explain it to others, so I recently did a [webinar about table partitioning](https://pragmaticworks.com/Training/Details/Webinar-1743). (*Update in 2020: The webinar has now been archived. Please contact [Pragmatic Works](https://www.pragmaticworks.com/) if you would like to watch it, as they are the owners and publishers.*) I wanted to follow that up with focused blog posts that included answers to questions I received during the webinar. This post covers the *basics* of partitioned tables, partition columns, partition functions and partition schemes.



## What is Table Partitioning?

[![Partitioned Table](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionedTable-200x200.png)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionedTable.png)

Table partitioning is a way to divide a large table into smaller, more manageable parts without having to create separate tables for each part. Data in a partitioned table is physically stored in groups of rows called *partitions* and each partition can be accessed and maintained separately. Partitioning is not visible to end users, a partitioned table behaves like one logical table when queried.

This example illustration is used throughout this blog post to explain basic concepts. The table contains data from every day in 2012, 2013, 2014 and 2015, and there is one partition per year. To simplify the example, only the first and last day in each year is shown.

An alternative to partitioned tables (for those who don’t have Enterprise Edition) is to create separate tables for each group of rows, union the tables in a view and then query the view instead of the tables. This is called a *partitioned view*. (Partitioned views are not covered in this blog post.)

## What is a Partition Column?

[![Partition Column (Partition Key)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionKey-200x200.png)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionKey.png)

Data in a partitioned table is partitioned based on a single column, the partition column, often called the *partition key*. Only one column can be used as the partition column, but it is possible to use a computed column.

In the example illustration the date column is used as the partition column. SQL Server places rows in the correct partition based on the values in the date column. All rows with dates before or in 2012 are placed in the first partition, all rows with dates in 2013 are placed in the second partition, all rows with dates in 2014 are placed in the third partition, and all rows with dates in 2015 or after are placed in the fourth partition. If the partition column value is NULL, the rows are placed in the first partition.

It is important to select a partition column that is almost always used as a filter in queries. When the partition column is used as a filter in queries, SQL Server can access only the relevant partitions. This is called *partition elimination* and can greatly improve performance when querying large tables.

## What is a Partition Function?

[![Partition Function](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionFunction-200x183.png)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionFunction.png)

The partition function defines how to partition data based on the partition column. The partition function does not explicitly define the partitions and which rows are placed in each partition. Instead, the partition function specifies *boundary values*, the points *between* partitions. The total number of partitions is always the total number of boundary values + 1.

In the example illustration there are three boundary values. The first boundary value is between 2012 and 2013, the second boundary value is between 2013 and 2014, and the third boundary value is between 2014 and 2015. The three boundary values create four partitions. (The first partition also includes all rows with dates before 2012 and the last partition also includes all rows after 2015, but the example is kept simple with only four years for now.)

But what are the actual boundary values used in the example? How do you know which date values are the points *between* two years? Is it December 31st or January 1st? The answer is that it can actually be either December 31st or January 1st, it depends on whether you use a *range left* or a *range right* partition function.

### Range Left and Range Right

Partition functions are created as either range left or range right to specify whether the boundary values belong to their left or right partitions:

- Range left means that the actual boundary value belongs to its left partition, it is the *last value in the left partition*.
- Range right means that the actual boundary value belongs to its right partition, it is the *first value in the right partition*.

Left and right partitions make more sense if the table is rotated:

[![Partition Function Range Left and Range Right](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionFunction01-200x183.png)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionFunction01.png) → [![Partition Function Range Left and Range Right](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionFunction02-200x183.png)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionFunction02.png) → [![Partition Function Range Left and Range Right](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionFunction03-183x200.png)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionFunction03.png)

### Range Left and Range Right using Dates

The first boundary value is between 2012 and 2013. This can be created in two ways, either by specifying a range left partition function with December 31st as the boundary value, or as a range right partition function with January 1st as the boundary value:

[![Partition Function Range Left and Range Right](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionFunction04-183x200.png)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionFunction04.png) → [![Partition Function Range Left and Range Right](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionFunction05-183x200.png)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionFunction05.png) → [![Partition Function Range Left and Range Right](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionFunction06-183x200.png)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionFunction06.png)

Partition functions are created as either range left or range right, it is not possible to combine both in the same partition function. In a range left partition function, all boundary values are *upper* boundaries, they are the last values in the partitions. If you partition by year, you use December 31st. If you partition by month, you use January 31st, February 28th / 29th, March 31st, April 30th and so on. In a range right partition function, all boundary values are *lower* boundaries, they are the first values in the partitions. If you partition by year, you use January 1st. If you partition by month, you use January 1st, February 1st, March 1st, April 1st and so on:

[![Partition Function Range Left and Range Right](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionFunction07-183x200.png)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionFunction07.png) → [![Partition Function Range Left and Range Right](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionFunction08-183x200.png)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionFunction08.png)

### Range Left and Range Right using the *Wrong Dates*

If the wrong dates are used as boundary values, the partitions incorrectly span two time periods:

[![Partition Function Range Left and Range Right](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionFunction10-183x200.png)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionFunction10.png) → [![Partition Function Range Left and Range Right](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionFunction09-183x200.png)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionFunction09.png)

## What is a Partition Scheme?

[![Partition Scheme](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionScheme-500x200.png)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionScheme.png)

The partition scheme maps the logical partitions to physical filegroups. It is possible to map each partition to its own filegroup or all partitions to one filegroup.

A filegroup contains one or more data files that can be spread on one or more disks. Filegroups can be set to read-only, and filegroups can be backed up and restored individually. There are many benefits of mapping each partition to its own filegroup. Less frequently accessed data can be placed on slower disks and more frequently accessed data can be placed on faster disks. Historical, unchanging data can be set to read-only and then be excluded from regular backups. If data needs to be restored it is possible to restore the partitions with the most critical data first.

## How do I create a Partitioned Table?

The following script (for SQL Server 2012 and higher) first creates a numbers table function created by [Itzik Ben-Gan](https://www.itprotoday.com/sql-server/virtual-auxiliary-table-numbers) that is used to [insert test data](https://www.cathrinewilhelmsen.net/2015/04/14/using-a-numbers-table-in-sql-server-to-insert-test-data/). The script then creates a partition function, a partition scheme and a partitioned table. (It is important to notice that this script is meant to demonstrate the basic concepts of table partitioning, it does not create any indexes or constraints and it maps all partitions to the [PRIMARY] filegroup. This script is not meant to be used in a real-world project.) Finally it inserts test data and shows information about the partitioned table.

```sql
/* – ------------------------------------------------
 – Create helper function GetNums by Itzik Ben-Gan
 – https://www.itprotoday.com/sql-server/virtual-auxiliary-table-numbers
 – GetNums is used to insert test data
------------------------------------------------ – */

 – Drop helper function if it already exists
IF OBJECT_ID('GetNums') IS NOT NULL
	DROP FUNCTION GetNums;
GO

 – Create helper function
CREATE FUNCTION GetNums(@n AS BIGINT) RETURNS TABLE AS RETURN
  WITH
  L0   AS(SELECT 1 AS c UNION ALL SELECT 1),
  L1   AS(SELECT 1 AS c FROM L0 AS A CROSS JOIN L0 AS B),
  L2   AS(SELECT 1 AS c FROM L1 AS A CROSS JOIN L1 AS B),
  L3   AS(SELECT 1 AS c FROM L2 AS A CROSS JOIN L2 AS B),
  L4   AS(SELECT 1 AS c FROM L3 AS A CROSS JOIN L3 AS B),
  L5   AS(SELECT 1 AS c FROM L4 AS A CROSS JOIN L4 AS B),
  Nums AS(SELECT ROW_NUMBER() OVER(ORDER BY (SELECT NULL)) AS n FROM L5)
  SELECT TOP (@n) n FROM Nums ORDER BY n;
GO

/* – ----------------------------------------------------------
 – Create example Partitioned Table (Heap)
 – The Partition Column is a DATE column
 – The Partition Function is RANGE RIGHT
 – The Partition Scheme maps all partitions to [PRIMARY]
---------------------------------------------------------- – */

 – Drop objects if they already exist
IF EXISTS (SELECT * FROM sys.tables WHERE name = N'Sales')
	DROP TABLE Sales;
IF EXISTS (SELECT * FROM sys.partition_schemes WHERE name = N'psSales')
	DROP PARTITION SCHEME psSales;
IF EXISTS (SELECT * FROM sys.partition_functions WHERE name = N'pfSales')
	DROP PARTITION FUNCTION pfSales;

 – Create the Partition Function 
CREATE PARTITION FUNCTION pfSales (DATE)
AS RANGE RIGHT FOR VALUES 
('2013-01-01', '2014-01-01', '2015-01-01');

 – Create the Partition Scheme
CREATE PARTITION SCHEME psSales
AS PARTITION pfSales 
ALL TO ([Primary]);

 – Create the Partitioned Table (Heap) on the Partition Scheme
CREATE TABLE Sales (
	SalesDate DATE,
	Quantity INT
) ON psSales(SalesDate);

 – Insert test data
INSERT INTO Sales(SalesDate, Quantity)
SELECT DATEADD(DAY,dates.n-1,'2012-01-01') AS SalesDate, qty.n AS Quantity
FROM GetNums(DATEDIFF(DD,'2012-01-01','2016-01-01')) dates
CROSS JOIN GetNums(1000) AS qty;

 – View Partitioned Table information
SELECT
OBJECT_SCHEMA_NAME(pstats.object_id) AS SchemaName
,OBJECT_NAME(pstats.object_id) AS TableName
,ps.name AS PartitionSchemeName
,ds.name AS PartitionFilegroupName
,pf.name AS PartitionFunctionName
,CASE pf.boundary_value_on_right WHEN 0 THEN 'Range Left' ELSE 'Range Right' END AS PartitionFunctionRange
,CASE pf.boundary_value_on_right WHEN 0 THEN 'Upper Boundary' ELSE 'Lower Boundary' END AS PartitionBoundary
,prv.value AS PartitionBoundaryValue
,c.name AS PartitionKey
,CASE 
	WHEN pf.boundary_value_on_right = 0 
	THEN c.name + ' > ' + 
	CAST(ISNULL(LAG(prv.value) OVER(PARTITION BY pstats.object_id 
	ORDER BY pstats.object_id, pstats.partition_number), 'Infinity') 
    AS VARCHAR(100)) + ' and ' + c.name + ' <= ' + CAST(ISNULL(prv.value, 'Infinity') AS VARCHAR(100)) 
	ELSE c.name + ' >= ' 
	+ CAST(ISNULL(prv.value, 'Infinity') AS VARCHAR(100))  
	+ ' and ' + c.name + ' < ' 
	+ CAST(ISNULL(LEAD(prv.value) OVER(PARTITION BY pstats.object_id ORDER BY 
pstats.object_id, pstats.partition_number), 'Infinity') AS VARCHAR(100))
    END AS PartitionRange
	,pstats.partition_number AS PartitionNumber
	,pstats.row_count AS PartitionRowCount
	,p.data_compression_desc AS DataCompression
FROM sys.dm_db_partition_stats AS pstats
INNER JOIN sys.partitions AS p ON pstats.partition_id = p.partition_id
INNER JOIN sys.destination_data_spaces AS dds ON pstats.partition_number = dds.destination_id
INNER JOIN sys.data_spaces AS ds ON dds.data_space_id = ds.data_space_id
INNER JOIN sys.partition_schemes AS ps ON dds.partition_scheme_id = ps.data_space_id
INNER JOIN sys.partition_functions AS pf ON ps.function_id = pf.function_id
INNER JOIN sys.indexes AS i ON pstats.object_id = i.object_id 
AND pstats.index_id = i.index_id 
AND dds.partition_scheme_id = i.data_space_id 
AND i.type <= 1 /* Heap or Clustered Index */
INNER JOIN sys.index_columns AS ic ON i.index_id = ic.index_id 
AND i.object_id = ic.object_id AND ic.partition_ordinal > 0
INNER JOIN sys.columns AS c ON pstats.object_id = c.object_id 
AND ic.column_id = c.column_id
LEFT JOIN sys.partition_range_values AS prv ON pf.function_id = prv.function_id 
AND pstats.partition_number = 
(CASE pf.boundary_value_on_right WHEN 0 THEN prv.boundary_id ELSE (prv.boundary_id+1) END)
WHERE pstats.object_id = OBJECT_ID('Sales')
ORDER BY TableName, PartitionNumber;
```

## Summary

[![SQL Server Table Partitioning Cheat Sheet](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/SQLServerTablePartitioningCheatSheet-200x172.png)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/SQLServerTablePartitioningCheatSheet.png)

The partition function defines how to partition a table based on the values in the partition column. The partitioned table is created on the partition scheme that uses the partition function to map the logical partitions to physical filegroups.

If each partition is mapped to a separate filegroup, partitions can be placed on slower or faster disks based on how frequently they are accessed, historical partitions can be set to read-only, and partitions can be backed up and restored individually based on how critical the data is.

This post is the first in a series of [Table Partitioning in SQL Server](https://www.cathrinewilhelmsen.net/series/table-partitioning-in-sql-server/) blog posts. It covers the *basics* of partitioned tables, partition columns, partition functions and partition schemes. Future blog posts in this series will build upon this information and these examples to explain other and more advanced concepts.

## _Table Partitioning - Partition Switching_

This post is part 2 of 2 in the series [Table Partitioning in SQL Server](https://www.cathrinewilhelmsen.net/series/table-partitioning-in-sql-server/)

[![Partition Switching](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionSwitching.png)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionSwitching.png)

Inserts, updates and deletes on large tables can be very slow and expensive, cause locking and blocking, and even fill up the transaction log. One of the main benefits of table partitioning is that you can speed up loading and archiving of data by using *partition switching*.

Partition switching moves entire partitions between tables almost instantly. It is extremely fast because it is a metadata-only operation that updates the location of the data, no data is physically moved. New data can be loaded to separate tables and then *switched in*, old data can be *switched out* to separate tables and then archived or purged. All data preparation and manipulation can be done in separate tables without affecting the partitioned table.



## Partition Switching Requirements

There are always two tables involved in partition switching. Data is switched *from* a source table *to* a target table. The target table (or target partition) must always be empty.

(The first time I heard about partition switching, I thought it meant “*partition swapping*“. I thought it was possible to *swap* two partitions that both contained data. This is currently not possible, but I hope it will change in a future SQL Server version.)

Partition switching is easy – as long as the source and target tables meet [all the requirements](https://docs.microsoft.com/en-us/previous-versions/sql/sql-server-2008-r2/ms191160(v=sql.105)) :) There are many requirements, but the most important to remember are:

- The source and target tables (or partitions) must have *identical* columns, indexes and use the same partition column
- The source and target tables (or partitions) must exist on the *same filegroup*
- The target table (or partition) must be *empty*

If all the requirements are not met, SQL Server is happy to tell you exactly what went wrong and provides detailed and informative error messages. Some of the most common examples are listed near the end of this blog post.

## Partition Switching Examples

Partitions are switched by using the ALTER TABLE SWITCH statement. You ALTER the source table (or partition) and SWITCH to the target table (or partition). There are four ways to use the ALTER TABLE SWITCH statement:

1. [Switch from a non-partitioned table to another non-partitioned table](https://www.cathrinewilhelmsen.net/2015/04/19/table-partitioning-in-sql-server-partition-switching/#switch1)
2. **Load data by switching in:** [Switch from a non-partitioned table to a partition in a partitioned table](https://www.cathrinewilhelmsen.net/2015/04/19/table-partitioning-in-sql-server-partition-switching/#switch2)
3. **Archive data by switching out:** [Switch from a partition in a partitioned table to a non-partitioned table](https://www.cathrinewilhelmsen.net/2015/04/19/table-partitioning-in-sql-server-partition-switching/#switch3)
4. [Switch from a partition in a partitioned table to a partition in another partitioned table](https://www.cathrinewilhelmsen.net/2015/04/19/table-partitioning-in-sql-server-partition-switching/#switch4)

The following examples use code from the previous [Table Partitioning Basics](https://www.cathrinewilhelmsen.net/2015/04/12/table-partitioning-in-sql-server/) blog post. It is important to notice that these examples are meant to demonstrate the different ways of switching partitions, they do not create any indexes and they map all partitions to the [PRIMARY] filegroup. These examples are not meant to be used in real-world projects.

## 1. Switch from Non-Partitioned to Non-Partitioned

The first way to use the ALTER TABLE SWITCH statement is to switch all the data from a non-partitioned table to an empty non-partitioned table:

```sql
ALTER TABLE Source SWITCH TO Target
```

Before switch:

[![Partition Switch: Non-Partitioned to Non-Partitioned (Before)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionSwitch01-500x225.png)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionSwitch01.png)

After switch:

[![Partition Switch: Non-Partitioned to Non-Partitioned (After)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionSwitch02-500x225.png)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionSwitch02.png)

This is probably not used a lot, but it is a great way to start learning the ALTER TABLE SWITCH statement without having to create partition functions and partition schemes:

```sql
 – Drop objects if they already exist
IF EXISTS (SELECT * FROM sys.tables WHERE name = N'SalesSource')
  DROP TABLE SalesSource;
IF EXISTS (SELECT * FROM sys.tables WHERE name = N'SalesTarget')
  DROP TABLE SalesTarget;
 
 – Create the Non-Partitioned Source Table (Heap) on the [PRIMARY] filegroup
CREATE TABLE SalesSource (
  SalesDate DATE,
  Quantity INT
) ON [PRIMARY];
 
 – Insert test data
INSERT INTO SalesSource(SalesDate, Quantity)
SELECT DATEADD(DAY,dates.n-1,'2012-01-01') AS SalesDate, qty.n AS Quantity
FROM GetNums(DATEDIFF(DD,'2012-01-01','2016-01-01')) dates
CROSS JOIN GetNums(1000) AS qty;
 
 – Create the Non-Partitioned Target Table (Heap) on the [PRIMARY] filegroup
CREATE TABLE SalesTarget (
  SalesDate DATE,
  Quantity INT
) ON [PRIMARY];

 – Verify row count before switch
SELECT COUNT(*) FROM SalesSource; – 1461000 rows
SELECT COUNT(*) FROM SalesTarget; – 0 rows

 – Turn on statistics
SET STATISTICS TIME ON;

 – Is it really that fast...?
ALTER TABLE SalesSource SWITCH TO SalesTarget; 
 – YEP! SUPER FAST!

 – Turn off statistics
SET STATISTICS TIME OFF;

 – Verify row count after switch
SELECT COUNT(*) FROM SalesSource; – 0 rows
SELECT COUNT(*) FROM SalesTarget; – 1461000 rows

 – If we try to switch again we will get an error:
ALTER TABLE SalesSource SWITCH TO SalesTarget; 
 – Msg 4905, ALTER TABLE SWITCH statement failed. The target table 'SalesTarget' must be empty.

 – But if we try to switch back to the now empty Source table, it works:
ALTER TABLE SalesTarget SWITCH TO SalesSource; 
 – (...STILL SUPER FAST!)
```

## 2. Load data by switching in: Switch from Non-Partitioned to Partition

The second way to use the ALTER TABLE SWITCH statement is to switch all the data from a non-partitioned table to an empty specified partition in a partitioned table:

```sql
ALTER TABLE Source SWITCH TO Target PARTITION 1
```

Before switch:

[![Partition Switch: Non-Partitioned to Partition (Before)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionSwitch03-500x225.png)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionSwitch03.png)

After switch:

[![Partition Switch: Non-Partitioned to Partition (After)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionSwitch04-500x225.png)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionSwitch04.png)

This is usually referred to as **switching in to load data** into partitioned tables. The non-partitioned table must specify WITH CHECK constraints to ensure that the data can be switched into the specified partition:

```sql
 – Drop objects if they already exist
IF EXISTS (SELECT * FROM sys.tables WHERE name = N'SalesSource')
  DROP TABLE SalesSource;
IF EXISTS (SELECT * FROM sys.tables WHERE name = N'SalesTarget')
  DROP TABLE SalesTarget;
IF EXISTS (SELECT * FROM sys.partition_schemes WHERE name = N'psSales')
  DROP PARTITION SCHEME psSales;
IF EXISTS (SELECT * FROM sys.partition_functions WHERE name = N'pfSales')
  DROP PARTITION FUNCTION pfSales;
 
 – Create the Partition Function 
CREATE PARTITION FUNCTION pfSales (DATE)
AS RANGE RIGHT FOR VALUES 
('2013-01-01', '2014-01-01', '2015-01-01');
 
 – Create the Partition Scheme
CREATE PARTITION SCHEME psSales
AS PARTITION pfSales 
ALL TO ([Primary]);

 – Create the Non-Partitioned Source Table (Heap) on the [PRIMARY] filegroup
CREATE TABLE SalesSource (
  SalesDate DATE,
  Quantity INT
) ON [PRIMARY];
 
 – Insert test data
INSERT INTO SalesSource(SalesDate, Quantity)
SELECT DATEADD(DAY,dates.n-1,'2012-01-01') AS SalesDate, qty.n AS Quantity
FROM GetNums(DATEDIFF(DD,'2012-01-01','2013-01-01')) dates
CROSS JOIN GetNums(1000) AS qty;
 
 – Create the Partitioned Target Table (Heap) on the Partition Scheme
CREATE TABLE SalesTarget (
  SalesDate DATE,
  Quantity INT
) ON psSales(SalesDate);
 
 – Insert test data
INSERT INTO SalesTarget(SalesDate, Quantity)
SELECT DATEADD(DAY,dates.n-1,'2013-01-01') AS SalesDate, qty.n AS Quantity
FROM GetNums(DATEDIFF(DD,'2013-01-01','2016-01-01')) dates
CROSS JOIN GetNums(1000) AS qty;

 – Verify row count before switch
SELECT COUNT(*) FROM SalesSource; – 366000 rows
SELECT 
	pstats.partition_number AS PartitionNumber
	,pstats.row_count AS PartitionRowCount
FROM sys.dm_db_partition_stats AS pstats
WHERE pstats.object_id = OBJECT_ID('SalesTarget')
ORDER BY PartitionNumber; – 0 rows in Partition 1, 365000 rows in Partitions 2-4

 – Turn on statistics
SET STATISTICS TIME ON;

 – Is it really that fast...?
ALTER TABLE SalesSource SWITCH TO SalesTarget PARTITION 1; 
 – NOPE! We get an error:
 – Msg 4982, ALTER TABLE SWITCH statement failed. Check constraints of source table 'SalesSource' 
 – allow values that are not allowed by range defined by partition 1 on target table 'Sales'.

 – Add constraints to the source table to ensure it only contains data with values 
 – that are allowed in partition 1 on the target table
ALTER TABLE SalesSource
WITH CHECK ADD CONSTRAINT ckMinSalesDate 
CHECK (SalesDate IS NOT NULL AND SalesDate >= '2012-01-01');

ALTER TABLE SalesSource
WITH CHECK ADD CONSTRAINT ckMaxSalesDate 
CHECK (SalesDate IS NOT NULL AND SalesDate < '2013-01-01');

 – Try again. Is it really that fast...?
ALTER TABLE SalesSource SWITCH TO SalesTarget PARTITION 1; 
 – YEP! SUPER FAST!

 – Turn off statistics
SET STATISTICS TIME OFF;

 – Verify row count after switch
SELECT COUNT(*) FROM SalesSource; – 0 rows
SELECT 
	pstats.partition_number AS PartitionNumber
	,pstats.row_count AS PartitionRowCount
FROM sys.dm_db_partition_stats AS pstats
WHERE pstats.object_id = OBJECT_ID('SalesTarget')
ORDER BY PartitionNumber; – 366000 rows in Partition 1, 365000 rows in Partitions 2-4
```

## 3. Archive data by switching out: Switch from Partition to Non-Partitioned

The third way to use the ALTER TABLE SWITCH statement is to switch all the data from a specified partition in a partitioned table to an empty non-partitioned table:

```sql
ALTER TABLE Source SWITCH PARTITION 1 TO Target
```

Before switch:

[![Partition Switch: Partition to Non-Partitioned (Before)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionSwitch05-500x225.png)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionSwitch05.png)

After switch:

[![Partition Switch: Partition to Non-Partitioned (After)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionSwitch06-500x225.png)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionSwitch06.png)

This is usually referred to as **switching out to archive data** from partitioned tables:

```sql
 – Drop objects if they already exist
IF EXISTS (SELECT * FROM sys.tables WHERE name = N'SalesSource')
  DROP TABLE SalesSource;
IF EXISTS (SELECT * FROM sys.tables WHERE name = N'SalesTarget')
  DROP TABLE SalesTarget;
IF EXISTS (SELECT * FROM sys.partition_schemes WHERE name = N'psSales')
  DROP PARTITION SCHEME psSales;
IF EXISTS (SELECT * FROM sys.partition_functions WHERE name = N'pfSales')
  DROP PARTITION FUNCTION pfSales;
 
 – Create the Partition Function 
CREATE PARTITION FUNCTION pfSales (DATE)
AS RANGE RIGHT FOR VALUES 
('2013-01-01', '2014-01-01', '2015-01-01');
 
 – Create the Partition Scheme
CREATE PARTITION SCHEME psSales
AS PARTITION pfSales 
ALL TO ([Primary]);
 
 – Create the Partitioned Source Table (Heap) on the Partition Scheme
CREATE TABLE SalesSource (
  SalesDate DATE,
  Quantity INT
) ON psSales(SalesDate);
 
 – Insert test data
INSERT INTO SalesSource(SalesDate, Quantity)
SELECT DATEADD(DAY,dates.n-1,'2012-01-01') AS SalesDate, qty.n AS Quantity
FROM GetNums(DATEDIFF(DD,'2012-01-01','2016-01-01')) dates
CROSS JOIN GetNums(1000) AS qty;

 – Create the Non-Partitioned Target Table (Heap) on the [PRIMARY] filegroup
CREATE TABLE SalesTarget (
  SalesDate DATE,
  Quantity INT
) ON [PRIMARY];

 – Verify row count before switch
SELECT 
	pstats.partition_number AS PartitionNumber
	,pstats.row_count AS PartitionRowCount
FROM sys.dm_db_partition_stats AS pstats
WHERE pstats.object_id = OBJECT_ID('Sales')
ORDER BY PartitionNumber; – 366000 rows in Partition 1, 365000 rows in Partitions 2-4
SELECT COUNT(*) FROM SalesTarget; – 0 rows

 – Turn on statistics
SET STATISTICS TIME ON;

 – Is it really that fast...?
ALTER TABLE SalesSource SWITCH PARTITION 1 TO SalesTarget; 
 – YEP! SUPER FAST!

 – Turn off statistics
SET STATISTICS TIME OFF;

 – Verify row count after switch
SELECT 
	pstats.partition_number AS PartitionNumber
	,pstats.row_count AS PartitionRowCount
FROM sys.dm_db_partition_stats AS pstats
WHERE pstats.object_id = OBJECT_ID('SalesSource')
ORDER BY PartitionNumber; – 0 rows in Partition 1, 365000 rows in Partitions 2-4
SELECT COUNT(*) FROM SalesTarget; – 366000 rows
```

## 4. Switch from Partition to Partition

The fourth way to use the ALTER TABLE SWITCH statement is to switch all the data from a specified partition in a partitioned table to an empty specified partition in another partitioned table:

```sql
ALTER TABLE Source SWITCH PARTITION 1 TO Target PARTITION 1
```

Before switch:

[![Partition Switch: Partition to Partition (Before)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionSwitch07-500x225.png)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionSwitch07.png)

After switch:

[![Partition Switch: Partition to Partition (After)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionSwitch08-500x225.png)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionSwitch08.png)

This can be used when data needs to be archived in another partitioned table:

```sql
 – Drop objects if they already exist
IF EXISTS (SELECT * FROM sys.tables WHERE name = N'SalesSource')
  DROP TABLE SalesSource;
IF EXISTS (SELECT * FROM sys.tables WHERE name = N'SalesTarget')
  DROP TABLE SalesTarget;
IF EXISTS (SELECT * FROM sys.partition_schemes WHERE name = N'psSales')
  DROP PARTITION SCHEME psSales;
IF EXISTS (SELECT * FROM sys.partition_functions WHERE name = N'pfSales')
  DROP PARTITION FUNCTION pfSales;
 
 – Create the Partition Function 
CREATE PARTITION FUNCTION pfSales (DATE)
AS RANGE RIGHT FOR VALUES 
('2013-01-01', '2014-01-01', '2015-01-01');
 
 – Create the Partition Scheme
CREATE PARTITION SCHEME psSales
AS PARTITION pfSales 
ALL TO ([Primary]);
 
 – Create the Partitioned Source Table (Heap) on the Partition Scheme
CREATE TABLE SalesSource (
  SalesDate DATE,
  Quantity INT
) ON psSales(SalesDate);
 
 – Insert test data
INSERT INTO SalesSource(SalesDate, Quantity)
SELECT DATEADD(DAY,dates.n-1,'2012-01-01') AS SalesDate, qty.n AS Quantity
FROM GetNums(DATEDIFF(DD,'2012-01-01','2013-01-01')) dates
CROSS JOIN GetNums(1000) AS qty;

 – Create the Partitioned Target Table (Heap) on the Partition Scheme
CREATE TABLE SalesTarget (
  SalesDate DATE,
  Quantity INT
) ON psSales(SalesDate);
 
 – Insert test data
INSERT INTO SalesTarget(SalesDate, Quantity)
SELECT DATEADD(DAY,dates.n-1,'2013-01-01') AS SalesDate, qty.n AS Quantity
FROM GetNums(DATEDIFF(DD,'2013-01-01','2016-01-01')) dates
CROSS JOIN GetNums(1000) AS qty;

 – Verify row count before switch
SELECT 
	pstats.partition_number AS PartitionNumber
	,pstats.row_count AS PartitionRowCount
FROM sys.dm_db_partition_stats AS pstats
WHERE pstats.object_id = OBJECT_ID('SalesSource')
ORDER BY PartitionNumber; – 366000 rows in Partition 1, 0 rows in Partitions 2-4
SELECT 
	pstats.partition_number AS PartitionNumber
	,pstats.row_count AS PartitionRowCount
FROM sys.dm_db_partition_stats AS pstats
WHERE pstats.object_id = OBJECT_ID('SalesTarget')
ORDER BY PartitionNumber; – 0 rows in Partition 1, 365000 rows in Partitions 2-4

 – Turn on statistics
SET STATISTICS TIME ON;

 – Is it really that fast...?
ALTER TABLE SalesSource SWITCH PARTITION 1 TO SalesTarget PARTITION 1; 
 – YEP! SUPER FAST!

 – Turn off statistics
SET STATISTICS TIME OFF;

 – Verify row count after switch
SELECT 
	pstats.partition_number AS PartitionNumber
	,pstats.row_count AS PartitionRowCount
FROM sys.dm_db_partition_stats AS pstats
WHERE pstats.object_id = OBJECT_ID('SalesSource')
ORDER BY PartitionNumber; – 0 rows in Partition 1-4
SELECT 
	pstats.partition_number AS PartitionNumber
	,pstats.row_count AS PartitionRowCount
FROM sys.dm_db_partition_stats AS pstats
WHERE pstats.object_id = OBJECT_ID('SalesTarget')
ORDER BY PartitionNumber; – 366000 rows in Partition 1, 365000 rows in Partitions 2-4
```

## Error messages

SQL Server provides detailed and informative error messages if not all requirements are met before switching partitions. You can see all messages related to ALTER TABLE SWITCH by executing the following query, it is also quite a handy requirements checklist:

```sql
SELECT message_id, text 
FROM sys.messages 
WHERE language_id = 1033
AND text LIKE '%ALTER TABLE SWITCH%';
```

## Summary

[![Partition Switching](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionSwitching.png)](https://www.cathrinewilhelmsen.net/scribbles/wp-content/uploads/2015/04/PartitionSwitching.png)

Partition switching moves entire partitions between tables almost instantly. New data can be loaded to separate tables and then *switched in*, old data can be *switched out* to separate tables and then archived or purged. There are many requirements for switching partitions. It is important to understand and test how partition switching works with filegroups, indexes and constraints.

This post is the second in a series of [Table Partitioning in SQL Server](https://www.cathrinewilhelmsen.net/series/table-partitioning-in-sql-server/) blog posts. It covers the *basics* of partition switching. Future blog posts in this series will build upon this information and these examples to explain other and more advanced concepts.







