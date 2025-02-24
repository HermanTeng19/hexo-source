---
title: SQL Server Table Partitioning in Large Scale Data Warehouse 2
date: 2020-12-26 12:06:21
categories: [Database, BI]
tags: [SQL Server, partitioning]
toc: True
cover: /img/partitionswitchribbon.webp
thumbnail: /img/partitionswitchicon.png
---

This post is part 2, we are focusing on design and create data process to extract, transform and load  (ETL) big amount data into partitioned tables to be able to integrate to SSIS package then operate ETL process by scheduled job automatically. 

In this case, we suppose transaction data with 100 columns and 100 million records need to be loaded into data warehouse from staging table. Technically, we can do partition switching from staging table (non partitioned table) to data warehouse table (partitioned table), but under the business reality, we can't just simply do it like that, because

1. Source table and target table usually are in different database, it's not able to switch data directly because of violating same file group condition by partition switching basic in part 1. 
2. Staging table would be empty after partition switching, but the most of data transformations are applied to staging table then load final results into target table in data warehouse, so in business point of view, we can't do partition switching from staging to target neither because big impact for entire daily, weekly or monthly data process.

There might be another question: why don't we bulk load data from source to target or what benefits do we get from partition switching? Technically, yes, we can do bulk insert, however, for such big volume of data  movement, the lead time is going to be hours (2-3 hours), if the process ran on business hours, it would cause big impact and it's hard to tolerant by business users for critical data like transaction so from efficiency and performance perspective,  we have to leverage partition switching to deliver transaction data in short period of time without interrupt BI reports, dashboard refresh and business analysis.

## What is the main idea for production operation?

In order to promise transaction data is always available, we need to create a middle table to hold transaction data as source table for partition switching, we call that middle table as switch table. To satisfy all the requirements for partition switching, the switch table has to be created in as the same file group as target transaction table, identical columns, indexes, use the same partition column. We do the bulk insert from staging table to switch table then partition switch to target transaction table in data warehouse, as part 1 mentioned, this step will finish in flash as long as there is no blocking. At last, we drop switch table so the entire data process completes. Now let's dig litter deeper on details for each steps.

<!-- more -->

![partitionSwitchiingWorkflow.png](/img/screenshots/partitionSwitchiingWorkflow.png)

## Create switch table task

From part 1 table partitioning basics, we know to create partition table we need to create partition function then partition schemes to associate with partition functions, and also need to replicate all indexes from target table such as cluster index and non cluster columnstore index. To be able to let us code more reusable, we'd better encapsulate those functions into stored procedures then create a single script to call those stored procedures to finish the task.

### Create partition function and scheme for switch table

Before we jump to the code some condition need to be clarified there:

* Transaction table is partitioned by month
* Partition column is `month_key` which is 6 digit integer like 202010 (Oct, 2020), 202011 (Nov, 2020)
* Target table file group naming convention: [database name]\_FG\_[table name]\_[year]
* Partition function naming convention: [database name]\_PF\_[table name]
* Partition scheme naming convention: [database name]\_PS\_[table name]
* Cluster indexes naming convention: PK\_[table name]
* Non cluster columnstore index naming convention: CSI\_[table name]

Now, we are ready to create stored procedure to generate partition function and scheme for switch table:

