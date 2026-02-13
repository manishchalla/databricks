# 📘 Databricks Zero to Hero: Master Notes

**Scope:** Core Architecture, Unity Catalog, and Data Engineering Patterns.
**Focus:** Technical implementation and Data Management (No Admin/Setup).

---

## 🏗️ 1. High-Level Architecture
**Concept:** Databricks uses a **Split-Plane Architecture** to ensure security and scalability.

### **The Control Plane (The Brain)**
* Managed entirely by Databricks in their cloud account.
* **Responsibilities:**
    * Hosting the Web UI (the website you log into).
    * Managing Notebooks and User accounts.
    * Scheduling Jobs and Workflows.
* *Note:* Your actual data **never** enters the Control Plane.

### **The Data Plane (The Muscle)**
* Lives in **Your Cloud Account** (e.g., your Azure Subscription).
* **Responsibilities:**
    * **Compute:** Runs the actual Virtual Machines (Clusters) that process data.
    * **Storage:** Connects to your Blob Storage/ADLS Gen2 where the data files sit.
* *Key Takeaway:* Databricks processes data "in place" within your security boundary.

---

## 💧 11. Next-Gen Optimization: Liquid Clustering
**Concept:** A new dynamic data layout that replaces standard Partitioning and Z-Ordering.

### **Liquid Clustering**
* **The Problem with Partitioning:** You have to pick physical columns (e.g., Year/Month). If you pick the wrong one, you get "Small File" issues. Changing it requires rewriting the whole table.
* **The Solution:** Liquid Clustering automatically clusters data based on usage patterns without creating rigid physical folders.
* **Benefits:**
    * Solves the "Small File Problem" automatically.
    * Adapts to uneven data (skew).
    * **Replaces** both Partitioning and Z-Ordering.

**Command:**
```sql
CREATE TABLE sales (id INT, city STRING)
CLUSTER BY (city); -- No 'PARTITIONED BY' needed!


---

## 📓 2. Notebooks & Magic Commands
**Concept:** Interactive coding environment that supports multiple languages in the same file.

| Command | Usage | Example |
| :--- | :--- | :--- |
| `%python` | Default for Data Engineering logic (PySpark). | `df = spark.read.csv(...)` |
| `%sql` | SQL queries on tables. | `SELECT * FROM sales` |
| `%md` | Documentation (Markdown). | `# Title` |
| `%fs` | Interacting with the File System (DBFS). | `%fs ls /databricks-datasets` |
| `%run` | Executing another notebook. | `%run ./setup/config` |

---

## ☁️ 3. Storage & Locations
**Concept:** Connecting Databricks to Azure Storage (ADLS Gen2).

* **Legacy Way (Mounting):** Insecure. Everyone on the cluster has access to the mount point.
* **Modern Way (Unity Catalog):**
    * **External Location:** A secure object that combines a **Storage Path** (URL) with a **Storage Credential** (Managed Identity).
    * **Hierarchy:** You can assign locations at the **Metastore**, **Catalog**, or **Schema** level. Managed tables inherit this location automatically.

---

## 🔐 4. Unity Catalog & Namespaces
**Concept:** Centralized Governance and the 3-Level Namespace.

* **The Old Way (Hive Metastore):** `schema.table`
* **The New Way (Unity Catalog):** `catalog.schema.table`

### **Object Hierarchy**
1.  **Metastore:** Top level container (Region-specific).
2.  **Catalog:** Highest grouping (e.g., `prod_catalog`, `dev_catalog`).
3.  **Schema:** Logical grouping of tables (e.g., `sales_db`).
4.  **Table:** The actual data.

```sql
-- Selecting data using the 3-level namespace
SELECT * FROM prod_catalog.sales_db.orders;
```

---

## 🗄️ 5. Managed vs. External Tables
**Concept:** The "Lifecycle" of data ownership. **(Crucial Interview Topic)**

### **Comparison Table**

| Feature | **Managed Table** | **External Table** |
| :--- | :--- | :--- |
| **Creation** | `CREATE TABLE xyz...` (No location specified) | `CREATE TABLE xyz LOCATION '...'` |
| **Ownership** | Databricks manages Metadata + Data. | Databricks manages Metadata ONLY. |
| **Storage** | Stored in the Default Catalog/Schema path. | Stored in your custom Azure container. |
| **Dropping** | **Deletes Data AND Metadata.** | **Deletes Metadata ONLY.** (Files stay safe). |

