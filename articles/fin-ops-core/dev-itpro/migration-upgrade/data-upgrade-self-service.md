---
# required metadata

title: Upgrade from AX 2012 - Data upgrade in self-service environments
description: This topic explains how to do a data upgrade from Microsoft Dynamics AX 2012 in self-service environments.
author: sarvanisathish
ms.date: 06/29/2021
ms.topic: article
audience: IT Pro
ms.reviewer: sericks
ms.search.region: Global
ms.author: sarvanis
ms.search.validFrom: 2021-06-30

---

# Upgrade from AX 2012 - Data upgrade in self-service environments

[!include[banner](../includes/banner.md)]

> [!IMPORTANT]
> Some or all of the functionality noted in this topic is available as part of a preview release. The content and the functionality are subject to change.

This Microsoft Dynamics AX 2012 data upgrade process is for self-service environments. Complete the sections of this topic in the following order:

1. **[Prerequisites](data-upgrade-self-service.md#prerequisites)**
2. **[Data upgrade process](data-upgrade-self-service.md#data-upgrade-process)** – Run the AX2012DataUpgradeToolKit.exe application to complete the upgrade process.
3. **[Reporting section of the application](data-upgrade-self-service.md#reporting-section-of-the-application)** – Review the reports of the replication validation, replication status, and data upgrade status.
4. **[Tooling section of the application](data-upgrade-self-service.md#tooling-section-of-the-application)**  – This section will help you reset the process parameters and restart any of the processes.

## Prerequisites

1. Download the AX 2012 Database Upgrade Toolkit for Dynamics 365 from Microsoft Dynamics Lifecycle Services (LCS). In the Shared asset Library, select **Model** as the asset type, and then select the model file.
2. Create a self-service environment in LCS. The environment should be in a **Deployed** state.
3. Download and install the [.NET Framework version 4.7.1](https://dotnet.microsoft.com/download/dotnet-framework/net471) if it isn't already installed.
4. Make sure that the replication feature is installed and enabled for the source SQL Server instance. To determine whether replication is enabled, run the following SQL script.

    ```sql
    -- If @installed is 0, replication must be added to the SQL Server installation.

    USE master;

    GO

    DECLARE @installed int;

    EXEC @installed = sys.sp_MS_replication_installed;

    SELECT @installed;
    ```

    If the replication components aren't installed, follow the steps in [Install SQL Server replication](/sql/database-engine/install-windows/install-sql-server-replication?view=sql-server-ver15) to install them.

5. Enable and start the SQL Server Agent on the source database server.

    > [!NOTE]
    > A user should have the **DB\_Owner** privilege in the source database, and should have access to the master database and the source database.

6. **Migration toolkit setup:** If you don't want some of the source database tables to be replicated in the target database, you can specify them in the IgnoreTables.xml file. Likewise, if you don't want some of the functions to be replicated, you can specify them in the IgnoreFunctions.xml file.

    - **Path of the IgnoreTables.xml file:** Data\\IgnoreTables.xml
    - **Path of the IgnoreFunctions.xml file:** Data\\IgnoreFunctions.xml

    The following examples show how to specify tables and functions in the XML files.

    ```xml
    <?xml version="1.0" encoding="utf-8"?>
    <IgnoreTables>
        <Name>
            <Table>USERADDHISTORYLIST</Table>
            <Table>TAXRECONCILIATIONREPORTTMP</Table>
            <Table>CASELOG</Table>
            <Table>SHAREDCATEGORYROLETYPE</Table>
            <Table>VATCSREPORTXMLATTRIBUTE_CZ</Table>
        </Name>
    </IgnoreTables>
    ```

    ```xml
    <?xml version="1.0" encoding="utf-8"?>
    <IgnoreFunctions>
        <Name>
            <Function>if_WHSInventReserveUnionDelta</Function>
        </Name>
    </IgnoreFunctions>
    ```

    > [!WARNING]
    > The tables and functions that are specified in these XML files won't be replicated in the target database, and the same format should be followed.

## Data upgrade process

Before you begin the replication process, note that the LCS environment will be in a **Deployed** state when it's created.

1. Run the **AX2012DataMigration.exe** application.

    A console window is opened. Use the Microsoft sign-in page for authentication.

2. Provide the credentials that are used to sign in to LCS.

3. After you're successfully authenticated, in the console window, provide the **Project-Id** value and then the **Environment-Id** value.

    > [!NOTE]
    > You can find the **Project-Id** and **Environment-Id** values on the **Manage environment** page in LCS. You can also find the **Environment-Id** value on the **Environment details** page.

    After the validation is successful, the application presents a set of menu options that correspond to the steps in the data upgrade process. To complete the data replication and upgrade, you should perform the steps in the following order.

4. **Data upgrade preparation: Environment setup activity**

    This step prompts you for the following information:

    - Details of the source database:

        - Source server (in the format *servername\\serverinstance*)
        - Source database name
        - User name
        - Password

    - IP address of the source database server (for the allowlist)
    - Distribution database path (for example, **D:\\SQLServer\\Data**)
    - Replication snapshot path (for example, **D:\\SQLServer\\Snapshot**)

    > [!WARNING]
    > The specified distribution database and replication snapshot paths should have enough space. We recommend that the amount of space be at least the size of the source database.

    This step performs the following actions:

    - Validate the connection to the source database.
    - Validate the version of the AX 2012 database.
    - Authorize the source IP address.
    - Validate the target databases.

5. **Data upgrade preparation: Prepare the target environment for the data upgrade**

    This step changes the state of the LCS environment from **Deployed** to **Ready for replication**.

6. **Replication: Clean-up target database**

    This step performs the following actions:

    1. Change the state of the LCS environment from **Ready for replication** to **Replication in progress**.
    2. Delete all AX product tables, views, stored procedures, and user-defined functions in the target database.

7. **Replication: Set up distributor**

    This step creates a distribution database under the **System Databases** folder on the source server. This distribution database is used for replication.

8. **Replication: Set up publication for primary key tables**

    This step creates publications for primary key tables under the **Replication** folder on the source server and replicates them in the target database. If  any **ignore-table** entries are specified, the specified tables are exempted from replication.

    **Created publishers:** AXDB\_PUB\_TABLE\_Obj\_\[\*\]

    > [!NOTE]
    > After this replication configuration step is completed, actual data replication will occur as a SQL job that runs in the background. This job will take some time to be completed. You can view the status of the replication by providing the **'rs'** option. To learn more about the **'rs'** option, see the [Reporting section of the application](data-upgrade-self-service.md#reporting-section-of-the-application) section later in this topic.

9. **Replication: Set up publication for other objects (functions)**

    This step creates a publication for other objects (functions) and replicates them in the target database. If you don't want some of the functions to be replicated, you can specify them in the IgnoreFunctions.xml file.

    **Created publisher:** AX\_PUB\_OtherObjects

    > [!NOTE]
    > The replication will take some time to be completed. You can view the replication status by providing the **'rs'** option.
    >
    > If there no functions to replicate, the publication won't be created.

    > [!WARNING]
    > Don't move on to next step until the **DataReplicationStatus** property for this step is shown as completed.

10. **Cutover: Set up publication for non-primary key tables**

    This step creates two publications: one that is used to replicate non-primary key tables, and one that is used to replicate locked tables.

    **Publication names:** AX\_PUB\_NoPKTable, AX\_PUB\_TABLE\_LockedTable

    If AX Service acquires a schema lock during creation of the primary key publication, those tables will be ignored and omitted from the publication. They will be added to temporary tables and marked for replication during creation of the cutover publication.

    > [!WARNING]
    > Don't move on to next step until the **DataReplicationStatus** property for this step is shown as completed.

11. **Cutover: Remove non-primary key publication and temporary tables**

    This step performs the following actions:

    1. Clean up the temporary tables that were created for non-primary key tables in the source database.
    2. Delete the **AX\_PUB\_NoPKTable** publication.

12. **Cutover: Create constraint for non-primary key tables**

    This step extracts constraints for the non-primary key tables from the source database and creates them in the target database.

13. **Cutover: Remove replication setup**

    This step deletes all the publications that were created in the source database, the distribution database, and the replication snapshot.

    > [!NOTE]
    > You can validate the replicated data by using the **'dv'** option. To learn more about this option, see the [Reporting section of the application](data-upgrade-self-service.md#reporting-section-of-the-application) section later in this topic.
    >
    > To remove the **Snapshot** folder without causing an exception, run the following script in the source database. Even if you don't run this script, you can ignore the exception message that you receive.
    >
    > ```sql
    > EXEC master.dbo.sp_configure 'show advanced options', 1
    >
    > RECONFIGURE WITH OVERRIDE
    > 
    > EXEC master.dbo.sp_configure 'xp_cmdshell', 1
    > 
    > RECONFIGURE WITH OVERRIDE
    > ```

14. **Post-replication: Update environment state to Replicated**

    This step changes the state of the LCS environment from **Replication in progress** to **Replication completed**.

15. **Data Upgrade: Trigger upgrade**

    This step triggers the data upgrade. When the action is successful, the state of the LCS environment changes from **Replication completed** to **Data upgrade in progress**.

    At this point, only the data upgrade trigger occurs. The actual data upgrade occurs in the self-service environment. To learn the status of the data upgrade, use the **'ds'** option. To learn more about this option, see the [Reporting section of the application](data-upgrade-self-service.md#reporting-section-of-the-application) section later in this topic.

    If data upgrade is successful, the **'ds'** option is shown as **AX 2012 upgrade topology (LCS) status: Deployed**, and all the upgrade steps will be in a **Completed** state.

    If data upgrade fails, the **'ds'** option is shown as **AX 2012 upgrade topology (LCS) status: Failed**, and one or more upgrade steps will be in a **Failed** state. The **Menu option (12)** tool will show a status of **Resume**.

    After you address and fix the reasons for the failure, you can perform the **Resume** operation. When the action is successful, the state of the LCS environment will change from **Failed** to **Data upgrade in progress**.

    > [!NOTE]
    > Repeat this step until the data upgrade is successful.

## Reporting section of the application

You can use the following options to review the reports of the replication validation, replication status, and data upgrade status:

- **dv) Report:** Validate the replication.

    This option compares the number of tables and records in the source server database and the target server database, and then shows the report. You should use this option only after step 12 is completed.

    You can find the report data at **output/PostValidationInfo.csv**.

- **rs) Report:** Get the replication status.

    This option shows the report of the replication process for the publications that were created. You should use this option only after step 3 is started (that is, during the replication process for any publication).

- **ds) Report:** Get the data upgrade status.

    This option shows the report of the data upgrade process. You should use this option only after step 12 is started.

## Tooling section of the application

- **Reset:** Reset the replication setup by removing all the replication configurations. Publications and the distribution database are deleted. The status of all **Replication** and **Cutover** menu options is reset from **Completed** to **Reset** mode to help you redo the replication from the beginning.

- **Reset-all:** Reset all the menu options.

    **Reset** option and **Clear** option will be performed. All the options will be changed to **Not Started**.

- **Clear:** Clear the environment setup activity. All information is cleared from the cache, such as the **project-Id** value, **Environment-Id** value, and source database details. The status of step 1 is changed to **Not Started**.
- **Help:** Show the data upgrade migration options with the updated status.
- **Exit:** Close the application.

## Learn about the replication configuration and status via SQL Server Management Studio

In SQL Server Management Studio (SSMS), if Object Explorer includes a **Replication** folder, the replication feature is installed on the server and available.

After step 3 of the data upgrade process is completed, you should find the publisher configured under the **Replication** folder. To learn the replication status, select and hold (or right-click) the **Replication** folder, and then select **Launch Replication Monitor**.

- In the replication monitor, you can view all the publishers that have been created for replication.
- On the **Snapshot** tab, you can view the status of the snapshot.
- To view the detail log/transaction, double-tap (or double-click) a grid item.
- To view the data replication to the target, on the **All Subscription** tab, double-tap (or double-click) the subscription from the grid item.
