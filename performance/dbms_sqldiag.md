

---

# 🛠️ Oracle Cheatsheet: DBMS_SQLDIAG Package

The `DBMS_SQLDIAG` package is the interface for the **SQL Diagnosability Framework**. It is designed to identify, analyze, and bypass problematic SQL statements that cause performance regressions or ORA-errors.

## 1. Package Overview

`DBMS_SQLDIAG` operates across three main pillars:

1. **Incident Diagnosis:** Automated analysis of SQL that produces errors (e.g., ORA-00600).
2. **SQL Test Case (STC):** Packaging all dependencies (metadata, stats, data samples) to reproduce a plan on another environment.
3. **SQL Patching:** Applying "emergency" optimizer hints to a `SQL_ID` without changing code.

---

## 2. SQL Test Case (The "Portability" Tool)

This is used when you need to send a problematic query to Oracle Support or a Dev environment. It creates a `.zip` or a directory of scripts to recreate the exact environment.

### Exporting a Test Case

```sql
BEGIN
  DBMS_SQLDIAG.EXPORT_SQL_TESTCASE (
    directory => 'DATA_PUMP_DIR',
    sql_id    => '8998pf78abc12',
    filename  => 'sql_issue_8998.zip'
  );
END;
/

```

### Key Parameters for STC:

* **`export_data`**: (Boolean) Whether to include a sample of the actual data (default is FALSE, metadata only).
* **`export_pkg_body`**: Includes PL/SQL package bodies if the SQL calls them.
* **`explain`**: Runs an explain plan and includes it in the bundle.

---

## 3. SQL Diagnosis (The "Doctor" Tool)

Used to diagnose SQL statements that are failing or triggering internal errors.

### Identify a Problematic SQL

```sql
DECLARE
  v_report CLOB;
BEGIN
  -- Generates a report on a specific SQL execution incident
  v_report := DBMS_SQLDIAG.REPORT_DIAGNOSIS (
    incident_id => 12345, 
    type        => 'TEXT'
  );
  DBMS_OUTPUT.PUT_LINE(v_report);
END;
/

```

### Common Procedures:

| Procedure | Description |
| --- | --- |
| `DUMP_TRACE` | Dumps the optimizer trace (10053) for a specific SQL. |
| `INCIDENT_CHECK` | Checks if a SQL statement is likely to trigger a known incident. |

---

## 4. SQL Patching (The "Fix" Tool)

A SQL Patch is a "hint-on-the-side." It allows you to inject hints into a query's compilation phase based on its `SQL_ID`.

**Quick Creation:**

```sql
BEGIN
  SYS.DBMS_SQLDIAG.CREATE_SQL_PATCH(
    sql_id => 'g33p678abc',
    hints  => 'INDEX(@SEL$1 "TAB1" "TAB1_PK")',
    name   => 'FIX_TAB1_SCAN'
  );
END;
/

```

---

## 5. Related Views & Useful Queries

### View the SQL Framework Repository

```sql
-- See recorded incidents and diagnostic data
SELECT incident_id, problem_key, create_time 
FROM v$sql_diag_repository 
ORDER BY create_time DESC;

```

### Check SQL Patch Status

```sql
SELECT name, status, force_matching 
FROM dba_sql_patches;

```

### Identify SQL using Patches in the Cursor Cache

```sql
SELECT sql_id, child_number, sql_patch, last_active_time
FROM v$sql
WHERE sql_patch IS NOT NULL;

```

---

## 6. Summary Table: Functions vs. Use Cases

| Function/Procedure | Target | Use Case |
| --- | --- | --- |
| `CREATE_SQL_PATCH` | Optimizer | Force a better plan without code change. |
| `EXPORT_SQL_TESTCASE` | Portability | Package a bug for Oracle Support/QA. |
| `ACCEPT_SQL_PATCH` | Management | Accept a patch recommended by the SQL Repair Advisor. |
| `REPORT_DIAGNOSIS` | Analysis | Get a detailed report on why a SQL failed. |

