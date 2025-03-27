[SQL Tuning Sets](https://github.com/revius-rcz/ora-cheatsheets/blob/main/performance.md#SQL-Tuning-Sets)  
[Redo](https://github.com/revius-rcz/ora-cheatsheets/blob/main/performance.md#Redo)  

### SQL Tuning Sets

[Delete SQL Tuning Set](https://github.com/revius-rcz/ora-cheatsheets/blob/main/performance.md#How-to-delete-existing-SQL-Tuning-Set)  
[Create SQL Tuning Set](https://github.com/revius-rcz/ora-cheatsheets/blob/main/performance.md#How-to-create-new-SQL-Tuning-Set)  
[Query SQL Tuning Set](https://github.com/revius-rcz/ora-cheatsheets/blob/main/performance.md#How-to-query-SQL-Tuning-Set)  


#### How to delete existing SQL Tuning Set
    execute dbms_sqltune.drop_sqlset(sqlset_name=>'sts_name>', sqlset_owner=>'<sts_owner>');

#### How to create new SQL Tuning Set
    execute dbms_sqltune.create_sqlset(sqlset_name=>'<sts_name>', sqlset_owner=>'<sts_owner>', description=>'<some_description>');  

[Populate SQL Tuning Set from cache](https://github.com/revius-rcz/ora-cheatsheets/blob/main/performance.md#Populate-SQL-Tuning-Set-with-queries-from-cache)  
[Populate SQL Tuning Set from AWR](https://github.com/revius-rcz/ora-cheatsheets/blob/main/performance.md#Populate-SQL-Tuning-Set-from-AWR)  
[Populate SQL Tuning Set from AWR](https://github.com/revius-rcz/ora-cheatsheets/blob/main/performance.md#Populate-SQL-Tuning-Set-from-another-SQL-Tuning-Set)  

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


### Redo

#### Historical Redo Per Second

    SELECT * FROM DBA_HIST_SYSMETRIC_HISTORY WHERE metric_name='Redo Generated Per Sec' ORDER BY snap_id DESC;  


