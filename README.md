# ETL Pipeline Documentation

This document outlines the steps for an ETL (Extract, Transform, Load) process, including configuring FTP access, downloading files, loading them into a PostgreSQL database, and constructing a data pipeline in SSIS (SQL Server Integration Services) to join multiple tables.

## Configuring vsftpd for FTP Access

To ensure proper FTP functionality, you might need to adjust your `vsftpd` configuration. Add or modify the following parameters in your `vsftpd.conf` file:
- ```force_local_logins_ssl=NO```
- ```force_local_data_ssl=NO```
- ```require_ssl_reuse=NO```

**Note:** When working with FTP within WSL (Windows Subsystem for Linux), it's crucial to keep the WSL terminal open. This is because the FTP server running within WSL relies on the active WSL environment for network connectivity and process execution. Closing the WSL terminal will terminate the FTP server and interrupt file transfers.

## Downloading Files from FTP using SSIS

This section describes how to download files from an FTP server to your local machine using the FTP Task in SSIS.

1.  **Create a Local Folder:** In your Visual Studio project, create a dedicated folder to store the downloaded files (e.g., `Extracts`).

2.  **Rename SSIS Packages (if applicable):** If your SSIS package files have a `.dtsx` extension within a folder named "SSIS Packages," note their location.

3.  **Add an FTP Task:** In your SSIS Control Flow, drag and drop an "FTP Task" onto the design surface.

4.  **Create an FTP Connection:**
    * Navigate to the "Connection Managers" pane (usually at the bottom of the SSIS designer).
    * Right-click and select "New FTP Connection...".
    * Configure the FTP connection details (Server name, Port, Username, Password, etc.) and click "OK".

5.  **Configure the FTP Task Editor:**
    * Double-click the FTP Task to open its editor.
    * **Connection:** Select the FTP connection you just created.
    * **Name:** Provide a descriptive name for the task.
    * **OverwriteFileAtDest:** Set this property to `True` if you want to overwrite existing files at the destination.
    * **IsLocalPathVariable:** Set this property to `True` to use a variable for the local destination path.
    * **Operation:** Select "Receive files".

6.  **Define Variables for Source and Destination:**
    * Open the "Variables" window by navigating to "View" -> "Other Windows" -> "Variables".
    * **Create a variable for the destination folder:**
        * **Name:** `DestinationFolder`
        * **DataType:** `String`
        * **Value:** `C:\Users\Harvy Jones\source\repos\FtpToPostgre\Extracts` (Adjust this path to your local folder)
    * **Create a variable for the source file(s) on the FTP server:**
        * Open a new command prompt and run `wsl -u ftpuser`. This will likely put you in the `/home/ftpuser` directory on your FTP server within WSL.
        * **Name:** `SourceFile`
        * **DataType:** `String`
        * **Value:** `/home/ftpuser/OFAC_*.csv` (This pattern will download all `.csv` files starting with "OFAC_" in the specified FTP directory).

7.  **Assign Variables in the FTP Task Editor:**
    * In the FTP Task Editor:
        * For the "LocalPath" property, click the dropdown and select "<Variable...>". Choose the `DestinationFolder` variable.
        * For the "RemotePath" property, click the dropdown and select "<Variable...>". Choose the `SourceFile` variable.

8.  **Execute the Package:** Run your SSIS package to download the files from the FTP server to your local directory.

## Loading CSV Files to PostgreSQL Database using SSIS

This section details how to load the downloaded CSV files into a PostgreSQL database using the Data Flow Task in SSIS.

1.  **Add a Data Flow Task:** In your SSIS Control Flow, drag and drop a "Data Flow Task" onto the design surface.

2.  **Configure Source (Flat File Source):**
    * Inside the Data Flow Task, drag and drop a "Flat File Source" component.
    * Double-click the "Flat File Source" to open its editor.
    * **Create a Connection Manager:** Click "New..." next to the "Flat file connection manager" dropdown.
    * **Configure Flat File Connection Manager Editor:**
        * **Name:** `CSV OFAC SDN` (Provide a descriptive name).
        * **File name:** Browse to the location where you downloaded the CSV files (e.g., `C:\Users\Harvy Jones\source\repos\FtpToPostgre\Extracts\OFAC_*.csv`). You might need to select one specific file initially, and later adjust for multiple files if needed.
        * **Text qualifier:** Set this to `"`.
        * **Header row delimiter:** `{CR}{LF}` (usually the default).
        * **Column names in the first data row:** Check this box if your CSV files have headers.
        * **Format:** Delimited.
        * **Columns:** Review the detected columns and adjust data types as necessary.
        * **Preview:** Verify that the data is being read correctly.
        * Click "OK" on both editors.

