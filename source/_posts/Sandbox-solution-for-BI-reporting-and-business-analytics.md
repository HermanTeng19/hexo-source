---
title: Sandbox solution for BI reporting and business analytics
date: 2020-12-02 10:41:48
categories: [Database, BI]
tags: [sandbox, SQL, SQL Server, database]
toc: true
cover: /img/sqlsandbox.jpg
thumbnail: /img/sqlserver.png
---

At beginning, I'd like to share a story from my client and business partner. One day, my team worked on a big marketing project, data from all kinds of source like spreadsheet, csv, mainframe and SQL Server, we had to do cross reference all those data to generate analysis reports but the headache thing was they were isolated and no way to put them into a single unique environment to run one time query then return the results so we could only open multiple windows to compare them by eyeballs. 

During the project, we often composed some complex queries then ran for a long time to return the result dataset, those datasets were quite important for future further analysis and share with the whole team, but the another panic thing was we could not save those dataset into database due to insufficient access so what we did was copy and paste everything in excel spreadsheet, after for a while, we found number of excel file explode and hard to find the report among those huge files, we feed up with the tedious work and decided created a bunch of views in database but that was also not controlled by us but infrastructure team, all we could do was submit the request then followed infrastructure team's schedule and waited for month end deployment, no matter how urgent those reports would be.

That is the story, I think if you had ever experienced that, that solution might be right for you. 

<!-- more -->

You might get the common points from above story, it's inefficient and even painful if you can only leverage data from data warehouse tables, that is reason why the sandbox database comes up which is a brand new data play zone on production with ability of pipeline to bring multiple sources of data based on major data warehouse you are using. In a brief mark, sandbox is aiming to build a ***homogeneous solution*** for ***heterogenous environment***.

How to build up and what is the foundation of sandbox database, now let's take a look into it.

