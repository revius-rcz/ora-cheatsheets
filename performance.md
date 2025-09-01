[SQL Tuning Sets](https://github.com/revius-rcz/ora-cheatsheets/blob/main/performance.md#SQL-Tuning-Sets)  
[AWR](https://github.com/revius-rcz/ora-cheatsheets/blob/main/performance.md#Automatic-Workload-Repository)  
[SQL Plan Baselines](https://github.com/revius-rcz/ora-cheatsheets/blob/main/performance.md#SQL-Plan-Baselines)  
[Redo](https://github.com/revius-rcz/ora-cheatsheets/blob/main/performance.md#Redo)  

---

### SQL Tuning Sets

[List SQL Tuning Sets](https://github.com/revius-rcz/ora-cheatsheets/blob/main/performance.md#List-SQL-Tuning-Sets)  
[Delete SQL Tuning Set](https://github.com/revius-rcz/ora-cheatsheets/blob/main/performance.md#How-to-delete-existing-SQL-Tuning-Set)  
[Create SQL Tuning Set](https://github.com/revius-rcz/ora-cheatsheets/blob/main/performance.md#How-to-create-new-SQL-Tuning-Set)  
[Query SQL Tuning Set](https://github.com/revius-rcz/ora-cheatsheets/blob/main/performance.md#How-to-query-SQL-Tuning-Set)  

#### List SQL Tuning Sets
    select name, owner, created, statement_count from dba_sqlset;  

#### How to delete existing SQL Tuning Set
    execute dbms_sqltune.drop_sqlset(sqlset_name=>'sts_name>', sqlset_owner=>'<sts_owner>');

#### How to create new SQL Tuning Set
    execute dbms_sqltune.create_sqlset(sqlset_name=>'<sts_name>', sqlset_owner=>'<sts_owner>', description=>'<some_description>');  

[Populate SQL Tuning Set from cache](https://github.com/revius-rcz/ora-cheatsheets/blob/main/performance.md#Populate-SQL-Tuning-Set-with-queries-from-cache)  
[Populate SQL Tuning Set from AWR](https://github.com/revius-rcz/ora-cheatsheets/blob/main/performance.md#Populate-SQL-Tuning-Set-from-AWR)  
[Populate SQL Tuning Set from another SQL Tuning Set](https://github.com/revius-rcz/ora-cheatsheets/blob/main/performance.md#Populate-SQL-Tuning-Set-from-another-SQL-Tuning-Set)  

##### Populate SQL Tuning Set with queries from cache

###### All at once
    DECLARE  
    cur dbms_sqltune.SQLSET_CURSOR;  
    BEGIN  
    OPEN cur FOR  
       SELECT VALUE(P)  
       FROM dbms_sqltune.select_cursor_cache(basic_filter =>  
       'parsing_schema_name = ''<schema_name>'' and sql_text like ''SELECT%''') P;  
       dbms_sqltune.load_sqlset(sqlset_name => '<sts_name>', populate_cursor => cur);  
      CLOSE cur;  
    END;  
    /

###### Incrementally
    execute dbms_sqltune.capture_cursor_cache_sqlset(-
       sqlset_name     => '<sts_name>',-
       time_limit      => <seconds, how long to gather information>, -
       repeat_interval => <seconds, the interval between getting data>, -
       basic_filter    => 'parsing_schema_name = ''<schema_name>'' AND elapsed_time > <time>');

##### Populate SQL Tuning Set from AWR
    DECLARE
    cur dbms_sqltune.sqlset_cursor;
    BEGIN
    OPEN cur FOR
      SELECT VALUE(P) 
      FROM dbms_sqltune.select_workload_repository(begin_snap=><start_snap_id>, end_snap=> <end_snap_id>, basic_filter => 'sql_text like ''SELECT%''   
       and parsing_schema_name = ''<schema_name>''' ) P; 
      dbms_sqltune.load_sqlset (sqlset_name => '<sts_name>', populate_cursor => cur); 
    END;
    /

##### Populate SQL Tuning Set from another SQL Tuning Set
    DECLARE cur dbms_sqltune.sqlset_cursor; 
    BEGIN 
    OPEN cur FOR
         SELECT VALUE (P)
         FROM table(DBMS_SQLTUNE.SELECT_SQLSET(sqlset_name =>'<sts_name_source>', basic_filter => 'parsing_schema_name = ''<schema_name>'' AND elapsed_time > <time>')) P;
      DBMS_SQLTUNE.LOAD_SQLSET(sqlset_name => '<sts_name_target>', populate_cursor => cur);
    CLOSE cur;
    END;
    /

#### How to query SQL Tuning Set
    select executions, cpu_time/1000 cpu_in_ms, elapsed_time/1000 elapsed_in_ms, sql_id, substr(sql_text,1,80) 
    from dba_sqlset_statements
    where sqlset_name like '<sts_name>' and sqlset_owner='<sts_owner>'
    ORDER BY elapsed_time desc;


### Automatic Workload Repository  
  
[List AWR snapshots](https://github.com/revius-rcz/ora-cheatsheets/blob/main/performance.md#List-AWR-snapshots)  
[PDB related configuration for AWR](https://github.com/revius-rcz/ora-cheatsheets/blob/main/performance.md#PDB-related-configuration-for-AWR)  
[AWR configuration](https://github.com/revius-rcz/ora-cheatsheets/blob/main/performance.md#AWR-configuration)  
[How to query AWR repository](https://github.com/revius-rcz/ora-cheatsheets/blob/main/performance.md#How-to-query-AWR-repository)
  
#### List AWR snapshots  

    SELECT snap_id, begin_interval_time begin_snap, end_interval_time end_snap FROM dba_hist_snapshot ORDER BY snap_id;  

#### PDB related configuration for AWR  

    select inst_id, con_id, name, display_value from gv$system_parameter where name in ('awr_pdb_autoflush_enabled', 'awr_snapshot_time_offset') order by inst_id, con_id;  

#### AWR configuration  

    select * from dba_HIST_WR_CONTROL where dbid in (select dbid from v$containers where name != 'PDB$SEED');  

#### How to query AWR repository  


### SQL Plan Baselines

List SQL Plan Baselines  

    select * from dba_sql_plan_baselines;  


### Redo

#### Historical Redo Per Second

    SELECT * FROM DBA_HIST_SYSMETRIC_HISTORY WHERE metric_name='Redo Generated Per Sec' ORDER BY snap_id DESC;  

#### Redo log switches map  

    set pages 999 lines 400  
    col h0 format 999  
    col h1 format 999  
    col h2 format 999  
    col h3 format 999  
    col h4 format 999  
    col h5 format 999  
    col h6 format 999  
    col h7 format 999  
    col h8 format 999  
    col h9 format 999  
    col h10 format 999  
    col h11 format 999  
    col h12 format 999  
    col h13 format 999  
    col h14 format 999  
    col h15 format 999  
    col h16 format 999  
    col h17 format 999  
    col h18 format 999  
    col h19 format 999  
    col h20 format 999  
    col h21 format 999  
    col h22 format 999  
    col h23 format 999  
    SELECT TRUNC (first_time) "Date", inst_id, TO_CHAR (first_time, 'Dy') "Day",  
    COUNT (1) "Total",  
    SUM (DECODE (TO_CHAR (first_time, 'hh24'), '00', 1, 0)) "h0",  
    SUM (DECODE (TO_CHAR (first_time, 'hh24'), '01', 1, 0)) "h1",  
    SUM (DECODE (TO_CHAR (first_time, 'hh24'), '02', 1, 0)) "h2",  
    SUM (DECODE (TO_CHAR (first_time, 'hh24'), '03', 1, 0)) "h3",  
    SUM (DECODE (TO_CHAR (first_time, 'hh24'), '04', 1, 0)) "h4",  
    SUM (DECODE (TO_CHAR (first_time, 'hh24'), '05', 1, 0)) "h5",  
    SUM (DECODE (TO_CHAR (first_time, 'hh24'), '06', 1, 0)) "h6",  
    SUM (DECODE (TO_CHAR (first_time, 'hh24'), '07', 1, 0)) "h7",  
    SUM (DECODE (TO_CHAR (first_time, 'hh24'), '08', 1, 0)) "h8",  
    SUM (DECODE (TO_CHAR (first_time, 'hh24'), '09', 1, 0)) "h9",  
    SUM (DECODE (TO_CHAR (first_time, 'hh24'), '10', 1, 0)) "h10",  
    SUM (DECODE (TO_CHAR (first_time, 'hh24'), '11', 1, 0)) "h11",  
    SUM (DECODE (TO_CHAR (first_time, 'hh24'), '12', 1, 0)) "h12",  
    SUM (DECODE (TO_CHAR (first_time, 'hh24'), '13', 1, 0)) "h13",  
    SUM (DECODE (TO_CHAR (first_time, 'hh24'), '14', 1, 0)) "h14",  
    SUM (DECODE (TO_CHAR (first_time, 'hh24'), '15', 1, 0)) "h15",  
    SUM (DECODE (TO_CHAR (first_time, 'hh24'), '16', 1, 0)) "h16",  
    SUM (DECODE (TO_CHAR (first_time, 'hh24'), '17', 1, 0)) "h17",  
    SUM (DECODE (TO_CHAR (first_time, 'hh24'), '18', 1, 0)) "h18",  
    SUM (DECODE (TO_CHAR (first_time, 'hh24'), '19', 1, 0)) "h19",  
    SUM (DECODE (TO_CHAR (first_time, 'hh24'), '20', 1, 0)) "h20",  
    SUM (DECODE (TO_CHAR (first_time, 'hh24'), '21', 1, 0)) "h21",  
    SUM (DECODE (TO_CHAR (first_time, 'hh24'), '22', 1, 0)) "h22",  
    SUM (DECODE (TO_CHAR (first_time, 'hh24'), '23', 1, 0)) "h23",  
    ROUND (COUNT (1) / 24, 2) "Avg"  
    FROM gv$log_history  
    WHERE thread# = inst_id  
    AND first_time > sysdate -7  
    GROUP BY TRUNC (first_time), inst_id, TO_CHAR (first_time, 'Dy')  
    ORDER BY 1,2;  
