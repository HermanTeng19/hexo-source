---
title: >-
  An Application of Microsoft Purview Authentication Workflow for API and Data
  Sources
date: 2024-04-09 10:37:38
Categories: [IT, Data Catalog, Cloud]
tags: [Microsoft, Azure, Purview, ADLS, SQL Database, API, Key vault, Azure Databricks]
toc: true
cover: https://i.ibb.co/0yxH4FG/Sans-titre.jpg
thumbnail: https://i.ibb.co/FVLhBG4/Microsoft-Purview-data-center-700x467.jpg

---

`Microsoft Purview` is a powerful data catalog and governance solution that helps organization discover metadata and manage their data assets. In front end, it provides web portal for business user to browse metadata and manage data mapping; in the back end, it provides API to access data source for scanning and Purview instance for business metadata updating through data process pipeline in batch job so that one critical aspect of using Purview is API authentication, especially when integrating with other services and applications. 

In this article, we'll take a deep dive into how Purview uses OAuth2.0 authentication for service principals and the secure way to access and manage data source for scanning, providing a comprehensive understanding of the process.

<!-- more -->

## Microsoft Purview OAuth2.0 API Authentication Process for Service Principals

### Understanding OAuth2.0 Authentication

OAuth2.0 is an authorization framework that allows a third-party application to obtain limited access to an HTTP service on behalf of  a resource owner. In the context of `Microsoft Purview`, OAuth2.0 is used to authenticate service principals, which are identities used by applications and services to access resources.

### Create a Service Principal (application)

The first step to access API services of Purview Data catalog and Scanning is creating a client side identity which Purview recognizes and is configured to trust. When we make API calls, that service principal's identity will be used for authorization. We can create a service principal through `Azure Entra ID` portal then we get a bunch of properties and IDs for OAuth authentication workflow. Let's create a SP called **SP_Purview_Test**

1. Application (Client) ID
2. Client Secret: we need to manually create it through `credential & secret` panel 

