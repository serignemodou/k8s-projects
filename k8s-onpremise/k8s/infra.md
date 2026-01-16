# 🏗️ Architecture Overview
![alt text](HAPROXY.drawio-1.png)
# 📋 Prerequisites and system Requirements
# 🖥️ Server Specifications

| Name | Role | CPU | RAM | Disk | OS | IP Address | VLAN | Port |
|:-------- |:--------:| --------:| --------:| --------:| --------:| --------:| --------:| --------:|
| haproxy-1 | HAProxy  | 2 vCPU | 2GB | 20GB | Ubuntu 24.04 LTS  | 172.16.144.10 |  Vlan106 | 6443, 80, 443, 8080 |
| haproxy-2 | HAProxy  | 2 vCPU | 2GB | 20GB | Ubuntu 24.04 LTS | 172.16.144.11 |  Vlan106 | 6443, 80, 443, 8080 |
| master-1 | Control Plan  | 4 vCPU | 4GB | 50GB | Ubuntu 24.04 LTS | 172.16.144.21 |  Vlan106 | 6443, 80, 443, 2379 |
| master-2 | Control Plan  | 4 vCPU | 4GB | 50GB | Ubuntu 24.04 LTS | 172.16.144.22 |  Vlan106 | 6443, 80, 443, 2379 |
| master-3 | Control Plan  | 4 vCPU | 4GB | 50GB | Ubuntu 24.04 LTS | 172.16.144.23 |  Vlan106 | 6443, 80, 443, 2379 |
| worker-1 | Data Plan  | 4 vCPU | 8GB | 100GB | Ubuntu 24.04 LTS | 172.16.144.31 |  Vlan106 | 80, 443, 8080 |
| worker-2 | Data Plan  | 4 vCPU | 8GB | 100GB | Ubuntu 24.04 LTS | 172.16.144.32 |  Vlan106 | 80, 443, 8080 |
| NFS-Server | Storage  | 2 vCPU | 4GB | 200GB | Ubuntu 24.04 LTS | 172.16.144.40 |  Vlan106 | 2049 |

# VIP

| VIP Name | Private IP | Public IP | Potrs |
|:-------- |:--------:| --------:| --------:|
|  apiserver  |  172.16.144.100  | -  | 8443 => 6443 |
|  worker  |  172.16.144.110  | 43.10.89.16  | 8443 => 6443 |