How to identify the Reasons for the Increasing TEMP Tablespace Usage in Oracle 19c (Doc ID 3006775.1)

### SQL query for assessing both used and available space within the temporary tablespace  

    select a.tablespace_name tablespace,
    d.TEMP_TOTAL_MB,
    sum (a.used_blocks * d.block_size) / 1024 / 1024 TEMP_USED_MB,
    d.TEMP_TOTAL_MB - sum (a.used_blocks * d.block_size) / 1024 / 1024 TEMP_FREE_MB
    from v$sort_segment a,
    (
    select b.name, c.block_size, sum (c.bytes) / 1024 / 1024 TEMP_TOTAL_MB
    from v$tablespace b, v$tempfile c
    where b.ts#= c.ts#
    group by b.name, c.block_size
    ) d
    where a.tablespace_name = d.name
    group by a.tablespace_name, d.TEMP_TOTAL_MB;

### Check TEMP Space Usage by session

    SELECT b.tablespace,
    ROUND(((b.blocks*p.value)/1024/1024),2)||'M' AS temp_size,
    a.inst_id as Instance,
    a.sid||','||a.serial# AS sid_serial,
    NVL(a.username, '(oracle)') AS username,
    a.program, a.status, a.sql_id
    FROM gv$session a, gv$sort_usage b, gv$parameter p
    WHERE p.name = 'db_block_size' AND a.saddr = b.session_addr
    AND a.inst_id=b.inst_id AND a.inst_id=p.inst_id
    ORDER BY temp_size desc;


### Check TEMP Space Usage by SQL Statement

    SELECT sql_id, SUM(blocks) AS total_blocks FROM v$sort_usage GROUP BY sql_id;  

_

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

_

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