```sql
use business_db
go
create procedure dbo.usp_makePartitionOnTable
 @table_name varchar(100) --target (structure source) table name
,@created_table_name varchar(100) --switch table name
,@lrang int --left range
,@rrang int --right range
,@partition_col varchar(100) = 'month_key'
,@dbname varchar(100) = 'business_db'
as
begin
begin try
declare @sp_msg varchar(max)
--check input parameter value, can't be '' or NULL
set @table_name = ltrim(rtrim(@table_name))
if (@table_name = '' or @table_name is null)
begin
	set @sp_msg = '@table_name is empty,check input value'
	goto errorProc --error capture block
end
set @created_table_name = ltrim(rtrim(@created_table_name))
if (@created_table_name = '' or @created_table_name is null)
begin
	set @sp_msg = '@create_table_name is empty, check input value'
	goto errorProc
end
set @lrang = ltrim(rtrim(@lrang))
if (@lrang = '' or @lrang is null)
begin
	set @sp_msg = '@lrang is empty, check input value'
	goto errorProc
end
set @rrang = ltrim(rtrim(@rrang))
if (@rrang = '' or @rrang is null)
begin
	set @sp_msg = '@rrang is empty, check input value'
	goto errorProc
end	
if (@rrange < 100000)
begin
	set @sp_msg = 'the boundary is out of range, check input value'
	goto errorProc
end	
declare @sSQ varchar(max)
,@iRange int
,@pfName varchar(100) --partition function
,@srcpfName varchar(100) --target table partition function
,@filegropus varchar(max)
,@psName varchar(100)
,@filegroup_name varchar(100)
,@param varchar(200)
,@minrange int
,@maxrange int

set @pfName = @dbname+'_PF_'+@created_table_name
set @srcpfName = @dbname+'PF'+@table_name
--check if partition function exists, if not, create, if so drop for rerun
declare @name varchar(100),@year int
set @sSQL = 'use '+@dbname+'; if(exists(select name from sys.objects where 
OBJECT_NAME(OBJECT_ID)='''+@created_table_name+''' and type in (''U'')))' +
'drop table dbo.'+@created_table_name
print @sSQL
exec sp_executeSQL @sSQL
--check partition scheme
set @sSQL='use '+@dbname+ '; select @name=PS.name from '+@dbname+'.sys.partition_schemes as PS
inner join '+@dbname+'.sys.partition_functions as PF on PF.function_id=PS.function_id'+'
where PF.name='''+@pfName+''''
print @sSQL
exec sp_executeSQL @sSQL, '@name varchar(100) output', @name=@spName output

if(@psName != '')
begin
set @sSQL = 'use '+@dbname+ '; drop partition scheme '+@psName
exec sp_executeSQL @sSQL
end
--check partition function
set @name=''
set @sSQL='use '+@dbname+'; select @name=name from '+@dbname+'.sys.partition_functions where
name='''+@pfName+''''
print @sSQL
exec sp_executeSQL @sSQL '@name varchar(100) output', @name=@name output

if(@name != '')
begin
set @sSQL = 'use '+@dbname+'; drop partition function '+@pfName
print @sSQL
exec sp_executeSQL @sSQL
end
--create partition function
--check if lrange is in the lrange for the src table, and rrange is in the rrange. otherwise,
--add one more in left and one more in right
if(@created_table_name<>@table_name)
begin
set @sSQL='use '+@dbname+'; select @lrange=convert(int,min(prv.value)), @rrange=convert(int,max(pre.value))'+
' from sys.partition_range_values as prv join sys.partition_functions as pfs'+
' on prv.function_id=pfs.function_id '+
' where pfs.name='''+@srcpfName+''' '
set @param = '@lrange int output, @rrange int output'
print @sSQL
exec sp_executeSQL @sSQL, @param, @lrange=minrange output, @rrange=@maxrange output
end
else
begin
set @minrange=0
set @maxrange=0
end
if(@lrange>@minrange and @rrange<@maxrange)--check the input parameter value if fall in the right range
begin
--month_key calculate for Jan for lrange
if(@lrange%100=1)
set iRange = @lrange-89
else
set @iRange = @lrange-1
end
else
set @iRange = @lrange --set default value for @iRange
if(@rrange<=@maxrange)
begin
if(@rrange%100=12)
set @rrange=@rrange+89
else
set @rrange=@rrange+1
end

set @filegroups=''
set @sSQL='use '+@dbname+';'+
'create partition function '+@pfName+'(int) as range left for values('
while(@iRange<=@rrange)
begin
set @filegroup_name=@dbname+'_FG_'+@table_name+'_'+convert(char(4),@year)
if(convert(int, right(convert(char(6),@iRange),2))=12)
set @iRange=(convert(int,left(convert(char(6),@iRange),4))+1)*100+1
else
set @iRange+=1
end
--last boundary
if(@iRange>@rrange)
begin
set @sSQL=@sSQL+')'
set @filegroups=@filegroups+@filegroup_name+')'
end
else
begin
set @sSQL=@sSQL+','
end
end
print @sSQL
exec sp_executeSQL @sSQL
--create partition scheme
set @psName=@dbname+'_PS_'+@created_table_name
set @sSQL='use '+@dbname+'; '+
'create partition scheme '+@psName+
'as partition '+@pfName+
' to ('+@filegroups

exec sp_executeSQL @sSQL
return 0
end try
begin catch
--add some try catch statement
return -1
end catch
errorProc:
--add some error catch statement
return -1
end
```

### Replicate target table structure (column and data type) for switch table