### **Code Example**

```sql
-- Managed Table (Data deleted if table dropped)
CREATE TABLE managed_sales (id INT, amount INT);

-- External Table (Data safe if table dropped)
CREATE TABLE external_sales (id INT, amount INT)
LOCATION 'abfss://my-container@storage.dfs.core.windows.net/data/sales';
```

---

## 📸 6. Views & Clones
**Concept:** Virtual tables and Table Copying strategies.

### **Views**
* **Temp View:** `CREATE TEMP VIEW`. Valid for **1 Session** only. Disappears when cluster restarts.
* **Permanent View:** `CREATE VIEW`. Stored in Catalog. Valid for **Everyone**.

### **Clones (Deep vs Shallow)**

| Feature | Deep Clone | Shallow Clone |
| :--- | :--- | :--- |
| **Data Copy** | Copies **Data + Metadata**. | Copies **Metadata Only**. |
| **Independence** | Independent backup. | Dependent on original files. |
| **Cost/Speed** | Slow & costs storage. | Instant & free. |
| **Use Case** | Archiving/DR. | Testing scripts/Blue-Green Deploy. |

---

## 🛡️ 7. The Safety Net: UNDROP
**Concept:** Recovering from accidental deletion.

Unlike the Legacy Metastore, **Managed Tables** in Unity Catalog have a safety net.
* **Window:** 7 Days.
* **Command:**

```sql
UNDROP TABLE critical_data;
```
* **Note:** Does not work for External Tables.

---

## 🔄 8. MERGE & Upserts (SCD Type 1)
**Concept:** Handling Incremental Data, Updates, and Soft Deletes.

**Scenario:** We want to update a user's status based on a daily feed.
1.  If they are new -> **INSERT**.
2.  If they exist -> **UPDATE** details.
3.  If source says "DELETE" -> **SOFT DELETE** (Mark inactive).

### **The Master MERGE Script**

```sql
MERGE INTO target_table AS target
USING source_data AS source
ON target.id = source.id

-- Soft Delete Logic (If source says delete, mark inactive)
WHEN MATCHED AND source.operation = 'DELETE' THEN
  UPDATE SET target.is_active = false

-- Update Logic (SCD Type 1 - Overwrite old values)
WHEN MATCHED AND target.value <> source.value THEN
  UPDATE SET target.value = source.value

-- Insert Logic (New records)
WHEN NOT MATCHED THEN
  INSERT (id, value, is_active) VALUES (source.id, source.value, true);
```


---

---

## 🚀 9. Optimization Essentials (Video 16 Prerequisites)
**Concept:** Before diving into Delta Lake internals, you must understand how Spark reads data efficiently.

### **1. Partitioning (The "Folder" Strategy)**
* **What is it?** Breaking a large table into sub-folders based on a column (e.g., `year=2023/month=01`).
* **Benefit:** **Partition Pruning**. If you query `WHERE year = 2023`, Spark reads *only* that folder and ignores the rest.
* **Best Practice:** Only partition columns frequently used in filters and with low cardinality (e.g., Date, Country). Do **not** partition by unique IDs.

### **2. Data Skipping & Z-Ordering**
* **Data Skipping:** Delta Lake automatically stores **Min/Max statistics** for every column in every file. If your query looks for `id = 50` and a file's range is `100-200`, Spark skips that file entirely.
* **Z-Ordering:** A technique to co-locate related data. It sorts data by multiple columns so that similar data points sit in the same files.
    * **Command:** `OPTIMIZE table_name ZORDER BY (col1, col2)`
    * **Result:** Dramatically improves Data Skipping.



### **3. Compaction (Small File Problem)**
* **The Problem:** In Big Data, opening 1,000 tiny files (1KB each) is much slower than opening 1 large file (1GB).
* **The Solution:** The `OPTIMIZE` command (Bin-packing).
* **Command:** `OPTIMIZE table_name`
    * This merges small files into larger, efficient files without stopping reads/writes.

---

## 🕰️ 10. Deep Dive: Delta Lake Internals
**Concept:** How Databricks achieves ACID properties, Time Travel, and Scalability using the `_delta_log`.