![Purview Service Principal](https://i.ibb.co/nb1DHzX/IMG-20240409-151305.jpg)

### Purview Role Assignment for Service Principal

Based on Azure Role Based Access Control (RBAC), we need to assign proper roles for our service principal to access different Purview services. We can do that on `Data map` through `Collection` then `Role assignment`.  Role assignment is on collection level which is hierarchical structure, role on high level collection can be inherited to lower level connection.  

| Role              | Service                                           |
| ----------------- | ------------------------------------------------- |
| Data Source Admin | Scan Data Plane                                   |
| Collection Admin  | Account Data Plane and Metadata Policy Data Plane |
| Data Curator      | Catalog Data Plane                                |

![Purview Role Assignment](https://i.ibb.co/6nC6j64/IMG-20240409-151137.jpg)

### Get the Access Token from OAuth2.0 Workflow

As mentioned above, to be able to access OAuth2.0 service framework, the first thing first we need to call its endpoint with mandatory payload parameters to get the access token (Bear token). We can send a `POST` request to the following URL to get the access token.

> https://login.microsoftonline.com/{Purview-tenant-id}/oauth2/token

We can easily find Purview **tenant id** from properties page. 

![Purview Tenant ID](https://i.ibb.co/NCpB2Vw/IMG-20240411-093420.jpg)

The following parameters need to be passed to the above URL as payload values:

* client_id: Client ID of service principal registered in Microsoft Entra ID and is assigned to a data plane role for the Microsoft Purview Account
* client_secret: client secret created for service principal
* grant_type: this should be 'client_credentials'
* resource: 'https://purview.azure.net'

![Purview API Authentication](https://i.ibb.co/6YHCJLn/IMG-20240409-153347.jpg)

```json
payload= {
    "client_id": cliendId,
    "client_secret": clientSecret,
    "grant_type": "client_credentials",
    "resource": "https://purview.azure.net"
}
```

Sample response

```json
{
        "token_type": "Bearer",
        "expires_in": "86399",
        "ext_expires_in": "86399",
        "expires_on": "1621038348",
        "not_before": "1620951648",
        "resource": "https://purview.azure.net",
        "access_token": "<<access token>>"
    }
```

Then, we make API call to acquire access token

```python
res = requests.post(url=endpoint_url, data=payload)
response = res.json()
accessToken = response["access_token"]
tokenType = response["token_type"]
accessTokenFull = tokenType + ' ' + accessToken
headers["authorization"] = accessTokenFull
```

### Client Tool OAth2.0 Authentication Process Use Case - Azure Databricks

Azure Databricks is the most common tool to access Purview on data process, now let's see how do we authenticate ADB on Microsoft Purview. First, we need to create a secret scope, under the secret scope, create one secret for SP (service principal) `client_id` and another secret for SP `client_secret`.

#### Create An Azure Key Vault secret scope

Go to `https://<databricks-instance>#secrets/createScope`. This URL is case sensitive; scope in `createScope` must be uppercase.

Enter the name of the secret scope. Secret scope names are case insensitive.

In our case, we will create secret scope called **purview-test**

#### Create Secrets for Client ID and Client Secret

A secret is a key-value pair that stores secret material, with a key name unique within a [secret scope](https://learn.microsoft.com/en-us/azure/databricks/security/secrets/secret-scopes).

In our case, we will create **SP_Purview_Test_AppID** for SP client id and **SP_Purview_Test_PWD** for sp client secret

after we create those, we can list them and check by `dbutils` function in ADB

```python
dbutils.secret.list('purview-test')
```

It would return the result

```python
Out[1]: [SecretMetadata(key='SP_Purview_Test_AppID'), SecretMetadata(key='SP_Purview_Test_PWD')]
```

Now, we can assign those value to variables in ADB

```python
clientId = dbutils.secrets.get('purview-test', 'SP_Purview_Test_AppID')
clientSecret = dbutils.secrets.get('purview-test', 'SP_Purview_Test_PWD')

payload= {
    "client_id": cliendId,
    "client_secret": clientSecret,
    "grant_type": "client_credentials",
    "resource": "https://purview.azure.net"
}

res = requests.post(url=endpoint_url, data=payload)
response = res.json()
accessToken = response["access_token"]
tokenType = response["token_type"]
accessTokenFull = tokenType + ' ' + accessToken
headers["authorization"] = accessTokenFull
```

Now, ADB can access Purview API services after get access token. One of the most important the function is creating and updating business metadata, Purview called it asset properties and **Managed Attributes**

![Purview Managed Attributes](https://i.ibb.co/M2wwfCv/IMG-20240411-093516.jpg)

## Microsoft Purview MSI for Accessing Scan Data Sources

Microsoft Purview offers **Managed Service Identity** (MSI) as a secure way to access and manage data sources with `Private endpoint`, ensuring that our data remains secure and compliant.

### Understanding Purview Managed Service Identity (MSI)

Managed Service Identity (MSI) is a feature of Azure Active Directory that provides an identity for services running in Azure. Purview uses MSI to authenticate itself when accessing data sources for scanning, eliminating the need to manage credentials.

### Registering Purview MSI

When we set up Microsoft Purview, a Managed Service Identity (MSI) is automatically created for it, usually it is Purview account. This identity is used to authenticate Purview when accessing data sources. We can view and manage this identity in the Azure portal Purview page.

![Purview MSI](https://i.ibb.co/7jgJfFf/IMG-20240411-093534.jpg)

### Purview Source Data Scan Authentication

Scan is one of the most important function in Purview against source data.

* Technical metadata harvesting
* Apply data classification through scan rule set
* Apply sensitivity labels to indicate the data security level

Scan is applied on data source level and returned by data assets as result.

### The Process Procedure

The first step is create `Managed private endpoints` to get provisioning approval on source data side. 

![Purview Private Endpoints](https://i.ibb.co/YbBYS9h/IMG-20240411-093614.jpg)

Next, we should register data source in Purview through `Data map`, we can register almost all kinds of data source such as ADLS Gen2 storage account, Azure SQL Database.

Then, we need to Create a AZ group **AZ_Purview_StorageBlobDataReader** through `Azure Entra ID` (previously it was called `Azure Active Directory`), go to the data source Access Control (IAM) and assigned ***Storage Blob Data Reader*** role to this AZ group by Azure RBAC (Role based Access Control) 

![Data Source Role Assignment](https://i.ibb.co/m05rxgB/IMG-20240411-111757.jpg)

Last, add Purview MSI into the AZ group we just created.

![Purview MSI in AZ Group](https://i.ibb.co/DpNPDCp/IMG-20240411-141930.jpg)

Now, Purview can access data source so that we can create new scan to harvest metadata from data source.