```sql
use business_db
go
create procedure dbo.usp_getTableModel
(
	@dbName as varchar(100)
    ,@table_name as varchar(100)
    ,@create_table as varcahr(100)
    ,@sSQL as varchar(max) output
)
as
begin
declare @colTemp as table
(
	name varchar(100)
    ,seqNo int
    ,IsNullable bit
    ,col_def varchar(max)
    ,data_type varchar(128)
    ,char_len int
    ,num_precs int
    ,num_dec int
)
set @sSQL = '
select ltrim(rtrim(column_name)) as name
,ordinal_position as seqNo
,case when ltrim(rtrim(is_nullable))=''no'' then 1 else 0 end as IsNullable
,ltrim(rtrim(column_default)) as col_def
,upper(ltrim(rtrim(data_type))) as data_type
,character_maximum_length as char_len
,numeric_precision as num_precs
,numeric_scale as num_dec
from '+@dbname+'.information_schema.columns
where table_name=@table_name and table_catalog=@dbname'

insert into @colTemp
exec sp_executeSQL @sSQL, '@table_name varchar(100),@dbname varchar(100)'
,@table_name=@table_name,@dbname=@dbname

declare @i int
,@name varchar(100)
,@seqNo int
,@IsNull int
,@col_def varchar(max)
,@data_type varchar(100)
,@char_len int
,@num_precs int
,@num_dec int
,@maxcount int

select @maxcount = max(seqNo) from @colTemp
;
select @i=min(seqNo) from @colTemp
;
select
@name=name
,@seqNo=seqNo
,@IsNull=IsNullable
,@col_def=col_def
,@data_type=data_typ
,@char_len=char_len
,@num_precs=num_precs
,@num_dec=num_dec
from @colTemp
where seqNo=@i

set @sSQL='create table '+quotename(@dbname)+'.dbo.'+quotename(@created_table) +'('
while not @i is Null
begin
set @sSQL = @sSQL + @Name
set @sSQL = @sSQL+' '+@data_type
if(charindex('char',lower(@date_type))>0 or charindex('var',lover(@data_type))>0)
set  @sSQL=@sSQL+'('+ltrim(rtrim(convert(varchar(4),@char_len)))+')'
if(charindex('decimal',lower(@data_type))>0)
set @sSQL = @sSQL+'('+ltrim(rtrim(convert(varchar(4),@num_precs)))+','+
ltrim(rtrim(convert(4),@num_dec)))+')'
if(@col_def<>'')
set @sSQL=@sSQL+' '+'default '+@col_def
if(@IsNull=0)
set @sSQL=@sSQL+' Null'
else
set @sSQL=@sSQL+' not Null'
if(@i<@maxcount)
set @sSQL=@sSQL+','
else
set @sSQL=@sSQL+')'
delete from @colTemp
where seqNo=@i
select @i=min(seqNo) from @colTemp
select
@name=name
,@seqNo=seqNo
,@IsNull=IsNullable
,@col_def=col_def
,@data_type=data_typ
,@char_len=char_len
,@num_precs=num_precs
,@num_dec=num_dec
from @colTemp
where seqNo=@i
end
end
go
```

 ### Replicate target table cluster index for switch table

we also need to clone cluster index from src (target) to switch table

```sql
use business_db
go
create procedure dbo.usp_ReplicateClusterIndex @table_name varchar(100)
,@created_table varchar(100)
,@dbname varchar(100)='business_db'
as
begin
declare @i int
,@sSQL varchar(max)
,@ncount int
,@colname varchar(100)
,@nloop int
,@return_result int
,@sp_msg varchar(100)
begin try
set @table_name=ltrim(rtrim(@table_name))
if(@table_name='' or @table_name is null)
begin
set @sp_msg='ended with @table_name empty, check input value'
goto errorProc
end
declare @name varchar(100)
--find out if the object exists
set @sSQL = 'use '+@dbname+ '; select @name=name from sys.indexes where
name=''PK_'+@created_table+''''
exec sp_executeSQL @sSQL, '@name varchar(100) output', @name=@name output
if @name != ''
begin
set @sSQL='use '+@dbname+'; drop index PK_'+@created_table+' on dbo.'+@created_table
exec sp_executeSQL @sSQL
end
if (@name='' or @name is null)
begin
declare @tblcol table(colID int not null, colNmae varchar(100) not null)
set @sSQL='use '+@dbname+'; select b._key,''[''+c.name+'']''
from sys.indexes a join sys.index_columns b on a.object_id=b.object_id
and a.index_id=b.index_id
join sys.columns c on b.column_id=c.column_id and b.object_id=c.object_id
where a.name=''PK_'+@table_name+''''

insert into @tblCol
exec sp_executeSQL @sSQL
select @ncount=count(*) from @tblcol;
select @i=min(colID) from @tblcol;

set nloop=o
set @sSQL = 'use '+@dbname+ '; create clustered index PK_'+@created_table+' on '+quotename(@created_table)+'('
while not @i is null
begin
select @colName=colName from @tblcol where colID=@i
if(@nloop=0)
set @sSQL=@sSQL+@colName
else
set @sSQL=@sSQL+', '+@colName
set @nloop+=1
delete from @tblcol where colID=@i
set @i=min(colID) from @tblcol
end
set @sSQL=@sSQL+')'
exec @return_result=sp_executeSQL @sSQL
if(return_result<0)
begin
--log statement
return -1
end
else
return 0
end
end try
begin catch
--try catch block statement
return -1
end catch
errorProc:
--error log statement
return -1
end
```

### Create partitioned switch table

Now we have all store procedures to meet partitioned table requirement, it's time to create a script (store procedure) to integrate in order to create partitioned switch table.

