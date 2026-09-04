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
   go to the cluster and find copy kubeconfig file, mkdir .kube, vim .kube/config and paste the copied content, and chmod 700 .kube/config
   export PATH=$PATH:/var/lib/rancher/rke2/bin
   kubectl get pods -A
   
  #2-get license key : INTERNAL-USE-ONLY-1602-fa7f

  #3- install longhorn 

  #4- Install Observability
  helm repo add suse-observability https://charts.rancher.com/server-charts/prime/suse-observability
  helm repo update
  vim values.yaml (get content from docs)
  helm upgrade --install \
    --namespace suse-observability \
    --create-namespace \
    --values values.yaml \
    suse-observability \
    suse-observability/suse-observability
  
  


remove rancher-agent:
sudo systemctl disable --now rancher-system-agent

curl https://raw.githubusercontent.com/rancher/system-agent/main/system-agent-uninstall.sh \
  -o system-agent-uninstall.sh

sudo sh system-agent-uninstall.sh
