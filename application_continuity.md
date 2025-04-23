The Application continuity is the feature built on top of RAC. When it is configured and drain timeout is reached, the connection is still terminated, but the operation is not interrupted - it is moved and replayed on available instance. This way instance unavailability can be hidden and be unnoticed by the application, it will only experience longer execution of the replayed operations. What is worth to notice, the Application Continuity can hide failures from application in following scenarios:  

* planned maintenance tasks on RAC node  
* unplanned failures  
* Data Guard switchover  
  
  
So it is not only for planned activities, but can also maintain application availability during unplanned incidents involving subset of RAC instances. 
  
  
  
Application Continuity - Planned Maintenance Demo (short video) - https://videohub.oracle.com/media/Application%20Continuity%20-%20Planned%20Maintenance%20Demo/1_xbd2fstx  

How Draining and Application Continuity Work for Maintenance with Oracle RAC - https://database-heartbeat.com/2023/11/01/draining-ac-rac/  
  
  
  
In short, steps to configure Application Continuity involve:  
  
**DBA steps**  
  
1. Setup database services  
2. Open port (default 6200)  
3. Grant keep on mutables  
  
**Developer or App Owner steps**  
  
1. Configure to use recommended connection string (tnsnames or ldap)
2. Update client to most recently available long-term version
3. Explicit request boundaries or Connection tests
4. Evaluate coverage (AWR, ACCHK)




Get started with Oracle Application Continuity - https://database-heartbeat.com/2023/03/06/get-started-appcon/

Application Continuity checklist - checklist-ac-6676160.pdf (oracle.com)



How to Configure Application Continuity, part 1 (video) - https://videohub.oracle.com/media/Exadata%20Master%20Class%20%7C%20How%20to%20Configure%20Application%20Continuity%2C%20part%201%20%E2%80%93%20What%20does%20the%20DBA%20need%20to%20do/1_enmfc9zo

How to Configure Application Continuity, part 2 (video) - https://videohub.oracle.com/media/Exadata%20Master%20Class%20%7C%20How%20to%20Configure%20Application%20Continuity%2C%20part%202/1_e7o4fh15

Services configuration


srvctl add service -d ac_rac -s sqlac -commit_outcome true -failovertype transaction -failover_restore level1 -preferred ac1 -available ac2 -pdb sales -replay_init_time 3600 -drain_timeout 60 -stopoption immediate -role primary




The following service properties are significant in context of Application Continuity:



notification

TRUE | FALSE

Specify a value of TRUE to enable Fast Application Notification (FAN) for Oracle Call Interface (OCI) connections



failover_restore

NONE|LEVEL1|AUTO

For Application Continuity, when you set the -failover_restore parameter, session states are restored before replaying. Use LEVEL1 for ODP.NET and Java with Application Continuity to restore the initial state.

Set this parameter to AUTO to enable Transparent Application Continuity to restore session states.

For OCI applications using TAF or Application Continuity, setting -failover_restore to LEVEL1 restores the current state. If the current state differs from the initial state, then a TAF callback is required. This restriction applies only to OCI.



failovertype

NONE|SESSION BASIC|TRANSACTION

The failover type for a service



failovermethod

NONE|BASIC

Specify the TAF failover method (for backward compatibility only).



commit_outcome 

TRUE | FALSE

Enable Transaction Guard; when set to TRUE, the commit outcome for a transaction is accessible after the transaction's session fails due to a recoverable outage.



replay_init_time

For Application Continuity, this parameter specifies the difference between the time, in seconds, of original execution of the first operation of a request and the time that the replay is ready to start after a successful reconnect. Application Continuity will not replay after the specified amount of time has passed. This parameter is intended to avoid the unintentional execution of a transaction when a system is recovered after a long period. The default is 5 minutes (300). The maximum value is 24 hours (86400). If the -failover_type parameter is not set to TRANSACTION, then you cannot use this parameter.



session_state

STATIC | DYNAMIC | AUTO