---

## 7. Useful Tips

* **Permissions:** You need the `ADMINISTER SQL DIAGNOSIS ANY` or `ADMINISTER SQL MANAGEMENT OBJECT` privilege.
* **The "Note" Section:** Always check the bottom of `dbms_xplan.display_cursor`. If `DBMS_SQLDIAG` is active, it will say: `SQL patch "NAME" used for this statement`.
* **Force Matching:** Use `force_matching => TRUE` if your application uses literals instead of bind variables.


This is a powerful combination. The **SQL Repair Advisor** is the "automated doctor" that analyzes a failing query (e.g., one hitting an `ORA-0600` or `ORA-07445`), and `DBMS_SQLDIAG` is the "pharmacist" that applies the patch recommended by the advisor.

Here is the markdown-ready guide for using the **SQL Repair Advisor** with **DBMS_SQLDIAG**.

---

# 🛠️ SQL Repair Advisor & DBMS_SQLDIAG Guide

When a SQL statement fails with a critical error (internal ORA- errors), the SQL Repair Advisor can analyze the failure and, if possible, recommend a **SQL Patch** that bypasses the problematic code path (usually by disabling a specific optimizer transformation).

## 1. The Workflow

The process follows four distinct steps: **Create → Execute → Report → Accept**.

---

## 2. Practical Example: Repairing a Failing SQL_ID

Assume `SQL_ID: 7v98abc12345` is causing an `ORA-00600` internal error every time it runs.

### Step 1: Create the Diagnosis Task

We initialize a "Repair Task" specifically for the problematic SQL ID.

```sql
DECLARE
  v_task_name  VARCHAR2(100);
BEGIN
  v_task_name := DBMS_SQLDIAG.CREATE_DIAGNOSIS_TASK(
    sql_id      => '7v98abc12345',
    task_name   => 'REPAIR_ORA_00600_TASK',
    description => 'Analyze and fix internal error in sales query'
  );
END;
/

```

### Step 2: Execute the Diagnosis Task

The advisor now simulates the query execution to identify the exact cause of the crash.

```sql
BEGIN
  DBMS_SQLDIAG.EXECUTE_DIAGNOSIS_TASK(task_name => 'REPAIR_ORA_00600_TASK');
END;
/

```

### Step 3: Report the Findings

Check if the advisor found a workaround (a SQL Patch).

```sql
SET LONG 100000;
SET PAGESIZE 1000;
SET LINESIZE 200;
SELECT DBMS_SQLDIAG.REPORT_DIAGNOSIS_TASK('REPAIR_ORA_00600_TASK', 'TEXT') AS report 
FROM dual;

```

> **Note:** Look for a section in the report titled **"Recommendations"**. It will tell you if a patch can bypass the error.

### Step 4: Accept and Apply the Patch

If the report recommends a patch, use `ACCEPT_SQL_PATCH` to activate it. This uses the `DBMS_SQLDIAG` engine to link the fix to the `SQL_ID`.

```sql
DECLARE
  v_patch_name VARCHAR2(30);
BEGIN
  v_patch_name := DBMS_SQLDIAG.ACCEPT_SQL_PATCH(
    task_name => 'REPAIR_ORA_00600_TASK',
    name      => 'BUG_FIX_PATCH_7v98',
    replace   => TRUE
  );
END;
/

```

---

## 3. Useful Queries for Repair Management

### Check the Status of Repair Tasks

```sql
SELECT task_name, status, created, last_modified
FROM dba_advisor_tasks
WHERE advisor_name = 'SQL Repair Advisor';

```

### List Patches Created by the Advisor

```sql
SELECT name, sql_text, status, description
FROM dba_sql_patches
WHERE description LIKE '%Repair%';

```

---

## 4. Summary Table: Repair Advisor Procedures