### **1. The Transaction Log (`_delta_log`)**
The "Brain" of the Delta Table. It is a folder stored at the root of your table directory.
* **JSON Files (Commits):** Every single action (Insert, Merge, Delete) creates a specific JSON file (e.g., `000000.json`, `000001.json`). These record **Metadata**, not data.
* **Checkpoint Files (.parquet):** Every **10 Commits**, Databricks creates a summary file. Spark reads this instead of processing thousands of tiny JSONs.



### **2. The "Add/Remove" Protocol (ACID)**
Delta Lake files are **Immutable**. When you update a row:
1.  Delta reads the original file (File A).
2.  It writes a **New File** (File B) with the change.
3.  It records in the Log: `remove(File A)` and `add(File B)`.
* **Result:** File A still exists physically (allowing Time Travel), but the "Current Version" ignores it.

### **3. Deletion Vectors (Optimization)**
**Concept:** A feature to speed up Updates/Deletes on large files.
* **The Old Way (Copy-on-Write):** If you change 1 row in a 1GB file, Delta must rewrite the entire 1GB file. This is slow ("Write Amplification").
* **The New Way (Deletion Vectors):**
    * Delta creates a tiny **Bitmap File** linked to the original file.
    * This bitmap simply marks specific rows as "Deleted".
    * **Benefit:** Instant writes because no massive data copy happens. The file is only rewritten later during `OPTIMIZE`.



### **4. Time Travel & Restore**
Query past versions using the log history.

```sql
-- Query History
SELECT * FROM my_table VERSION AS OF 5;

-- Restore (Undo Mistake)
RESTORE TABLE my_table TO VERSION AS OF 3;
```

### **5. Maintenance: VACUUM**
Databricks keeps old/deleted files physically in storage to enable Time Travel. To save costs, you must remove them eventually.

* **Concept:** Permanently deletes files that are no longer in the latest state of the table and are older than the retention period.
* **Command:** `VACUUM table_name [RETAIN num HOURS]`
* **Default Retention:** 7 Days (168 Hours).
* **Safety Check:** Databricks prevents you from setting retention < 168 hours to prevent accidental data loss. You must disable this check to vacuum aggressively.

> **⚠️ CRITICAL WARNING:** Once you run VACUUM, the physical files are gone forever. You **cannot** Time Travel back to versions older than the retention limit.

```sql
-- Standard cleanup (Safe - Defaults to 7 days)
VACUUM employee_table;

-- Aggressive cleanup (Removes ALL history instantly)
-- 1. Turn off the safety check
SET spark.databricks.delta.retentionDurationCheck.enabled = false;
-- 2. Run Vacuum with 0 hours retention
VACUUM employee_table RETAIN 0 HOURS;

```
---

## 📦 12. Unity Catalog: Tables & Volumes
**Concept:** How Unity Catalog manages Structured Data (Tables) vs. Unstructured Data (Volumes).

### **1. Managed vs. External Tables (Recap for UC)**
In Unity Catalog, the distinction determines **who deletes the data**.

* **Managed Table:**
    * **Data Location:** Stored in the root storage of the Metastore or Catalog.
    * **Creation:** `CREATE TABLE my_table ...` (No path needed).
    * **Deletion:** `DROP TABLE` deletes **Metadata AND Data**.
* **External Table:**
    * **Data Location:** Stored in your specific Cloud Storage path (ADLS/S3).
    * **Creation:** `CREATE TABLE my_table ... LOCATION 's3://...'`
    * **Deletion:** `DROP TABLE` deletes **Metadata ONLY**. Data remains safe.

### **2. Volumes (New Feature)**
**Concept:** A Unity Catalog object used to manage **Non-Tabular Data** (PDFs, Images, CSVs, JSONs, ML Models).

* **Why use Volumes?**
    * Tables are for rows and columns.
    * Volumes are for files. They replace the old "Mount Points" (`/dbfs/mnt/...`).
* **Path Structure:**
    * Access files just like a local folder:
    * `/Volumes/<catalog_name>/<schema_name>/<volume_name>/file.csv`

### **3. Managed vs. External Volumes**
Just like tables, Volumes come in two flavors:

* **Managed Volume:**
    * Created in the default storage location of the Schema.
    * Good for: Temporary files, scratchpad data.
* **External Volume:**
    * Points to a specific URL in your Cloud Storage (e.g., `s3://my-bucket/raw-data`).
    * Good for: Landing zones where raw data arrives from outside.



