---
title: A little thought on SQL query performance optimization 1
date: 2020-11-30 20:44:25
categories: [Data]
tags: [SQL, query, SQL Server, optimization]
toc: true
cover: /img/querytuning.jpg
thumbnail: /img/sqlicon.png
---

Working with data and database, writing query is daily routine, query running time might be big different for different queries but serve the same purpose, which shows query performance is not 100 percent determined by database system configuration, server hard ware, table index and statistics etc. but how well do you use SQL to construct your query. 

To satisfy user strong demand, we built sandbox database on production for business partners to server their needs of BI reporting and business analytical, in the meanwhile, we are also in charge of database maintenance and monitoring so that we got chance to collect all kinds of user queries. Some of them even crash the system or drag the database performance down apparently, here I want to share my thinking of query optimization on large amount of data.

## Before writing a query

Just like a project, a query is able to achieve the things with backend business logics even if it is small, so make a plan before creation is very necessary to be able to let the query return result in flash and only consume minimum system resources.

- What level detail of data do you want to query from? Business data is built on certain level, e.g. in common scenario, there are customer level, account level and transaction level data. It's better clarify target data is on which level or mix different level.
- What the time scope? The data amount volume would be big if you want to query multiple years even months, so if you evaluate the data is big, it's better use loop or paging technic instead of returning outcome in one single page.
- What are query tables look like? A table contains a lot of information, not only business attributes or columns but data type, keys, index, constraints, triggers. Familiar with table structure is going to give additional benefits when you write your queries against those tables.

<!-- more -->

## SQL query optimization tips

* **Correlated and uncorrelated subquery**

Business often asks about the highest transaction information in terms of product, product group, merchant category etc. in certain month or quarter. For this kind of request is not straight forward to be solved by single select from query but it needs subquery. Now let's see what I got from a business partner

```sql
select tran_date, Product_cd, Merchant_Name, Trans_Amt
from trans a
where tran_date between '2020-10-01' and '2020-10-31' and trans_Amt=(
	select Max(b.Trans_Amt)
	from trans b
    where a.product_cd=b.product_cd and tran_date between '2020-10-01' and '2020-10-31'
)
order by Product_cd
```

subquery is correlated with main query so `trans` data is loaded into memory again and make a calculation to return data to main query, that query execution time is 32s based on 90mm data. What if change correlated to uncorrelated subquery like below

```sql
select tran_date, Product_cd, Merchant_Name, Trans_Amt
from trans a
join (
select Product_cd, Max(Trans_Amt) as Trans_Amt, tran_date
    from trans
    where tran_date between '2020-10-01' and '2020-10-31'
    group by Producct_cd, tran_date
) as b
on a.Product_cd=b.Product_cd and a.Trans_Amt=b.Trans_Amt and a.tran_date=b.tran_date
order by a.Product_cd
```

the query execution time reduce to 26s.

* **Exists clause and table join**

The most active accounts and their corresponding spending amount is another important KPI for product manage from account management perspective, additionally, if some condition applied such as account opened on certain month, we got the query from one business partner using `exists` statement down below

```sql
select acct_id, trans_dt,count(*) as num_trans, sum(trans_amt) as tot_trans_amt
from trans a
where date_key=202010 and exists(
select 1 from acct b where a.date_key=b.date_key and a.acct_id=b.acct_id and year(acct_open_dt)*100+month(acct_open_dt)>=202009
)
group by trans_dt, acct_id
having count(acct_id)>10
order by num_trans desc
```

from the query structure, we might be able to tell `exists` statement was put in `where` clause to get the account open date, its execution time is 13s, but it's obviously not thinking about the whole business logic before, if the query changed to get target accounts for account open date first then did the calculation, the execution time will reduce to 3s like below

```sql
select a.acct_id, a.trans_dt, count(*) as num_trans, sum(trans_amt) as tot_trans_amt
from trans a join acct b on a.acct_id=b.acct_id and a.date_key=b.date_key
where year(b.acct_open_dt)*100+month(b.acct_open_dt)>=202009 and a.date_key=202010
group by a.trans_dt, a.acct_id
having count(a.acct_id)>10
order by num_trans desc
```

* **Reduce the data scope by subquery**

It's very necessary to think about your query data scope first when you do a bunch of left outer join, especially, data volume is quite big on your main left table . Below query does a series left join by using all full tables data, but based on business requirement, only partial data is required on the major left table, so it cause somehow resources wasted in reflect on the execution time of 7s

```sql
select a.*
from wfs_trans a
left join credit_decision b on a.tran_id=b.tran_id
left join finance_request c on a.req_id=c.req_id
where cast(a.creation_date as date)>='2019-01-01'
order by creation_date desc
```

after revised above query on left table, the execution time reduced to 5s (2000ms)

```sql
select a.*
from (
select *
    from wfs_trans 
    where cast(a.creation_date as date)>='2019-01-01'
) as a
left join credit_decision b on a.tran_id=b.tran_id
left join finance_request c on a.req_id=c.req_id
order by a.creation_date desc
```

* **Paging browse data**

Business users sometime complaint the SSRS or Power BI report is slow when they browse data by skipping pages, that is not the programming problem because on the backend the query to support the BI report like below

```sql
select prod_cd, post_dt, tran_dt, tran_amt
from trans
where date_key=202010 and prod_cd='ABCD'
order by post_dt
offset 1000000 rows fetch next 100 only;
```

that will take 22s to return results based on 80mm data in single month. Thing thing is even the data is ordered by index column post_dt, database engine doesn't know where is 1000000 row, it needs to recalculate again, so to answer that business concern, we recommend to apply some conditions parameter then start to browse data like below

```sql
select prod_cd, post_dt, tran_dt, tran_amt
from trans
where date_key=202010 and prod_cd='ABCD' and post_dt='2020-10-15'
order by post_dt
offset 1000 rows fetch next 100 only;
```

by this way, the report will be presented by less than 1s.



