| Procedure | Purpose |
| --- | --- |
| `CREATE_DIAGNOSIS_TASK` | Defines the scope (SQL ID or Text) for the advisor. |
| `EXECUTE_DIAGNOSIS_TASK` | Runs the internal analysis engine. |
| `REPORT_DIAGNOSIS_TASK` | Generates a human-readable CLOB report of findings. |
| `ACCEPT_SQL_PATCH` | Confirms and implements the recommended fix. |
| `CANCEL_DIAGNOSIS_TASK` | Stops a currently running analysis. |

---

## 💡 Pro-Tip: When to use this?

Don't use the Repair Advisor for "slow" queries—use the **SQL Tuning Advisor** (`DBMS_SQLTUNE`) for that. Use the **SQL Repair Advisor** (`DBMS_SQLDIAG`) specifically when a query **crashes** or produces an **ORA- error** during execution.

Here is a complete, **one-click automation script** that integrates the SQL Repair Advisor with `DBMS_SQLDIAG`.

This script handles the entire lifecycle: it creates the task, executes the analysis, generates the report, and provides the command to accept the patch if a recommendation is found.

---

# 🚀 One-Click SQL Repair Automation Script

Copy and run this block in SQL*Plus or SQL Developer. Replace `&&target_sql_id` with the failing SQL ID when prompted.

```sql
SET SERVEROUTPUT ON
SET FEEDBACK OFF
SET LINESIZE 200
SET LONG 100000
SET PAGESIZE 1000

ACCEPT target_sql_id CHAR PROMPT 'Enter the failing SQL_ID: '

DECLARE
  v_task_name   VARCHAR2(64);
  v_sql_id      VARCHAR2(13) := '&&target_sql_id';
  v_report      CLOB;
  v_patch_count NUMBER;
BEGIN
  -- 1. Generate a unique Task Name
  v_task_name := 'REPAIR_TASK_' || v_sql_id || '_' || TO_CHAR(SYSDATE, 'YYYYMMDD_HH24MI');
  
  DBMS_OUTPUT.PUT_LINE('---------------------------------------------------------');
  DBMS_OUTPUT.PUT_LINE('Starting SQL Repair Advisor for SQL_ID: ' || v_sql_id);
  DBMS_OUTPUT.PUT_LINE('Task Name: ' || v_task_name);
  DBMS_OUTPUT.PUT_LINE('---------------------------------------------------------');

  -- 2. Create the Diagnosis Task
  v_task_name := DBMS_SQLDIAG.CREATE_DIAGNOSIS_TASK(
    sql_id      => v_sql_id,
    task_name   => v_task_name,
    description => 'Automated repair task for ' || v_sql_id
  );

  -- 3. Execute the Task
  DBMS_SQLDIAG.EXECUTE_DIAGNOSIS_TASK(task_name => v_task_name);

  -- 4. Fetch and Display the Report
  v_report := DBMS_SQLDIAG.REPORT_DIAGNOSIS_TASK(v_task_name, 'TEXT');
  DBMS_OUTPUT.PUT_LINE(v_report);

  -- 5. Check if a Patch was recommended
  -- We check the advisor data for recommendations
  SELECT COUNT(*) INTO v_patch_count
  FROM dba_advisor_recommendations
  WHERE task_name = v_task_name;

  IF v_patch_count > 0 THEN
    DBMS_OUTPUT.PUT_LINE('SUCCESS: A SQL Patch recommendation was found!');
    DBMS_OUTPUT.PUT_LINE('To apply the patch, execute:');
    DBMS_OUTPUT.PUT_LINE('EXEC :p := DBMS_SQLDIAG.ACCEPT_SQL_PATCH(task_name => ''' || v_task_name || ''');');
  ELSE
    DBMS_OUTPUT.PUT_LINE('NOTICE: No SQL Patch was recommended for this specific error.');
  END IF;
  
  DBMS_OUTPUT.PUT_LINE('---------------------------------------------------------');

EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('ERROR: ' || SQLERRM);
    RAISE;
END;
/

```