### **4. Commands**
```sql
-- Create a Volume
CREATE VOLUME my_volume;

-- Upload a file (UI or CLI)
-- Then read it directly:
SELECT * FROM csv.`/Volumes/prod/sales/my_volume/data.csv`;

-- List files in a Volume
LIST '/Volumes/prod/sales/my_volume/';
```


---
---

## 🛠️ 13. Databricks Utilities (DBUtils) & Widgets
**Concept:** A built-in library (`dbutils`) that acts as the "Swiss Army Knife" for Databricks. It allows you to interact with the file system, chain notebooks, and create input parameters.

### **1. File System Utilities (`dbutils.fs`)**
While `%fs` is good for quick checks, `dbutils.fs` allows you to manage files using Python code (loops, variables, logic).

| Command | Description | Example |
| :--- | :--- | :--- |
| `ls(path)` | List files in a directory. | `dbutils.fs.ls("/tmp")` |
| `cp(src, dst)` | Copy a file. | `dbutils.fs.cp("/src/file.csv", "/dst/backup.csv")` |
| `mv(src, dst)` | Move/Rename a file. | `dbutils.fs.mv("/src/old.csv", "/src/new.csv")` |
| `rm(path, recurse)` | Remove a file or folder. | `dbutils.fs.rm("/tmp/folder", True)` (True = delete contents) |
| `put(path, content)` | Create a simple file. | `dbutils.fs.put("/tmp/hello.txt", "Hello World!", True)` |
| `head(path)` | Read the first few bytes of a file. | `dbutils.fs.head("/tmp/hello.txt")` |
| `mount(...)` | **Legacy:** Connect to Cloud Storage. | *(Use Unity Catalog Volumes instead)* |

### **2. Widgets (Parameters)**
**Concept:** Make your notebook dynamic by adding input fields at the top. This allows you to pass arguments when running a notebook as a Job.

* **Types:** Text, Dropdown, Combobox, Multiselect.
* **Workflow:**
    1.  Create the widget.
    2.  Get the value into a variable.
    3.  Use the variable in your logic.

```python
# 1. Create a Dropdown for "Department"
dbutils.widgets.dropdown("dept_param", "HR", ["HR", "IT", "Sales"])

# 2. Capture the user's selection
selected_dept = dbutils.widgets.get("dept_param")

# 3. Use it in SQL or Python
# spark.sql(f"SELECT * FROM employees WHERE dept = '{selected_dept}'")
print(f"Selected Department: {selected_dept}")
```

### **3. Notebook Chaining (`dbutils.notebook`)**
**Concept:** Running one notebook inside another. This allows you to build modular pipelines (e.g., a "Setup" notebook called by an "Analysis" notebook).

* **Command:** `dbutils.notebook.run(path, timeout_seconds, arguments)`
* **Exit:** `dbutils.notebook.exit(value)` (Returns a value to the parent notebook).

```python
# Run the "Setup_Config" notebook and wait 60 seconds
# result = dbutils.notebook.run("./Setup_Config", 60, {"env": "prod"})
# print(result)
```

### **4. Secrets (`dbutils.secrets`)**
**Concept:** securely storing passwords/keys instead of hardcoding them.
* **Command:** `password = dbutils.secrets.get(scope="my-scope", key="db-password")`

---

### **Hands-On Practice Code**

```python
# --- PART 1: WIDGET PRACTICE ---

# 1. Create a Widget (Look at the top of your notebook after running this!)
dbutils.widgets.dropdown("environment", "DEV", ["DEV", "QA", "PROD"])

# 2. Get the value
current_env = dbutils.widgets.get("environment")

# 3. Use logic based on the widget
print(f"🚀 Running logic for: {current_env}")

if current_env == "PROD":
    print("⚠️ CAUTION: You are in Production!")
else:
    print("✅ Safe to test.")

# --- PART 2: FILE SYSTEM (DBUtils FS) PRACTICE ---

# 1. Create a directory for practice
base_path = "dbfs:/tmp/dbutils_practice"
# dbutils.fs.mkdirs(base_path) # Uncomment to run

# 2. Create a dummy file programmatically
file_path = f"{base_path}/config.txt"
content = "Database=SalesDB\nUser=Admin"

# The 'True' argument means "Overwrite if exists"
# dbutils.fs.put(file_path, content, True)
print(f"File path defined: {file_path}")

# 3. List the files to verify
# files = dbutils.fs.ls(base_path)
# for f in files:
#    print(f"Found: {f.name} (Size: {f.size} bytes)")

# 4. Read the file content
# read_content = dbutils.fs.head(file_path)
# print("\n--- File Content ---")
# print(read_content)
# print("--------------------")

# 5. Cleanup (Delete everything)
# dbutils.fs.rm(base_path, True)
```
---

