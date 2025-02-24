---
title: Blocking Process Monitoring and Auto Email Notification in SQL Server
date: 2020-12-17 21:54:30
categories: [IT, BI]
tags: [SQL Server, database, monitoring, system optimization]
toc: true
cover: /img/blockingribbon.webp
thumbnail: /img/sqlfnicon.png
---

From function perspective, to maintain a large scale business data warehouse is for making database system stable, robust and fast, which is a essential part to boost business team productivity and performance. However, for business users, the fundamental thing is **data**, so data can be delivered in high frequency and in time is the cornerstone for all business analysis and BI reporting. Obviously, the primary mandate for data management team is highly monitor ETL jobs to promise data process running well and smooth and never being blocked or corrupt by using process.

In this article, I am going to talk about how to build up a ETL job monitoring system to watch user query automatically in designed frequency. To accomplish this task, we need 

1. Create a view to collect user query in real time
2. Create a stored procedure to detect blocking in different situations
3. Create another stored procedure to handle the notification email sending
4. Create a console script to overall control the workflow

<!-- more -->

to make clear with those step by a process workflow chart, we can easily to see the logic on behand the scene

![blockingworkflow.png](/img/screenshots/blockingworkflow.png)

Firstly, SQL agent job is going to check blocking transaction every 30 minutes, if there are some user queries blocked ETL job execution service account, then check back database to determine if those transactions are new, if they are new then write into database `Blocking_Transaction` table and send email notification  to data management team to raise awareness; but if those blocking are not new, update `last_time` and `count` column in table then calculate if count number reach to 4 if yes, then send email to user and data management team.

## Create a view to collect user query in real time

 The first and most important thing for us is get the all transaction records against database so that we are able to know which user query transactions would block service account conduct ETL job. We need to utilize sys schema tables or dmv (dynamic management views) 

```sql
use controlDB;
go

create view dbo.usrQueryMonioring
as
with t1 as
(
select b.spid, c.DBName as [DB_Name]
    ,a.total_scheduled_time/1000 as TotalScheTime_SS
    ,a.total_scheduled_time/1000/60 as TotalScheTime_MM
    ,a.total_elapsed_time/1000 as TotalElapTime_SS
    ,a.total_elapsed_time/1000/60 as TotalElapTime_MM
    ,a.last_request_start_time
    ,a.last_request_end_time
    ,a.login_time
    ,a.login_name
    ,a.host_name
    ,a.program_name
    ,a.nt_domain
    ,a.nt_user_name
    ,a.status
    ,a.is_user_process
    ,b.blocked --add blocking query information to quick locate the blocking transactions
    ,b.cmd,
    ,b.memusage*8 as Memusage_KB
    ,b.memusage*8/1024 as Memusage_MB
    ,c.Query
    from master.sys.dm_exec_sessions a
    join master.sys.sysprocesses b
    on a.session_id=b.spid and a.login_time=b.login_time
    cross apply
    (
        select
        from master.sys.dm_exec_sql_text(b.sql_handle)
    ) c
    where a.is_user_process=1 --consider spids for user only, no system spids
    and a.session_id!=@@SPID --don't include request from current spid
),
t2 as
(
select
    spid
    ,DB_Name
    ,TotalScheTime_SS
    ,TotalScheTime_MM
    ,TotalElapTime_SS
    ,TotalElapTime_MM
    ,blocked as BlockedBy
    ,last_request_start_tiem
    ,last_request_end_time
    ,login_time
    ,login_name
    ,case when host_name like 'your server name%' then 'Server'
    else 'Client' end as 'Host Name'
    ,case when program_name like 'Microsoft SQL Server Mangement Studio%' then 'SQL Query'
    when program_name like 'Python' then 'Python Access'
    else 'Server general job or data provider' end as 'Program Name'
    ,nt_domain
    ,nt_user_name
    ,status
    ,case is_user_process when 0 then 'SA System'
    when 1 then 'User Initiate' end as 'Is_User_Process'
    ,cmd
    ,Memusage_KB
    ,Query
    from t1
)
select *
from t2
;
go
```

Before moving forward, we need a middle table to store all service account blocking transaction records 