For Application Continuity; this parameter describes how the non-transactional session state is changed by the application within a request. Examples of session state are NLS settings, optimizer preferences, event settings, PL/SQL global variables, and temporary tables. For Transparent Application Continuity session_state is always set to AUTO. Session state is tracked automatically.

This parameter is considered only if -failovertype is set to TRANSACTION for Application Continuity or AUTO for Transparent Application Continuity.

If failover_type is set to TRANSACTION, then Oracle recommends a value of DYNAMIC for session_state.
If failover_type is set to AUTO, then session_state defaults to AUTO.
If failover_type is set to any value other than TRANSACTION or AUTO, then the value of session_state is not set.
If non-transactional values change after the request starts, then set this parameter to either DYNAMIC or AUTO. Most applications should use DYNAMIC or AUTO mode.



retention

For Transaction Guard (with the -commit_outcome parameter set to TRUE); this parameter determines the amount of time (in seconds) that the commit outcome is retained in the database.





More about srvctl and service properties - https://docs.oracle.com/en/database/oracle/oracle-database/19/racad/server-control-utility-reference.html?source=%3Aso%3Ach%3Aor%3Aawr%3A%3A%3A&source=%3Aso%3Ach%3Aor%3Aawr%3A%3A%3A&source=%3Aso%3Ach%3Aor%3Aawr%3A%3A%3A#GUID-C2D37BAB-DA98-49B4-A777-F2B3AA8D2E7A

How to create Oracle Database Services for High Availability - https://database-heartbeat.com/2023/04/18/create-service-ha/



Connection string
Recommended options in connections string in context of Application Continuity:



description = (

              (connect_timeout=90)

              (retry_count=20) (retry_delay=3)

              (transport_connect_timeout=3)

              (address_list =

                             (load_balance=on)

                             (address = (protocol = tcp) (host=primary-scan) (port=1521)))

              (address_list =

                             (load_balance=on)

                             (address = (protocol = tcp) (host=second-scan) (port=1521)))

(connect_data=(service_name=gold)))





retry_count

To specify the number of times an ADDRESS list is traversed before the connection attempt is terminated.

retry_delay

To specify the delay in seconds between subsequent retries for a connection. This parameter works in conjunction with RETRY_COUNT parameter.

connect_timeout

To specify the timeout duration in ms, sec, or min for a client to establish an Oracle Net connection to an Oracle database.

transport_connect_timeout

To specify the transport connect timeout duration in ms, sec, or min for a client to establish an Oracle Net connection to an Oracle database.



More about these parameters - https://docs.oracle.com/en/database/oracle/oracle-database/19/netrf/local-naming-parameters-in-tns-ora-file.html#GUID-B1EEB283-CBD7-4ED8-9B94-AB890660EB3C

Application Continuity Health Check (ACCHK)


Application continuity is configured and delivered on service level. So some workload is protected while other might not. In order to monitor coverage and to see which operations are not protected, Oracle provides utility ACCHK - Application Continuity Health Check.



Demo about ACCHK to download (video) - https://www.oracle.com/a/otn/docs/downloads/acchk_demo_video.mp4



More about usage of ACCHK - https://database-heartbeat.com/2024/03/21/acchk/ 

More about package DBMS_APP_CONT_ADMIN - https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/dbms_app_cont_admin1.html

More about package DBMS_APP_CONT_REPORT - https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/dbms_app_cont_report.html



AWR reports also contains information about Application Continuity.

Licensing


RAC or ADG license required
Can be single-instance if with ADG
Must deploy with oracle Clusterware (for full FAN support)
ADG - configure with Max Availability (recommended, not requirement)
Additional resources


Fast Application Notification (FAN) - Oracle Notification Client and FAN

Continuous Availability for Applications -https://docs.oracle.com/en/database/oracle/oracle-database/19/haovw/configuring-continuous-availability-applicationsconfiguring-continuous-availability-applicati.html#GUID-5EBF37EA-48AB-4508-A14E-86A2583A24BF 
