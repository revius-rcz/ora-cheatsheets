
### AHF update

1. Get staging directory (as root):  
    ``ahfctl getupgrade | grep autoupgrade.swstage | cut -d: -f2 | sed 's/ //g' ``   
Return error if empty string received.  
2. Get status and version of AHF before the upgrade(as root):  
``ahfctl statusahf -tfa``  
3. Check if staging directory exists and create it if it doesn’t  
4. Remove all existing content of staging directory  
5. Get AHF package, copy it to staging directory and unzip it.  
Here we need to discuss what are the options where we could put the package in the first place, so it would be available to Ansible. Perhaps Artifactory ?  
6. Run the upgrade (as root):  
``ahfctl upgrade``  
7. Check status after the upgrade (as root):  
``ahfctl statusahf -tfa``  
8. Clean up staging directory  



### Exachk


#### Checks to exclude  
While running exachk, following checks can be skipped as they are of no importance to NCI environments:  


| CHECK ID | DESCRIPTION |
| -------- | ----------- |
| D9321653EB0CA241E053D498EB0A6CE1 | E-Business Suite: ETCC RDBMS Patch recommendations |
| A98AC77CA837A489E040E50A1EC014A6 | os_authent_prefix |
| 0E74F80547F37AF7E0639912F50A7B06 | Verify SYS and SYSTEM accounts are using default profile |
| ED2A765922CCB9A0E053D298EB0A512A | Verify if cluster has the latest dbaastools rpm version |
| AACE397A971608F0E040E50A1EC03350 | Verify Platform Configuration and Initialization Parameters for Consolidation |
| E34A916A4A5E6B15E04312C0E50A509B | Verify control_file_record_keep_time value is in recommended range |
| 082611CF6909295EE05313C0E50A6BCF | Ensure db_unique_name is unique across the enterprise |
| FF4C0F0453F19778E0539A12F50A3007 | Verify number of inactive patches for Grid Infrastructure home |
| FF397079FD8D96B3E0539912F50AC837 | Verify number of inactive patches for database home |


#### In order to skip the checks, the exachk can be run like this:  

    ahfctl compliance -a -dball -noupgrade -excludecheck D9321653EB0CA241E053D498EB0A6CE1,A98AC77CA837A489E040E50A1EC014A6,0E74F80547F37AF7E0639912F50A7B06,ED2A765922CCB9A0E053D298EB0A512A,AACE397A971608F0E040E50A1EC03350,E34A916A4A5E6B15E04312C0E50A509B,082611CF6909295EE05313C0E50A6BCF,FF4C0F0453F19778E0539A12F50A3007,FF397079FD8D96B3E0539912F50AC837  

#### Default excluded checks  
Print default excluded checks:  
``ahf configuration get --property ahf.compliance.excluded-check-ids``  

To exclude check id:  
``ahf configuration set --property ahf.compliance.excluded-check-ids --value F3B1F6668255A1E2E053D298EB0A88A7,DDAC922A1762467BE053D498EB0A9DF6,D9321653EB0CA241E053D498EB0A6CE1,A98AC77CA837A489E040E50A1EC014A6,0E74F80547F37AF7E0639912F50A7B06,ED2A765922CCB9A0E053D298EB0A512A,AACE397A971608F0E040E50A1EC03350,082611CF6909295EE05313C0E50A6BCF,FF4C0F0453F19778E0539A12F50A3007,FF397079FD8D96B3E0539912F50AC837``  

To unset all excluded checks:  
``ahf configuration reset --property ahf.compliance.excluded-check-ids``  

To unset selected check ids:  
``ahf configuration unset --property ahf.compliance.excluded-check-ids --value FB7BBD8E8BC78623E0539812F50A0B81 ``  

#### Include Oracle Security Assessment Tool checks  
It is possible to run also checks from Oracle Security Assessment Tool (SAT)  

    ahfctl compliance -a -dball -noupgrade -includeprofile security  


#### Exachk execution before patching  
    ahfctl compliance -profile prepatch  


#### Compare exachk reports  
    ahfctl compliance -diff <report1>.zip <report2>.zip  


### TFA  