```sql
use business_db
go
create procedure dbo.usp_createPartitionTable @table_name varchar(100)
,@partition_col varchar(100)='month_key'
,@created_table varchar(100)
,@lrange int
,@rrange int
,@dbname varchar(100)='business_db'
as
begin
set nocount on;
begin try
declare @return_result int
,@sSQL varchar(max)
,sp_msg varchar(100)
set @table_name=ltrim(rtrim(@table_name))
if(@table_name ='' or @table_name is null)
begin
set @sp_msg='end with @table_name is empty, check input value'
goto errorProc
end
set @created_table =ltrim(rtrim(@created_table))
if(@created_table='' or @created_table is null)
begin
set @sp_msg='end with @created_table is empty, check input value'
goto errorProc
end
if(@lrange is null)
begin
set sp_msg='end with @lrange empty, check input value'
goto errorProc
end
if(@rrange is null)
begin
set sp_msg='end with @rrange empty, check input value'
goto errorProc
end
declare @name varchar(100),@ps_name varchar(100)
set @sSQL='select @name=name from '+quotename(@dbname)+'.sys.tables where name='''+@created_table''''
print @sSQL
exec sp_executeSQL @sSQL, '@name varchar(100) output', @name=@name output

if(@name !='')
begin
set @sSQL='drop table '+quotename(@dbname)+'.dbo.'+@created_table
exec sp_executeSQL @sSQL
end
--create partition function and scheme
exec business_db.dbo.usp_makePartitionOnTable @table_name=@table_name
,@created_table_name=@created_table
,@lrange=@lrange
,@rrange=@rrange
,@partiton_col=@partition_col
,@dbname=@dbname
--create swich table structure
exec business_db.dbo.usp_getTableModel @dbname=@dbname
,@table_name=@table_name
,@created_table=@created_table
,@sSQL=@sSQL output
set @ps_name=@dbname+'_PS_'+@created_table
set @sSQL='use '+@dbname+';'+@sSQL+' on '+@ps_name+'('+@partition_col+')'
print @sSQL
exec sp_executedSQL @sSQL
--create cluster index for switch table
exec @return_result=business_db.dbo.usp_ReplicateClusterIndex @table_name=@table_name
,@dbname=@dbname
if(@return_result<0)
begin
--add some log statement
return -1
end
else
return 0
end try
begin catch
--add try catch block statement
return -1
end catch
errorProc:
--add some error handling statement
return -1
end
```

### Business data API

We have everything to create a partitioned table so far, the only thing left is create a data API to pass in input value in terms of business RSD (Requirement Specifics document).

```sql
use business_db
go
create procedure switch_transaction
as 
begin
begin try
declare @left_range int
,@right_range int
,@return_result int
select top 1 @left_range=month_key from staging.dbo.transaction
exec @return_result=business_db.dbo.usp_createPartitionTable @table_name='transaction'
,@partition_col='month_key'
,@created_table='switch_transaction'
,@lrange=@left_range
,@rrange=@left_range
,@dbname='business_db'
if(return_result<0)
begin
--add log statement
return -1
end
else
return 0
end try
end
```

## Load staging transaction data into switch table

This step, we need to prepare data for switch table in order to make partition switching later on. Because switch table is a kind of temp table and it's not awareness by business data users, so no matter how long the loading time is, it won't cause down time on production. For the bulk insert, the simplest and fastest way is use SSIS `data flow` which provides bulk load functionality.

![dataflowstaging2switch.png](/img/screenshots/dataflowstaging2switch.png)

![targetLoadFn.png](/img/screenshots/targetLoadFn.png)

for `Data access mode` option, there is drop-down list, make sure **Table or view - fast load** being selected.

## Create columnstore index for switch table

In this step, we create columnstore index for switch table, you might ask why we don't create it along with cluster index prior data loading? We separate columnstore index from cluster index is by its special properties, because we can't load data with columnstore index enabled. Now let's brief take a look at the definition by Microsoft

> Columnstore indexes are the standard for storing and querying large data warehousing fact tables. This index uses column-based data storage and query processing to achieve gains up to 10 times the query performance in your data warehouse over traditional row-oriented storage. You can also achieve gains up to 10 times the data compression over the uncompressed data size. Beginning with SQL Server 2016 SP1, columnstore indexes enable operational analytics: the ability to run performant real-time analytics on a transactional workload.
>
> A columnstore index is a technology for storing, retrieving, and managing data by using a columnar data format, called a ***columnstore***

Obviously, we can gain a lot of performance benefits from `columnstore` index, but its specialty decides we have to treat it very carefully, because it costs a quite long time to build it for large data warehousing fact table.

we are going to create a stored procedure to achieve the function of building columnstore index then write another API to create columnstore index for switch table in terms of business requirement.

