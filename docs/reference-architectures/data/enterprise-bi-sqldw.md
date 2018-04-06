---
title: Gain business insights from relational data
description: Gain business insights from relational data
author: alexbuckgit
ms.date: 03/21/2018
---

# Enterprise BI with SQL Data Warehouse
 
This reference architecture implements an ELT (extract-load-transform) pipeline that moves the data into SQL Data Warehouse and transforms the data for analysis.

Scenario: An organization has a large OLTP data set stored in a SQL Server database on premises. The organization wants to use SQL Data Warehouse to perform analysis using Power BI. 

This reference architecture is designed for one-time or on-demand jobs. If you need to move data on a continuing basis (hourly or daily), we recommend using Azure Data Factory to define an automated workflow.

## Architecture

**SQL Server**. The source data is located in a SQL Server database on premises. To simulate the on-premises environment, the deployment scripts for this architecture provision a virtual machine in Azure with SQL Server installed. 

**Blob Storage**. Before moving the data in SQL Data Warehouse, it is copied over the network into Blob storage. 

**Azure SQL Data Warehouse**. SQL Data Warehouse is a distributed system designed to perform analytics on large data. It supports massive parallel processing (MPP), which makes it suitable for running high-performance analytics. 

Consider SQL Data Warehouse when you have large amounts of data (more than 1 TB) and are running an analytics workload. SQL Data Warehouse is not a good fit for OLTP workloads or smaller data sets (< 250GB). For data sets less than 250GB, consider Azure SQL Database or SQL Server.

**Azure Analysis Services**. Analysis Services is a full managed service that provides data modeling capabilities and data modeling functionality. It is used to create a semantic model that can users can query to gain insight. Analysis Services is especially useful in a BI dashboard scenario. In this architecture, Analysis Service reads data from the data warehouse to process the semantic model, and efficiently serves dashboard queries. It also supports elastic concurrency, by scaling out replicas for faster query processing.

**Power BI**. Power BI is a suite of business analytics tools to analyze data for business insights. In this architecture, it queries the semantic model stored in Analysis Services.

**Azure Active Directory** (Azure AD) authenticates users who connect to the Analysis Services server through Power BI.

## Data Pipeline
 
At a high level, this data pipeline has the following logical stages:

1.	Data is moved from a source (SQL Server) into the data warehouse. 
2.	A semantic model is applied to the data stored in the warehouse. 
3.	The semantic model is consumed by reporting/visualization tools 

The rest of this section describes the pipeline in more detail.
 
This reference architecture uses the WorldWideImporters sample database as the source of data. 

### Export data from SQL Server

Use the bcp (bulk copy program) utility to export data from SQL Server to flat files.

The bcp utility is a fast way to create flat files from SQL tables. In this step, you select the columns that you want to export, but do not otherwise transform the data. Any data transformations should happen in SQL Data Warehouse. 

If possible, schedule data extraction during off-peak hours, to minimize resource contention in the production environment. 

Avoid running bcp on the database server. Instead, run it from another machine. Write the files to a local drive. Ensure that you have sufficient I/O resources to handle the concurrent writes. For best performance, export the files to dedicated fast storage drives.

You can speed up the network transfer by saving the exported data in Gzip compressed format. However, loading compressed files into the warehouse is slower than loading uncompressed files. If you use Gzip compression, don't create a single Gzip file. Instead, split the data into multiple compressed files.

## Copy flat files into blob storage

Use the AzCopy utility to copy the files over the network into Azure blob storage. 
AzCopy utility is designed for high-performance copying of data to and from Azure Storage.

Create the storage account in a region that is closest to where the source data resides. Deploy the storage account and the SQL Data Warehouse instance in the same region. 

Don't run AzCopy on the same machine that runs your production workloads, because the CPU and I/O consumption can interfere with the production workload. 