---

## 📋 Understanding the Workflow

The script above follows the official Oracle Diagnosability workflow:

1. **Scope Identification:** It targets the specific `SQL_ID` currently causing `ORA-` errors in your environment.
2. **Internal Analysis:** `EXECUTE_DIAGNOSIS_TASK` performs a "dry run" of the Optimizer's transformations to see which one triggers the internal failure.
3. **Workaround Generation:** If the Advisor identifies that disabling a specific transformation (like `STAR_TRANSFORMATION` or `SUBQUERY_UNNESTING`) stops the crash, it prepares a **SQL Patch**.
4. **Deployment:** Using `ACCEPT_SQL_PATCH` (via `DBMS_SQLDIAG`) physically writes the patch to the data dictionary (`SYS.SQLOBJ$`), ensuring all future executions of that SQL ID use the bypass.

---

## 🛠️ Post-Repair Checklist

Once you have applied a patch using the script above, you should perform these three checks:

### 1. Verify Patch Creation

```sql
SELECT name, status, category 
FROM dba_sql_patches 
WHERE last_modified > SYSDATE - 1/24; -- Created in the last hour

```

### 2. Force a Re-Parse

For the patch to take effect, the existing "bad" plan must be flushed or aged out.

```sql
-- Replace with your SQL_ID and Address/Hash from V$SQL
SELECT address, hash_value FROM v$sqlarea WHERE sql_id = '&&target_sql_id';
EXEC DBMS_SHARED_POOL.PURGE('address, hash_value', 'C');

```

### 3. Check the Note Section

Run the query and then check the execution plan.

```sql
SELECT * FROM table(DBMS_XPLAN.DISPLAY_CURSOR('&&target_sql_id', NULL, 'ADVANCED'));

```

> **Look for:** `Note: SQL patch "BUG_FIX_..." used for this statement`


As requested, here is a cleanup script to keep your diagnostic environment tidy. It is good practice to remove these tasks once the analysis is complete and the SQL Patch is successfully deployed to the data dictionary.

---

# 🧹 DBMS_SQLDIAG Cleanup & Management Script

Diagnostic tasks consume space in the `SYSAUX` tablespace. Use this script to list and purge old tasks that are no longer needed.

## 1. Identify Existing Tasks

Run this to see which tasks are still sitting in your repository.

```sql
SELECT 
    task_name, 
    status, 
    created, 
    last_modified,
    advisor_name
FROM dba_advisor_tasks
WHERE advisor_name = 'SQL Repair Advisor'
ORDER BY created DESC;

```

---

## 2. Bulk Cleanup Script

This script allows you to delete a specific task by name or purge all tasks older than a certain number of days.

```sql
SET SERVEROUTPUT ON
DECLARE
  v_days_to_keep NUMBER := 7; -- Change this to your retention preference
  CURSOR c_old_tasks IS
    SELECT task_name 
    FROM dba_advisor_tasks 
    WHERE advisor_name = 'SQL Repair Advisor'
      AND created < SYSDATE - v_days_to_keep;
BEGIN
  DBMS_OUTPUT.PUT_LINE('--- Cleaning up SQL Repair Tasks older than ' || v_days_to_keep || ' days ---');
  
  FOR r IN c_old_tasks LOOP
    DBMS_SQLDIAG.DROP_DIAGNOSIS_TASK(task_name => r.task_name);
    DBMS_OUTPUT.PUT_LINE('Dropped Task: ' || r.task_name);
  END LOOP;
  
  DBMS_OUTPUT.PUT_LINE('Cleanup complete.');
EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('Error during cleanup: ' || SQLERRM);
END;
/

```

---

## 3. Managing the SQL Patches (Permanent Objects)

Deleting the **Task** does **not** delete the **SQL Patch**. The patch is a permanent object. If you need to manage the patches themselves, use these commands:

