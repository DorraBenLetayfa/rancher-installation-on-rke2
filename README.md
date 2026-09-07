# 1-Provide an RKE2 cluster with 3 nodes:
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
   
  # 2-get license key : C51FR-AVWZH-A31RA for INTERNAL-USE-ONLY-ff33-d2b8
  

  # 3- install longhorn 

  # 4- Install Observability
  helm repo add suse-observability https://charts.rancher.com/server-charts/prime/suse-observability
  helm repo update
  vim values.yaml (get content from docs)
  helm upgrade --install \
    --namespace suse-observability \
    --create-namespace \
    --values values.yaml \
    suse-observability \
    suse-observability/suse-observability
  
# 5- after deployment create an ingress 
ingress:
  enabled: true
  ingressClassName: traefik
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
  hosts:
    - host: suse-observability.MY_DOMAIN

helm upgrade --namespace suse-observability --reuse-values --values ingress_values.yaml suse-observability suse-observability/suse-observability
and access the URL.

# 6-Install agent on managed cluster

Configure TLS for agent:
cat agent-tls-values.yaml
global:
  customCertificates:
    enabled: true
    configMapName: observability-ca

  skipSslValidation: false

stackstate:
  url: https://observability.example.com/receiver/stsAgent

checksAgent:
  skipSslValidation: false

clusterAgent:
  skipSslValidation: false

nodeAgent:
  skipSslValidation: false

logsAgent:
  skipSslValidation: false


helm upgrade suse-observability-agent \
  suse-observability/suse-observability-agent \
  -n suse-observability-agent \
  --reuse-values \
  --values agent-tls-values.yaml

  follow steps on Observability UI to install agent. 

remove rancher-agent:
sudo systemctl disable --now rancher-system-agent

curl https://raw.githubusercontent.com/rancher/system-agent/main/system-agent-uninstall.sh \
  -o system-agent-uninstall.sh

sudo sh system-agent-uninstall.sh
