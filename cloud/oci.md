#### List compartments  
    oci iam compartment list --all --query "data[*].[name, description, id]" --output table --compartment-id-in-subtree true  
  
#### List DB images
    oci db database-software-image list --compartment-id <compartment_ocid> --image-type DATABASE_IMAGE --sort-by TIMECREATED --sort-order ASC --query "data[*].{\"ImageName\":\"display-name\", Created:\"time-created\", Version:\"patch-set\", Ocid:id}" --output table --all  

#### List GI images  
    oci db database-software-image list --compartment-id <compartment_ocid> --image-type GRID_IMAGE --sort-by TIMECREATED --sort-order ASC --query "data[*].{\"ImageName\":\"display-name\", Created:\"time-created\", Version:\"patch-set\", Ocid:id}" --output table --all
  
#### List VM Clusters  
    oci db vm-cluster list --compartment-id <compartment_ocid> --query "data[*].[\"display-name\", id]" --output table  
  
#### Create database home
    oci db db-home create --display-name <display_name> --database-software-image-id <image_ocid> --vm-cluster-id <vmcluster_ocid> --db-version <version>  

#### List database homes
    oci db db-home list --compartment-id <compartment_id> --vm-cluster-id <vmcluster_ocid> --query "data[].[\"display-name\",\"db-version\",\"lifecycle-state\",\"is-unified-auditing-enabled\",\"db-home-location\", id]" --output table --sort-order ASC --sort-by DISPLAYNAME  

#### Delete database home
    oci db db-home delete --db-home-id <dbhome_ocid>  

#### Run precheck for GI patching using image
    oci db vm-cluster update --vm-cluster-id <vmcluster_ocid> --gi-image-id <image_ocid> --patch-action PRECHECK  

#### Run GI patching using image
    oci db vm-cluster update --vm-cluster-id <vmcluster_ocid> --gi-image-id <image_ocid> --patch-action APPLY  
