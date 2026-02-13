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

## 🚀 9. Optimization Essentials 
**Concept:** Before understanding Delta Lake internals, you must understand how Spark reads data efficiently.

### **1. Partitioning (The "Folder" Strategy)**
* **What is it?** Breaking a large table into sub-folders based on a column (e.g., `year=2023/month=01`).
* **Benefit:** **Partition Pruning**. If you query `WHERE year = 2023`, Spark reads *only* that folder and ignores the rest.
* **Best Practice:** Only partition columns frequently used in filters and with low cardinality (e.g., Date, Country). Do **not** partition by unique IDs.

### **2. Data Skipping & Z-Ordering**
* **Data Skipping:** Delta Lake automatically stores **Min/Max statistics** for every column in every file. If your query looks for `id = 50` and a file's range is `100-200`, Spark skips that file entirely.
* **Z-Ordering:** A technique to co-locate related data. It sorts data by multiple columns so that similar data points sit in the same files.
    * **Command:** `OPTIMIZE table_name ZORDER BY (col1, col2)`
    * **Result:** dramatically improves Data Skipping.

### **3. Compaction (Small File Problem)**
* **The Problem:** In Big Data, opening 1,000 tiny files (1KB each) is much slower than opening 1 large file (1GB).
* **The Solution:** The `OPTIMIZE` command (Bin-packing).
* **Command:** `OPTIMIZE table_name`
    * This merges small files into larger, efficient files without stopping reads/writes.

---

## 🕰️ 10. Delta Lake Architecture & Time Travel
**Concept:** How Databricks tracks changes (`_delta_log`) and allows you to query history or rollback.

### **The Transaction Log (`_delta_log`)**
Every Delta Table has a hidden folder named `_delta_log` at the root.
* **The Source of Truth:** It contains JSON files (commits).
* **How it works:** Every INSERT, UPDATE, or DELETE creates a new JSON file (e.g., `00001.json`) that says "Add File A, Remove File B".
* **ACID Compliance:** If the JSON file isn't fully written, the transaction doesn't happen. This prevents half-written data.

### **Time Travel (Version History)**
Because Delta Lake keeps the "Removed" files (for a default of 7 days) and the Log history, you can query the table as it existed in the past.

**Use Cases:**
1.  **Auditing:** Checking data state before a bad update.
2.  **Reproduction:** Re-running a report with last month's data.

**Commands:**

```sql
-- 1. View all versions/history
DESCRIBE HISTORY employee_table;

-- 2. Query by Version Number
SELECT * FROM employee_table VERSION AS OF 3;

-- 3. Query by Timestamp
SELECT * FROM employee_table TIMESTAMP AS OF '2023-10-25 14:30:00';
```

**Concept:** A performance feature that speeds up `UPDATE` and `DELETE` operations on large files.

* **The Problem (Copy-on-Write):**
    * In standard Delta Lake, if you modify **one single row** in a 1GB Parquet file, Databricks must read the whole file, remove the row, and rewrite a **new 1GB file**.
    * This is called "Write Amplification." It is slow and wastes I/O.

* **The Solution (Deletion Vectors / Merge-on-Read):**
    * Instead of rewriting the whole file, Databricks creates a tiny auxiliary file (a **Bitmap**) linked to the original file.
    * This bitmap simply says: *"Rows 5, 10, and 500 are deleted."*
    * **Write Speed:** Instant (because we aren't moving 1GB of data).
    * **Read Speed:** Slight overhead (Spark has to check the bitmap while reading), but generally faster than waiting for a full rewrite.



* **Maintenance:**
    * The file remains "fragmented" (Original Data + Deletion Vector) until you run `OPTIMIZE` or `VACUUM`.
    * During Optimization, Databricks finally rewrites the file cleanly, permanently removing the deleted rows and deleting the vector file.

* **Enabling it:**
    ```sql
    ALTER TABLE my_table SET TBLPROPERTIES ('delta.enableDeletionVectors' = true);
    ```