Test the upload first to see what the upload speed is like. You can use the /NC option in AzCopy to specify the number of concurrent copy operations. Start with the default value, then experiment with this setting to tune the performance. In a low-bandwidth environment, too many concurrent operations can overwhelm the network connection and prevent the operations from fully completing.  

AzCopy moves data to storage over the public internet. If this isn't fast enough, consider setting up an ExpressRoute circuit. ExpressRoute is a service that routes your data through a dedicated private connection to Azure. Another option, if your network connection is too slow, is to physically ship the data on disk to an Azure datacenter. For more information, see Transferring data to and from Azure.

During a copy operation, AzCopy creates a temporary journal file, which enables AzCopy to restart the operation if it gets interrupted (for example, due to a network error). Make sure there is enough disk space to store the journal files. You can specify the location using the /Z option.

### Load data into SQL Data Warehouse

Use PolyBase to load the files from blob storage into the data warehouse. 

PolyBase is designed to leverage the MPP (Massively Parallel Processing) architecture of SQL Data Warehouse, so it is the fastest way to load data in SQL Data Warehouse. 

This stage has the following steps:

1.	Create a set of external tables for the data. An external table is a table that points to data stored outside of the warehouse -- in this case, the flat files in blob storage. This step does not move any data into the warehouse.
2.	Create staging tables, and use an INSERT INTO SELECT query to load the data from the external tables into the staging tables. This step copies the data into the warehouse.  
3.	Transform the data and move it into production tables. In this step, the data is transformed into a star schema with dimension tables and fact tables, suitable for semantic modeling.

The external tables and staging tables are temporary tables. You can delete them after the production tables are created. For more information, see Load data with PolyBase in SQL Data Warehouse.

The following queries show how to create an external table, create the corresponding staging table, and load the data into the staging table.

```sql
CREATE EXTERNAL TABLE [ext].[Application_PaymentMethods]
(
	[PaymentMethodID] [int] NOT NULL,
	[PaymentMethodName] [nvarchar](50) NOT NULL,
	[LastEditedBy] [int] NOT NULL,
	[ValidFrom] [datetime2](7) NOT NULL,
	[ValidTo] [datetime2](7) NOT NULL
)
WITH (DATA_SOURCE = [WAREHOUSEEXTERNALDATASOURCE],LOCATION = N'/WideWorldImporters_Application_PaymentMethods/',FILE_FORMAT = [UNCOMPRESSEDCSV],REJECT_TYPE = VALUE,REJECT_VALUE = 0)

CREATE TABLE stg.Application_PaymentMethods
(
	[PaymentMethodID] [int] NOT NULL,
	[PaymentMethodName] [nvarchar](50) NOT NULL,
	[LastEditedBy] [int] NOT NULL,
	[ValidFrom] [datetime2](7) NOT NULL,
	[ValidTo] [datetime2](7) NOT NULL
)
WITH (HEAP);

INSERT INTO stg.Application_PaymentMethods
SELECT
	[PaymentMethodID] ,
	[PaymentMethodName] ,
	[LastEditedBy] ,
	[ValidFrom] ,
	[ValidTo]
FROM ext.Application_PaymentMethods
```
Create the staging tables as heap tables, which are not indexed. Step 3 involves a full table scan, so there is no reason to index the staging tables.

Create the production tables with clustered columnstore indexes, which offer the best overall query performance. Columnstore indexes are optimized for queries that scan many records. Columnstore indexes don't perform as well for singleton lookups (that is, looking up a single row). If you need to perform frequent singleton lookups, you can add a non-clustered index to a table. Singleton lookups can run significantly faster using a non-clustered index. However, singleton lookups are less common in traditional data warehouse workloads than in OLTP systems.

PolyBase automatically takes advantage of parallelism in the warehouse. The load performance scales as you increase DWUs. For best performance, use a single load operation. There's no need to break up the input data into chunks and perform multiple concurrent loads.