---

## 🎻 14. Orchestration: Jobs & Notebook Chaining
**Concept:** Moving from manual "Run All" to automated, scheduled pipelines where notebooks trigger each other.

### **1. Notebook Chaining (`dbutils.notebook.run`)**
**Concept:** You can call a "Child" notebook from a "Parent" notebook, pass arguments to it, and get a result back.
* **Why use it?** To modularize code (e.g., `Ingest_Notebook` -> `Process_Notebook` -> `Reporting_Notebook`).
* **Scope:** The child notebook runs in a **separate job** (ephemeral scope), so variables are *not* shared automatically. You must pass them explicitly.

#### **Command Syntax**
```python
# Parent Notebook calling Child
result = dbutils.notebook.run(
    "path/to/child_notebook",  # Path
    600,                       # Timeout in seconds
    {"input_param": "Value"}   # Arguments (Dictionary)
)
```

#### **Returning Values (`dbutils.notebook.exit`)**
* The child notebook can send a string back to the parent.
* *Tip:* To return complex data (like a list or dictionary), wrap it in JSON.

```python
# Child Notebook finishing
import json
dbutils.notebook.exit(json.dumps({"status": "Success", "rows": 100}))
```

### **2. Scheduling Jobs (The "Jobs" Tab)**
**Concept:** Turning a notebook into a production workflow.
* **Task:** A single unit of work (e.g., Run Notebook A).
* **Job:** A collection of tasks with dependencies (A -> B -> C).

#### **Passing Parameters from Job to Notebook**
When you schedule a Job, you can define **Parameters** in the Job UI.
1.  **In Job UI:** Add Key=`env`, Value=`prod`.
2.  **In Notebook:** Use `dbutils.widgets.get("env")` to read it.
    * *Result:* The same notebook can run as "Dev" or "Prod" just by changing the Job Parameter.

### **3. Retry & Timeout Policies**
* **Retries:** If a cell fails (e.g., API glitch), the Job can auto-retry.
* **Timeout:** If a job hangs for too long (e.g., > 2 hours), kill it to save money.

---

### **Hands-On Practice: Parent-Child Pipeline**

To practice this, you need to create **two separate notebooks**.

#### **Step 1: Create "Child_Worker" Notebook**
*Create a new notebook named `Child_Worker` and paste this code:*

```python
# --- CHILD NOTEBOOK ---
import json

# 1. Accept Inputs
dbutils.widgets.text("date_param", "2024-01-01")
date = dbutils.widgets.get("date_param")

print(f"👷 Child Worker processing data for: {date}")

# 2. Simulate Work
status = "Success"
count = 500

# 3. Return Result to Parent
# We bundle the results into a JSON string
result_json = json.dumps({
    "processed_date": date,
    "status": status,
    "row_count": count
})

dbutils.notebook.exit(result_json)
```

#### **Step 2: Create "Parent_Orchestrator" Notebook**
*Create a new notebook named `Parent_Orchestrator` and paste this code:*

```python
# --- PARENT NOTEBOOK ---
import json

print("🚀 Starting Pipeline...")

# 1. Run the Child Notebook
# We pass the 'date_param' dynamically
raw_result = dbutils.notebook.run(
    "./Child_Worker",   # Path to child (assuming same folder)
    60,                 # Timeout (60 seconds)
    {"date_param": "2025-12-31"} # Arguments
)

# 2. Process the Result
parsed_result = json.loads(raw_result)

print(f"✅ Pipeline Finished.")
print(f"Child Status: {parsed_result['status']}")
print(f"Rows Processed: {parsed_result['row_count']}")
```
---

---

## 🖥️ 15. Databricks Compute Architecture & Security (Video 20)
**Concept:** Understanding the different "Engines" (Clusters) that process your data, how to secure them using Access Modes, and how to control costs with Policies.

