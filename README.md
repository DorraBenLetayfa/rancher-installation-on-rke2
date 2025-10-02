# rancher-installation-on-rke2

1- provision 3 machines for rke2              
OS: opensuse leap 15            
CPU: 4                     
RAM: 16GB                            
DISK: 80GB                      

2- configure firewall on machines 

3- Configure DNS on bastion

4- test DNS resolution for 3 nodes

5- deploy RKE2

6- config NGINX 

7- Install RANCHER 

# troubleshooting 
in case you face issues: 
- get the rancher pods 
- describe the not working pod
- if health probe issue : change the deploy to port 443 and HTTPS for health probes 
- restart the rke2-server service if needed 