```sql
use controlDB;
go
create table dbo.blocking_transaction
(
TraceID int identity(1,1) not null
    ,spid smallint not null
    ,blokedBy
    ,login_name nvarchar(128) not null
    ,nt_user_name nvarchar(128) null
    ,TotalElapTime_MM int not null
    ,ps_time datetime not null
    ,fr_Time datetime not null --first record time
    ,lr_Time datetime not null --last record time
    ,counter smallint not null
    ,cmd nchar(16) not null
    ,query varchar(max) null
)
```



## Create a stored procedure to detect blocking in different situations

Next we will apply the main logics to detect service account blocking transactions in terms of different scenarios when we have the query transaction records data. In this time, we need to create a stored procedure to be executed by SQL job.

```sql
create proc dbo.usp_blocking_trans (@counter smallint output, @rowcnt int output)
as
if OBJECT_ID(N'tempdb..#blked') is not null
drop table #blked
;
declare @row_cnt int
declare @sSQL nvarchar(max)
declare @sParam nvarchar(4)
begin try
with t1 as
(
select spid, BlockedBy, login_name, nt_user_name, TotalElapTime_in_MM,last_request_start_time,cmd,query
    from controlDB.dbo.usrQueryMonioring
    where BlockedBy!=0 and login_name='your service account'
),t2 as
(
select spid, BlockedBy, login_name, nt_user_name, TotalElapTime_in_MM,last_request_start_time,cmd,query
    from t1
    except
    select spid, BlockedBy, 
    login_name, nt_user_name, TotalElapTime_in_MM,last_request_start_time,cmd,query
    from t1 a join t1 b
    on a.spid=b.BlockedBy --remove duplicates
),t3 as
(
select spid, BlockedBy, login_name, nt_user_name, TotalElapTime_in_MM,last_request_start_time,cmd,query
    from controlDB.dbo.usrQueryMonioring
    where spid in (select BlockedBy from t2) -- in operator to hold potential multiple transactions
),t4 as
(
select *
    from t2
    union all
    select *
    from t3
)
select *
into #blked
from t4
/*return 1 if no record in temp table*/
if not exists(select * from #blked)
begin
print 'no record in temp table'
return 1
end
/*check for incremental load*/
else
select @row_cnt=count(a.spid)
from #blked a left join controlDB.dbo.blocking_transaction b
on a.spid=b.spid and a.last_request_start_time=b.ps_time
where b.spid is null and b.ps_time is null
;
if @row_cnt>=1
begin
set @sSQL=N'
insert into dbo.blocking_transaction
select a.spid,a.BlockedBy,a.login_name,a.nt_user_name,
a.TotalElapTime_MM,a.last_request_start_time,getdate(),getdate(),1,a.cmd,a.query
from #blked a
'
exec sp_executesql @sSQL
select @rowcnt=@@ROWCOUNT --output number of row inserted
return 2
end
/*update the old records on last_record_time and count if counter>=3*/
else
select top(1) @counter = counter from controlDB.dbo.blocking_transaction order by lr_time desc
;
update a 
set lr_time=getdate(),counter=@counter+1
from blocking_transaction a inner join #blked b
on a.spid=b.spid and a.ps_time=b.last_reqeust_start_time
;
if (select top(1) counter from blocking _transaction order by lr_time desc)%4 = 0
return 3
else
return 4
drop table #blked
begin catch
declare @err_msg nvarchar(1000),@err_num int,@err_line int,@syserr nvarchar(1000)
select @err_msg=ERROR_MESSAGE(),@err_num=ERROR_NUMBER(),@err_line=ERROR_LINE()
SET @syserr=N'usp_blocking_trans ended with error: 
Line='+convert(nvarchar(3),@err_line)+', Error_Msg='+@err_msg+''
return -1
end catch
end
```

## Create another stored procedure to handle the notification email sending

Sending email to raise awareness is another important task for the entire monitoring system, in this case there are two server levels, if user query blocking time less than 4 (4*30=120 mins) only email to data management team, otherwise, email to both data management team and user to cancel the transaction. If user ignores then data management team would do something to clear the lock by policy.

we use parameters output from stored procedure `usp_blocking_trans` as input to be the key conditions

