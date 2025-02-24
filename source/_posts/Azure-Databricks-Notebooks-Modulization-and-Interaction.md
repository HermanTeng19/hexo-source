---
title: Azure Databricks Notebooks Modulization and Interaction
date: 2024-03-28 10:29:30
Categories: [Cloud, IT, Data]
tags: [Azure, Databricks, Notebook, Azure Data Factory]
toc: true
cover: https://i.ibb.co/0q74Tzp/databrick-notebooks.png
thumbnail: https://i.ibb.co/tL2zTJv/databrick-workspace.png
---

Azure Databricks is the most common and popular platform and engine for data engineering and machine learning in Azure cloud. Notebook is the most used tool and application to do the data processing and data analysis, it not only inherits `Jupyter Notebook` powerful functionalities in `Python` but integrates `Scala`, `R`, `Java` even `markdown` to be able to create storyboard for data process. One common use case for a large data process development is notebooks need to call each. Main notebook calls sub-notebooks to retrieve classes, functions or properties, sub-notebooks call to parameter notebook to retrieve values for parameters. Notebook can be modularized and imported by other notebooks, this post is about the methods on notebook modulization and reference.

<!-- more -->

In one recent project, we collaborate with our partner engineering team to build a data solution based on their product in Azure through the APIs they provide, we use Databricks to call their API to fetch data from their application, then apply our business logics to performance data transformation and report generating. We create environmental configuration and utility notebooks for reuse and reference and we summarize different ways on notebook modulization by leveraging Databricks PySpark command and functionality

## Situation and Background

we have three module notebooks to support the main data process notebook.

1. util: generic and common functions used by main notebooks
2. env_config: environment information and parameters' value and common libraries for the main process
3. access_token_output: access token generator for obtaining time-bind API access

we need use all variables, parameters and functions from `util` and `env_config` notebook, but token output from `access_token_output` notebook so that we are going to use 2 different methods to solve this problem.

* **%run** command
* **dbutils.notebook.run()** function

## Solution

### Method 1: %run command

%run is spark magic command and it's usage syntax is very simple

> %run <notebook path>  $parameter_1= "value" $parameter_2="value" ... $parameter_n="value"

