---
title: Data Manipulation and ETL with Pandas
date: 2020-12-20 21:18:08
categories: [data, BI]
tags: [Python, Pandas, SQL, ETL]
toc: true
cover: /img/pandassqlribbon.jpg
thumbnail: /img/pandasicon.jpg
---

Pandas is the most used library in Python to manipulate data and deal with data transformation, by leveraging `Pandas` in memory data frame and abundant build-in functions in terms of data frame, it almost can handle all kinds of ETL task. We are going to talk about a data process to read input data from `Excel Spreadsheet`, make some data transformations by business requirement then load reporting data into `SQL Server` database.

<!-- more -->

## Pandas functions brief introduction

First, we use `read_excel()` function to read in input Excel file, then use slicing function `iloc()` or `loc()` to get the data which need to be proceeded. The difference between `iloc()` and `loc()` is `iloc()` use row and column index number to slice data but `loc()` is use column name instead of index number.

If we need to rename the column, we use `rename(column={},inplace=True/False)` function to do that. We also can use `drop()` function to drop columns. If we want to re-order the sequence of columns, the `reindex()` will help us to do that. For the purpose of merging or joining multiple tables, we can use `merge()` function. we can use `to_sql()` function to write data into database table after transformations. If we want to create UDF (user defined function) to apply to `Pandas` data frame then we can use `apply(UDF name)` function.

For SQL windows function, `Pandas` also has equivalent way on those kind of function such as Top N rows per group which is equivalent to row_number()over() sql window function. Use `assign(().groupby().cumcount()).query().sort_values()` function 

## Pandas use cases to manipulate Excel data to load into SQL Server database

```python
import numpy as np
import pandas as pd
import pyodbc
from datetime import datetime
from sqlalchemy import create_engine

def py2mssql():
    engine = create_engine("mssql+pyodbc://yourServerName/database?driver=SQL Server Native Client 11.0")
    con_str = pyodbc.connect("DRIVER={SQL Server Native Client 11.0}; SERVER=yourServerName; DATABASE=yourDBName; Trusted_connection=yes")
    return engine, con_str

def dfstaging(df, sql_cmd):
    df.to_sql('your target table',py2mssql()[0],if_exists='replace',schema='dbo',index=False)
    conn = py2mssql()[1]
    cur = conn.cursor()
    conn.commit()
    cur.close()
    conn.close()
def rpt2mssql():
    df_pdr = pd.read_excel(r'C:\your input file path\fileName.xlsx',sheet_name='Sheet1')
    df_pd1 = df_pdr.iloc[:,[0,1,2,3,4,5,6,8,9,10]] ## iloc[row,col in num]
    df_temp = df_pdr.loc[:,['col1','col2','col3']] ## loc[row,col in name]
    up_col = """
    	sql statement
    """
    dfstaging(df_temp, up_col)
    df1=pd.read_sql_table('your target table', py2mssql()[0], schema='dbo', index_col=None)
    ## rename columns in data frame to match with target table
    df1.rename(column={"user ID":"User_ID",
                      "item Key":"Item_Key",
                      "connect ID":"Connect_ID"},inplace=True)
    df1["Cust_Id"] = np.nan
    up_custid = """
    	update sql statement
    """
    dfstaging(df1, up_custid)
    df2 = pd.read_sql_table('your target table', py2mssql()[0], schema='dbo', index_col=None)
    ## drop data frame column
    df3 = df2.drop(columns=["Connect_ID","Item_Key"])
    ## join two data frames
    df_join = pd.merge(df_pd1, df3, left_index=True, right_index=True)
    df_join["Process_dt"] = datetime.today().strftime("%Y-%m-%d %H:%M:%S")
    df_final = df_join.drop(columns=["Connect_Id","Item_Key"])
    ## loop through all columns need to be converted to datetime datatype
    cvt_lt = ["Work_Queue_Datetime","Complete_Datetime","Current_StatementDate"]
    for col in cvt_lt:
        df_final[col] = pd.to_datetime(df_final[col])
	## get duplicate record from column value
	df_final["Dup_Ind"] = [1 if x == "Dup Submission" else 0 for x in df_final["Reason_Comment"]]
    ## re-order the columns
    new_index = ["col5","col3","col1","col2","col4"]
    ## apply the new order
    df_final = df_final.reindex(columns=new_index)
    df_final.to_sql('your target table',py2mssql()[0], if_exists='append', schema='dbo', index=None)
    up_dup = """
    	sql statement
    """
    dfstaging(df_final,up_dup)

    
if __name__=="__main__":
    rpt2mssql()
```

```python
import numpy as np
import pandas as pd
import pyodbc
from sqlalchemy import create_engine

def split_dt(ef_dt):
    ef_day=(ef_dt//10000)%100 ## extract day info
    ef_mo=ef_dt//1000000 ## extract month info
    ef_yr=ef_dt%10000
    return pd.Series({'dd': ef_day, 'mm': ef_mo, 'yy': ef_yr})
def py2mssql():
    engine = create_engine("mssql+pyodbc://yourServerName/database?driver=SQL Server Native Client 11.0")
    con_str = pyodbc.connect("DRIVER={SQL Server Native Client 11.0}; SERVER=yourServerName; DATABASE=yourDBName;
    Trusted_connection=yes")
    return engine, con_str
def df2staging(df, sql_cmd):
    df.to_sql('your target table',py2mssql()[0],if_exists='replace',schema='dbo',index=False)
    conn = py2mssql()[1]
    cur = conn.cursor()
    conn.commit()
    cur.close()
    conn.close()
def df2db(df_type1,df_type2):
    engine = py2mssql()[0]
    df_type1.to_sql('your target table', engine, schema='dbo', if_exists='append', index=False)
    df_type2.to_sql('your target table', engine, schema='dbo', if_exists='append', index=False)
def buz2df():
    df = pd.read_excel(r'\\your input file path\filename.xlsx', sheet_name='Sheet1', index_col=None)
    df1 = df.iloc[:,[0,2,3,4,5,8]]
    ## Pandas equivalents for some SQL analytic and aggregate function
    ## Top N rows per group which is equivalent to row_number()over() sql window function
    df2 = df1.assign(rn=df1.sort_values(['your order by column name'],ascending=True)
    .groupby(['your group column']).cumcount()+1).query('rn == 1').sort_values(['your group column'])
    df3 = df2['the date column you want to split'].apply(split_dt)
    df3 = df3.astype(str) ## convert data frame elements as string
    df3['the data column you want to split'] = df3['mm'] + '/' + df3['dd'] + '/' + df3['yy']
    df3['the data column you want to split'] = pd.to_datetime(df3['the data column you want to split'])
    df2 = pd.concat([df2,df3['the data column you want to split']], axis=1)
    df_staging = df2.iloc[:,[0,1,5,7,8]]
    df_type1 = df_staging.loc[:['col1','col2','col3','col4']]
    df_type2 = df_staging.iloc[:,[2,3,4,5]]
    df2staging(df_staging)
    df2db(df_type1,df_type2)

if __name__=="__main__":
    buz2df()
    
```