```sql
use controlDB;
go
create procedure dbo.usp_blocking_Email
(
	@case smallint,
    @times smallint,
    @rownum int --handle multiple blocking case
)
as
begin try
declare @result int
declare @runtime nvarchar(20)
declare @subject nvarchar(500)
declare @sSQL nvarchar(200)
declare @sParam nvarchar(100)
declare @spid nvarchar(4)
declare @body nvarchar(max)
declare @counter int

set @runtime=convert(nvarchar,GETDATE())
set @sSQL=N'select top 1 @bID=blockedBy from controlDB.dbo.blocking_transaction where 
nt_user_name=''your service account'' and blockedBy!=0 by tranceID desc'
set @sParam=N'@bID nvarchar(4) output'
exec sp_executesql @sSQL, @sParam, @bId=spid output
set @subject=N'Warning: ETL job being blocked 
on '+@runtime+N'by SPID'+@spid+N', check it out!'

if @case=2
set @body=N'
<p style="color:#000000;font-family: Georgia; font-size: 18px; line-height: 18px">
Hi Team,<br /><br /></p>
<p style="color:#000000;font-family: Georgia; font-size: 18px; line-height: 18px">
A job is now being blocked by a user process with <font color="#FF5733">
<b>SPIS='+@spid+'</b></font><br /><br />
Please check [controlDB].dbo.[blocking_transaction] table.<br /><br /></i>
<font size="4">More details information shown on below</font><br /><br />
Thanks and Best Regards,<br /><br />
</p>'+
N'
<style>
	th
	{
		background: #33FFBD
		height: 30px;
	}
	table,th,td
	{
		border: 2px solid green;
	}
	table
	{
		width: 80%;
		color: black;
		text-align: center;
	}
</style>'+
	N'<h2><center><font color="red">ETL Job Info for Blocking</font></center></h2>'+
	N'<table align="center">' +
	N'<tr>'+
	N'<th>SPID</th><th>BLkdBY</th>' +
	N'<th>User Name</th>' +
	N'<th>Elapse Time in MM</th><th>Process Run Time</th>' +
	N'<th>First Record Time</th><th>Last Record Time</th>' +
	N'<th>Counter</th><th>CMD</th>' +
	N'</tr>' +
	cast((
    	select top (@rownum)
        td=[spid],''
        ,td=[blockedBy],''
        ,td=[nt_user_name],''
        ,td=[TotalElapTime_in_MM],''
        ,td=[ps_time],''
        ,td=[fr_time],''
        ,td=[lr_time],''
        ,td=[counter],''
        ,td=[cmd],''
        from controlDB.dbo.blocking_transaction
        order by TranceID desc
        FOR XML PATH('tr'), TYPE
    ) as nvarchar(max)) +
N'</table>' +
N'<p><br /></p>' +
N'your team name'
;
else
if @case=3
set @body=N'
<p><br /></p>
<p style="color:#000000;font-family: Georgia; font-size: 18px; line-height: 18px">Hi Team,<br /><br /></p>
A ETL job is now still being blocked by a user process with 
spid='+@spdi+' for more than '+@times+' checks adn '+@times*30+' minutes
check previous email or controlDB.dbo.blocking_transaction table for more details!
<br /><br />
your team name
'
execute @result=msdb.dbo.sp_send_dbmail
@profile_name=N'your SMTP profile name',
@recipients = N'email1@yourcompany.com;email2@yourcompany.com',
@copy_recipients = N'email3@yourcompany.com',
@subject = @subject,
@body = @body,
@body_format='HTML'
;
return 0
end try
end
```



## Create a console script to overall control the workflow

at the last step, we need a control script put all above stored procedure together to be able to schedule `SQL Agent` job.

```sql
use controlDB;
go

declare @result_status smallint
declare @counter smallint
declare @rowcnt int

exec @result_status=usp_blocking_trans @counter=@counter output, @rowcnt=@rowcnt output
if @result_status=1
begin
print('no blocking process')
return
end
else
if @result_status=4
begin
print('no new blocking process')
return
end
else
begin
set @counter=@counter+1
exec usp_blocked_Email @case=@result_status, @times=@counter, @rownum=rowcnt
```









