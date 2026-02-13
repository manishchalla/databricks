# 📘 Databricks Zero to Hero: Core Concepts & Notes

**Scope:** Videos 1-15 (Architecture, Table Management, Unity Catalog, Data Manipulation)  
**Focus:** Pure Data Engineering concepts (skipping Azure administration/setup).

---

## 🏗️ 1. High-Level Architecture
Databricks uses a **Split-Plane Architecture** to ensure security and scalability.

### **The Control Plane (The Brain)**
* Managed entirely by Databricks in their cloud account.
* **Responsibilities:** * Hosting the Web UI (the website you log into).
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
Databricks Notebooks allow you to mix languages. You set a default language for the notebook (e.g., Python), but can switch for specific cells using **Magic Commands**.

### **Common Magic Commands**
| Command | Name | Usage |
| :--- | :--- | :--- |
| `%python` | Python | Write PySpark logic, define variables, or use libraries like Pandas. |
| `%sql` | SQL | Run standard ANSI SQL queries on tables and views. |
| `%md` | Markdown | Write documentation, titles, and notes (like this file!). |
| `%fs` | File System | Interact with DBFS (Databricks File System) to list/move files. |
| `%run` | Run | Execute another notebook inside the current one (great for modular code). |

---

## 🗄️ 3. Managed vs. External Tables
This is the most critical concept for a Data Engineer. It determines **who owns the data** and **what happens when you delete a table**.

### **Managed Tables**
* **Created by:** Simply running `CREATE TABLE table_name`.
* **Data Location:** Stored in a default location managed by Databricks (e.g., the Hive Warehouse or Unity Catalog root).
* **Lifecycle:** "Fully Managed." If you `DROP` the table, Databricks deletes **both** the metadata (table name) and the actual data files.

### **External (Unmanaged) Tables**
* **Created by:** Specifying a `LOCATION 'path/to/storage'` during creation.
* **Data Location:** Stored in your own Azure Data Lake Storage (ADLS) container.
* **Lifecycle:** "Loose Pointer." If you `DROP` the table, Databricks **only deletes the metadata**. The actual files remain safe in your storage.

### **📝 Code Example: Managed vs. External**

```sql
-- 1. Creating a MANAGED Table
-- We do NOT specify a location. Databricks picks the folder.
CREATE TABLE managed_users (
    user_id INT,
    user_name STRING
);

-- ⚠️ DANGER: If we run this, the data is GONE forever (unless using Unity Catalog UNDROP).
-- DROP TABLE managed_users;


-- 2. Creating an EXTERNAL Table
-- We explicitly tell Databricks where the files live.
CREATE TABLE external_users (
    user_id INT,
    user_name STRING
)
LOCATION 'abfss://my-container@mystorage.dfs.core.windows.net/raw_data/users';

-- ✅ SAFE: If we run DROP on this, the table disappears from the UI,
-- but the folder 'raw_data/users' in Azure still has all the CSV/Parquet files.