### **1. Compute Types (The "Workhorses")**
Choosing the wrong compute type can cost you **3x more money**.

| Feature | **All-Purpose Compute** (Interactive) | **Job Compute** (Automated) |
| :--- | :--- | :--- |
| **Purpose** | Development, Ad-hoc analysis, Building Notebooks. | Running scheduled Production workflows (ETL). |
| **Lifecycle** | **Long-Running.** Stays up until you manually terminate it (or auto-termination kicks in). | **Ephemeral.** Starts when the job begins, dies immediately after. |
| **Cost** | **Expensive ($$$).** You pay a premium for interactivity. | **Cheap ($).** Heavily discounted (approx. 60% cheaper). |
| **Sharing** | Shared by many developers (e.g., "Team_Shared_Cluster"). | Dedicated to that specific job run. Isolated. |
| **Best For** | Coding, Debugging, BI Dashboards. | Nightly Batch Jobs, Streaming Pipelines. |



### **2. Access Modes (Crucial for Unity Catalog)**
**Concept:** This setting defines *who* can use the cluster and *what* data security rules apply.

| Access Mode | **Single User** | **Shared** (Standard) | **No Isolation** (Legacy) |
| :--- | :--- | :--- | :--- |
| **Target User** | **One person only.** | **Multiple users.** | **Everyone.** |
| **Unity Catalog?** | **YES.** (Full Support) | **YES.** (Full Support) | **NO.** (Legacy Hive only) |
| **Languages** | Python, SQL, Scala, R. | Python, SQL. (**No Scala/R**). | All Languages. |
| **Isolation** | User is isolated. | Process isolation (Secure). | None (Shared Java VM). |
| **Use Case** | Data Scientists needing R or custom libraries. | Data Engineering Teams, BI Analysts. | Legacy jobs, non-UC data. |



### **3. Cluster Policies (Governance & Cost Control)**
**Concept:** A set of rules (JSON) that restrict what users can configure.
* **Why?** Prevents a Junior Engineer from accidentally spinning up a **$500/hour** GPU cluster.

**Common Rules:**
1.  **Limit Size:** "Max Workers = 4".
2.  **Enforce Type:** "Must use `i3.xlarge` (Spot Instances)".
3.  **Auto-Termination:** "Must terminate after 30 mins".

**Example Policy (JSON):**
```json
{
  "spark_version": { "type": "fixed", "value": "13.3.x-scala2.12" },
  "autotermination_minutes": { "type": "range", "minValue": 10, "maxValue": 60 },
  "node_type_id": { "type": "allowlist", "values": ["Standard_DS3_v2", "Standard_DS4_v2"] }
}
```

### **4. Cluster Permissions (ACLs)**
**Concept:** Controlling *who* can touch the cluster itself.

| Permission | Capability | Typical Role |
| :--- | :--- | :--- |
| **Can Attach To** | Can connect their notebook to the cluster and run code. | Developers / Analysts |
| **Can Restart** | Can reboot the cluster (clearing cache) but not change size. | Senior Developers / Leads |
| **Can Manage** | Can edit config, resize, change policy, or delete. | Admins / DevOps |



---

### **5. Real-World Scenario: The "Thinking Process"**
**Situation:** You are a Senior Data Engineer at "ShopSmart". You need to create a cluster for your team of 5 engineers to process sensitive customer data.

**The Analysis Checklist:**

1.  **Question:** "Is this for just me, or the whole team?"
    * *Answer:* Whole team (5 people).
    * *Decision:* **Cannot use Single User.** Must be a multi-user cluster.

2.  **Question:** "Do we need Unity Catalog (Security)?"
    * *Answer:* Yes, we have PII data.
    * *Decision:* **Must use Shared Mode.** (No Isolation would expose all data).