PolyBase can read gzip compressed files. However, only a single reader is used per compressed file, because uncompressing the file is a single threaded operation. Therefore, avoid loading a single large compressed file. Instead, split the data into multiple compressed files, in order to take advantage of parallelism. 

Clustered columnstore tables do not support varchar(max), nvarchar(max), or varbinary(max) data types. In that case, consider heap or clustered index. You might put this data into a separate table.

In addition, PolyBase supports a maximum column size of varchar(8000), nvarchar(4000), and varbinary(8000). If you have data that exceeds these limits, one option is to break the data up into chunks when you export it, and then reassemble the chunks after import. Another option is to use bcp or bulk insert to load the files. However, this will be significantly slower than using PolyBase, because they don't take advantage of parallelism in the warehouse. 
  
### Load semantic model

Load the data into a tabular model in Azure Analysis Services.
In this step, you create a semantic data model by using SQL Server Data Tools (SSDT). You can also create a model by importing it from a Power BI Desktop file.
Because SQL Data Warehouse does not support foreign keys, you must add the relationships to the semantic model, so that you can join across tables.

Currently Azure Analysis Services supports tabular models but not multidimensional models. Tabular models use relational modeling constructs (tables and columns), whereas multidimensional models use OLAP modeling constructs (cubes, dimensions, and measures). If you require multidimensional models, use SQL Server Analysis Services (SSAS). For more information, see Comparing tabular and multidimensional solutions.

### Use PowerBI to visualize the data

When PowerBI connects to a data source, you can either import the data into PowerBI or use DirectQuery, which sends queries directly to the data source. When connecting to Azure Analysis Services, we recommend DirectQuery because is does not require copying data into the PowerBI model. Also, when using DirectQuery, results are always consistent with the latest source data. For more information, see Connect with Power BI.

Avoid running BI dashboard queries directly against the data warehouse. BI dashboards require very low response times, which direct queries against warehouse may not be able to meet. Also, refreshing the dashboard will count against the number of concurrent queries, which could impact performance. 

Azure Analysis Services is designed to handle the query requirements of a BI dashboard, so the recommended practice is to query Analysis Services from Power BI.

## Scalability Considerations

### SQL Data Warehouse

With SQL Data Warehouse, you can scale out your compute resources on demand. The query engine optimizes queries for parallel processing based on the number of compute nodes, and moves data between nodes as necessary. For more information, see Manage compute in Azure SQL Data Warehouse.

### Analysis Services

For production workloads, we recommend the Standard Tier, because it supports partitioning and DirectQuery. Within a tier, the instance size determines the memory and processing power. Processing power is measured in Query Processing Units (QPUs). Monitor your QPU usage to select the appropriate tier; see Monitor server metrics.

Under high load, query performance can become degraded due to query concurrency. You can scale out Analysis Services by creating a pool of replicas to process queries, so that more queries can be performed concurrently. The work of processing the data model always happens on the primary server. By default, the primary server also handles queries. Optionally, you can designate the primary server to run processing exclusively, so that all queries are run against the query pool.  If you have high processing requirements, you should separate the processing from the query pool. If you have high query loads, and relatively light processing, you can include the primary server in the query pool. For more information, see Azure Analysis Services scale-out. 

To reduce the amount of unnecessary processing, consider using partitions to divide the tabular model into logical parts. Each partition can be processed separately. For more information, see Partitions.

## Security Considerations

### IP whitelisting of Analysis Services clients

Currently Azure Analysis Services does support VNet service endpoints. Consider using the Analysis Services firewall feature to whitelist client IP addresses. If enabled, the firewall blocks all client connections other than those specified in the firewall rules. 

Note: The default rules whitelist the Power BI service, but you can disable this rule if desired.

For more information, see Hardening Azure Analysis Services with the new firewall capability.

### Authorization

Azure Analysis Services uses Azure Active Directory (Azure AD) to authenticate users who connect to an Analysis Services server. You can restrict what data a particular user is able to view, by creating roles and then assigning Azure AD users or groups to those roles. For each role, you can: 

