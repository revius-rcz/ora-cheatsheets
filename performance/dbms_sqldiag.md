
---

# Oracle DBMS_SQLDIAG Cheatsheet

`DBMS_SQLDIAG` is a powerful administrative package used primarily for **SQL Patching**, diagnosing SQL failures, and creating **SQL Test Cases**. It allows DBAs to influence the Optimizer without changing application source code.

## 1. What is a SQL Patch?

A **SQL Patch** is a collection of optimizer hints stored in the data dictionary and associated with a specific `SQL_ID` or SQL text.

* **Use Case:** When a query performs poorly but the source code cannot be modified (e.g., vendor software).
* **Mechanism:** When the Optimizer parses a statement, it checks for an associated patch and transparently applies the hints.
* **Stability:** Unlike Baselines, Patches are "lightweight" and don't lock a full plan; they only apply specific corrective hints (e.g., `INDEX`, `LEADING`, `PARALLEL`).

---

## 2. Core Functionalities

| Feature | Description |
| --- | --- |
| **SQL Patching** | Create, drop, and alter hints applied to a specific `SQL_ID`. |
| **SQL Test Case** | Export/Import all metadata, stats, and environment data to reproduce a SQL issue on another DB. |
| **Diagnostic Tracing** | Identify why a SQL statement failed with an error (e.g., ORA-00600). |

---

## 3. Usage Examples: SQL Patches

### A. Create a Patch for a specific SQL_ID

This applies hints to a query currently sitting in the Library Cache.

```sql
DECLARE
  v_patch_name VARCHAR2(30);
BEGIN
  v_patch_name := DBMS_SQLDIAG.CREATE_SQL_PATCH(
    sql_id      => '8998pf78abc12',
    hints       => 'INDEX(@SEL$1 "EMPLOYEES" "EMP_IDX") USE_NL(e d)',
    name        => 'EMP_NL_PATCH',
    description => 'Force Nested Loops and specific index usage'
  );
END;
/

```

### B. Create a Patch via SQL Text (For AWR/Historical SQL)

Use this if the SQL is no longer in the cache but you have the text.

```sql
BEGIN
  SYS.DBMS_SQLDIAG.CREATE_SQL_PATCH(
    sql_text    => 'SELECT * FROM orders WHERE status = :1',
    hints       => 'PARALLEL(4)',
    name        => 'ORDER_PARALLEL_PATCH'
  );
END;
/

```

### C. Disable or Drop a Patch

If the patch is no longer needed or you want to test the original plan.

```sql
-- Disable the patch temporarily
BEGIN
  DBMS_SQLDIAG.ALTER_SQL_PATCH(
    name            => 'EMP_NL_PATCH',
    attribute_name  => 'STATUS',
    value           => 'DISABLED'
  );
END;
/

-- Drop the patch permanently
BEGIN
  DBMS_SQLDIAG.DROP_SQL_PATCH(name => 'EMP_NL_PATCH');
END;
/

```

---

## 4. Useful Data Dictionary Views

| View | Purpose |
| --- | --- |
| **`DBA_SQL_PATCHES`** | Shows patch name, status (ENABLED/DISABLED), and the hint text. |
| **`V$SQL`** | Check the `SQL_PATCH` column to see if a cursor is actively using a patch. |
| **`V$SQL_DIAG_REPOSITORY`** | Contains history of diagnostic incidents and SQL failures. |

---

## 5. Essential Monitoring Queries

### List all active SQL Patches

```sql
SELECT 
    name, 
    status, 
    force_matching, 
    last_modified, 
    description 
FROM dba_sql_patches;

```

### Verify if a query is using a Patch

Check the `SQL_PATCH` column. If it is NOT NULL, the patch is active for that cursor.

```sql
SELECT 
    sql_id, 
    child_number, 
    plan_hash_value, 
    sql_patch 
FROM v$sql 
WHERE sql_id = '&your_sql_id';

```

### Extract hints from an existing Patch

Hints are stored in a CLOB in the `comp_data` column.

```sql
SELECT 
    name, 
    extractValue(value(t), '/hint') as hint
FROM 
    dba_sql_patches, 
    table(xmlsequence(extract(xmltype(comp_data), '/outline_data/hint'))) t
WHERE name = 'EMP_NL_PATCH';

```

---

## 6. Pro-Tips

* **Force Matching:** By default, patches only match if the SQL text is identical. You can set `force_matching => TRUE` in `CREATE_SQL_PATCH` to ignore literal values (similar to `CURSOR_SHARING=FORCE`).
* **Validation:** Always run `EXPLAIN PLAN` or check `DBMS_XPLAN.DISPLAY_CURSOR`. A successful patch will show a "Note" section at the bottom:
> *Note: SQL patch "PATCH_NAME" used for this statement*


* **Security:** Executing `DBMS_SQLDIAG` typically requires the `ADMINISTER SQL MANAGEMENT OBJECT` privilege.

---

Would you like me to generate a script to automate the export of a **SQL Test Case** using this package?
