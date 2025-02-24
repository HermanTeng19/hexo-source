---
title: Oracle client side configuration for 12C
date: 2020-12-11 13:40:31
categories: [Database, BI]
tags: [oracle, SSIS, SAS, config]
toc: true
cover: /img/oracleconfig.jpg
thumbnail: /img/oracleicon.png
---

The client side need to do some configurations after Oracle 11g upgrade to 12C on Server in order to make database server is connectable. Before starting to configurate your clients, you have to get the below new server information from DBA

1. Host name
2. Port number
3. Service name
4. Your user name(usually it won't be changed and replicated from old version)
5. Password(initial password for test connection then you need to update it)

and then you need to make a new connection strings to add it into ORA file(*.ora) 

![oracle12cinfo.png](/img/screenshots/oracle12cinfo.png)

<!-- more -->

## Oracle SQL Developer configuration

The simplest way to connect to oracle 12C is by using Oracle sql client tool `SQL Developer`, it uses build-in JDBC driver to make connections and GUI connection wizard guide you fill in the server information without updating ora file. It's better use version 19.1 and above.

![oraclesqldeveloper19.png](/img/screenshots/oraclesqldeveloper19.png)

fill in all info in connection window, then click `Test` button to test connection

![oracle12cconn2.png](/img/screenshots/oracle12cconn2.png)

## Other client tool connection by ODBC driver such as SSIS and WinSql

The first step, you need to install the ODBC driver, there is another client tool comes with ODBC driver called `Oracle Database Client12cSQLDev 12.1.0.2 R02`, install that in your local machine, then update your environment variable to make sure Oracle home directory being added into it.

### Update environment variable

If you installed both `Oracle SQL Developer` and `Client12cSQLDev 12.1.0.2 R02`, there is going to be 2 Oracle folders, one is client_32 which is 32 bit, the other one is client_1 which is 64 bit application, you need to add both of them into you environment variable. The path is usually `C:\app\product\12.1.0\client_*`

![oracleenvironmentvar.png](/img/screenshots/oracleenvironmentvar.png)

### Configurate ora file after setup oracle environment variable

The tricky thing is the ODBC driver only works well for 32 bit version not 64 bit so we can only edit the ora file for ***client_32***. You can find out the ora file on below directory

`C:\app\product\12.1.0\client_32\network\admin\tnsnames.ora`

append following string to the ora file

![orafile.png](/img/screenshots/orafile.png)

save andclose the ora file, then go to windows ODBC data source center to finalize ODBC driver configuration.

### Configurate ODBC driver and DSN to test connection

![odbccenter.png](/img/screenshots/odbccenter.png)

`Data Source Name` is customizable; `TNS Service Name` is the connection string name in ora file **CAP12CPRD_32bit** in this case; fill in user ID then click test connection button, a prompt window will pop out to let you input password. 

![oracleconnect.png](/img/screenshots/oracleconnect.png)

### SSIS package connection manager configuration

Open SSIS client SSDT to configure connection manager for Oracle data source. SSDT -> open a project -> right click data source on right side panel -> follow the wizard

![ssiswizard.png](/img/screenshots/ssiswizard.png)

both preinstalled .net provider and native ole db provider for Oracle are working well

![oledboracle.png](/img/screenshots/oledboracle.png)

server name is as the same as connection string name in ora file in this case is **CAP12CPRD_32bit**. Give a name for oracle new data source then connection manager can be created to Oracle database.

![oraclessis.png](/img/screenshots/oraclessis.png)

![connmgr.png](/img/screenshots/connmgr.png)

## SAS connection to Oracle 12C

* prerequisite:

  * SAS grid server local DSN need to be created before testing;
  * PC SAS login server is necessary

* SAS grid local DSN for Oracle 12C (ask for system admin create local DSN and return the name to you and it is going to be the value for `path` when you create new library for Oracle 12C)

* SAS EG connection to 12C:

  * open SAS EG 7.1 and make sure it connects to grid server

    ![sasgrid.png](/img/screenshots/sasgrid.png)

  * open a new code window and create a new library by running below statement

    ```SAS
    libname ora12C oracle user="yourUserName" password="xxxx" path=sas_grid_local_DSN schema=oracle_schema
    ```

    ![sasnewlib.png](/img/screenshots/sasnewlib.png)

  * make sure the program run against SASApp rather than localhost

  * new Oracle library would be created under the SASApp dataset list

    ![sasliblist.png](/img/screenshots/sasliblist.png)

    

  







