# 🐳  HAProxy & Keepalive : HAProxy-1 and HAProxy-2 servers
## Install HAProxy and keepalive packages
```
apt-get update && apt-get upgrade -y
apt-get install haproxy keepalived rsyslog -y

systemctl daemon-reload
systemctl start haproxy
systemctl enable haproxy
systemctl status haproxy

systemctl start keepalived
systemctl enable keepalived
systemctl status keepalived
```

## Configure HAProxy
```
cat <<EOF | sudo tee /etc/haproxy/haproxy.cfg
global
    log 127.0.0.1 local0
    maxconn 2000
    user haproxy
    group haproxy
    daemon

    # SSL Configuration
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256
    ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
    mode http
    log global
    option httplog
    option dontlognull
    retries 3
    timeout http-request 10s
    timeout queue  1m
    timeout connect  10s
    timeout client  1m
    timeout server  1m
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http

listen stats
    bind *:8080
    stats enable
    stats uri /stats
    stats refresh 10s
    stats admin if TRUE
    stats auth admin: h@pr!y

# Kubernetes API Server
frontend k8s-api-frontend
    bind *:6443
    mode: tcp 
    option tcplog
    default_backend k8s-api-backend

backend k8s-api-backend
    mode tcp 
    balance roundrobin
    option tcp-check
    tcp-check connect port 6443
    server master-1 172.16.144.21:6443 check
    server master-2 172.16.144.22:6443 check
    server master-3 172.16.144.23:6443 check
EOF

systemctl restart haproxy
systemctl status haproxy
```

## Configuration Keepalive
```
cat <<EOF | sudo tee /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    state MASTER
    interface ens3
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass passw123
    }
    virtual_ipaddress {
        172.16.144.100
    }
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens3
    virtual_router_id 51
    priority 99
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass passw123
    }
    virtual_ipaddress {
        172.16.144.100
    }
}
EOF

systemctl restart keepalived
systemctl status keepalived
```

## Configure sysctl
```
cat <<EOF | sudo tee /etc/sysctl.conf
net.ipv4.ip_forward                 = 1

sysctl --system
EOF
```

# 🔧 Kubernetes Master and Worker Nodes Setup 
## Install prerequisites 
```
apt-get update && apt-get upgrade -y
apt-get install unzip tar apt-transport-https libseccomp2 util-linux ca-certificates curl gpg gnupg nfs-common -y
```

#### Disable swap configuration
```
swapoff -a
sed -e '/swap/s/^/#/g' -i /etc/fstab
```

#### Configure required modules
```
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
EOF

modprobe overlay
modprobe br_netfilter
modprobe ip_vs
modprobe ip_vs_rr
modprob ip_vs_wrr
modprob ip_vs_sh

cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system
```

### Local DNS
```
In /etc/hosts ; hostname -> IP address
```
### Install and configure containerd runtime (1.7.30)
```
mkdir -p /opt/cni/bin/
mkdir -p /etc/cni/net.d/
mkdir -p /etc/containerd

wget https://github.com/containerd/containerd/releases/download/v1.7.30/cri-containerd-cni-1.7.30-linux-amd64.tar.gz
tar Cxzvf /opt/cni/bin/ cri-containerd-cni-1.7.30-linux-amd64.tar.gz
rm -f cri-containerd-cni-1.7.30-linux-amd64.tar.gz

systemctl daemon-reload
systemctl start containerd
systemctl enable containerd
systemctl status containerd

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

systemctl restart containerd
systemctl status containerd

```

### Install kubernetes components binaire (Add dpkg packages on ubuntu and install)
```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubelet=1.31.5-1.1 kubeadm=1.31.5-1.1 kubectl=1.31.5-1.1
apt-mark hold kubelet kubeadm kubectl

```

### Insert this line below with vi in /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf file after section [Service] (it's kubelet attributes parameters)
```
Environment="KUBELET_EXTRA_ARGS= --runtime-cgroups=/system.slice/containerd.service --container-runtime=remote --runtime-request-timeout=15m --container-runtime-endpoint=unix:///run/containerd/containerd.sock"
```

# 🏗️ Install Kubernetes
## Only on first Master Node Initialization
### Cluster initialization

```
mkdir /opt/kubernetes

cat <<EOF | sudo tee /opt/kubernetes/kubeadm-config.yaml
kind: ClusterConfiguration
apiVersion: kubeadm.k8s.io/v1beta4
clusterName: swap-sandbox
kubernetesVersion: v1.31.5
controlPlaneEndpoint: "api.proxima.swap.io:6443"
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
    - 
  certSANs:
    - "api.proxima.swap.io"
    - "172.16.144.100" #vip-private ip
etcd:
  local:
    dataDir: "/var/lib/etcd"
    extraArgs:
      - name: listen-metrics-urls
        value: http://0.0.0.0::2379
controllerManager:
  extraArgs:
  - name: node-cidr-mask-size
    value: "24"
networking:
  serviceSubnet: "192.168.128.0/17"
  podSubnet: "192.168.0.0/17"


---

kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authorization:
  anonymous:
    enabled: false
  webhook:
    enabled: true
authorization:
  mode: Webhook
systemReserved:
  cpu: 100m
  memory: 100Mi
kubeReserved:
  cpu: 100m
  memory: 100Mi
evictionHard:
  imagefs.available: "15%"
  memory.available: "100Mi",
  nodefs.available: "10%",
  nodefs.inodesFree: "5%",
  imagefs.inodesFree: "5%"
maxPods: 250
containerLogMaxSize: "10Mi",
containerLogMaxFiles: 5,
cgroupDriver: systemd


EOF

kubeadm init --config /opt/kubernetes/kubeadm-config.yaml --skip-phases=addon/kube-proxy --upload-certs | tee /opt/kubernetes/kubeadm-init.out

```

### Install cilium cni (version 1.18.5 with helm)
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

export API_SERVER_IP="api.proxima.swap.io"
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

### JoinAdditional Master Nodes (connect to the second then the thirty master vm)

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

## 👷 Worker Nodes Setup

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