3.  **Configure Destination (ODBC Destination):**
    * **Install PostgreSQL Driver (if not already installed):** Ensure you have the PostgreSQL ODBC driver installed on your machine. You can usually download it from the official PostgreSQL website.
    * **Open pgAdmin:** Launch pgAdmin, the administration tool for PostgreSQL.
    * **Open ODBC Data Source Administrator:** Search for "ODBC Data Sources (x64)" in the Windows search bar and open it.
    * **Configure ODBC Connection:**
        * Go to the "System DSN" tab and click "Add...".
        * Select the "PostgreSQL Unicode(x64)" driver and click "Finish".
        * Configure the data source name (e.g., `PostgreSQL_ETL`), Description (optional), Database, Server, Port, User Name, and Password for your PostgreSQL database.
        * Click "Test" to verify the connection and then click "OK" on all dialogs.

4.  **Add and Configure ODBC Destination in SSIS:**
    * In your Data Flow Task, drag and drop an "ODBC Destination" component.
    * Double-click the "ODBC Destination" to open its editor.
    * **Create a Connection Manager:** Click "New..." next to the "ODBC connection manager" dropdown.
    * Select the ODBC data source you configured in the ODBC Data Source Administrator (e.g., `PostgreSQL_ETL`) and click "OK".
    * **Data access mode:** Select "Table or view - fast load".
    * **Name of the table or the view:** Enter the name of the PostgreSQL table where you want to load the data (e.g., `your_target_table`).
    * **Mappings:** Click on the "Mappings" tab and ensure that all columns from your Flat File Source are correctly mapped to the corresponding columns in your PostgreSQL table. Adjust mappings as needed.
    * Click "OK".

5.  **Connect Source to Destination:** Drag the green arrow from the "Flat File Source" component to the "ODBC Destination" component.

6.  **Test the Data Flow Task:**
    * **Truncate the target table:** In pgAdmin or using a SQL query tool, execute a `TRUNCATE TABLE your_target_table;` command to clear any existing data.
    * **Select data:** Execute a `SELECT * FROM your_target_table;` to confirm the table is empty.
    * **Execute the Data Flow Task:** Right-click on the "Data Flow Task" in the SSIS designer and select "Execute Task".
    * **Verify:** After successful execution, query your PostgreSQL table again to ensure the data has been loaded correctly.

## Constructing the Data Pipeline in SSIS

This section describes how to build a more robust data pipeline by adding tasks for truncating the target table before loading.

1.  **Add "Execute SQL Task":**
    * Go back to the Control Flow.
    * Drag and drop an "Execute SQL Task" onto the design surface.
    * Connect the "Start" event to the "Execute SQL Task" using the green arrow.
    * Double-click the "Execute SQL Task" to open its editor.
    * **Connection:** Select the same ODBC connection manager you used for the PostgreSQL destination.
    * **SQLStatement:** Enter the SQL command to truncate your target table: `TRUNCATE TABLE your_target_table;`.
    * Click "OK".

2.  **Add "Sequence Container":**
    * Drag and drop a "Sequence Container" onto the Control Flow.
    * Drag the "Data Flow Task" (the one responsible for loading data) *inside* the "Sequence Container".
    * Connect the "Execute SQL Task" to the "Sequence Container" using the green arrow (On Success precedence constraint). This ensures the truncation happens before the data loading process.

    *(Optional: You can add more Data Flow Tasks inside the Sequence Container if you have multiple loading processes that should run sequentially or in parallel within this stage.)*

## Joining Multiple Tables in SSIS

This section outlines how to join data from multiple tables within your PostgreSQL database using a new Data Flow Task.

