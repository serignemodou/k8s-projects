# Deploy kubernetes cluster with kubeadm
## 🔧 Compute Charasteristic
1. Compute
| VM Name | VM IP | Roles | OS | CPU (vCPU) | RAM | Disk Storage | Openning Ports |
|:-------- |:--------:| --------:| --------:| --------:| --------:| --------:|
| master01 | 172.24.0.2  | Control Plane  | Ubuntu24.04 LTS| 2 | 8 | 20Go | 6443, 2379-2380, 10250, 10259, 10257 |
| master02 | 172.24.0.3  | Control Plane  | Ubuntu24.04 LTS| 2 | 8 | 20Go | 6443, 2379-2380, 10250, 10259, 10257 |
| master03 | 172.24.0.4  | Control Plane  | Ubuntu24.04 LTS| 2 | 8 | 20Go | 6443, 2379-2380, 10250, 10259, 10257 |

## 🏗️ Setup Infrastructure
### Set up the network and subnets
1. Create the custom mode VPC network
```
gcloud compute networks create vpc-k8s \
  --subnet-mode=custom
```
2. create a subnet for backends
```
gcloud compute networks subnets create snet-k8s \
  --network=vpc-k8s \
  --range=172.24.0.0/24 \
  --region=us-central1
```

### Create the zonal managed instance groups
1. Create an instance template
```
gcloud compute instance-templates create control-plane-template \
  --region=us-central1 \
  --network=vpc-k8s \
  --subnet=snet-k8s \
  --stack-type=IPV4_ONLY \
  --machine-type=e2-standard-2 \
  --boot-disk-type=pd-balanced \
  --tags=lb-tag \
  --image-family=ubuntu-2404-lts \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=20GB \
  --metadata=enable-oslogin=TRUE
```
2. Create a managed instance group
```
gcloud compute instance-groups managed create instance-group-k8s \
  --size 3 \
  --template control-plane-template
```

### Configure firewall rules
1. Allow IPv4 8443 traffic
```
gcloud compute firewall-rules create allow-network-lb-ipv4 \
  --network=vpc-k8s \
  --target-tags=lb-tag \
  --allow=tcp:8443 \
  --source-ranges=0.0.0.0/0
```
2. Allow IPv4 6443 traffic
```
gcloud compute firewall-rules create allow-network-lb-ipv4 \
  --network=vpc-k8s \
  --target-tags=lb-tag \
  --allow=tcp:6443 \
  --source-ranges=0.0.0.0/0
```
3. Allow IPv4 SSH and RDP traffic
```
gcloud compute firewall-rules create allow-network-lb-ipv4 \
  --network=vpc-k8s \
  --target-tags=lb-tag \
  --allow=tcp:22,3389 \
  --source-ranges=0.0.0.0/0
```

### Configure the load balancer
1. Reserve a static external IP address
```
gcloud compute addresses create network-lb-ipv4 \
  --region us-central1
```
2. Create a TCP health check.
```
gcloud compute health-checks create tcp tcp-health-check \
  --region us-central1 \
  --port 6443
```
3. Create a backend service
```
gcloud compute backend-services create network-lb-backend-service \
  --protocol TCP \
  --health-checks tcp-health-check \
  --health-checks-region us-central1 \
  --region us-central1
```
4. Add the instance groups to the backend service
```
gcloud compute backend-services add-backend network-lb-backend-service \
  --instance-group instance-group-k8s \
  --region us-central1
```
5. Create Frontend IPv4 and port
```
gcloud compute forwarding-rules create network-lb-frontend-ipv4 \
  --load-balancing-scheme EXTERNAL \
  --region us-central1 \
  --ports 8443 \
  --address network-lb-ipv4 \
  --backend-service network-lb-backend-service
```

## 🐳 Installation (bootstrap k8s cluster)
### Install prerequisites (install ubuntu package ; desable swap ; install modules ; configure iptables)
1. Install packages
```
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install unzip tar apt-transport-https libseccomp2 util-linux ca-certificates curl gpg nfs-common -y
```
2. Disable swap configuration
```
sudo swapoff -a
sudo sed -e '/swap/s/^/#/g' -i /etc/fstab
```

3. Configure required modules
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
sudo modprobe ip_vs
sudo modprobe ip_vs_rr
sudo modprobe ip_vs_wrr
sudo modprobe ip_vs_sh

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

4. DNS record (local DNS)
On each VM name
```
IPV4=$(hostname -I | awk '{print $1}')
HOSTNAME=$(hostname -s)
sudo echo "$IPV4 $HOSTNAME" >> /etc/hosts
```

5. Install and configure containerd runtime (1.7.30)
```
sudo mkdir -p /opt/cni/bin/
sudo mkdir -p /etc/cni/net.d/
sudo mkdir -p /etc/containerd

sudo wget https://github.com/containerd/containerd/releases/download/v1.7.30/cri-containerd-cni-1.7.30-linux-amd64.tar.gz
sudo tar --no-overwrite-dir -C / -xzf cri-containerd-cni-1.7.30-linux-amd64.tar.gz
sudo rm -f cri-containerd-cni-1.7.30-linux-amd64.tar.gz

sudo systemctl daemon-reload
sudo systemctl start containerd
sudo systemctl enable containerd
sudo systemctl status containerd

cat <<EOF | sudo tee /etc/containerd/config.toml
version = 2
imports = ["/etc/containerd/conf.d/*.toml"]
[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
    sandbox_image = "registry.k8s.io/pause:3.10"
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
      runtime_type = "io.containerd.runc.v2"
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
        SystemdCgroup = true
EOF

sudo systemctl restart containerd
sudo systemctl status containerd

```

