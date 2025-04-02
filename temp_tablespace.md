How to identify the Reasons for the Increasing TEMP Tablespace Usage in Oracle 19c (Doc ID 3006775.1)

### SQL query for assessing both used and available space within the temporary tablespace  

    SELECT *  
    FROM  
    (SELECT a.tablespace_name,  
    SUM(a.bytes/1024/1024) allocated_mb  
    FROM dba_temp_files a  
    WHERE a.tablespace_name = 'TEMP'  
    GROUP BY a.tablespace_name  
    ) x,  
    (SELECT SUM(b.bytes_used/1024/1024) used_mb,  
    SUM(b.bytes_free /1024/1024) free_mb  
    FROM v$temp_space_header b  
    WHERE b.tablespace_name = 'TEMP'  
    GROUP BY b.tablespace_name  
    );  

### Retrieve active sessions utilizing TEMP tablespace  

    SELECT sysdate,a.username, a.sid, a.serial#, a.osuser,  
    (b.blocks*d.block_size)/1048576 MB_used, c.sql_text  
    FROM v$session a, v$tempseg_usage b, v$sqlarea c,  
    (select block_size from dba_tablespaces where tablespace_name='TEMP') d  
    WHERE b.tablespace = 'TEMP'  
    and a.saddr = b.session_addr  
    AND c.address= a.sql_address  
    AND c.hash_value = a.sql_hash_value  
    AND (b.blocks*d.block_size)/1048576 > 1024  
    ORDER BY b.tablespace, 6 desc;  


### Check TEMP Space Usage by SQL Statement

    SELECT sql_id, SUM(blocks) AS total_blocks FROM v$sort_usage GROUP BY sql_id;  

    SELECT sql_id, sql_text, disk_reads, buffer_gets, executions FROM v$sql WHERE sql_id IN ( SELECT sql_id FROM v$sort_usage WHERE tablespace = 'TEMP' ) ORDER BY disk_reads DESC;  


### Top 10 Sessions Consuming Significant TEMP Space

    cursor bigtemp_sids is  
    select * from (  
    select s.sid,  
    s.status,  
    s.sql_hash_value sesshash,  
    u.SQLHASH sorthash,  
    s.username,  
    u.tablespace,  
    sum(u.blocks*p.value/1024/1024) mbused ,  
    sum(u.extents) noexts,  
    nvl(s.module,s.program) proginfo,  
    floor(last_call_et/3600)||':'||  
    floor(mod(last_call_et,3600)/60)||':'||  
    mod(mod(last_call_et,3600),60) lastcallet  
    from v$sort_usage u,  
    v$session s,  
    v$parameter p  
    where u.session_addr = s.saddr  
    and p.name = 'db_block_size'  
    group by s.sid,s.status,s.sql_hash_value,u.sqlhash,s.username,u.tablespace,  
    nvl(s.module,s.program),  
    floor(last_call_et/3600)||':'||  
    floor(mod(last_call_et,3600)/60)||':'||  
    mod(mod(last_call_et,3600),60)  
    order by 7 desc,3)  
    where rownum < 11;  


### Historical TEMP usage  

    select min(sample_time), instance_number, user_id, session_id, session_serial#, sql_id,max(TEMP_SPACE_ALLOCATED)/(1024*1024*1024) gig from DBA_HIST_ACTIVE_SESS_HISTORY where sample_time between to_date('22.01.2025 10:20','DD.MM.YYYY HH24:MI') and to_date('22.01.2025 10:50','DD.MM.YYYY HH24:MI') group by instance_number, user_id, session_id, session_serial#, sql_id order by min(sample_time);  

    SELECT ASH.inst_id,  
      ASH.user_id,  
      ASH.session_id sid,  
      ASH.session_serial# serial#,  
      ASH.sql_id,  
      ASH.sql_exec_id,  
      ASH.sql_opname,  
      ASH.module,  
      MIN(sample_time) sql_start_time,  
      MAX(sample_time) sql_end_time,  
      ((CAST(MAX(sample_time) AS DATE)) - (CAST(MIN(sample_time) AS DATE))) * (3600*24) etime_secs ,  
      ((CAST(MAX(sample_time) AS DATE)) - (CAST(MIN(sample_time) AS DATE))) * (60*24) etime_mins ,  
      MAX(temp_space_allocated)/(1024*1024) max_temp_mb  
    FROM gv$active_session_history ASH  
    WHERE ASH.session_type = 'FOREGROUND'  
    AND ASH.sql_id        IS NOT NULL  
    AND sample_time BETWEEN to_timestamp('14-11-2024 00:00', 'DD-MM-YYYY HH24:MI') AND to_timestamp('21-11-2024 20:00', 'DD-MM-YYYY HH24:MI')  
      --and  ASH.sql_id = &amp;amp;SQL_ID  
    GROUP BY ASH.inst_id,  
      ASH.user_id,  
      ASH.session_id,  
      ASH.session_serial#,  
      ASH.sql_id,  
      ASH.sql_opname,  
      ASH.sql_exec_id,  
      ASH.module  
    HAVING MAX(temp_space_allocated) > 0;  