1.  **Add a New "Data Flow Task":** Go back to the Control Flow and add a new "Data Flow Task" to handle the table joining process. Connect the previous "Sequence Container" to this new Data Flow Task using an "On Success" precedence constraint.

2.  **Add ODBC Sources for Each Table:**
    * Inside the new Data Flow Task, drag and drop an "ODBC Source" component for each table you want to join.
    * Double-click each "ODBC Source" to open its editor.
    * Select the PostgreSQL ODBC connection manager.
    * **Data access mode:** Choose "SQL command".
    * **SQL command text:** Enter a `SELECT * FROM your_table_name` query for each source, selecting all necessary columns.
    * Click "OK" for each source.

3.  **Sort the Tables:**
    * For each "ODBC Source", drag and drop a "Sort" component and connect the output of the source to the input of the sort.
    * Double-click each "Sort" component.
    * Select the primary key column(s) of the corresponding table as the sort key(s). Ensure the "Sort type" is "Ascending" or "Descending" as needed.

4.  **Add "Merge Join" Transformations:**
    * Drag and drop a "Merge Join" component onto the Data Flow.
    * Connect the output of two "Sort" components to the two input pins of the "Merge Join".
    * Double-click the "Merge Join" to configure it:
        * **Join kind:** Choose the appropriate join type (e.g., Inner Join, Left Outer Join).
        * **Join keys:** Select the columns from each input that should be used for the join condition. Ensure the data types of the join keys are compatible.

5.  **Continue Merging (if more than two tables):** If you have more than two tables, add another "Merge Join" component. Connect the output of the previous "Merge Join" to one input, and the output of the next "Sort" component (from another table) to the other input. Configure the join as described in the previous step. Repeat this process until all tables are merged.

6.  **Create a Target Table in PostgreSQL:** Using pgAdmin or a SQL query tool, create a new table in your PostgreSQL database that will hold the results of the merged data. Ensure the table schema matches the columns resulting from your merge operations.

7.  **Add an ODBC Destination for the Merged Data:**
    * Drag and drop an "ODBC Destination" component onto the Data Flow.
    * Connect the output of the final "Merge Join" component to the input of the "ODBC Destination".
    * Double-click the "ODBC Destination" to configure it:
        * Select the PostgreSQL ODBC connection manager.
        * **Data access mode:** Select "Table or view - fast load".
        * **Name of the table or the view:** Enter the name of the target table you created for the merged data.
        * **Mappings:** Ensure all columns from the merged output are correctly mapped to the columns in your target table.
        * Click "OK".

## Deploying the Pipeline using SSMS and SQL Server

To deploy your SSIS package to SQL Server for scheduled execution or management:

1.  **Save your SSIS package:** Ensure your SSIS project and package are saved in Visual Studio.

2.  **Connect to SQL Server Integration Services:** Open SQL Server Management Studio (SSMS) and connect to your SQL Server Integration Services instance.

3.  **Import the package:**
    * In Object Explorer, navigate to "Integration Services Catalogs" -> "SSISDB" -> Your desired folder (or create a new one).
    * Right-click on the folder and select "Import Package...".
    * Choose the "File system" as the source and browse to the location of your `.dtsx` file.
    * Follow the wizard to specify the package name and location within the SSISDB catalog.

4.  **Configure Connections and Parameters (if necessary):** After importing, you might need to configure the connection managers within the deployed package to point to the correct server names, databases, usernames, and passwords in your production environment. You can do this through SSMS.

5.  **Create an SQL Server Agent Job (for scheduling):**
    * In SSMS, connect to your SQL Server Database Engine instance.
    * Navigate to "SQL Server Agent" -> "Jobs".
    * Right-click on "Jobs" and select "New Job...".
    * Provide a name for the job.
    * Go to the "Steps" page and click "New...".
    * **Step name:** Provide a name for the step (e.g., "Run ETL Package").
    * **Type:** Select "SQL Server Integration Services Package".
    * **Server:** Select your Integration Services server.
    * **Package source:** Choose "SSIS Catalog".
    * Browse to and select the SSIS package you deployed.
    * Configure any necessary parameters or connection overrides.
    * Click "OK".
    * Go to the "Schedules" page to define when the job should run (e.g., daily, weekly).
    * Click "OK" to create the job.
