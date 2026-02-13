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

## 🕰️ 10. Deep Dive: Delta Lake Internals (Video 16)
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