```sql
use business_db
go
create procedure usp_createColmart @table_name varchar(100)
,@src_table varchar(100) --target table 
,@dbname varchar(100)='business_db'
as
begin
declare @i int
,@sSQL varchar(max)
,@ncount int
,@colName varchar(100)
,@nloop int
,@return_result int
,@sp_msg varchar(100)
begin try
set @table_name=ltrim(rtrim(@table_name))
if(@table_name='' or @table_name is null)
begin
set sp_msg='end with @table_name is empty, check the input value'
goto errorProc
end
set @dbname=ltrim(rtrim(@dbname))
declare @name varchar(100)
set @sSQL='use '+@dbname+'; select @name=name from sys.indexes where name=''CSI_'+@table_name+''''
exec sp_executeSQL @sSQL, '@name varchar(100) output', @name=@name output
if(@name !='')
set @sSQL='use '+@dbname+'; drop index CSI_'+@table_name+' on dbo.'+@table_name
exec sp_executeSQL @sSQL
end
else
declare @tblcol table(colID int not null, colName varchar(100) not null)
set @sSQL='use '+@dbname+'; select c.column_id,c.name from sys.indexes a
inner join sys.index_columns b on a.object_id=b.object_id and a.index_id=b.index_id
inner join sys.columns c on b.object_id=c.object_id and b.column_id=c.column_id
where a.is_primary_key=0 and
a.is_unique=0 and
a.is_unique_constraint=0 and
a.name=''CSI_'+@scr_table+''''
insert into @tblcol
exec sp_executeSQL @sSQL
select @ncount=count(*)
from @tblcol

set @nloop=0
set @sSQL='use '+@dbname+'; create nonclustered columnstore index CSI_'+@table_name+' on '+
quotename(@table_name)+'('
while not @i is null
begin
select @colName=colName from @tblcol where colID=@i
if(@nloop=0)
set @sSQL = @sSQL + @colName
else
set @sSQL=@sSQL+','+@colName
set nloop+=1
delete from @tblcol where colID=@i
select @i=min(colID) from @tblcol
end
set @sSQL+=')'
exec @return_result=sp_executeSQL @sSQL
if(@return_result<0)
begin
--add log statement
return -1
end
else
return 0
end try
begin catch
--add try catch block statment
return -1
end catch
errorProc:
--error handling statement
return -1
end
```

Now, we create business data API to pass in parament value based on business requirement, in this case, we deal with transaction table

```sql
use business_db
go
create procedure switch_transaction_csi
as
begin
begin try
declare @left_range int
,@right_range int
,@return_result int

exec @return_result=business_db.dbo.usp_createColmart @table_name='switch_transaction'
,@scr_table='transaction'
,@dbname='business_db'
if(@return_result<0)
begin
--add log statement
return -1
end
else
return 0
end try
begin catch
--add try catch block statment
return -1
end catch
errorProc:
--error handling statement
return -1
end
```

## Partition switching to load data to target table

So far we create switch table and load transaction data into it then create `columnstore` index, next we are going to conduct major task to switch partition from switch table to target transaction table. For code reusable and more generic scenarios, we still need do some condition checks before we jump to partition switching.

### Check data in switch (staging) table is in the right range and ready to be proceeded

```sql
use business_db
go
create procedure usp_IsStgDataReady
(
	@stgtbl_name varchar(100)
    ,@partition_col varchar(100)='month_key'
    ,@lrange int
    ,@urange int
    ,@dbname varchar(100)='business_db'
)
as
begin
begin try
declare @sSQL varchar(max)
,@month_key int
,@sParam varchar(100)
,@range int
,@sp_msg varchar(100)

set @stgtbl_name=ltrim(rtrim(@stgtbl_name))
if(@stgtbl_name='' or @stgtbl_name is null)
begin
set sp_msg='end up with @stgtbl_name empty, check input value'
goto errorProc
end
set @partition_col=ltrim(rtrim(@partition_col))
if(@partition_col='' or @partition_col is null)
begin
set @sp_msg='end up with @partition_col empty, check input value'
goto errorProc
end
if(not len(convert(char(6),@urange))=6)
begin
set sp_msg='end up with @urange is out of range, check input value'
goto errorProc
end
set @range=@lrange
while @range <= @urange
begin
set @sParam='@month_keyOut int output'
set @sSQL='select top 1 @month_keyOut='+@partition_col+
' from'+@dbname+'.dbo.'+@stgtbl_name+'where'+@partition_key+
' ='+convert(varchar(6),@range)
exec sp_executeSQL @sSQL,@sParam,@month_keyOUt=@month_key output
if(@month_key!=@range or @month_key is null)
begin
set @sp_msg='data for '+convert(char(6),@range)+' not ready for loading'
return -1
end
if(right(convert(char(6),@range),2)='12')
begin
set @range=convert(int,(left(convert(char(6),@range),4)+1))*100+1
else
set @range+=1
end
return 0
end
end try
begin catch
--add try catch block statment
return -1
end catch
errorProc:
--error handling statement
return -1
end
```

### Check up target table partition

From previous post about table partitioning basics, the last requirement for partition switching is partition in target table must be empty, so it's necessary to check whether target table partition has data in it, if it's empty, that is good and we are ready to do the partition switching, but in production environment, we usually need to handle more complicated situations like rerun after job failed so definitely have to take it into account when target partition has data and deal with it. Firstly, let's check target partition:

