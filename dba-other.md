### Read alert log and trace files using SQL  

**v$diag_alert_ext** for alert log - [doc](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-DIAG_ALERT_EXT.html)  
**v$diag_trace_file** for list of available trace files - [doc](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-DIAG_TRACE_FILE.html)  
**v$diag_trace_file_contents** for content of trace files - [doc](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-DIAG_TRACE_FILE_CONTENTS.html)  


Display alert log content  

    select originating_timestamp, message_text from v$diag_alert_ext order by originating_timestamp;  

Display count for different types and criticality level of messages in the alert log  

    select message_type, message_level, message_group, count(*) from v$diag_alert_ext group by message_type, message_level, message_group;  

Display messages only for critical errors  

    select originating_timestamp, message_text from v$diag_alert_ext where message_type=3 and message_level=1 order by originating_timestamp;

List all available trace files  

    select adr_home, trace_filename from v$diag_trace_file

Display content of specific trace file

    select payload from v$diag_trace_file_contents where trace_filename=’<trace_filename>’ order by line_number;
