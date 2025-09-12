### Resources  

DBMS_DATAPUMP - [doc](https://docs.oracle.com/en/database/oracle/oracle-database/21/arpls/DBMS_DATAPUMP.html)  
Using Data Pump API - [doc](https://docs.oracle.com/en/database/oracle/oracle-database/19/sutil/using-oracle_datapump-api.html#GUID-EAD7AE4B-778A-4369-9842-68E026409045)  


### Basic steps
  
1. To create an Oracle Data Pump job and its infrastructure, run the DBMS_DATAPUMP.OPEN procedure.
When you run the procedure, the Oracle Data Pump job is started.  
        `DBMS_DATAPUMP.OPEN()`  
3. Define any parameters for the job.  
       `DBMS_DATAPUMP.ADD_FILE()`  
       `DBMS_DATAPUMP.METADATA_FILTER()`  
       `DBMS_DATAPUMP.DATA_FILTER()`  
       `DBMS_DATAPUMP.METADATA_REMAP()`  
       `DBMS_DATAPUMP.DATA_REMAP()`  
       `DBMS_DATAPUMP.METADATA_TRANSFORM()`  
5. Start the job.  
       `DBMS_DATAPUMP.START_JOB`
7. (Optional) Monitor the job until it completes.  
       `DBMS_DATAPUMP.GET_STATUS`   
9. (Optional) Detach from the job, and reattach at a later time.  
       `DBMS_DATAPUMP.DETACH()`  
10. (Optional) Stop the job.  
       `DBMS_DATAPUMP.STOP_JOB()`  
11. (Optional) Restart the job, if desired.  


### Examples  

[Examples from documentation](https://docs.oracle.com/en/database/oracle/oracle-database/19/sutil/using-oracle_datapump-api.html#GUID-5AAC848B-5A2B-4FD1-97ED-D3A048263118)