```sql
use business_db
go
create procedure IsPartitionLoaded @table_name
,@partition_col varchar(100)='month_key'
,@range int
,@ret int=0 output
,@dbname varchar(100)='business_db'
as
begin
begin try
declare @sSQL varchar(max)
,@sParam varchar(100)
,@monthKey int
,@sp_msg varchar(100)
set @table_name=ltrim(rtrim(@table_name))
if(@table_name='' or @table_name is null)
begin
set sp_msg='end up with @table_name empty, check input value'
goto errorProc
end
set @partition_col=ltrim(rtrim(@partition_col))
if(@partition_col='' or @partition_col is null)
begin
set sp_msg='end up with @partition_col empty, check input value'
goto errorProc
end
if(isnumeric(@range)!=1 or not(len(convert(char(8),@range))=6))
begin
set @sp_msg='end up with @range out of range, check input value'
goto errorProc
end
set @ret=1 --target partition has data
set sParam='@retOut int output'
begin
set @sSQL='select top 1 @retOut='+@partition_col+' from '+'.dbo.'+@table_name+
'where '+@partition_col+' ='+convert(char(6),@range)
exec sp_executeSQL @sSQL, @sParam, @retOut=@monthKey output
end
if (@monthKey is null or @monthkey='')
begin
set @ret=0 --target partition is empty
return 0
end
return 0
end try
begin catch
set @ret = -1
--add try catch block statment
return -1
end catch
errorProc:
set @ret = -1
--error handling statement
return -1
end
```

### Get proper partition numbers for source (switch) and target table

Let's first handle ideal key which is empty in target partition, the last preparation we have to know is find out proper partition number on both source and target sides so that we are able to switch to right target partition.

```sql
use business_db
go
create procedure usp_getPatitionNum @pf_name varchar(100)
,@range int
,@pn int output
,@dbname varchar(100)='business_db'
as
begin
begin try
declare @sp_msg varchar(100)
set @pf_name=ltrim(rtrim(@pf_name))
if(@pf_name='' or @pf_name is null)
begin
set sp_msg='end up with @pf_name is empty, check input value'
goto errorProc
end
if((not isnumeric(@range)=1)or (@range<100000))
begin
set sp_msg='end up with @range is out of range, check input value'
goto errorProc
end
declare @sParam varchar(100)
,@sSQL varchar(max)

if @range<10000000
set @sSQL='use '+@dbname+'; select @pnOut=$PARTITION.'+@pf_name+'('+convert(char(6),@range)+')'
set @sParam='@pnOut int output'
exec sp_executeSQL @sSQL, @sParam, @pnOut=@pn output
return 0
end try
begin catch
--add try catch block statment
return -1
end catch
errorProc:
--error handling statement
return -1
end
```

### Partition switching

Now, it's time for us to switch partition to target table 

```sql
use business_db
go
create procedure usp_switchPartition @srctbl_name varchar(100)
,@destbl_name varchar(100)
,@src_pn int=0
,@des_pn int=0
,@dbname varchar(100)='business_db'
as
begin
declare @pn int
,@sSQL varchar(max)
,@sParam varchar(100)
,@ncount int
,@sp_msg varchar(100)
begin try
set @srctbl_name=ltrim(rtrim(@srctbl_name))
if(@srctbl_name='' or @srctbl_name is null)
begin
set sp_msg='end up with @srctbl_name empty, check input value'
goto errorProc
end
set @destbl_name=ltrim(rtrim(@destbl_name))
if(@destbl_name='' or @srctbl_name is null)
begin
set sp_msg='end up with @destbl_name empty, check input value'
end
set @sSQL='use '+@dbname+'; alter table '+@dbname+'.dbo.'+quotename(@srctbl_name)
if(@src_pn>0)
set @sSQL=@sSQL+' switch partition '+convert(char(4),@src_pn)
else
set @sSQL=@sSQL+' switch '
if(@des_pn > 0)
set @sSQL=@sSQL+' to'+@dbname+'.dbo.'
+ quotename(@destbl_name)+' partition '
+ convert(char(4),@des_pn)
else
set @sSQL=@sSQL+' to '+@dbname+'.dbo.'
+ quotename(#destbl_name)
exec sp_executeSQL @sSQL
return 0
end try
begin catch
--add try catch block statment
return -1
end catch
errorProc:
--error handling statement
return -1
end
```

### Clean the switch table after partition switching

The switch table is going to be empty after partition switching so it's useless now and can be drop from database as well as partition function and scheme

