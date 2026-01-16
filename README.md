# What we will achieve
- ✅ 3 Master nodes for HA control plane
- ✅ 2 Worker nodes for workload distribution
- ✅ NFS server for persistent storage
- ✅ HAProxy + Keepalived for production load balancing (VIP Control Plan associate to HAProxy)
- ✅ Kube-vip for application exposition (VIP worker associate to kube-vip (daemonset on kube))
- ✅ Cert-Manager for SSL certificates and security hardening 
- ✅ Contour and HttpProxy
- ✅ Fully automated deployment with Ansible (comming soon)
- ✅ Monitoring and log management (comming soon)