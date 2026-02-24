

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

Would you like me to create a script that combines **SQL Repair Advisor** with `DBMS_SQLDIAG` to automatically find and patch failing SQL?