```sql
use business_db
go
create procedure usp_cleanTable @table_name varchar(100)
,@partition_col varchar(100)='month_key'
,@dbname varchar(100)='business_db'
as 
begin 
begin try
declare sp_msg varchar(100)
set @table_name=ltrim(rtrim(@table_name))
if(@table_name='' or @table_name is null)
begin
set sp_msg='end up with @table_name empty, check input value'
goto errorProc
end
declare @sSQL varchar(max)
,@constraint_name varchar(100)
,@return_result int
,@pf_name varchar(100)
,@ps_name varchar(100)
@name varchar(100)

set @pf_name=@dbname+'_PF_'+@table_name
set @ps_name=@dbname+'_PS_'+@table_name
--drop table
set @name=''
set @sSQL='use '+@dbname+'; select @name=name from sys.tables where name='''+@table_name+''''
exec sp_executeSQL @sSQL, '@name varchar(100) output',@name=@name output
if(@name!='')
begin
set @sSQL='use '+@dbname+'; drop table dbo.'+@table_name
exec sp_executeSQL @sSQL
--drop partition scheme
set @name=''
set @sSQL='use '+@dbname+'; select @name=name from sys.partition_schemes where name='''+@ps_name+''''
exec sp_executeSQL @sSQL, '@name varchar(100) output', @name=@name output
if(@name!='')
begin
set @sSQL='use '+@dbname+'; drop partition scheme'+@ps_name
exec sp_executeSQL @sSQL
end
--drop partition function
set @name=''
set @sSQL='use '+@dbname+'; select @name=name from sys.partition_functions where name='''+@pf_name+''''
exec sp_executeSQL @sSQL, '@name varchar(100) output', @name=@name output
if(@name!='')
begin
set @sSQL='use '+@dbname+'; drop partition function'+@ps_name
exec sp_executeSQL @sSQL
end
end
return 0
end try
begin catch
--add try catch block statment
return -1
end catch
errorProc:
--error handling statement
return -1
end
```

### Swap data out of target partition to temp table

A special stored procedure is needed to switch data out of target partition if there would be data in there.

```sql
use business_db
go
create procedure dbo.unloadData @table_name varchar(100)
,@tempTbl varchar(100)
,@range int,
,@partition_col varchar(100)='month_key'
,@dbname varchar(100)='business_db'
as
begin
begin try
declare @pn int
,@pnd int
,@return_result int
,@filegroup_name varchar(100)
,@pf_name varchar(100)
,@isRead int
,@sp_msg varchar(100)
,@ps_name varchar(100)

set @table_name=ltrim(rtrim(@table_name))
if(@table_name='' or @table_name is null)
begin
set @sp_msg='end up with @table_name empty, check input value'
goto errorProc
end
if(not isnumeric(@range)=1 or @range<100000)
begin
set @sp_msg='end up with @range out of range, check input value'
goto errorProc
end
set @pf_name=@dbname+'_PF_'+@table_name
--get src table partition
exec @return_result=business_db.dbo.usp_getPatitionNum @pf_name=@pf_name
,@range=@range
,@pn=@pn output
,@dbname=@dbname
if(@return_result<0)
begin
--add some log statement
return -1
end
if(@tempTbl='')
set @tempTbl='Temp_'+@table_name
--get des table partition
set @pf_name=@dbname+'_PF_'+@tempTbl
exec @return_result=business_db.dbo.usp_getPatitionNum @pf_name=@pf_name
,@range=@range
,@pn=@pnd output
,@dbname=@dbname
if(@return_result<0)
begin
--add some log statement
return -1
end
--swap data out from target partition to temp table
exec @return_result=business_db.dbo.usp_switchPartition @srctbl_name=@table_name
,@destbl_name=@tempTbl
,@src_pn=@pn
,@des_pn=@pnd
,@dbname=@dbname
if(@return_result<0)
begin
--add some log statement
return -1
end
return 0
end try
begin catch
--add try catch block statment
return -1
end catch
errorProc:
--error handling statement
return -1
end
```

### Control console script to integrate all functions

We are now all set so it's time to integrate all those stored procedure into one control console to implement the partition switching task.