### Disable a Patch (Safety First)

If you suspect a patch is causing side effects, disable it first instead of dropping it.

```sql
BEGIN
  DBMS_SQLDIAG.ALTER_SQL_PATCH(
    name           => 'BUG_FIX_PATCH_7v98',
    attribute_name => 'STATUS',
    value          => 'DISABLED'
  );
END;
/

```

### Drop a Patch (Permanent Removal)

```sql
BEGIN
  DBMS_SQLDIAG.DROP_SQL_PATCH(name => 'BUG_FIX_PATCH_7v98');
END;
/

```

---

## 💡 Summary of DBMS_SQLDIAG Best Practices

* **Task vs. Patch:** Always remember that `CREATE_DIAGNOSIS_TASK` is for the *process* of finding a fix, while `ACCEPT_SQL_PATCH` creates the *actual* fix.
* **AWR Retention:** If you are diagnosing a SQL from AWR, ensure your `STATISTICS_LEVEL` is set to `TYPICAL` or `ALL` so `DBMS_SQLDIAG` has enough metadata to work with.
* **Documentation:** Always include the `Bug Number` or `Ticket ID` in the `description` parameter when creating a patch. This makes future audits much easier.

This final query is essential for any DBA managing a production environment. It provides a "single pane of glass" view to see which SQL statements are currently being manipulated by `DBMS_SQLDIAG` and what hints are being injected.

---

# 📊 SQL Patch Audit & Monitoring Report

This query joins `DBA_SQL_PATCHES` with `V$SQL` to show you exactly which active cursors are currently under the influence of a patch.

```sql
SET LINESIZE 300
SET PAGESIZE 100
COL patch_name FOR A25
COL status FOR A10
COL sql_text FOR A60 TRUNC
COL hints FOR A50

SELECT 
    p.name AS patch_name,
    p.status,
    p.force_matching,
    s.sql_id,
    s.child_number,
    s.plan_hash_value,
    p.description,
    -- Extracting hints from the XML structure for readability
    EXTRACTVALUE(VALUE(h), '/hint') AS hint_details
FROM 
    dba_sql_patches p
LEFT JOIN 
    v$sql s ON p.name = s.sql_patch
CROSS JOIN 
    TABLE(XMLSEQUENCE(EXTRACT(XMLTYPE(p.comp_data), '/outline_data/hint'))) h
ORDER BY 
    p.created DESC, p.name;

```

---

### 📝 Report Breakdown

| Column | Importance |
| --- | --- |
| **PATCH_NAME** | The unique identifier you provided in `ACCEPT_SQL_PATCH` or `CREATE_SQL_PATCH`. |
| **FORCE_MATCHING** | If `YES`, the patch applies even if literal values in the SQL change. If `NO`, the SQL text must match exactly. |
| **SQL_ID** | If this is NULL, the patch exists but no sessions have executed that query since the last shared pool flush. |
| **HINT_DETAILS** | This shows you the actual internal fix (e.g., `IGNORE_OPTIM_EMBEDDED_HINTS`, `NO_QUERY_TRANSFORMATION`) applied by the Advisor. |

### 💡 Final Maintenance Tip: Transporting Patches

If you find a fix in **Test** using `DBMS_SQLDIAG` and want to move it to **Production** without re-running the Advisor:

1. Use `DBMS_SQLDIAG.EXPORT_SQL_TESTCASE` to bundle the environment.
2. Alternatively, use **Staging Tables**:
* `DBMS_SQLDIAG.CREATE_STGTAB_SQLPATCH` (Create a table to hold the patch).
* `DBMS_SQLDIAG.PACK_STGTAB_SQLPATCH` (Move the patch from Data Dictionary to the table).
* **DataPump** the table to the new DB.
* `DBMS_SQLDIAG.UNPACK_STGTAB_SQLPATCH` (Move the patch from the table to the new Data Dictionary).

