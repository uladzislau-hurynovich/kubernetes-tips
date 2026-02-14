# On-prem K8S
![Example Image](media/on-prem-k8s-main.jpg)
## 🧭 Overview
This project provides a simple **on-premises Kubernetes (K8S)** setup that can be launched locally on a personal computer.  
The deployment includes a complete local Kubernetes cluster with:
- **1 Master Node**
- **2 Worker Nodes**
- **NFS Server** for shared storage

This setup is ideal for testing, learning, and developing Kubernetes workloads without relying on a cloud provider.

---

## ⚙️ Components
| Component     | Description |
|----------------|-------------|
| **Master Node** | Controls the cluster, manages API server, scheduler, and controller manager. |
| **Worker Nodes** | Run application workloads and connect to the master node. |
| **NFS Server** | Provides shared persistent storage for deployments. |

---

## 🚀 Deployment Plan

### 1. Prepare control environment (MacOS)
Install required tools:

- **Freelens** ([GitHub](https://github.com/freelensapp/freelens))
```bash
brew install --cask freelens
```
- Helm ([Helm](https://helm.sh/docs/intro/install/))
```bash
brew install helm
```
- Kubectl ([kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/))
```bash
brew install kubectl
```
2. **Prepare host environment for VM** \
Use Ubuntu or Windows with installed VirtualBox. Launch VM with Ubuntu 24, network in bridged mode.

3. **Configure the Kubernetes Master and Worked Nodes**
- Configure static IP
```bash
nano /etc/netplan/50-cloud-init.yaml
```
```YAML
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      dhcp6: false
      addresses:
      - 192.168.1.100/24
      routes:
      - to: default
        via: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```
```bash
netplan apply
```
- Install container runtime ([Link](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)) \
Enable IPv4 packet forwarding
```bash
sudo -i
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
sysctl --system
```
For VIP/Keepalived
```bash
cat <<EOF | tee /etc/sysctl.d/k8s.conf
sysctl net.ipv4.ip_nonlocal_bind =1
EOF
sysctl --system
```
Enable kernel modules
```bash
modprobe overlay
modprobe br_netfilter
sudo tee /etc/modules-load.d/kubernetes.conf > /dev/null <<EOF
overlay
br_netfilter
EOF
```

Disable SWAP. Comment a line with /swap.img in */etc/fstab*
```bash
swapoff -a
```
Install containerd ([Link](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)). ***Check versions!!!*** \
Install runc
```bash
wget https://github.com/opencontainers/runc/releases/download/v1.4.0/runc.amd64
install -m 755 runc.amd64 /usr/local/sbin/runc
```
Install and configure containerd
```bash
wget https://github.com/containerd/containerd/releases/download/v2.2.1/containerd-2.2.1-linux-amd64.tar.gz
tar Cxzvf /usr/local containerd-2.2.1-linux-amd64.tar.gz
mkdir /etc/containerd
containerd config default | tee /etc/containerd/config.toml
#Add to the /etc/containerd/config.toml
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
#After
  [plugins.'io.containerd.grpc.v1.cri'.x509_key_pair_streaming]
    tls_cert_file = ''
    tls_key_file = ''
curl -L https://raw.githubusercontent.com/containerd/containerd/main/containerd.service -o /etc/systemd/system/containerd.service
systemctl daemon-reload
systemctl enable --now containerd
systemctl status containerd
```
Install kubelet kubeadm kubectl ([Link](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/))
```bash
apt-get update
apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
systemctl enable --now kubelet
```
4.1. Init a cluster and join worker nodes
Master node
```bash
kubeadm init --pod-network-cidr=10.244.0.0/16
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Install a networking and network policy provider (Calico) ([Link](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises))
```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.31.3/manifests/calico.yaml -O
```
Open calico.yaml and uncomment the following lines. 
Update the CIDR to match the one you used for --pod-network-cidr=10.244.0.0/16:
```bash
- name: CALICO_IPV4POOL_CIDR
  value: "10.244.0.0/16"
```
```bash
kubectl apply -f calico.yaml
```
Join worker nodes
```bash
kubeadm token create --print-join-command
kubeadm join 192.168.1.100:6443 --token i8vsnp.ezpdqmusb01qhihd \
        --discovery-token-ca-cert-hash sha256:c887a0e1fe692d1c1c652c8d1840c00facc08ea236411045ba886f47060c7055
```
4.2. Init a cluster and join control planes and worker nodes (HA, stacked etcd) \
4.2.1. Configure NGINX tcp balancer \
Install NGINX and add config to the /etc/nginx/nginx.conf file
```bash
stream {
    log_format proxy '$remote_addr [$time_local] '
                     '$protocol $status $bytes_sent $bytes_received '
                     '$session_time "$upstream_addr" '
                     '"$upstream_bytes_sent" "$upstream_bytes_received" "$upstream_connect_time"';

    access_log /var/log/nginx/k8s-api-tcp-access.log proxy;

    upstream kubernetes_apiserver {
        least_conn;
        server <ip_control_plane_1>:6443 max_fails=3 fail_timeout=10s;
        server <ip_control_plane_2>:6443 max_fails=3 fail_timeout=10s;
        server <ip_control_plane_3>:6443 max_fails=3 fail_timeout=10s;
    }

    server {
        listen 6443;
        proxy_pass kubernetes_apiserver;
        proxy_timeout 10m;
        proxy_connect_timeout 5s;
    }
}
```
-If you are going to use DNS, you need to add record to the /etc/hosts file
```bash
sudo vi /etc/hosts

192.168.1.200  control-plane.local
```
Master node. Init and join control planes.
```bash
sudo kubeadm init --control-plane-endpoint "<tcp_balancer_ip>or<tcp_balancer_dns>:<tcp_balancer_port>" --upload-certs --pod-network-cidr=10.244.0.0/16
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Install a networking and network policy provider (Calico) ([Link](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises))
```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.31.3/manifests/calico.yaml -O
```
Open calico.yaml and uncomment the following lines. 
Update the CIDR to match the one you used for --pod-network-cidr=10.244.0.0/16:
```bash
- name: CALICO_IPV4POOL_CIDR
  value: "10.244.0.0/16"
```
```bash
kubectl apply -f calico.yaml
```
Join worker nodes
```bash
kubeadm token create --print-join-command
kubeadm join 192.168.1.100:6443 --token i8vsnp.ezpdqmusb01qhihd \
        --discovery-token-ca-cert-hash sha256:c887a0e1fe692d1c1c652c8d1840c00facc08ea236411045ba886f47060c7055
```
4.2.2. Configure keepalived (VIP)
Install keepalived on all control-plane nodes.
```bash
sudo apt-get install -y keepalived psmisc
```
Create the script on all control-plane nodes.
```bash
sudo vim /etc/keepalived/check_apiserver.sh

#!/bin/bash
err=0
for k in $(seq 1 5)
do
    if ss -tulpn | grep -q ":6443"; then
        if timeout 3 curl -kfs https://127.0.0.1:6443/healthz >/dev/null 2>&1; then
            echo "API Server is healthy, keepalived in control"
            exit 0
        fi
    fi
    
    echo "Waiting for API Server startup (attempt $k/5)"
    err=$((err+1))
    sleep 3
done
echo "API Server is unhealthy"
exit 1
```
Make this script executable
```bash
sudo chmod +x /etc/keepalived/check_apiserver.sh
```
Create config on control_plane_1 and do the same on other nodes (change priotity (make lower), router_id, state set BACKUP)
```bash
sudo vim /etc/keepalived/keepalived.conf

! Configuration File for keepalived
global_defs {
    router_id KUBE_MASTER1
    script_user root root
    enable_script_security
}

vrrp_script check_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 3
    weight -20
    fall 5
    rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface ens160
    virtual_router_id 51
    priority 100
    advert_int 5
    
    authentication {
        auth_type PASS
        auth_pass keepalived123
  }
  virtual_ipaddress {
        x.x.x.x/(xx) dev ens160 label ens160:1
    }
     track_script {
        check_apiserver
    }
}
```
Run and check keepalived (VIP)
```bash
sudo systemctl enable keepalived
sudo systemctl start keepalived
sudo systemctl status keepalived
ip addr show <network_interface>
```
Master node. Init and join control planes.
```bash
sudo kubeadm init --control-plane-endpoint "<keepalived_ip>:6443" --upload-certs --pod-network-cidr=10.244.0.0/16
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Install a networking and network policy provider (Calico) ([Link](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises))
```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.31.3/manifests/calico.yaml -O
```
Open calico.yaml and uncomment the following lines. 
Update the CIDR to match the one you used for --pod-network-cidr=10.244.0.0/16:
```bash
- name: CALICO_IPV4POOL_CIDR
  value: "10.244.0.0/16"
```
```bash
kubectl apply -f calico.yaml
```
Join worker nodes
```bash
kubeadm token create --print-join-command
kubeadm join 192.168.1.100:6443 --token i8vsnp.ezpdqmusb01qhihd \
        --discovery-token-ca-cert-hash sha256:c887a0e1fe692d1c1c652c8d1840c00facc08ea236411045ba886f47060c7055
```

5. **Set up NFS Server (1 vCPU 1GB RAM)**
- Configure static IP
```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```
```YAML
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      dhcp6: false
      addresses:
      - 192.168.1.150/24
      routes:
      - to: default
        via: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```
```bash
sudo netplan apply
```
- Configure NFS server
```bash
sudo apt install nfs-kernel-server
sudo systemctl enable --now nfs-kernel-server
sudo mkdir -p /data
sudo chown nobody:nogroup /data
echo "/data *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee /etc/exports
sudo exportfs -a
```

---
## 📦 Next Steps
Helm charts and values for:
- NFS storage class
Install nfs tools on worker nodes
```bash
apt install nfs-common -y
```
Add and install helm chart
```bash
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm upgrade --install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --namespace=kube-system \
    --set nfs.server=192.168.1.150 \
    --set nfs.path=/data \
    --set storageClass.archiveOnDelete=false \
    --set storageClass.defaultClass=true
helm delete nfs-subdir-external-provisioner --namespace=kube-system
```
- Grafana
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm upgrade --install grafana grafana/grafana -f grafana-values.yaml --namespace monitoring --version 10.1.2 --create-namespace
```
- Loki (SingleBinary)
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm upgrade --install loki grafana/loki -f loki-values.yaml -n monitoring --version 6.44.0 --create-namespace
```
- Promtail
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm install promtail grafana/promtail -f promtail-values.yaml -n monitoring --version 6.17.0 --create-namespace
```
- Prometheus
```bash
helm repo add prometheus https://prometheus-community.github.io/helm-charts
helm upgrade --install prometheus prometheus/prometheus -f prometheus-values.yaml -n monitoring --version 27.39.0 --create-namespace
```
- MetalLB
```bash
helm repo add metallb https://metallb.github.io/metallb
helm upgrade --install metallb metallb/metallb -n metallb --create-namespace
kubectl apply -f metallb.yaml
```
- Ingress
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx -f ingress-values.yaml -n ingress-nginx --version 4.13.3 --create-namespace
```
- Cert Manager
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.9.2/cert-manager.crds.yaml
helm repo add jetstack https://charts.jetstack.io
helm upgrade --install cert-manager jetstack/cert-manager -f cert-manager-values.yaml -n cert-manager  --version v1.9.2 --create-namespace
kubectl apply -f self-signed-cert.yaml
kubectl run nginx --image nginx -n test
kubectl expose pod nginx --port=80 --name=nginx -n test
kubectl apply -f nginx-app-ingress.yaml
```
- Priority Classes
```bash
kubectl apply -f priorityclass.yaml
```
- Istio SideCar mode
```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm upgrade --install istio-base istio/base -n istio-system --set defaultRevision=default --create-namespace
helm upgrade --install istiod istio/istiod -n istio-system --wait
kubectl create namespace istio-ingress
helm upgrade --install istio-ingress istio/gateway -n istio-ingress --wait
#istio API (not Kubernetes Gateway API)
kubectl create ns test  
kubectl label namespace test istio-injection=enabled  
kubectl run nginx --image nginx -n test
kubectl expose pod nginx --port=80 --name=nginx -n test
kubectl apply -f istio-gateway
```
- Opensearch
```bash
helm repo add opensearch https://opensearch-project.github.io/helm-charts/
helm repo add fluent https://fluent.github.io/helm-charts
helm iupgrade --install opensearch opensearch/opensearch -f opensearch-values.yaml -n logi --create-namespace
helm upgrade --install opensearch-dashboard opensearch/opensearch-dashboards -n logi --create-namespace
helm upgrade --install fluent-bit fluent/fluent-bit -f opensearch-fluentbit-values.yaml --namespace logi --create-namespace
```
- Argocd
```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm upgrade --install argocd argo/argo-cd -f argocd-values.yaml --namespace argocd --create-namespace --version 9.3.7
```
- Longhorn
Install **open-iscsi**,**nfs-common**, enable **iscsi_tcp**
```bash
apt-get install open-iscsi
systemctl enable iscsid
systemctl start iscsid
modprobe iscsi_tcp
echo "iscsi_tcp" | sudo tee /etc/modules-load.d/iscsi_tcp.conf
helm repo add longhorn https://charts.longhorn.io
helm upgrade --install longhorn longhorn/longhorn -f longhorn-values.yaml --namespace longhorn-system --create-namespace --version 1.10.1
```

- Gateway API (Traefik)
```bash
# Install Gateway API CRDs from the Standard channel.
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml
# Install Traefik RBACs.
kubectl apply -f https://raw.githubusercontent.com/traefik/traefik/v3.6/docs/content/reference/dynamic-configuration/kubernetes-gateway-rbac.yml
helm repo add traefik https://traefik.github.io/charts
helm upgrade --install traefik traefik/traefik -f traefik-values.yaml --namespace traefik --create-namespace
```

- Elasticsearch, Kibana, logstash
```bash
helm repo add elastic https://helm.elastic.co
helm install elastic-operator elastic/eck-operator -n elastic-system --create-namespace
helm upgrade --install es-kb-quickstart elastic/eck-stack  -f elasticsearch-values.yaml --namespace elastic-stack --create-namespace --version 0.17.0
helm upgrade --install fluent-bit fluent/fluent-bit -f elasticsearch-fluentbit-values.yaml --namespace elastic-stack --create-namespace
```

- Kafka
```bash
kubectl create namespace kafka
kubectl create -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka
kubectl apply -f https://strimzi.io/examples/latest/kafka/kafka-single-node.yaml -n kafka
kubectl apply -f kafdrop.yaml
kubectl -n kafka delete $(kubectl get strimzi -o name -n kafka)
kubectl delete -f kafdrop.yaml
kubectl delete -f https://strimzi.io/examples/latest/kafka/kafka-single-node.yaml -n kafka
kubectl delete -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka
kubectl delete namespace kafka
```

https://strimzi.io/
https://opensource.adobe.com/koperator/

- Ceph
```bash
helm repo add rook-release https://charts.rook.io/release
helm upgrade --install --create-namespace --namespace rook-ceph rook-ceph rook-release/rook-ceph -f ceph-operator-values.yaml
kubectl apply -f ceph-cluster.yaml
```


https://blog.risingstack.com/ceph-storage-deployment-vm/
https://medium.com/@satishdotpatel/ceph-integration-with-kubernetes-using-ceph-csi-c434b41abd9c
https://habr.com/ru/articles/465399/#vm


---

## 🧰 Requirements Host 
- Virtualization support enabled
- Minimum recommended resources:
  - **CPU:** 8 cores  
  - **Memory:** 32 GB  
  - **Storage:** 200 GB free space
---


