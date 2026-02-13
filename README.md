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
