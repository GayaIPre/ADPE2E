# Lab 2: Transform Big Data using Azure Data Factory and Azure SQL Data Warehouse
In this lab you will use Azure Data Factory to download large data files to your data lake and use Azure SQL Data Warehouse to generate a summary dataset and store it. The dataset you will use contains detailed New York City Yellow Taxi rides for 2018. You will generate a daily aggregated summary of all rides and save the result in your data warehouse. You will use Power BI to visualise summarised taxi ride data.

The estimated time to complete this lab is: **60 minutes**.

## Microsoft Learn & Technical Documentation

Azure Service | Microsoft Learn | Technical Documentation|
--------------|-----------------|------------------------|
Azure Data Lake Gen2 | [Large Scale Data Processing with Azure Data Lake Storage Gen2](https://docs.microsoft.com/en-us/learn/paths/data-processing-with-azure-adls/) | [Azure Data Lake Gen2 Technical Documentation](https://docs.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-introduction)

## Lab Architecture
![Lab Architecture](./Media/Lab2-Image01.png)

Step     | Description
-------- | -----
![](./Media/Green1.png) | Build an Azure Data Factory Pipeline to copy big data files from shared Azure Storage
![](./Media/Green2.png) | Save data files to your data lake
![](./Media/Green3.png) | Use Polybase to load data into staging tables in your Azure SQL Data Warehouse. Call a Stored Procedure to perform data aggregations and save results in the final table.
![](./Media/Green4.png) | Visualize data from your Azure SQL Data Warehouse using Power BI

**IMPORTANT**: Some of the Azure services provisioned require globally unique name and a “-suffix” has been added to their names to ensure this uniqueness. Please take note of the suffix generated as you will need it for the following resources in this lab:

Name	                     |Type
-----------------------------|--------------------
MDWDataFactory-*suffix*	     |Data Factory (V2)
mdwdatalake*suffix*	         |Storage Account
mdwsqlvirtualserver-*suffix* |SQL server

## Create Azure SQL Data Warehouse database objects
In this section you will connect to Azure SQL Data Warehouse to create the database objects used to host and process data.

![](./Media/Lab2-Image02.jpg)

**IMPORTANT**|
-------------|
**Execute these steps inside the MDWDesktop remote desktop connection**|

**IMPORTANT**: The full solution script can be found ![here](./Solution/Create Staging NYCTaxiData.sql)

1.	Open Azure Data Studio. 
2.	If you already have a connection to MDWSQLVirtualServer, then **go to step 6**.
3.	On the **Servers** panel, click **New Connection**.

    ![](./Media/Lab2-Image03.png)

4.	On the Connection Details panel, enter the following connection details:
    <br>- **Server**: mdwsqlvirtualserver-*suffix*.database.windows.net
    <br>- **Authentication Type**: SQL Login
    <br>- **User Name**: mdwadmin
    <br>- **Password**: P@ssw0rd123!
    <br>- **Database**: MDWASQLDW
5.	Click **Connect**.

    ![](./Media/Lab2-Image04.png)

6.	Right click the MDWSQLVirtualServer name and then click **New Query**.

7.	Create a new round robin distributed table named Staging.NYCTaxiData. Use the script below:

```sql
create schema [Staging]
go

create table [Staging].[NYCTaxiData]
(
    [VendorID] [int] NULL,
    [tpep_pickup_datetime] [datetime] NULL,
    [tpep_dropoff_datetime] [datetime] NULL,
    [passenger_count] [smallint] NULL,
    [trip_distance] [decimal](8, 2) NULL,
    [RatecodeID] [smallint] NULL,
    [store_and_fwd_flag] [char](1) NULL,
    [PULocationID] [int] NULL,
    [DOLocationID] [int] NULL,
    [payment_type] [smallint] NULL,
    [fare_amount] [decimal](10, 2) NULL,
    [extra] [decimal](10, 2) NULL,
    [mta_tax] [decimal](10, 2) NULL,
    [tip_amount] [decimal](10, 2) NULL,
    [tolls_amount] [decimal](10, 2) NULL,
    [improvement_surcharge] [decimal](10, 2) NULL,
    [total_amount] [decimal](10, 2) NULL
)
with
(
    distribution = round_robin,
    heap
)
go

```

8.	Create a new table called Staging.NYCTaxiLocationLookup to host NYC taxi location lookup data. Use the script below:

```sql
create table [Staging].[NYCTaxiLocationLookup]
(
    [LocationID] [int] NULL,
    [Borough] [varchar](200) NULL,
    [Zone] [varchar](200) NULL,
    [service_zone] [varchar](200) NULL
)
with
(
    distribution = round_robin,
    clustered columnstore index
)
```

9.	Create a stored procedure that will transform and aggregate data and save the result in a new table called NYC.TaxiDataSummary . Use the script below:

```sql
create procedure Staging.spNYCLoadTaxiDataSummary
as
    --Drop and re-create Staging.idx_NYCTaxiData index
    if (exists(select top 1 1 from sys.indexes where name = 'idx_NYCTaxiData' and object_id = object_id('Staging.NYCTaxiData')))
        drop index idx_NYCTaxiData on Staging.NYCTaxiData 
    
    create index idx_NYCTaxiData on Staging.NYCTaxiData(tpep_pickup_datetime, PULocationID, payment_type, passenger_count, trip_distance, tip_amount, fare_amount, total_amount)

    --Drop and re-create NYC.TaxiDataSummary table
    if (exists(select top 1 1 from sys.objects where name = 'TaxiDataSummary' and schema_id = schema_id('NYC') and type = 'U'))
        drop table NYC.TaxiDataSummary 

    create table NYC.TaxiDataSummary 
    with (
        distribution = round_robin
    )
    as
    select 
        cast(tpep_pickup_datetime as date) as PickUpDate
        , PickUp.Borough as PickUpBorough
        , PickUp.Zone as PickUpZone
        , case payment_type
            when 1 then 'Credit card'
            when 2 then 'Cash'
            when 3 then 'No charge'
            when 4 then 'Dispute'
            when 5 then 'Unknown'
            when 6 then 'Voided trip'
        end as PaymentType
        , count(*) as TotalTripCount
        , sum(passenger_count) as TotalPassengerCount
        , sum(trip_distance) as TotalDistanceTravelled
        , sum(tip_amount) as TotalTipAmount
        , sum(fare_amount) as TotalFareAmount
        , sum(total_amount) as TotalTripAmount
    from Staging.NYCTaxiData
        inner join Staging.NYCTaxiLocationLookup as PickUp
            on NYCTaxiData.PULocationID = PickUp.LocationID
    group by cast(tpep_pickup_datetime as date) 
        , PickUp.Borough
        , PickUp.Zone
        , payment_type

    --drop index idx_NYCTaxiData so it does not impact future loads
    drop index idx_NYCTaxiData on Staging.NYCTaxiData
go
```

## Create NYCTaxiData Container on Azure Blob Storage
In this section you will create a container in your MDWDataLake that will be used as a repository for the NYC Taxi Data files. You will copy 12 files from the MDWResources Storage Account into your NYCTaxiData container. These files contain data for all Yellow Taxi rides in 2018, one file for each month of the year.

![](./Media/Lab2-Image05.jpg)

**IMPORTANT**|
-------------|
**Execute these steps on your host computer**|

1.	In the Azure Portal, go to the lab resource group and locate the Azure Storage account **mdwdatalake*suffix***. 
2.	On the **Overview** panel, click **Blobs**.

    ![](./Media/Lab2-Image06.png)

3.	On the **mdwdalalake*suffix* – Blobs** blade, click **+ Container**.

    ![](./Media/Lab2-Image07.png)

4.	On the New container blade, enter the following details:
    <br>- **Name**: nyctaxidata
    <br>- **Public access level**: Private (no anonymous access)
5.	Click **OK** to create the new container.

    ![](./Media/Lab2-Image08.png)

## Create Linked Service connection to MDWResources
In this section you will create a linked service connection to a shared storage accounnt called MDWResources hosted in an external Azure subscription. This storage account hosts the NYC Taxi data files you will copy to your data lake. As this storage account sits in an external subscription you will connect to it using a SAS URL token.

**IMPORTANT**|
-------------|
**Execute these steps on your host computer**|

1.	Open the **Azure Data Factory** portal and click the **Author option *(pencil icon)*** on the left-hand side panel. Under **Connections** tab, click **Linked Services** and then click **+ New** to create a new linked service connection.

    ![](./Media/Lab2-Image09.png)

2.	On the **New Linked Service** blade, type “Azure Blob Storage” in the search box to find the **Azure Blob Storage** linked service. Click **Continue**.

    ![](./Media/Lab2-Image10.png)

3.	On the **New Linked Service (Azure Blob Storage)** blade, enter the following details:
    <br>- **Name**: MDWResources
    <br>- **Connect via integration runtime**: AutoResolveIntegrationRuntime
    <br>- **Authentication method**: SAS URI
    <br>- **SAS URL**: 
    ```
    https://mdwresources.blob.core.windows.net/?sv=2018-03-28&ss=b&srt=sco&sp=rwl&se=2050-12-30T17:25:52Z&st=2019-04-05T09:25:52Z&spr=https&sig=4qrD8NmhaSmRFu2gKja67ayohfIDEQH3LdVMa2Utykc%3D
    ```
4.	Click **Test connection** to make sure you entered the correct connection details and then click **Finish**.

    ![](./Media/Lab2-Image11.png)

## Create Source and Destination Data Sets
In this section you are going to create 5 datasets that will be used by your data pipeline:

Dataset | Description
--------|---------------
**MDWResources_NYCTaxiData**| References MDWResources shared storage account container that contains source data files.
**MDWResources_NYCTaxiLookup_CSV**| References MDWResources shared storage account that contains a .csv file with all taxi location codes and names.
**MDWASQLDW_StagingNYCTaxiData**| References the table Staging.NYCTaxiData in the Azure SQL Data Warehouse database MDWASQLDW.
**MDWASQLDW_StagingNYCLocationLookup**| References the table [Staging].[NYCTaxiLocationLookup] in the Azure SQL Data Warehouse database MDWASQLDW and acts as destination of lookup data copied from MDWResources_NYCTaxiLookup.
**MDWDataLake_NYCTaxiData_Binary**| References your MDWDataLake-*suffix* storage account. It acts as the destination for the files copied from MDWResources_NYCTaxiData. 
**MDWDataLake_NYCTaxiData_CSV**| References your MDWDataLake-*suffix* storage account. It functions as a data source when copying data to MDWASQLDW_StagingNYCTaxiData.

**IMPORTANT**|
-------------|
**Execute these steps on your host computer**|

1.	Open the **Azure Data Factory** portal and click the **Author *(pencil icon)*** option on the left-hand side panel. Under **Factory Resources** tab, click the ellipsis **(…)** next to **Datasets** and then click **Add Dataset** to create a new dataset.

    ![](./Media/Lab2-Image12.png)

2.	Type “Azure Blob Storage” in the search box and select **Azure Blob Storage**. Click **Continue**.

    ![](./Media/Lab2-Image13.png)

3.	On the **Select Format** blade, select **Binary** and click **Continue**.

    ![](./Media/Lab2-Image14.png)

4.	On the **Set Properties** blade, enter the following details:
    <br>- **Name**: MDWResources_NYCTaxiData
    <br>- **Linked service**: MDWResources
    <br>- **File Path**: **Container**: nyctaxidata, **Directory**: [blank], **File**: [blank]
    
    ![](./Media/Lab2-Image41.png)

    Alternatively you can copy and paste the Dataset JSON definition below:

    ```json
    {
        "name": "MDWResources_NYCTaxiData",
        "properties": {
            "linkedServiceName": {
                "referenceName": "MDWResources",
                "type": "LinkedServiceReference"
            },
            "annotations": [],
            "type": "Binary",
            "typeProperties": {
                "location": {
                    "type": "AzureBlobStorageLocation",
                    "container": "nyctaxidata"
                }
            }
        }
    }
    ```
5.	Leave remaining fields with default values.

    ![](./Media/Lab2-Image15.png)

6.	Repeat the process to create another Azure Storage Binary dataset, this time referencing the NYCTaxiData container in your MDWDataLake storage account. This dataset acts as the destination for the NYC taxi data files you will copy from the previous dataset.

7.	Type “Azure Blob Storage” in the search box and select **Azure Blob Storage**. Click **Continue**.

    ![](./Media/Lab2-Image13.png)

8.	On the **Select Format** blade, select **Binary** and click **Continue**.

    ![](./Media/Lab2-Image14.png)

9.	On the **Set Properties** blade, enter the following details:
    <br>- **Name**: MDWDataLake_NYCTaxiData_Binary
    <br>- **Linked Service**: MDWDataLake
    <br>- **File Path**: **Container**: nyctaxidata, **Directory**: [blank], **File**: [blank]

    ![](./Media/Lab2-Image42.png)

    Click **Continue**.    

    Alternatively you can copy and paste the Dataset JSON definition below:

    ```json
    {
        "name": "MDWDataLake_NYCTaxiData_Binary",
        "properties": {
            "linkedServiceName": {
                "referenceName": "MDWDataLake",
                "type": "LinkedServiceReference"
            },
            "annotations": [],
            "type": "Binary",
            "typeProperties": {
                "location": {
                    "type": "AzureBlobStorageLocation",
                    "container": "nyctaxidata"
                }
            }
        }
    }
    ```
10.	Leave remaining fields with default values.

    ![](./Media/Lab2-Image16.png)

11.	Repeat the process to create a new Azure Storage CSV dataset referencing the NYCTaxiData container in your MDWDataLake storage account. This dataset acts as the data source of NYC taxi records (CSV) you will copy to your Azure SQL Data Warehouse.

12.	Type “Azure Blob Storage” in the search box and select **Azure Blob Storage**. Click **Continue**.

    ![](./Media/Lab2-Image13.png)

13.	On the **Select Format** blade, select **DelimitedText** and click **Continue**.

    ![](./Media/Lab2-Image43.png)

14.	On the **Set Properties** blade, enter the following details:
    <br>- **Name**: MDWDataLake_NYCTaxiData_CSV
    <br>- **Linked Service**: MDWDataLake
    <br>- **File Path**: **Container**: nyctaxidata, **Directory**: [blank], **File Path**: [blank]
    <br>- **First row as header**: Checked
    <br>- **Import schema**: None

    ![](./Media/Lab2-Image44.png)

    Click **Continue**.

15.	On the dataset properties, set the following property values:
    <br>- **Connection > Escape character**: No escape character
    <br>- **Connection > Quote character**: No quote character

    Leave remaining fields with default values.

    ![](./Media/Lab2-Image46.png)

    Alternatively you can copy and paste the Dataset JSON definition below:

    ```json
    {
        "name": "MDWDataLake_NYCTaxiData_CSV",
        "properties": {
            "linkedServiceName": {
                "referenceName": "MDWDataLake",
                "type": "LinkedServiceReference"
            },
            "annotations": [],
            "type": "DelimitedText",
            "typeProperties": {
                "location": {
                    "type": "AzureBlobStorageLocation",
                    "container": "nyctaxidata"
                },
                "columnDelimiter": ",",
                "escapeChar": "",
                "firstRowAsHeader": true,
                "quoteChar": ""
            },
            "schema": []
        }
    }
    ```

16.	Repeat the process to create another Azure Blob CSV dataset, this time referencing the NYCTaxiLookup container in your MDWResources storage account. 

17.	Type “Azure Blob Storage” in the search box and select **Azure Blob Storage**. Click **Continue**.

    ![](./Media/Lab2-Image13.png)

18.	On the **Select Format** blade, select **DelimitedText** and click **Continue**.

    ![](./Media/Lab2-Image43.png)

19.	On the **Set Properties** blade, enter the following details:
    <br>- **Name**: MDWResources_NYCTaxiLookup_CSV
    <br>- **Linked Service**: MDWResources
    <br>- **File Path**: **Container**:nyctaxilookup, **Directory*: [blank], **File**: [blank]
    <br>- **First row as header**: Checked
    <br>- **Import schema**: None.

    ![](./Media/Lab2-Image47.png)

20.	Leave remaining fields with default values.  

    Alternatively you can copy and paste the Dataset JSON definition below:

    ```json
    {
        "name": "MDWResources_NYCTaxiLookup_CSV",
        "properties": {
            "linkedServiceName": {
                "referenceName": "MDWResources",
                "type": "LinkedServiceReference"
            },
            "annotations": [],
            "type": "DelimitedText",
            "typeProperties": {
                "location": {
                    "type": "AzureBlobStorageLocation",
                    "container": "nyctaxilookup"
                },
                "columnDelimiter": ",",
                "escapeChar": "\\",
                "firstRowAsHeader": true,
                "quoteChar": "\""
            },
            "schema": []
        }
    }
    ```


21.	Repeat the process to create another dataset, this time referencing the Staging.NYCTaxiData in your Azure SQL Data Warehouse database. 
22.	Type “Azure SQL Data Warehouse” in the search box and select **Azure SQL Data Warehouse**. Click **Continue**.

    ![](./Media/Lab2-Image17.png)

23.	On the Set Properties blade, enter the following details:
    <br>- **Name**: MDWASQLDW_StagingNYCTaxiData
    <br>- **Linked Service**: MDWSQLVirtualServer_MDWASQLDW
    <br>- **Table**: [Staging].[NYCTaxiData]

    Alternatively you can copy and paste the Dataset JSON definition below:

    ```json
    {
    "name": "MDWASQLDW_StagingNYCTaxiData",
    "properties": {
        "linkedServiceName": {
            "referenceName": "MDWSQLVirtualServer_MDWASQLDW",
            "type": "LinkedServiceReference"
        },
        "type": "AzureSqlDWTable",
        "typeProperties": {
            "tableName": "[Staging].[NYCTaxiData]"
        }
    },
    "type": "Microsoft.DataFactory/factories/datasets"
    }
    ```

24.	Leave remaining fields with default values.

    ![](./Media/Lab2-Image18.png)

25.	Repeat the process to create another dataset, this time referencing the Staging.NYCLocationLookup in your Azure SQL Data Warehouse database. 

26.	Type “Azure SQL Data Warehouse” in the search box and select **Azure SQL Data Warehouse**. Click **Finish**.

    ![](./Media/Lab2-Image17.png)

27.	On the Set Properties blade, enter the following details:
    <br>-**Name**: MDWASQLDW_StagingNYCLocationLookup
    <br>-**Linked Service**: MDWSQLVirtualServer_MDWASQLDW
    <br>-**Table**: [Staging].[NYCTaxiLocationLookup]

    Alternatively you can copy and paste the Dataset JSON definition below:

    ```json
    {
    "name": "MDWASQLDW_StagingNYCLocationLookup",
    "properties": {
        "linkedServiceName": {
            "referenceName": "MDWSQLVirtualServer_MDWASQLDW",
            "type": "LinkedServiceReference"
        },
        "type": "AzureSqlDWTable",
        "typeProperties": {
            "tableName": "[Staging].[NYCTaxiLocationLookup]"
        }
    },
    "type": "Microsoft.DataFactory/factories/datasets"
    }
    ```

28.	Leave remaining fields with default values.

    ![](./Media/Lab2-Image19.png)

29.	Publish your dataset changes by clicking the **Publish all** button.

    ![](./Media/Lab2-Image20.png)

## Create and Execute Pipeline
In this section you create a data factory pipeline to copy data in the following sequence:

1. Copy NYC Taxi CSV Data files from shared storage account **MDWResources** to your the **nyctaxidata** container in your **MDWDataLake-*suffix*** storage account;
2. Copy data from the files sitting in your data lake to a staging table in your SQL Data Warehouse;
3. Execute a stored procedure in your SQL Data Warehouse to generate a daily summary and save it in a new table used for Power BI reporting.

**IMPORTANT**: You can either follow the steps in this section to create the pipeline or you can copy and paste the pipeline JSON code below:

<details>
<summary>Click to expand pipeline JSON code</summary>
<p>

```json
{
    "name": "Copy NYC Taxi Data",
    "properties": {
        "activities": [
            {
                "name": "CopyTaxiDataFiles",
                "type": "Copy",
                "dependsOn": [],
                "policy": {
                    "timeout": "7.00:00:00",
                    "retry": 0,
                    "retryIntervalInSeconds": 30,
                    "secureOutput": false,
                    "secureInput": false
                },
                "userProperties": [],
                "typeProperties": {
                    "source": {
                        "type": "BinarySource",
                        "storeSettings": {
                            "type": "AzureBlobStorageReadSettings",
                            "recursive": true
                        }
                    },
                    "sink": {
                        "type": "BinarySink",
                        "storeSettings": {
                            "type": "AzureBlobStorageWriteSettings",
                            "copyBehavior": "PreserveHierarchy"
                        }
                    },
                    "enableStaging": false
                },
                "inputs": [
                    {
                        "referenceName": "MDWResources_NYCTaxiData",
                        "type": "DatasetReference"
                    }
                ],
                "outputs": [
                    {
                        "referenceName": "MDWDataLake_NYCTaxiData_Binary",
                        "type": "DatasetReference"
                    }
                ]
            },
            {
                "name": "CopyTaxiDataToDW",
                "type": "Copy",
                "dependsOn": [
                    {
                        "activity": "CopyTaxiDataFiles",
                        "dependencyConditions": [
                            "Succeeded"
                        ]
                    }
                ],
                "policy": {
                    "timeout": "7.00:00:00",
                    "retry": 0,
                    "retryIntervalInSeconds": 30,
                    "secureOutput": false,
                    "secureInput": false
                },
                "userProperties": [],
                "typeProperties": {
                    "source": {
                        "type": "DelimitedTextSource",
                        "storeSettings": {
                            "type": "AzureBlobStorageReadSettings",
                            "recursive": true,
                            "wildcardFileName": "*.*"
                        },
                        "formatSettings": {
                            "type": "DelimitedTextReadSettings"
                        }
                    },
                    "sink": {
                        "type": "SqlDWSink",
                        "allowPolyBase": true,
                        "preCopyScript": "truncate table Staging.NYCTaxiData",
                        "polyBaseSettings": {
                            "rejectValue": 0,
                            "rejectType": "value",
                            "useTypeDefault": true
                        }
                    },
                    "enableStaging": false
                },
                "inputs": [
                    {
                        "referenceName": "MDWDataLake_NYCTaxiData_CSV",
                        "type": "DatasetReference"
                    }
                ],
                "outputs": [
                    {
                        "referenceName": "MDWASQLDW_StagingNYCTaxiData",
                        "type": "DatasetReference"
                    }
                ]
            },
            {
                "name": "CopyTaxiLookupDataToDW",
                "type": "Copy",
                "dependsOn": [],
                "policy": {
                    "timeout": "7.00:00:00",
                    "retry": 0,
                    "retryIntervalInSeconds": 30,
                    "secureOutput": false,
                    "secureInput": false
                },
                "userProperties": [],
                "typeProperties": {
                    "source": {
                        "type": "DelimitedTextSource",
                        "storeSettings": {
                            "type": "AzureBlobStorageReadSettings",
                            "recursive": true,
                            "wildcardFileName": "*.*"
                        },
                        "formatSettings": {
                            "type": "DelimitedTextReadSettings"
                        }
                    },
                    "sink": {
                        "type": "SqlDWSink",
                        "allowPolyBase": true,
                        "preCopyScript": "truncate table [Staging].[NYCTaxiLocationLookup]",
                        "polyBaseSettings": {
                            "rejectValue": 0,
                            "rejectType": "value",
                            "useTypeDefault": true
                        }
                    },
                    "enableStaging": true,
                    "stagingSettings": {
                        "linkedServiceName": {
                            "referenceName": "MDWDataLake",
                            "type": "LinkedServiceReference"
                        },
                        "path": "polybase"
                    }
                },
                "inputs": [
                    {
                        "referenceName": "MDWResources_NYCTaxiLookup_CSV",
                        "type": "DatasetReference"
                    }
                ],
                "outputs": [
                    {
                        "referenceName": "MDWASQLDW_StagingNYCLocationLookup",
                        "type": "DatasetReference"
                    }
                ]
            },
            {
                "name": "LoadTaxiDataSummary",
                "type": "SqlServerStoredProcedure",
                "dependsOn": [
                    {
                        "activity": "CopyTaxiDataToDW",
                        "dependencyConditions": [
                            "Succeeded"
                        ]
                    },
                    {
                        "activity": "CopyTaxiLookupDataToDW",
                        "dependencyConditions": [
                            "Succeeded"
                        ]
                    }
                ],
                "policy": {
                    "timeout": "7.00:00:00",
                    "retry": 0,
                    "retryIntervalInSeconds": 30,
                    "secureOutput": false,
                    "secureInput": false
                },
                "userProperties": [],
                "typeProperties": {
                    "storedProcedureName": "[Staging].[spNYCLoadTaxiDataSummary]"
                },
                "linkedServiceName": {
                    "referenceName": "MDWSQLVirtualServer_MDWASQLDW",
                    "type": "LinkedServiceReference"
                }
            }
        ],
        "annotations": []
    }
}
```
</details>
<br>

**IMPORTANT**|
-------------|
**Execute these steps on your host computer**|

1.	Open the **Azure Data Factory** portal and click the **Author *(pencil icon)*** option on the left-hand side panel. Under the **Factory Resources** tab, click the ellipsis **(…)** next to Pipelines and then click **Add Pipeline** to create a new dataset.
2.	On the New Pipeline tab, enter the following details:
    <br>- **General > Name**: Copy NYC Taxi Data
3.	Leave remaining fields with default values.

    ![](./Media/Lab2-Image21.png)

4.	From the Activities panel, type “Copy Data” in the search box. Drag the Copy Data activity on to the design surface. This copy activity will copy data files from MDWResources to MDWDatalake.
5.	Select the Copy Data activity and enter the following details:
    <br>- **General > Name**: CopyTaxiDataFiles
    <br>- **Source > Source dataset**: MDWResources_NYCTaxiData
    <br>- **Sink > Sink dataset**: MDWDataLake_NYCTaxiData_Binary
    <br>- **Sink > Copy Behavior**: Preserve Hierarchy
6.	Leave remaining fields with default values.

    ![](./Media/Lab2-Image22.png)
    ![](./Media/Lab2-Image23.png)

7.	Repeat the process to create another Copy Data Activity, this time to copy data from the files in your data lake to your SQL Data Warehouse.
8.	From the Activities panel, type “Copy Data” in the search box. Drag the Copy Data activity on to the design surface.
9.	Select the Copy Data activity and enter the following details:
    <br>- **General > Name**: CopyTaxiDataToDW
    <br>- **Source > Source dataset**: MDWDataLake_NYCTaxiData_CSV
    <br>- **Sink > Sink dataset**: MDWASQLDW_StagingNYCTaxiData
    <br>- **Sink > Pre Copy Script**: 
    ```sql
    truncate table Staging.NYCTaxiData
    ```
10.	Leave remaining fields with default values.
    
    ![](./Media/Lab2-Image24.png)
    ![](./Media/Lab2-Image25.png)

11.	Create a **Success *(green)*** precedence constraint between CopyTaxiDataFiles and CopyTaxiDataToDW. You can do it by dragging the green connector from CopyTaxiDataFiles and landing the arrow onto CopyTaxiDataToDW.

    ![](./Media/Lab2-Image26.png)

12.	Repeat the process to create another Copy Data Activity, this time to copy taxi location lookup data from MDWResources to your SQL Data Warehouse.
13.	From the Activities panel, type “Copy Data” in the search box. Drag the Copy Data activity on to the design surface.
14.	Select the Copy Data activity and enter the following details:
    <br>- **General > Name**: CopyTaxiLookupDataToDW
    <br>- **Source > Source dataset**: MDWResources_NYCTaxiLookup_CSV
    <br>- **Sink > Sink dataset**: MDWASQLDW_StagingNYCLocationLookup
    <br>- **Sink > Pre Copy Script**: 
    ```sql
    truncate table Staging.NYCTaxiLocationLookup
    ```
    <br>- **Settings > Enable staging**: Checked
    <br>- **Settings > Staging account linked service**: MDWDataLake
    <br>- **Settings > Storage Path**: polybase
15.	Leave remaining fields with default values.

    ![](./Media/Lab2-Image27.png)
    ![](./Media/Lab2-Image28.png)
    ![](./Media/Lab2-Image29.png)

16.	From the Activities panel, type “Stored Procedure” in the search box. Drag the Stored Procedure activity onto the design surface. This activity will execute the Staging.spNYCLoadTaxiDataSummary to generate aggregated taxi ride information.
17.	Select the Stored Procedure activity and enter the following details:
    <br>- **General > Name**: LoadTaxiDataSummary
    <br>- **SQL account > Linked service**: MDWSQLVirtualServer_MDWASQLDW
    <br>- **Stored procedure > Stored procedure name**: [Staging].[spNYCLoadTaxiDataSummary]
18.	Leave remaining fields with default values.

    ![](./Media/Lab2-Image30.png)
    ![](./Media/Lab2-Image31.png)

19.	Create **Success *(green)*** from CopyDataToDW to LoadTaxiDataSummary and from CopyTaxiLookupData to LoadTaxiDataSummary.

    ![](./Media/Lab2-Image32.png)

20.	Publish your pipeline changes by clicking the **Publish all** button.

    ![](./Media/Lab2-Image33.png)

21.	To execute the pipeline, click on **Add trigger** menu and then **Trigger Now**.
22.	On the **Pipeline Run** blade, click **Finish**.

    ![](./Media/Lab2-Image34.png)

23.	To monitor the execution of your pipeline, click on the **Monitor** menu on the left-hand side panel.
24.	You should be able to see the Status of your pipeline execution on the right-hand side panel.

    ![](./Media/Lab2-Image35.png)

25.	Click the **View Activity Runs** button for detailed information about each activity execution in the pipeline. The whole execution should last between 7-8 minutes.

    ![](./Media/Lab2-Image36.png)
    ![](./Media/Lab2-Image37.png)

## Visualize Data with Power BI
In this section you are going to use Power BI to visualize data from Azure SQL Data Warehouse. The Power BI report will use an Import connection to query Azure SQL Data Warehouse and visualise Motor Vehicle Collision data from the table you loaded in the previous exercise.

**IMPORTANT**|
-------------|
**Execute these steps inside the MDWDesktop remote desktop connection**|

1.	On MDWDesktop, download the Power BI report from the link https://aka.ms/MDWLab2 and save it in the Desktop.
2.	Open the file MDWLab2.pbit with Power BI Desktop.
3.	When prompted to enter the value of the MDWSQLVirtualServer parameter, type the full server name: **mdwsqlvirtualserver-*suffix*.database.windows.net**
4.	Click **Load**.

    ![](./Media/Lab2-Image38.png)

5.	When prompted to enter credentials, select **Database** from the left-hand side panel and enter the following details:
    <br>- **User name**: mdwadmin
    <br>- **Password**: P@ssw0rd123!
6.	Leave remaining fields with their default values.
7.	Click **Connect**.

    ![](./Media/Lab2-Image39.png)

8.	Once data finish loading interact with the report by changing the PickUpDate slicer and by clicking on the other visualisations.
9.	Save your work and close Power BI Desktop.

    ![](./Media/Lab2-Image40.png)