![sandbox_structure.png](https://p.130014.xyz/2020/12/03/sandbox_structure.png)

The Major data source is so called `big db`, but it's data warehouse so that user was only assigned read access which means you can do nothing but only select and query data. Now we carved out two new databases - `sandbox` and `control db`.

## Sandbox database brief introduction

sandbox is the new data loading zone for slicing and dicing data in production, like you own backyard playground, you can do almost anything you want to do with it, so user will be granted both read and write permission. control_db that is control console station to provide access gateway for sandbox and also monitor and logging the users' behaviors, just like a guardian to promise the safety and healthy for sandbox.

Let's take a closer look at these two databases. Sandbox, in general, user can perform 3 kinds of actions, DDL, DML as well as batch job execution based on stored procedure. With DDL command which is database define language, user can create table/view/stored procedure, alter them, even drop them. With DML command which is database manipulate language, user can insert/update/delete data from existing table and also import data from other source. The batch operation is fit for user with programming background and write multiple queries with logical sequence into it like conditional and looping function by using T-SQL scripting language in terms of stored procedure.

## What things should be considered before 

A very critical point for sandbox database is **object ownership**, which means you can _only create and deal with your own database objects plus read others' object_. Image you data or analytical work were deleted by other accidently, how would you feel at the moment, so the restriction must be setup along with the sandbox database creation to promise the user data safety.

![noDMLTable.png](https://p.130014.xyz/2020/12/03/noDMLTable.png)

![noDropTable.png](https://p.130014.xyz/2020/12/03/noDropTable.png)

Another important thing need to be considered before is user data volume control. Unlike data warehouse, data volume increasing is followed trend and stable for each year, but sandbox database size is unpredictable and totally depends on user personal behavior, in order to prevent non-critical user data dominants your server hard disk, it's very necessary to set quota for each users. When reach to the limit then

+ User won't be allowed to create table
+ User also won't be allowed insert new data into existing table

until manually clean data and release space.

![quotaLimit.png](https://p.130014.xyz/2020/12/03/quotaLimit.png)

## Creating sandbox database walk through

### Define and create a new database 

```sql
use master;
go
if DB_ID('sandbox') is not null
drop database sandbox
go
```

in real production, you might need to create a new file group and bunch of file under it for better management.

```sql
create database sandbox
on primary
(name=N'sandbox', filename=N'G:\sqldata\sandbox.mdf', size=1048kB, maxsize=5124KB, filegrowth=512KB),
filegroup [yourTeam_sandbox]
(name=N'yourTeam_sandbox_FG1',filename=N'G:\sqldata\yourTeam_sandbox1.ndf',size=1048KB, maxsize=5124KB, filegrowth=512KB),
(name=N'yourTeam_sandbox_FG2',filename=N'G:\sqldata\yourTeam_sandbox2.ndf',size=1048KB, maxsize=5124KB, filegrowth=512KB),
(name=N'yourTeam_sandbox_FG3',filename=N'G:\sqldata\yourTeam_sandbox3.ndf',size=1048KB, maxsize=5124KB, filegrowth=512KB)
log on
(name=N'yourTeam_sandbox_log',filename=N'H:\sqllog\yourTeam_sandbox_log.ldf',size=1048KB, maxsize=5124KB, filegrowth=512KB)
go
```

make sure your sandbox and control console database have the same database ownership user

```sql
use master;
go
alter authorization on database::sandbox
to sa
go
alter authorization on database::control_db
to sa
go
```

make sure your new created file group to be the default file group so that coming data is going to be allocated there

```sql
use sandbox;
go
if not exists (select name from sys.filegroups where is_default=1 and name=N'sandbox')
alter database sandbox modify filegroup [yourTeam_sandbox] default
go
```

### Grant user permissions

First, you need to provide the basic read access to your business user

```sql
use sandbox;
go
if exists (select * from sys.database_principals where name=N'yourDomain\yourUserGroup')
drop user [yourDomain\yourUserGroup]
else
create user [yourDomain\yourUserGroup] for login [yourDomain\yourUserGroup] with default_schema=dbo
exec sp_addrolemember 'db_datareader','yourDomain\yourUserGroup';
```

then assign to them DDL and DML permissions

```sql
use sandbox;
go
grant alter any schema to [yourDomain\yourUserGroup];
go
grant create procedure to [yourDomain\yourUserGroup];
go
grant create table to [yourDomain\yourUserGroup];
go
grant create view to [yourDomain\yourUserGroup];
go
grant delete to [yourDomain\yourUserGroup];
go
grant insert to [yourDomain\yourUserGroup];
go
grant select to [yourDomain\yourUserGroup];
go
grant update to [yourDomain\yourUserGroup];
go
grant execute to [yourDomain\yourUserGroup];
go
```

### Create database level trigger on DDL query control for rules and restrictions

1. encourage give table name starting with id number which is unique identifier for employee in your company like nameInitial2_tableName
2. must be eligible user, identity check by control database
3. no special characters are allowed in table name like !@#$%^&-
4. no over quota allowed
5. not allowed user drop or delete others' object and data

let's connect above constraints with workflow chart to be able to see the big picture

![sandbox_trigger_workflow.png](https://p.130014.xyz/2020/12/05/sandbox_trigger_workflow.png)

First of all, we create the database trigger for enforcing object creation rules

```sql
create trigger ddltrg_create_table
on database
with execute as 'dbo'
for create_table
as
begin try
	set noncount on;
	declare @eventdata xml
	declare @orig_object_name varchar(100)
	declare @orig_user_name varchar(25)
	declare @alt_object_name varchar(100)
	declare @alt_user_name varchar(25)
	declare @valid_chars varchar(50)
	declare @create_status char(1)
	declare @create_err_msg varchar(250)
	declare @rename_required char(1)
	declare @trigger_sql varchar(max)
	declare @schema_name varchar(25)
	declare @trigger_insert varchar(max)
	declare @guid varchar(32)
	declare @overQuota_user table (userId varchar(25) null)
	
	set @guid='250'
	set @valid_chars = '%[^a-zA-Z0-9_]%'
	set @eventdata = EVENTDATA()
	set @orig_object_namne = @eventdata.value('(/EVENT_INSTANCE/objectName)[1]','nvarchar(100)')
	set @orig_user_name = upper(@eventdata.value('(/EVENT/INSTANCE/LoginName)[1]','nvarchar(25)'))
	set @create_status='Y'
	set @create_err_msg=''
	set @alt_user_name = right(@orig_user_name,len(@orig_user_name)-charindex('\\',@orig_user_name))
	select @schema_name = SCHEMA_NAME(SCHEMA_ID) from sys.tables where name = @orig_object_name
	
	if (@alt_user_name = 'yourServiceAccout')
	return
	if substring(@orig_object_name,1,len(@alt_user_name)) = @orig_object_name
	begin
		set @alt_object_name = @orig_object_name
		set @rename_required = 'N'
	end
	else
	begin
		set @alt_object_name = @orig_object_name
		set @rename_required = 'Y'
	end
	/*check the user eligibility*/
	if not exists (select * from control_db.dbo.Audit_Sandbox_User where User_ID = @alt_user_name)
	begin
		set @create_status = 'N'
		set @create_err_msg = 'User: ' + @alt_user_name + ' does not have privilege to use the sandbox database'
	end
	insert into @overQuota_user
	select user_id from control_db.dbo.Audit_Sandbox_User where sandbox_limit - current_usage < 0
	/*user control for over limit quota*/
	if @alt_user_name in (select userId from @overQuota_user)
	begin
		set @create_status = 'N'
		set @create_err_msg = 'User: ' + @alt_user_name + ' exceeded designed space quota, ' 
		+ @alt_user_name + ' is not able to  ' +
		N'create new table. Please delete data not needed' +
        N'to free up space so usage is less than quota 1024MB and rerun query again'
	end
	/*check object name eligibility*/
	if (patindex(@valid_chars,@orig_object_name) > 0)
	begin
		set @create_status = 'N'
		set @create_err_msg = 'Invalid characters in table name'
	end
	/*check object existency*/
	if exists(select * from sys.objects where object_id=OBJECT_ID(@alt_object_name) and type in (N'U'))
	begin
		if @rename_required = 'Y'
		begin
			set @create_status = 'N'
			set @create_err_msg = 'Table: ' + @alt_object_name + ' already exists'
		end
	end
	/*check schema name eligibility*/
	if @schema_name <> 'dbo'
	begin
		set @create_status = 'N'
		set @create_err_msg = 'Table: ' + @orig_object_name + 
		' not created. Schema is not specified: dbo.<table name>!'
	end
	if (@create_status = 'N')
	begin
		rollback
		print @create_err_msg
	end
	/*log sandbox event into tracking table in control_db*/
	insert control_db.dbo.Audit_Sandbox_Event_Tracking 
	(Event_Type, Event_Time, Event_User, Event_Database, Event_Object_Name
    ,Event_Object_Type, Event_SQK, Audit_Created_Status, Audit_Error_Message)
    	value (@eventdata.value('(/EVENT_INSTANCE/EventType)[1]','nvarchar(50)'),
               @eventdata.value('(/EVENT_INSTANCE/PostTime)[1]','datetime'),
               @Orig_user_name,
               @eventdata.value('(EVENT_INSTANCE/DatabaseName)[1]','nvarchar(25)'),
               @orig_object_name,
               @eventdata.value('(/EVENT_INSTANCE/ObjectType)[1]','nvarchar(25)'),
               @eventdata.value('(/EVENT_INSTANCE/TSQLCommand/CommandText)[1]','nvarchar(max)'),
               case when @create_status = 'Y' then 'success' else 'failed' end,
               @create_err_msg
              )                                                   
     if (@create_status = 'N') 
     return
     /*rename object to add user id prefix*/
     begin
     	exec sp_rename @orig_object_name, @alt_object_name
     	print 'Table name is renamed with prefix of your user id'
     end
     /*creaet DML trigger on table level*/
     set @trigger_sql = 'create trigger DML trig_'+@alt_object_name+ 'ON ' + @alt_object_name + ''
     set @trigger_sql = @trigger_sql + 'for delete, update, insert'
     set @trigger_sql = @trigger_sql + 'as'
     set @trigger_sql = @trigger_sql + 'begin '
     set @trigger_sql = @trigger_sql + '	set nocount on;'
     set @trigger_sql = @trigger_sql + '	declare @user varchar(25)'
     set @trigger_sql = @trigger_sql + '	select @user = right(SUSER_NAME(),LEN(SUSER_NAME())'
     set @trigger_sql = @trigger_sql + '	- CHARINDEX(''\\'',SUSER_NAME()))'
     set @trigger_sql = @trigger_sql + '	IF @user <> '''+@alt_user_name+''' '
     set @trigger_sql = @trigger_sql + '	begin'
     set @trigger_sql = @trigger_sql + '		rollback'
     set @trigger_sql = @trigger_sql + '		print ''User: ''+@user+'' does not have the privilege to perform a DML operation on table' +@alt_object_name + ''' '
     set @trigger_sql = @trigger_sql + '	end'
     set @trigger_sql = @trigger_sql + ' end'
     exec (@trigger_sql)
     ;
     /*create over quota DML trigger on table level*/
     create table #overQuota (userId varchar(25) Null)
     ;
 	set @trigger_sql = 'create trigger DML trig_'+@alt_object_name+ '_insert ON ' + @alt_object_name + ''
     set @trigger_sql = @trigger_sql + 'after insert'
     set @trigger_sql = @trigger_sql + 'as'
     set @trigger_sql = @trigger_sql + 'begin '
     set @trigger_sql = @trigger_sql + '	set nocount on;'
     set @trigger_sql = @trigger_sql + '	declare @user varchar(25)'
     set @trigger_sql = @trigger_sql + '	select @user = right(SUSER_NAME(),LEN(SUSER_NAME()) - CHARINDEX(''\\'',SUSER_NAME()))'
     set @trigger_sql = @trigger_sql + '	insert into #overQuota'
     set @trigger_sql = @trigger_sql + '	select user_id from control_db.dbo.Audit_Sandbox_User where sandbox_limit-current_usage<0;'
     set @trigger_sql = @trigger_sql + '	IF @user in (select userId from #overQuota)'
     set @trigger_sql = @trigger_sql + '	begin'
     set @trigger_sql = @trigger_sql + '		rollback'
     set @trigger_sql = @trigger_sql + '		print ''User: ''+@user+'' is over quota on usage so it can not perform a INSERT operation on table' +@alt_object_name + ', please delete data that is not needed to free up space then rerun the query'' '
     set @trigger_sql = @trigger_sql + '	end'
     set @trigger_sql = @trigger_sql + ' end'
     exec (@trigger_sql)
     ;
     drop table #overQuota
     ;
end try

begin catch
	declare @err_msg varchar(900),
		   @err_num int,
		   @err_line int,
		   @syserr varchar(900)
     select @err_msg = ERROR_MESSAGE(), @err_num = ERROR_NUMBER(), @err_line = ERROR_LINE()
     set @syserr = 'Ended in DDLTRIG_CREATE_TABLE with errors: Line= ' + convert(varchar(10), @err_line) + ', Error Num = ' + convert(varchar(10), @err_num) + ', Error Msg= ' + @err_msg
     /*save to log file or control_db table*/
end catch
go

enable trigger [ddltrg_create_table] on database
go
```

One more run need to be applied is prevent user drop object which is not belong to that user.

```sql
create trigger ddltrig_drop_table
on database
with execute as 'dbo'
	for drop_table
as
	set nocount on;
	declare @eventdata xml
	declare @orig_object_name varchar(100)
	declare @orig_user_name varchar(25)
	declare @alt_user_name varchar(25)
	declare @create_status char(1)
	declare @create_err_msg varchar(250)
	
	set @eventdata = EVENTDATA()
	set @orig_object_name = @eventdata.value('(/EVENT_INSTANCE/ObjectName)[1]','nvarchar(100)')
	set @orig_user_name = upper(@eventdata.value('(/EVENT_INSTANCE/LoginName)[1]','nvarchar(25)'))
	set @creaet_status = 'Y'
	set @create_err_msg = ''
	set @alt_user_name = right(@orig_user_name, len(@orig_user_name) - charindex('\\', @orig_user_name))
	
	if @alt_user_name <> 'domainName/yourServiceAccount'
	begin
		if not exists (select * from control_db.dbo.Audit_Sandbox_User where User_ID = @alt_user_name)
	begin
		set @create_status = 'N'
		set @create_err_msg = 'User: ' + @alt_user_name + ' does not have privilege to use the sandbox database'
	end
	if substring(@orig_object_name, 1, len(@alt_user_name)) <> @alt_user_name
	begin
		set @create_status = 'N'
		set @create_err_msg = 'user: ' + @alt_user_name + ' does not have privilege to drop table: ' + @orig_object_name
	end
end
if (@create_Status = 'N')
begin
	rollback
	print @create_err_msg
end
insert control_db.dbo.Audit_Sandbox_Event_Tracking (Event_Type, Event_Time, Event_User, Event_Database, Event_Object_Name
                                                       ,Event_Object_Type, Event_SQK, Audit_Created_Status, Audit_Error_Message)
    	value (@eventdata.value('(/EVENT_INSTANCE/EventType)[1]','nvarchar(50)'),
               @eventdata.value('(/EVENT_INSTANCE/PostTime)[1]','datetime'),
               @Orig_user_name,
               @eventdata.value('(EVENT_INSTANCE/DatabaseName)[1]','nvarchar(25)'),
               @orig_object_name,
               @eventdata.value('(/EVENT_INSTANCE/ObjectType)[1]','nvarchar(25)'),
               @eventdata.value('(/EVENT_INSTANCE/TSQLCommand/CommandText)[1]','nvarchar(max)'),
               case when @create_status = 'Y' then 'success' else 'failed' end,
               @create_err_msg
              )
go
```

At last, one very important setting need to be in placed in case cause issue when  [sa] user cross database reference data

```sql
alter database sandbox set TRUSTWORTHY ON;
```

about trustworthy for detail, you can see Microsoft official document on below link

[Trustworthy database property](https://docs.microsoft.com/en-us/sql/relational-databases/security/trustworthy-database-property?view=sql-server-ver15)