- Hide tables or individual columns. 
- Hide individual rows based on filter expressions. 

## Deploy the solution

A deployment for this reference architecture is available on [GitHub][ref-arch-repo-folder]. It deploys the following:

  * A Windows virtual machine to simulate an on-premises database server. It includes SQL Server 2017 and related tools.
  * An Azure storage account that provides Blob storage to hold data exported from the SQL Server database.
  * An Azure SQL Data Warehouse instance.
  * An Azure Analysis Services instance.

### Prerequisites

1. Clone, fork, or download the zip file for the [Azure reference architectures][ref-arch-repo] GitHub repository.

2. Install the [Azure Building Blocks][azbb-wiki] (azbb).

3. From a command prompt, bash prompt, or PowerShell prompt, login to your Azure account by using the command below and following the instructions.

  ```bash
  az login  
  ```

### Deploy the simulated on-premises server

First you'll deploy a virtual machine as a simulated on-premises server, which includes SQL Server 2017 and related tools. This step also loads the sample [Wide World Importers OLTP database](/sql/sample/world-wide-importers/wide-world-importers-oltp-database) into SQL Server.

1. Navigate to the `data\enterprise-bi-sqldw\onprem\templates` folder of the repository you downloaded in the prerequisites above.

2. In the `onprem.parameters.json` file, replace the values for `adminUsername` and `adminPassword`. Also change the values in the `SqlUserCredentials` section to match the user name and password. Note the `.\\` prefix in the userName property.
    
    ```bash
    "SqlUserCredentials": {
      "userName": ".\\username",
      "password": "password"
    }
    ```

3. Run `azbb` as shown below to deploy the on-premises server.

    ```bash
    azbb -s <subscription_id> -g <resource_group_name> -l <location> -p onprem.parameters.json --deploy
    ```

4. The deployment may take 20 to 30 minutes to complete, which includes running the [Desired State Configuration](/powershell/dsc/overview) (DSC) to install the tools and restore the database. Verify the deployment in the Azure portal by reviewing the resources in the resource group. You should see the `sql-vm1` virtual machine and its associated resources.

### Deploy the Azure resources

This step provisions Azure SQL Data Warehouse and Azure Analysis Services, along with a Storage account. If you want, you can run this step in parallel with the previous step.

1. Navigate to the `data\enterprise-bi-sqldw\azure\templates` folder of the repository you downloaded in the prerequisites above.

2. Run the following Azure CLI command to create a resource group, replacing the bracketed parameters specified. Note that you can deploy to a different resource group than you used for the on-premises server in the previous step. 

    ```bash
    az group create --name <resource_group_name> --location <location>  
    ```

