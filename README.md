set unique hostaname on all rke2 nodes.

"Edit the chrony configuration file: Open the configuration file using a text editor like vi.
vi /etc/chrony.conf

Specify NTP servers: In the configuration file, add or modify lines to point to your desired NTP servers. Use the server directive.
To add a new server, add a line like:
server 10.10.135.10 iburst
server 10.23.135.10 iburst


Restart the chronyd service to apply the changes:
systemctl restart chronyd

Enable the service to start automatically on boot (if not already enabled):
systemctl enable chronyd

Verify the synchronization status: Use the chronyc sources or chronyc tracking command to check the synchronization status.
chronyc sources
chronyc tracking
"
"Disable and stop firewalld (the default firewall manager):
sudo systemctl stop firewalld
sudo systemctl disable firewalld

Open a read-write shell using transactional-update:
This command creates a new snapshot and provides a shell where you can make changes to the system.
sudo transactional-update shell

Install the iptables package:
Inside the transactional-update shell, use zypper to install the package.
zypper in iptables

Exit the transactional-update shell:
The system will automatically apply the changes and reboot into the new snapshot.
exit
Reboot the system:

The reboot is necessary for the changes to the immutable system to take effect.
sudo reboot

Enable and start the iptables service:
After the system reboots, enable and start the service for persistence.
sudo systemctl enable iptables
sudo systemctl start iptables"
"curl -sfL https://get.rke2.io --output install.sh
chmod +x install.sh"
sudo INSTALL_RKE2_ARTIFACT_URL=https://prime.ribs.rancher.io/rke2 INSTALL_RKE2_VERSION="v1.34.10+rke2r1" ./install.sh
cp -f /opt/rke2/share/rke2/rke2-cis-sysctl.conf /etc/sysctl.d/60-rke2-cis.conf
systemctl restart systemd-sysctl

mkdir -p /etc/rancher/rke2

vi /etc/rancher/rke2/config.yaml

"token: WelcomeTOrke2
cni: calico
tls-san:
  - bastion.example.com
  - 192.168.122.69
  - 192.168.122.134
  - 192.168.122.170
  - 192.168.122.86
disable:
  - rke2-ingress-nginx
ingress-controller: traefik
profile: ""cis""
pod-security-admission-config-file: /etc/rancher/rke2/rancher-psa.yaml
kube-apiserver-arg:
  - 'service-account-extend-token-expiration=false'
#add this after the first restart
etcd-snapshot-schedule-cron: ""0 */5 * * *""
etcd-snapshot-retention: 5
etcd-snapshot-compress: true

systemctl enable --now rke2-server.service
/var/lib/rancher/rke2/bin/kubectl get nodes --kubeconfig /etc/rancher/rke2/rke2.yaml

"configure access to rke2 from bastion machine:
https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/kubernetes-cluster-setup/rke2-for-rancher"

"check certificate public or not : 
openssl verify cert.crt"

"create .pem from .crt and .key :
sudo sh -c 'cat rancher.crt rancher.key > rancher.pem'"

sudo useradd -r -c "etcd user" -s /sbin/nologin -M etcd -U

helm repo add rancher-prime https://charts.rancher.com/server-charts/prime

kubectl create namespace cattle-system

"helm install rancher rancher-prime/rancher \
  --namespace cattle-system \
  --set hostname=HOSTNAME \
  --set bootstrapPassword=STRONGPASS \
  --set tls=external \
  --set version=2.13.4

kubectl -n cattle-system rollout status deploy/rancher"

"CIS issue: 
vim /etc/rancher/rke2/rancher-psa.yaml
https://ranchermanager.docs.rancher.com/v2.13/reference-guides/rancher-security/psa-restricted-exemptions
#add cattle-scc-system namespace to the list"

"for rancher pods are runing 1/1 but not working on browser
vi /var/lib/rancher/rke2/server/manifests/rke2-traefik-config.yaml

apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-traefik
  namespace: kube-system
spec:
  valuesContent: |-
    ports:
      web:
        forwardedHeaders:
          #insecure: true
          trustedIPs:
            - ""192.168.122.86/32""

kubectl -n kube-system rollout restart daemonset/rke2-traefik

just wait for the pods to be redeployed cofngi will be detected automatically "
kubectl -n cattle-system create secret generic tls-ca   --from-file=cacerts.pem=/home/alpha/rancher.pem
kubectl -n cattle-system rollout restart deployment/rancher
