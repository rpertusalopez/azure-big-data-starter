# Getting Started

This project aims to simplify Azure Big Data environment setup. Involved Azure PaaS services require different development and deployment steps, and this initiative is a set of suggestions for improving the overall development experience. We use car telemetry data as an example. 

You can find more information about Big Data architectural style in Azure [here.](https://docs.microsoft.com/en-us/azure/architecture/guide/architecture-styles/big-data)

Azure offers many different PaaS services for stream and batch workloads. Since HDInsight offerings require extra effort and don't provide multitenancy, this project only focuses on full scope PaaS. Below you find the architecture and involved services. Green dots mark currently utilized services. 


![Silvrback blog image sb_float_center](https://silvrback.s3.amazonaws.com/uploads/63050ce6-d2f8-4e8c-a610-5184abdaac73/Azure%20Big%20Data%20(1).png)


#### Structure

* 0-Deployment - *PowerShell scripts*
* 1-Resources - *ARM templates and csv telemetry data files*
* 2-SqlServer - *Incremental database updates for Azure SQL Data Warehouse*
* 3-DataLakeAnalytics - *Empty*
* 4-DataFactory - *Empty*
* 5-MachineLearning - *Empty*
* 6-StreamAnalytics  - *WIP. Currently Stream Analytics objects are located in 1-Resources*
* 7-WebApp - *SignalR + Google Map dashboard*
* 8-EventApps - *Sending telemetry data to Event Hubs*

#### Prerequisites
* [Azure Subscription](https://azure.microsoft.com/en-us/free/)
* [Visual Studio 2015](https://www.visualstudio.com/vs/older-downloads/)
* [Azure PowerShell 2](https://docs.microsoft.com/en-us/powershell/azure/overview?view=azurermps-4.4.0)
* [Azure Stream Analytics for VS 2015](https://docs.microsoft.com/en-us/azure/stream-analytics/stream-analytics-tools-for-visual-studio)
* [Azure Data Factory for VS2015](https://marketplace.visualstudio.com/items?itemName=AzureDataFactory.MicrosoftAzureDataFactoryToolsforVisualStudio2015)

---

# Deployment

**1** Find **.sample* config files (5), make a copy without **.sample* extension ([dealing with passwords in GitHub](https://stackoverflow.com/questions/2397822/what-is-the-best-practice-for-dealing-with-passwords-in-github)). 

![Silvrback blog image sb_float_center](https://silvrback.s3.amazonaws.com/uploads/7c2cf059-cdda-4a2a-b53a-03ca2a28930b/1-Resources.jpg)

**2** Populate *1-Resources/azuredeploy.parameters.json* with your values (use only lower letters for service names and be creative since some require globally unique names).

![Silvrback blog image sb_float_center](https://silvrback.s3.amazonaws.com/uploads/809f97d3-b069-480e-b916-9b69826f61bf/azure-big-data-starter%20-%20Microsoft%20Visual%20Studio_2_medium.jpg)

**3** Run *0-Deployment\0-Initial.ps1* 
- Creates Azure Resources
- Populates local config files afterward
- Uploads reference data for data enrichment

**4** Go to Azure portal, SQL Server Firewall and add your IP to Azure SQL database.

![Silvrback blog image sb_float_center](https://silvrback.s3.amazonaws.com/uploads/b7369b26-9dc2-4cbf-b24b-f4e10bbdf0e7/Firewall%20settings%20-%20Microsoft%20Azure%20-%20Google%20Chrome.jpg)

**5** Build and run *2-SqlServer\SqlDWDbUp* project to deploy database objects.

![Silvrback blog image sb_float_center](https://silvrback.s3.amazonaws.com/uploads/c6c2dff4-f1b8-447c-8114-c2456df52baf/fileCCodeGitazure-big-data-starter2-SqlServerSqlDWDbUpbinDebugDataWarehouseDbUp.EXE.jpg)
 
---

# Execution
**1** Go to Azure portal and run Azure Stream Analytics.

![Silvrback blog image sb_float_center](https://silvrback.s3.amazonaws.com/uploads/3ff6cce3-8361-4e16-8ab0-75e88b678515/ridesasajob%20-%20Microsoft%20Azure%20-%20Google%20Chrome.jpg)

**2** Run locally  *7-WebApp\DashboardWebApp* to open a dashboard (refresh multiple times if Google map is not visible).

**3** Run locally *8-EventApps\CarEventsSenderApp* to send events to event hub.

![Silvrback blog image sb_float_center](https://silvrback.s3.amazonaws.com/uploads/780d84dd-0efe-45a7-809f-64985e1ad324/fileCCodeGitazure-big-data-starter8-EventAppsCarEventsSenderAppbinDebugazure-patterns-big-data.EXE.jpg)

**4** Connect to [SQL Data Warehouse](https://docs.microsoft.com/en-us/azure/sql-data-warehouse/sql-data-warehouse-query-visual-studio) and query *dbo.CarHealthStatus*

![Silvrback blog image sb_float_center](https://silvrback.s3.amazonaws.com/uploads/3846e1d4-b095-47aa-88af-08da9e53da62/azure-big-data-starter%20(Running)%20-%20Microsoft%20Visual%20Studio.jpg)

Azure Stream Analytics query that sends data to SQL Data Warehouse:

```sql
--Map into 10-second tumbling windows and send to SQL Warehouse

SELECT 
	Input.carid, 
	System.Timestamp as deviceTime,
	AVG(Input.SpeedOdb) as speedOdb
INTO SqlOutput 
FROM EventHubInput AS Input
GROUP BY Input.carid, TumblingWindow(SECOND, 10);
```

**5** Monitor dashboard web app

![Silvrback blog image sb_float_center](https://silvrback.s3.amazonaws.com/uploads/d7a731c5-3b29-4c71-9d27-e8fc605a40c2/localhost11282%20-%20Google%20Chrome.jpg)

Azure Stream Analytics query that sends data to Service Bus and eventually to the web app:

```sql
--Join data with reference blob file and map into 5-second tumbling windows.

SELECT
    Ref.Make,
    System.Timestamp as deviceTime,
    AVG(Input.Longitude) as longitude,
    AVG(Input.Latitude) as latitude,
    AVG(Input.EngineRpm) as engineRpm,
    AVG(Input.SpeedOdb) as speedOdb
INTO ServiceBusOutput 
FROM EventHubInput AS Input
INNER JOIN [ReferenceInput] AS Ref
ON Input.carid = Ref.carid
GROUP BY Ref.Make, TumblingWindow(SECOND, 5);
```

---

# Limitations
* Globally unique names are required for some services (in case of deployment failure due to already occupied name, change the name and execute the deployment script again).
* Stream Analytics projects cannot be under folders in a solution, due to some issues with the VS component. Due to this limitation Stream Analytics hangs at the root level.
* ARM template for Stream Analytics doesn't support Data Lake store and PowerBI as outputs. 
* Data Lake Store and required Azure AD App.
 
---

# To Do
* Azure ASA inputs, outputs, and the query deploys from 1-Resources\azuredeploy.json. Use 6-StreamAnalytics project* Figure out how to deal with sensitive info in ASA config files and GIT.
* Start ASA after deployment. 
* Deploy Dashboard web app to Azure (Issues with app.config).
* Introduce batch workload with Data Lake Analytics 
* Add orchestration using Azure Data Factory (v2?)
* Add an anomaly detection experiment using Machine Learning (v2?)