3.  **Question:** "What languages are we using?"
    * *Answer:* Python and SQL.
    * *Check:* Shared Mode supports these. (If we needed Scala, we'd have a problem).

4.  **Question:** "How do I prevent over-spending?"
    * *Answer:* This is for dev work.
    * *Decision:* Set **Auto-Termination** to 30 mins. Use a **Cluster Policy** that restricts max workers to 4.

**The Final Configuration:**
* **Name:** `data-eng-team-shared-dev`
* **Access Mode:** **Shared**
* **Policy:** `General_Dev_Policy`
* **Runtime:** `13.3 LTS`
* **Worker Type:** `Standard_D4s_v5` (Small/Cheap)
* **Permissions:** Group `data-engineers` gets **Can Attach To**.

---

---

## 🎻 17. Advanced Workflows: Logic, Loops & Values
**Concept:** Moving beyond simple linear jobs (A -> B -> C) to complex, logic-driven pipelines (A -> If True -> B -> Loop C).

### **1. The Hierarchy: Job vs. Run vs. Task**
* **Job:** The definition or "blueprint" of the workflow (e.g., "Daily Sales ETL"). It contains the schedule and permissions.
* **Job Run:** A specific instance of that job executing (e.g., "The Jan 1st Run").
* **Task:** A single step within the job.
    * **Types:** Notebook, Python Script, SQL Query, JAR, Delta Live Tables, dbt.
    * **Dependency:** You link tasks together to create a **DAG** (Directed Acyclic Graph). Task B waits for Task A to finish.



### **2. Passing Values Between Tasks (`taskValues`)**
**Concept:** You calculate a value in **Task A** (e.g., `row_count = 500`) and you need to use it in **Task B** (e.g., `if row_count > 0`).
* **Old Way:** Save to a file on disk (S3/ADLS) in Task A, read the file in Task B. Slow and messy.
* **New Way:** Use `dbutils.jobs.taskValues`. It passes variables in memory between tasks securely.

#### **Syntax:**
**Task A (The Setter):**
```python
# Calculate a value
records_processed = 1500

# Set the value with a Key ("row_count")
# dbutils.jobs.taskValues.set(key = "row_count", value = records_processed)
```

**Task B (The Getter):**
```python
# Get the value from Task A
# usage: get(taskKey, key, default, debugValue)
# count = dbutils.jobs.taskValues.get(taskKey = "Task_A", key = "row_count", default = 0)

# print(f"Received count from Task A: {count}")
```

### **3. Control Flow: If/Else Condition Task**
**Concept:** A specific **Task Type** in the Workflows UI that branches execution based on a condition. You don't need a notebook for this logic anymore.

* **How it works:**
    1.  Create a task and select Type: **If/Else Condition**.
    2.  **Condition:** Check a `taskValue` from a previous task.
        * *Example:* `{{tasks.Task_A.values.row_count}} > 0`
    3.  **True Path:** Add tasks that run if the condition is met (e.g., "Send_Email").
    4.  **False Path:** Add tasks that run if it fails (e.g., "Skip_Processing").



### **4. Control Flow: For Each Loop Task**
**Concept:** A **Task Type** that runs a nested task multiple times in parallel over a list of items.

* **Use Case:** You have a list of 50 tables to ingest. Instead of creating 50 separate tasks, you create **one** loop.
* **Input:** A JSON array (e.g., `["table1", "table2", "table3"]`) passed from a previous task or defined manually.
* **Concurrency:** You can set how many loops run at the same time (e.g., "Run 5 tables in parallel").



### **5. Repair and Re-Run**
**Concept:** When a job fails halfway through, you don't want to restart from the beginning (Task A). You want to fix the error and resume from where it broke.

* **Repair Run:**
    * Identify the failed task (e.g., Task C).
    * Fix the code in Task C.
    * Click **"Repair Run"** in the UI.
    * Databricks skips Task A and B (since they succeeded) and restarts **only** Task C and its downstream dependents.
* **Matrix Re-run:** If a "For Each" loop fails on just 1 out of 50 tables, a Repair Run will only retry that **1 failed iteration**.

---

### **Hands-On Practice: Task Values**

**Scenario:** Task 1 generates a random number. Task 2 prints it.

**Step 1: Create Notebook "Generate_Data" (Task 1)**
```python
import random

# 1. Generate a random value
val = random.randint(1, 100)
print(f"🎲 Generated Value: {val}")

# 2. Pass it to the Job context
# dbutils.jobs.taskValues.set(key = "my_random_number", value = val)
```

**Step 2: Create Notebook "Process_Data" (Task 2)**
```python
# 1. Get value from previous task (Use the exact Task Name you set in the UI)
# Note: 'debugValue' is what is returned when running interactively (not in a job)
# received_val = dbutils.jobs.taskValues.get(
#     taskKey = "Generate_Data", 
#     key = "my_random_number", 
#     default = 0, 
#     debugValue = 99
# )

# print(f"📥 Received Value: {received_val}")

# 2. Logic
# if int(received_val) > 50:
#     print("✅ High Value! Processing...")
# else:
#     print("⚠️ Low Value. Skipping.")
```
---
---

## 📥 18. Data Ingestion: COPY INTO & Idempotency
**Concept:** The standard SQL command to load data from files (S3/ADLS) into a Delta Table. It is designed to be **Idempotent** (safe to re-run).

### **1. What is `COPY INTO`?**
* **Definition:** A SQL command that loads data from a file location into a Delta table.
* **The Magic:** It keeps track of which files it has already loaded.
* **Use Case:** Batch ingestion (e.g., "Load all new files that arrived today").
* **Best For:** Small to Medium workloads (Thousands of files). For Millions of files, use **Auto Loader** (discussed later).

### **2. The "Metadata" Strategy (How it works)**
`COPY INTO` maintains a **File State** (Metadata) in the target Delta table's transaction log.
1.  **Scan:** It looks at the Source folder (e.g., S3 Bucket).
2.  **Check:** It compares the files in the source against its internal log of "already loaded files."
3.  **Filter:** It ignores any file that matches an entry in the log.
4.  **Ingest:** It only reads and commits the **New Files**.

### **3. Idempotency & Exactly-Once Processing**
**Concept:** "Idempotent" means you can run the same command 1,000 times, and the result is exactly the same as running it once.

* **Scenario:**
    * **Run 1:** You have `file_A.csv`. You run `COPY INTO`.
        * *Result:* `file_A` is loaded. Table count: 100 rows.
    * **Run 2:** You run `COPY INTO` again immediately.
        * *Result:* Databricks sees `file_A` is already in the metadata. It does **nothing**. Table count: 100 rows. (No Duplicates!).
    * **Run 3:** You add `file_B.csv` and run `COPY INTO`.
        * *Result:* It skips `file_A`, loads `file_B`. Table count: 200 rows.

**This guarantees "Exactly-Once Processing"**: Data is processed once and only once, even if the pipeline crashes and restarts.



### **4. Syntax**
```sql
COPY INTO target_table
FROM 's3://my-bucket/data-folder'
FILEFORMAT = CSV
FORMAT_OPTIONS ('header' = 'true', 'inferSchema' = 'true')
COPY_OPTIONS ('mergeSchema' = 'true'); -- Auto-evolve schema if new columns appear
```

---

### **Hands-On Practice: Idempotency Test**

**Step 1: Create a Dummy Source File**
*We will create a file in DBFS to act as our "S3 Bucket".*
```python
# Create a folder and a file
dbutils.fs.mkdirs("/tmp/copy_into_demo")
dbutils.fs.put("/tmp/copy_into_demo/file1.csv", "id,name\n1,Manish\n2,Rahul", True)
```

**Step 2: Create a Target Table**
```sql
CREATE TABLE IF NOT EXISTS my_ingest_table (
    id INT,
    name STRING
);
```

**Step 3: Run COPY INTO (First Load)**
```sql
COPY INTO my_ingest_table
FROM '/tmp/copy_into_demo'
FILEFORMAT = CSV
FORMAT_OPTIONS ('header' = 'true');

-- Check count. Should be 2.
SELECT count(*) FROM my_ingest_table;
```

**Step 4: The Idempotency Test (Run it again!)**
*Run the EXACT same command from Step 3 again.*
```sql
COPY INTO my_ingest_table
FROM '/tmp/copy_into_demo'
FILEFORMAT = CSV
FORMAT_OPTIONS ('header' = 'true');

-- Check count. Should STILL be 2. 
-- It detected that file1.csv was already loaded and skipped it.
SELECT count(*) FROM my_ingest_table;
```

**Step 5: Add New Data**
```python
# Add a new file
dbutils.fs.put("/tmp/copy_into_demo/file2.csv", "id,name\n3,Priya", True)
```

**Step 6: Run COPY INTO (Incremental Load)**
*Run the command from Step 3 one more time.*
```sql
-- It will now skip file1.csv and ONLY load file2.csv
COPY INTO my_ingest_table ... 

-- Check count. Should be 3.
SELECT * FROM my_ingest_table;
```
---


