### ExaCC: run commands on cells  

#### Get list of cells  
    cat /etc/oracle/cell/network-config/cellip.ora  

#### Get password for user  
    /opt/exacloud/get_cs_data.py –-dataonly  

#### Get cluster name  
    crsctl get cluster name  

#### Username  

The username is in format **cloud_user_<cluster_name>**, where <cluster_name> was retrieved in previous step  


#### Run command with exacli  

    exacli -l <username> -c <cell_ip> -e <command>  
