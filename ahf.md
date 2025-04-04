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


#### Include Oracle Security Assessment Tool checks  
It is possible to run also checks from Oracle Security Assessment Tool (SAT)  

    ahfctl compliance -a -dball -noupgrade -includeprofile security  


#### Exachk execution before patching  
    ahfctl compliance -profile prepatch  


#### Compare exachk reports  
    ahfctl compliance -diff <report1>.zip <report2>.zip  