3. Run the following Azure CLI command to deploy the Azure resources, replacing the bracketed parameters specified. The `storageAccountName` parameter must follow the [naming rules](../../best-practices/naming-conventions.md#naming-rules-and-restrictions) for Storage accounts. For the `analysisServerAdmin` parameter, use your Azure Active Directory user principal name (UPN).

    ```bash
    az group deployment create --resource-group <resource_group_name> --template-file azure-resources-deploy.json --parameters "dwServerName"="<server_name>" "dwAdminLogin"="<admin_username>" "dwAdminPassword"="<password>" "storageAccountName"="<storage_account_name>" "analysisServerName"="<analysis_server_name>" "analysisServerAdmin"="user@contoso.com"
    ```

4. Verify the deployment in the Azure portal by reviewing the resources in the resource group. You should see a storage account, Azure SQL Data Warehouse instance, and Analysis Services instance.

5. Use the Azure portal to get the access key for the storage account. Select the storage account to open it. Under **Settings**, select **Access keys**. Copy the primary key value. You will use it in the next step.

### Export the source data to Azure Blob storage 

In this step, you will run a PowerShell script that uses bcp to export the SQL database to flat files on the VM, and then uses AzCopy to copy those files into Azure Blob Storage.

1. Use Remote Desktop to connect to the simulated on-premises VM you created previously.

2. While logged into the VM, run the following commands from a PowerShell window.  

    ```powershell
    cd 'C:\SampleDataFiles\reference-architectures\data\enterprise_bi_sqldw\onprem'

    .\Load_SourceData_To_Blob.ps1 -File .\sql_scripts\db_objects.txt -Destination 'https://<storage_account_name>.blob.core.windows.net/wwi' -StorageAccountKey '<storage_account_key>'
    ```

    For the `Destination` parameter, replace `<storage_account_name>` with the name the Storage account that you created previously. For the `StorageAccountKey` parameter, use the access key for that Storage account.

3. In the Azure portal, verify that the source data was copied to Blob storage by navigating to the storage account, selecting the Blob service, and opening the `wwi` container. You should see a list of tables prefaced with `WorldWideImporters_Application_*`.

### Execute the data warehouse scripts

1. From your Remote Desktop session, launch SQL Server Management Studio (SSMS). 

2. Connect to SQL Data Warehouse

    - Server type: Database Engine
    
    - Server name: `<dwServerName>.database.windows.net`, where `<dwServerName>` is the name that you specified when you deployed the Azure resources. You can get this name from the Azure portal.
    
    - Authentication: SQL Server Authentication. Use the credentials that you specified when you deployed the Azure resources, in the `dwAdminLogin` and `dwAdminPassword` parameters.

2. Navigate to the `C:\SampleDataFiles\reference-architectures\data\enterprise_bi_sqldw\azure\sqldw_scripts` folder on the VM. You will execute the scripts in this folder in numerical order, `STEP_1` through `STEP_7`.

3. Select the `master` database in SSMS and open the `STEP_1` script. Change the value of the password in the following line, then execute the script.

    ```sql
    CREATE LOGIN LoaderRC20 WITH PASSWORD = '<change this value>';
    ```

4. Select the `wwi` database in SSMS. Open the `STEP_2` script and execute the script. If you get an error, make sure you are running the script against the `wwi` database and not `master`.

5. Open a new connection to SQL Data Warehouse, using the `LoaderRC20` user and the password indicated in the `STEP_1` script.

6. Using this connection, open the `STEP_3` script. Set the following values in the script:

    - SECRET: Use the access key for your storage account.
    - LOCATION: Use the name of the storage account as follows: `wasbs://wwi@<storage_account_name>.blob.core.windows.net`.

7. Using the same connection, execute scripts `STEP_4` through `STEP_7` sequentially. Verify that each script completes successfully before running the next.

In SMSS, you should see a set of `prd.*` tables in the `wwi` database. To verify that the data was generated, run the following query: 

```sql
SELECT TOP 10 * FROM prd.CityDimensions
```

### Build the Azure Analysis Services model

In this step, you will create a tabular model that imports data from the data warehouse. Then you will deploy the model to Azure Analysis Services.

1. From your Remote Desktop session, launch SQL Server Data Tools 2015.

2. Select **File** > **New** > **Project**.

3. In the **New Project** dialog, under **Templates**, select  **Business Intelligence** > **Analysis Services** > **Analysis Services Tabular Project**. 

4. Name the project and click **OK**.

5. In the **Tabular model designer** dialog, select **Integrated workspace**  and set **Compatibility level** to `SQL Server 2017 / Azure Analysis Services (1400)`. Click **OK**.

6. In the **Tabular Model Explorer** window, right-click the project and select **Import from Data Source**.

7. Select **Azure SQL Data Warehouse** and click **Connect**.

8. For **Server**, enter fully qualified name of your Azure SQL Data Warehouse server. For **Database**, enter `wwi`. Click **OK**.

9. In the next dialog box, choose **Database** authentication and enter your Azure SQL Data Warehouse admin user name and password, and click **OK**.

10. In the **Navigator** dialog box, select the checkboxes for **prd.CityDimensions**, **prd.DateDimensions**, and **prd.SalesFact**. 

    ![](./images/analysis-services-import.png)

11. Click **Load**. When processing is complete, click **Close**. You should now see a tabular view of the data.

12. In the **Tabular Model Explorer** window, right-click the project and select **Model View** > **Diagram View**.

13. Drag the **[prd.SalesFact].[WWI City ID]** field to the **[prd.CityDimensions].[WWI City ID]** field to create a relationship.  

14. Drag the **[prd.SalesFact].[Invoice Date Key]** field to the **[prd.DateDimensions].[Date]** field.  
    ![](./images/analysis-services-relations.png)

15. From the **File** menu, choose **Save All**.  

16. In **Solution Explorer**, right-click the project and select **Properties**. 

17. Under **Server**, enter the URL of your Azure Analysis Services instance. You can get this value from the Azure Portal. In the portal, select the Analysis Services resource, click the Overview pane, and look for the **Server Name** property. It will be similar to `asazure://westus.asazure.windows.net/contoso`. Click **OK**.

    ![](./images/analysis-services-properties.png)

18. In **Solution Explorer**, right-click the project and select **Deploy**. Sign into Azure if prompted. When processing is complete, click **Close**.

19. In the Azure portal, view the details for your Azure Analysis Services instance. Verify that your model appears in the list of models.

    ![](./images/analysis-services-models.png)

### Analyze the data in Power BI Desktop

In this step, you will use Power BI to create a report from the data in Analysis Services.

1. From your Remote Desktop session, launch Power BI Desktop.

2. In the Welcome Scren, click **Get Data**.

3. Select **Azure** > **Azure Analysis Services database**. Click **Connect**

    ![](./images/power-bi-get-data.png)

4. Enter the URL of your Analysis Services instance, then click **OK**. Sign into Azure if prompted.

5. In the **Navigator** dialog, expand the tabular project that you deployed, select the model that you created, and click **OK**.

2. In the **Visualizations** pane, select the **Stacked Bar Chart** icon. In the Report view, resize the visualization to make it larger.

6. In the **Fields** pane, expand **prd.CityDimensions**.

7. Drag **prd.CityDimensions** > **WWI City ID** to the **Axis well**.

8. Drag **prd.CityDimensions** > **City** to the **Legend** well.

9. In the Fields pane, expand **prd.SalesFact**.

10. Drag **prd.SalesFact** > **Total Excluding Tax** to the **Value** well.

    ![](./images/power-bi-visualization.png)

11. Under **Visual Level Filters**, select **WWI City ID**.

12. Set the **Filter Type** to `Top N`, and set **Show Items** to `Top 10`.

13. Drag **prd.SalesFact** > **Total Excluding Tax** to the **By Value** well

    ![](./images/power-bi-visualization2.png)

14. Click **Apply Filter**. The visualization shows the top 10 total sales by city.

    ![](./images/power-bi-report.png)

To learn more about Power BI Desktop, see [Getting started with Power BI Desktop](/power-bi/desktop-getting-started).

## Next steps

- For more information about this reference architecture, visit our [GitHub repository][ref-arch-repo-folder].
- Learn about the [Azure Building Blocks][azbb-repo].

<!-- links -->

[azure-cli-2]: /azure/install-azure-cli
[azbb-repo]: https://github.com/mspnp/template-building-blocks
[azbb-wiki]: https://github.com/mspnp/template-building-blocks/wiki/Install-Azure-Building-Blocks
[github-folder]: https://github.com/mspnp/reference-architectures/tree/master/data/enterprise_bi_sqldw
[ref-arch-repo]: https://github.com/mspnp/reference-architectures
[ref-arch-repo-folder]: https://github.com/mspnp/reference-architectures/tree/master/data/enterprise_bi_sqldw
