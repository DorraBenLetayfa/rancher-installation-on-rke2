#1-Provide an RKE2 cluster with 3 nodes:
each node has all the roles, 8vCPU, 16GB each. (150-HA profile)
   linux kernel version 4.4+
   Helm 3.0+
   kubectl on nodes 
   containerd(prefered)/docker/cri-o
   curl
   jq 
   openSSL
   for architecture choose a profile from here: https://documentation.suse.com/cloudnative/suse-observability/latest/en/setup/install-stackstate/requirements.html#_resource_requirements
 
   use TLS for secure access to Observability
   DNS configured
   apply network policies if needed  to control inter-components communication
  
  #2-get license key

  #3- install longhorn 

  #4- Install Observability