we can call notebook from another notebook by using it, all variables defined in reference notebook become available in main notebook. One thing we need to pay attention is %run must be in a cell by itself because it runs the entire notebook inline  [Databricks document](https://learn.microsoft.com/en-us/azure/databricks/notebooks/notebooks-code#--run-a-notebook-from-another-notebook).

Additionally, ***space*** is the separator among parameters, path of notebook should use ***relative directory*** with current main notebook.

```python
%run ../config/env_config $configuration_file=$configuration_file $environment_params=$environment_params
```

```python
%run ../config/util
```

If we want to comment out the command, just put cursor in the cell then stroke `ctrl + ?` key. Now let's show the way to receive parameters' value for reference notebook from main notebook. 

```python
executionStatus = []
processTimeStamp = str(datetime.now().strftime('%Y-%m-%d %H:%M:%S'))

try:
    configFile = dbutils.widgets.get("configuration_file")
    envParams = dbutils.widget.get("environment_params")
    process_config = spark.read.option("multiline", True).option("mode", "PERMISSIVE").json(configFile)
except Exception as e:
    logging.error(processTimeStamp + ": Failed to configure file in provided path")
    executionStatus.append({"status": "Failure", "error_code": "801", "message": "Failed to configuration file in provided path. " + str(e)[1000]});
    dbutils.notebook.exit(json.dumps(executionStatus))
   
```

As above shown, reference notebook needs values for variables `configFile` and `envParams` from main process notebook to parse below properties

```python
# Read Environment Configurations
sqlServerName = process_config.select(col(envParams).getField("sql.server.name")).first()[0]
sqlDatabaseName = process_config.select(col(envParams).getField("sql.database.name")).first()[0]
```

In main process notebook, we retrieve parameters' value from ADF (Azure Data Factory) then run %run command to bring all properties from reference notebook

```python
# Reading the parameters passed from the ADF
configuration_file = dbutils.widgets.get("configuration_file")
environment_params = dbutils.widgets.get("environment_params")
```

```python
%run ../config/env_config $configuration_file=$configuration_file $environment_params=$environment_params
```

If we want to use variables defined in reference notebook, we can directly call it from main process notebook

```python
sqlJdbcUrl = jdbcUrlPrefix + sqlServerName + ";" + "databaseName=" + sqlDatabaseName
```

If we want to use functions defined in utility notebook, we also can call it directly from main process notebook. 

```python
def writeSqlTable (jdbcUrl, write_df, sqlTable, writeMode, connectionProperties, execStatus):
    try:
        write_df.write.jdbc(url=jdbcUrl, table=sqlTable, mode=writeMode, properties=connectionProperties)
        return True
    except Exception as e:
        errorDesc = str(traceback.format_exc())
        exeStatus.append({"executionStatus": "Warning", "error_code": "702", "errorMessage": "Dataframe SQL table write failed. " + str(e)[:100]})
        return execStatus
```

### Method 2: dbutils.notebook.run() function

The syntax is `run(path: string, timeout_seconds: int, arguments: map): string`

map can be Python dictionary, for example:

> dbutils.notebook.run("sample_notebook", 60, {"param1": "value", "param2": "value"})

In this way, we can run a notebook and return its exit/return value. The method starts ephemeral job that runs immediately, which means a new instance of the executed notebook is created and the computations are done within it, in its own scope, and completely aside from the main notebook. This means that no functions and variables you define in the executed notebook can be reached from the main notebook. Now let's see how do we use that method in our main process notebook. 

**Use Case**

As we showed before, %run magic command must run in separated cell, it can't be used with other statement, but sometime if we want to run the notebook in a separate environment without affecting current main notebook, that method would be a better choice. It won't overwrite the variables and functions in our current main notebook, but only return executed notebook exit value so that it can be used with other code, e.g. when we use OAuth2 method to authenticate our SP (Service Principal) API access, we need to get the access token which is time-bind string, if our process needs to call million times API, we have to retrieve the new access token string before old one is expired, in that case, we have a notebook specifically is created for retrieving a new access token, the main process notebook has to integrate it into the logic with other codes in one cell. 

```python
if datetime.now() > tokenExpTime:
    apiCallTime = datetime.now()
    tokenExpTime = apiCallTime + timedelta(minutes=60)
    header_str = dbutils.notebook.run("../config/access_token_output", 60, {"configuration_file": f"{configuration_file}", "environment_params": f"{environment_params}"})
    header_valid = header_str.replace("\'", "\"")
    headers = json.loads(header_valid)
else:
    headers = headers
try:
    res = requests.post(url=endpoint_uri, headers=headers, params=q_params, json=payload)
    returnCode = res.status_code
    if returnCode // 100 != 2:
        logging.error()
        raise Exception("API call Failed")
except Exception as e:
    logging.error()
    dbutils.notebook.exit(json.dumps(executionStatus))
```

### The differences between method 1 and method 2

In method 1, the main process notebook includes the entire reference notebook `env_config`. The value of variables and functions are available for main process notebook, but they are also overwrite variables and functions in main process notebook.

In method 2, reference notebook `access_token_output` runs main process notebook in a separated job. The value of variables and functions are not overwritten by reference notebook and keep as they were.

### Summary

Here is the summary of the differences between them.

|                           | %run                                                         | deutils.notebook.run()                                   |
| ------------------------- | ------------------------------------------------------------ | -------------------------------------------------------- |
| Running Environment       | In the same instance of the executed notebook                | In a new instance of the executed notebook               |
| Running Method            | In a separated cell                                          | In one line of PySpark code                              |
| Supported Parameter Types | String (only)                                                | Map                                                      |
| Execution Output          | Doesn't return exit status unless adding "dbutils.notebook.exit()" in executed notebook explicitly | Offers details of notebook execution and command outputs |













