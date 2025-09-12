### Useful queries  
*Info about wallet*  

    select * from v$encryption_wallet  

*Info about encryption keys*  

    select con_id, key_id, tag, creator_instance_name, activating_instance_name, activating_pdbname, activation_time from v$encryption_keys order by con_id;  

### Migrate master encryption key for pdb
  
##### Export master encryption key from source  
    ADMINISTER KEY MANAGEMENT EXPORT ENCRYPTION KEYS WITH SECRET "<pass>" TO '<export_file_path>' FORCE KEYSTORE IDENTIFIED BY "<keystore_pass>";  
  
##### Drop autologin keystore on target  
    cd <wallet_dir>  
    mv cwallet.sso cwallet.sso.bkp  

*on cdb$root*  

    administer key management set keystore close;  
 
##### Open password wallet on target  

*on both cdb$root and pdb*  

    administer key management set keystore open identified by "<keystore_pass>";
 
##### Import the keys  
  
*on pdb*  

    ADMINISTER KEY MANAGEMENT IMPORT KEYS WITH SECRET "<pass>" FROM '<export_file_path>' FORCE KEYSTORE IDENTIFIED BY "<keystore_path>" with backup;  
  
*if needed, activate the specific imported key*  

    ADMINISTER KEY MANAGEMENT USE KEY '<key_id>' IDENTIFIED BY "<keystore_pass>" with backup;  
  
##### Create autologin wallet  
  
    administer key management create AUTO_LOGIN keystore from keystore '<password_keystore_path>' identified by "<keystore_pass>" ;  
    administer key management set keystore close identified by "<keystore_pass>";  
  
##### Restart pdb  
    alter pluggable database calycopy_pdb close;  
    alter pluggable database calycopy_pdb open;  