6. Install kubernetes components binaire (Add dpkg packages on ubuntu and install)
```
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

```

7. Insert this line below with vi in /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf file after section [Service] (it's kubelet attributes parameters)
```
sudo vi /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
```
```
Environment="KUBELET_EXTRA_ARGS=--runtime-cgroups=/system.slice/containerd.service --container-runtime=remote --runtime-request-timeout=15m --container-runtime-endpoint=unix:///run/containerd/containerd.sock"
```

### Install kubernetes
#### Only on the first master
1. Initialize cluster
```
sudo mkdir /opt/kubernetes

cat <<EOF | sudo tee /opt/kubernetes/kubeadm-config.yaml
kind: ClusterConfiguration
apiVersion: kubeadm.k8s.io/v1beta4
clusterName: swap-sandbox
kubernetesVersion: v1.34.3
apiServer:
  extraArgs:
    - name: "audit-log-path"
      value: "/var/log/kubernetes/audit/audit.log"
    - name: "audit-log-maxage"
      value: "30"
    - name: "audit-log-maxbackup"
      value: "3"
    - name: "audit-log-maxsize"
      value: "100"
controllerManager:
  extraArgs:
    - name: "cloud-provider"
      value: "external"
etcd:
  local:
    dataDir: /var/lib/etcd
networking:
  serviceSubnet: "192.168.128.0/17"
  podSubnet: "192.168.0.0/17"
controlPlaneEndpoint: "<LB_PUBLIC_IP>:8443"

---

kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
EOF

export export NODE_NAME=$(hostname -s)
sudo kubeadm init --config /opt/kubernetes/kubeadm-config.yaml --skip-phases=addon/kube-proxy --upload-certs --node-name $NODE_NAME | sudo tee /opt/kubernetes/kubeadm-init.out
```

2. Install cilium cni (version 1.18.5 with helm)
```
wget https://get.helm.sh/helm-v3.17.0-linux-amd64.tar.gz
tar -zxvf helm-v3.17.0-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
rm helm-v3.17.0-linux-amd64.tar.gz

helm repo add cilium https://helm.cilium.io/

export KUBECONFIG=/etc/kubernetes/admin.conf

helm install cilium cilium/cilium --version 1.18.5 \
--set ipam.operator.clusterPoolIPv4PodCIDRList={"192.168.0.0/17"} \
--set ipam.operator.clusterPoolIPv4MaskSize=24 \
--namespace kube-system

export API_SERVER_IP="<LB_PUBLIC_IP>"
export API_SERVER_PORT="8443"

helm upgrade --install cilium cilium/cilium --version 1.18.5 --namespace kube-system \
--set k8sServiceHost=${API_SERVER_IP} \
--set k8sServicePort=${API_SERVER_PORT} \
--set kubeProxyReplacement=true \
--set hubble.relay.enabled=true \
--set ipam.operator.clusterPoolIPv4PodCIDRList={"192.168.0.0/17"} \
--set ipam.operator.clusterPoolIPv4MaskSize=24 \
--set tunnelProtocol="geneve" \
--set devices="ens192" \
--set hubble.ui.enabled=true

kubectl get pods --all-namespaces -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,HOSTNETWORK:.spec.hostNetwork --no-headers=true | grep '<none>' | awk '{print "-n "$1" "$2}' | xargs -L 1 -r kubectl delete pod
```

3. Join master (connect to the second then the thirty master vm)
| Variable | Description | Location | 
|:--------|:-------- |:--------:| 
| JOIN_TOKEN | Token to join the control plan  | /opt/kubernetes/kubeadm-init.out  | 
| JOIN_TOKEN_CACERT_HASH | Token Certificate Authority hash to join the control plan  | /opt/kubernetes/kubeadm-init.out  | 
| JOIN_TOKEN_CERT_KEY | Token Certificat Key to join the control plan | /opt/kubernetes/kubeadm-init.out  | 

```
kubeadm join api.proxima.swap.io:8443 --token <JOIN_TOKEN> \
--discovery-token-ca-cert-hash <JOIN_TOKEN_CACERT_HASH> \
--control-plane --certificate-key <JOIN_TOKEN_CERT_KEY>
```

4. Join workers (connect on each worker and make the command below)
```
kubeadm join api.proxima.swap.io:8443 --token <JOIN_TOKEN> \
--discovery-token-ca-cert-hash <JOIN_TOKEN_CACERT_HASH>
```

### Check Nodes status
```
kubectl get node
```

### Check Pods
```
kubectl get pod -A
```

### Check Cilium connectivity status
```
kubectl -n kube-system exec ds/cilium -- cilium-health status

kubectl -n kube-system exec ds/cilium -- cilium-dbg status
```
