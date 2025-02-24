---
title: Python Environment Setup for Implementation
date: 2020-12-08 14:21:38
categories: [IT, BI]
tags: [python, sql, db2, CLI, SSIS, config]
toc: true
cover: /img/pysql.jpeg
thumbnail: /img/pytool.jpg
---

Python is a good scripting language to boost your productivity on data analysis and BI reporting. As open source language, you can easily get the binary installation file from [python official site](https://www.python.org) for windows and source code on vary versions for Linux, in production, it's better choose installation approach by source code.

We also need to setup python environment after installation so that we can not only use python interpreter to develop but also make it executable by CLI and even some ETL tool such as Microsoft SSIS.

## Python environment variable configuration and local folder set up for your file, package and library

If python is installed in system-wise, then you need to create some new folders to store you python file, package and library, e.g. python install path is "D:\Python36\", then you need to add python executable interpreter to be a part of the PATH variable. Next create python environment variable PYTHONPATH with the following paths and create empty file \_\_init\_\_.py file in each of these folders:

- create a new folder under D drive "D:\pyLib" and set that directory as value of PYTHONPATH and create \_\_init\_\_.py file in "D:\pyLib"
- you can also create subfolder to assign different permissions for different user group
  - create a subfolder "D:\pyLib\AD-group1" and create the \_\_init\_\_.py file in it.
  - create a subfolder "D:\pyLib\AD-group2" and create the \_\_init\_\_.py file in it.

For Linux, if you install python3 by source code and directory is `/usr/local/python3`, then edit `~/.bash_profile` file, append the python directory into PATH

```bash
# Python3
export PATH=/usr/local/python3/bin/:$PATH
```

then run `source ~/.bash_profile` let setting take effect

if your system pre-installed python2 then it's necessary to make a soft link

```bash
ln -s /user/local/python3/bin/python3 /user/bin/python3
ln -s /user/local/python3/bin/pip3 /user/bim/pip3
```

<!-- more -->

setup name space and package python scripts for development project to be able to importable

* create or edit environment variable and add your python files folder into your system directory
* enter your python file folder to create an empty file \_\_init\_\_.py file
* open terminal prompt type python to active python interactive console
* import sys
* execute sys.path to make sure your python file folder is recognizable by python

![edit_env_var1.png](/img/screenshots/edit_env_var1.png)

![edit_env_var2.png](/img/screenshots/edit_env_var2.png)

## Python readiness test in localhost for SQL database connection (Anaconda virtual environment)

First check python and Ipython version by issue command `python --version` and `ipython --version`. Anaconda almost pre-installs all python prevailing and popular libraries in its virtual environment, to check library list by using command `pip list`

### Python and SQL database connection facility with supported drivers

Depends on what python library do you install for database connectivity, it usually comes with function to show you available drivers to connect python to your database, e.g: pyodbc, use `drivers()` function to list the odbc drivers

![odbc_drivers.png](/img/screenshots/odbc_drivers.png)

### Python SQL Server database connection and data task

SQL Server database can be connected both by DB API (pyodbc) and ORM (sqlalchemy), create a py script and run sql query from user input

```python
import pyodbc
import sqlalchemy
from sqlalchemy import create_engine
import pandas as pd

def pyquery(conn):
    cnxn = pyodbc.connect(conn)
    cur = cnxn.cursor()
    sql_cmd = input("Input your sql command with database name: ")
    result = cur.execute(sql_cmd)
    for row in result:
        print(row)
    cur.close()
    cnxn.close()
    
def py2mssql():
    way2conn = input("Do you want to connect to db by [ORM] or [DBAPI]: ")
    if way2conn.upper()=="DPAPI":
        dsn_str = input("Do you want to connect to db by [connection string] or [DSN]: ")
        if dsn_str.upper()=="DSN":
            dsn_name=input("Please enter your DSN name: ")
            conn_dsn='DSN{};Trusted_connection=yes'.format(dsn_name)
            conn=conn_dsn
            pyquery(conn)
        else:
            conn_str='DRIVER={SQL Server Native Client 11.0}; SERVER=YOURSERVERNAME;DATABASE=YOURDB;Trusted_connection=yes'
            conn=conn_str
            pyquery(conn)
    else:
        ser_name=input('Please enter your server name: ')
        db_name=input('Please enter your database name: ')
        tbl_name=input('Enter the table name: ')
        orm_dict={'servername':'{}'.format(ser_name), 'database':'{}'.format(db_name), 'driver':'driver=SQL Server Native Client 11.0'}
        engine=create_engine('mssql+pyodbc://'+orm_dict['servername']+'/'+orm_dict['database']+'?'+orm_dict['driver'])
        df=pd.read_sql_table(tbl_name, engine, index_col=0, schema='dbo')
        print(df.head(10))
        
if __name__=="__main__":
    py2mssql()
```

script both can be ran directly or imported (recommend) after setup PYTHONPATH variable in your account and copy that script over to the path of environment variable.

![py2mssql.png](/img/screenshots/py2mssql.png)

### Python DB2 database connection and data task

We can only connect to DB2 by DBAPI (pyodbc), connection string doesn't work but only DSN (PRD1 was setup as system DSN in local and server) plus user id and password. Use below script to try to connect

```python
import pyodbc
import getpass
import pandas as pd

def py2edw():
    uid=getpass.getuser()
    print("Your user id is '{}'".format(uid))
    pwd=getpass.getpass("Please enter your db2 password: ")
    dsn=input("Please enter your db2 dsn name: ")
    conn_dsn='DSN={0}; UID={1}; PWD={2}'.format(dsn, uid, pwd)
    conn=pyodbc.connect(conn_dsn)
    cur=conn.cursor()
    sql_cmd= '''
    		select * from table
    		'''
    result=cur.execute(sql_cmd)
    for row in result:
        print(row)
    cur.close()
    
    df=pd.read_sql_query(sql_cmd, conn)
    print(df.info())
    print(df)
    conn.close()
    
if __name__=="__main__":
    py2edw()
```

one thing need to be aware is the script better runs on the shell than python interactive console because pyQT doesn't support password masking

![py2db2_m.png](/img/screenshots/py2db2_m.png)

## Productionize Python script by passing in parameters

In this section we will demonstrate how you can parameterize your code in python or pyspark so that you can use these techniques before deployed your script into production for automation. It's best practice to parameterize database names, tables names, and dates so that you can pass these values as inputs to your script. This is beneficial when writing code for values that are dynamic in nature, which can change depending on the environment and/or use case.

The key module is from python standard library: `sys`. Assign variables through `sys.argv[...]`

```python
import pandas as pd
from sqlalchemy import create_engine
import sys

def py2mssql():
    ser_name=sys.argv[1] # sys.argv[0] is assigned to python script itself, all other parameters start from 1
    db_name=sys.argv[2]
    tbl_name=sys.argv[3]
    orm_dict={'servername':'{}'.format(ser_name),'database':'{}'.format(db_name),
             'driver':'driver=SQL Server Native Client 11.0'}
    engine=create_engine('mssql+pyodbc://'+orm_dict['servername']+'/'+
                        orm_dict['database']+'?'+orm_dict['driver'])
    df=pd.read_sql_table(tbl_name, engine, index_col=0, schema='dbo')
    df.to_sql('pandas_sql_test', engine, schema='dbo', if_exists='replace', index=False)
    
if __name__=="__main__":
    py2mssql()
```

run python script in CLI (command line interface) by following parameter values

```bash
ipython py2mssql_argvdf.py yourSvrNm yourDBNm yourTblNm
```

![py2sql_argvdf.png](/img/screenshots/py2sql_argvdf.png)

## Encapsulate into SSIS to minimize change in production deployment -- python interpreter

An alternative way to apply python in production is leverage current SSIS package and embed python script in process task

![pyssis1.png](/img/screenshots/pyssis1.png)

you can hard code the configuration or use expression (VB) through variables in package

![pyssis2.png](/img/screenshots/pyssis2.png)



![pyssis3.png](/img/screenshots/pyssis3.png)

## Encapsulate into SSIS to minimize change in production deployment -- batch process

use batch process by .bat file also can achieve that task

![pyssis4.png](/img/screenshots/pyssis4.png)

## Call user defined module or function in python script

It's very efficient to create bunch of generic module packages to contain functions to be used widely by other python scripts for specific tasks.

| 1.   | Firstly, setup python environment variable to include directories which are recognized by python |
| ---- | :----------------------------------------------------------- |
| 2.   | Create \_\_init\_\_.py file (can be empty) in these folders  |
| 3.   | Create python programs and save script should end up with if \_\_name\_\_=="\_\_main\_\_": main() |
| 4.   | Ready to import user modules and functions                   |

```python
import pandas as pd
import py2mssql_module as db
import sys

tbl_name=sys.argv[1]
df=pd.read_sql_table(tbl_name, db.py2mssql('serverName','databaseName'), index_col=0, schema='dbo')
print(df.head(10))
```