```sql
use business_db
go
create procedure dbo.usp_switchData @scrtable_name varchar(100)
,@table_name varchar(100)
,@lrange int
,@urange int
,@partiton_col varchar(100)='month_key'
,@dbname varchar(100)='business_db'
as
begin
begin try
declare @sp_msg varchar(100)
--check input values
set @srctable_name=ltrim(rtrim(@srctable_name))
if(@srctable_name='' or @srctable_name is null)
begin
set @sp_msg='end up with @srctable_name empty, check input value'
goto errorProc
end
set @table_name=ltrim(rtrim(@table_name))
if(@table_name='' or @table_name is null)
begin
set @sp_msg='end up with @table_name empty, check input value'
goto errorProc
end
if((not isnumeric(@lrange)=1) or (@lrange < 100000))
begin
set sp_msg='end up with @lrange out of range, check input value'
goto errorProc
end
if(@urange=0)
set @urange=@lrange
set @partition_col=ltrim(rtrim(@partition_col))
--declare local variables
declare @sSQ varchar(max)
,@check_constraint varchar(100)
,@pf_name varchar(100)
,@ps_name varchar(100)
,@staging_pf_name varchar(100)
,@staging_ps_name varchar(100)
,@filegroup_name varchar(100)
,@desfilegroup_name varchar(100)
,@range_count int
,@range int
,@sParam varchar(100)
,@ret int
,@pn int
,@pn_staging int
,@index_name varchar(100)
,@return_result int
,@destbl_lrange int
--set values
set @pf_name=@dbname+'_PF_'+@table_name
set @ps_name=@dbname+'_PS_'+table_name
set @staging_pf_name=@dbname+'_PF_'+srctable_name
set @staging_pf_name=@dbname+'_PS_'+srctable_name
set @range=@lrange
set @range_count=0
set @pn_staging=0
--check if switch table data is ready for the range from @lrange to @urange
exec @return_result=business_db.dbo.usp_IsStgDataReady @stgtbl_name=@srctable_name
,@partition_col=@partiton_col
,@lrange=@lrange
,@urange=urange
,@dbname=@dbname
if(@return_result<0)
begin
--add some log statement
return -1
end
--begin while loop transaction
declare @tempTbl varchar(100)
set @temTbl='Temp'+@table_name
set @range=@lrange
while @range<=@urange
begin
set @range_count+=1
--check if data is in target partition
set @ret=0
exec return_result=business_db.dbo.IsPartitionLoaded @table_name=@table_name
,@partition_col=@partition_col
,@range=@range
,@ret=@ret output
,@dbname=@dbname
if(@ret=1)
begin
--create temp table to hold swapped data
exec @return_result=busienss_db.dbo.usp_createPartitionTable @table_name=@table_name
,@partition_col=@partition_col
,@created_table=@tempTbl
,@lrange=@lrange
,@rrange=@urange
,@dbname=@dbname
if(@return_result<0)
begin
--add some log statement
return -1
end
--create columnstore index for temTbl
exec @return_result=business_db.dbo.usp_createColmart @table_name=temTbl
,@scr_table=@table_name
,@dbname=@dbname
if(@return_result<0)
begin
--add some log statement
return -1
end
--swap data to temp table
exec @return_result=business_db.dbo.usp_unloadData @table_name=@table_name
,@tempTbl=@tempTbl
,@range=@range
,@partition_col=@partition_col
,@dbname=@dbname
if(@return_result<0)
begin
--add some log statement
return -1
end
--get partition number from destination table
exec @return_result=business_db.dbo.usp_getPatitionNum @pf_name=@pf_name
,@range=range
,@pn=@pn output
,@dbname=@dbname
if(@return_result<0)
begin
--add some log statement
return -1
end
--get partition number from switch table
exec @return_result=business_db.dbo.usp_getPatitionNum @pf_name=@staging_pf_name
,@range=range
,@pn=@pn_staging output
,@dbname=@dbname
if(@return_result<0)
begin
--add some log statement
return -1
end
--partition switching from switch to target partition
exec @return_result=business_db.dbo.usp_switchPartition @srctbl_name=@src_name
,@destbl_name=@table_name
,@src_pn=@pn_staging
,@des_pn@pn
,@dbname=@dbname
if(@return_result<0)
begin
--add some log statement
return -1
end
if(@yearload_flag=0)
begin
if(right(convert(char(6),@range),2)='12')
set @range=(convert(int,left(convert(6),@range)4))+1)*100+1
else
set @range+=1
end
else
set @range+=100
end
--drop temp table
exec @return_result=business_db.dbo.usp_cleanTable @table_name=@tempTbl
,@partition_col=@partition_col
,@dbname=@dbname
if(@return_result<0)
begin
--add some log statement
return -1
end
end
return 0
end try
begin catch
--add try catch block statment
return -1
end catch
errorProc:
--error handling statement
return -1
end
```

### Business data API for final transaction partition switching

All in all, we are going to integrate all encapsulated stored procedures into one data API to take input parameter value from outside then accomplish the task which is partition switch transaction data 100mm records in flash.

```sql
use business_db
go
create procedure transaction_partition_switch 
as
begin
begin try
declare @month_key int
,@return_result int

select top 1 @month_key=month_key from staging.dbo.transaction
exec @return_result=business_db.dbo.usp_switchData @scrtable_name='switch_transaction'
,@table_name='transaction'
,@lrange=@month_key
,@urange=0
,@partition_col='month_key'
,@yearload_flag=0
,@dbname='business_db'
if(@return_result<0)
begin
--add some log statement
return -1
end
end try
begin catch
--add try catch block statment
return -1
end catch
errorProc:
--error handling statement
return -1
end
```

## A sub-process of partition switching for error handling and restatement in production

[In previous chapter](Get proper partition numbers for source (switch) and target table), we talk about the ideal case which is there is no data or empty in target partition, but in real production environment, situation is more complicated, some unpredicted event might cause data process failed or business redefined some data elements so that reprocess and restatement is necessary, in those cases, the target partition usually has data in it, that is the thing we have to deal with.

The approach is the same, partition switching will help us out. The main idea is to create another partition temp table with the same table structure, partition column and index setting as target table in the same file group then swap data out of target partition to temp table counterpart partition, finally drop the temp table and it's partition function and scheme.

![partitionSwitchRestatement.png](/img/screenshots/partitionSwitchRestatement.png)

## Conclusion

Above, we go through the whole solution on how to apply SQL table partitioning technic to implement partition switching in real ETL process so that we can easily to schedule it run on regular basis, I believe process automation is a valuable AI technic for business operation, because the best use-case for AI projects will reduce costs, reduce risk and improve profits. more often than not, the best results are seen from implementing AI to handle the small, repetitive tasks that businesses do on a daily basis.
