podman seatch <image_name>  
podman pull <image>  
podman images  
podman run -it <image>  
podman run -it --rm <image>  
podman run --name <container_name> -p <ext_port>:<int_port> <container_image>   
podman ps  
podman ps -a  
podman start <container_name>  
podman inspect <container_name>  
podman port <container_name>  
podman stop <container_name>  
podman rm <container_name>  
podman rmi <container_name>  


DOCKERFILE  

vim Dockerfile  
podman build -t <image_name>  
podman run --name <container_name> -p <ext_port>:<int_port> <image_name>:<tag>  


podman pod create --name <pod_name>  
podman pod ls  
podman ps -a --pod  
podman run -dt --pod <pod_name> <container_name>  
podman pod start <pod_name>  
podman pod stop <pod_name>  
podman pod rm <pod_name>  
