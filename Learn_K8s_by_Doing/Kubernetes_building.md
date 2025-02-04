# Kubernetes


## Kubernetes Cluster Basics

##### Building a Kubernetes 1.27 Cluster with kubeadm

- Run this commands Control plane and 2 worker nodes
```bash
ssh cloud_user@<PUBLIC_IP_ADDRESS>

cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
sudo apt-get update && sudo apt-get install -y containerd.io
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl status containerd
sudo swapoff -a
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.27/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb [trusted=yes] https://pkgs.k8s.io/core:/stable:/v1.27/deb/ /
EOF

sudo apt-get update # wait for 2 minutes

sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl   # hold the updates, without automatic updates
```
- Initialize the Cluster
```bash
sudo kubeadm init --pod-network-cidr 192.168.0.0/16 --kubernetes-version 1.27.11
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes
kubeadm token create --print-join-command  # Initializing calico nework lool like below
# kubeadm join 10.0.1.100:6443 --token uhb4cx.vnvwlkcj2vcs9qrn \
        --discovery-token-ca-cert-hash sha256:126f4ac30baec439e5c09c91748d46dfd913ac0e1e0af1a5d6361cc49e15c1ad
```

- Controller plane to install Calico Network Add-on
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
kubectl get nodes
```
- Join worker nodes to the cluster
- Run this coomand to get the tokens ```kubeadm token create --print-join-command``` Initializing calico nework
```bash
sudo kubeadm join...  # full command in the controller plane 
# Control Plane 
kubectl get nodes
```

##### Deploying a Simple Service to Kubernetes
- Here is only Control Plane no worker nodes
```bash
ssh cloud_user@PUBLIC_IP_ADDRESS

# Create the deployment with four replicas:
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: store-products
  labels:
    app: store-products
spec:
  replicas: 4
  selector:
    matchLabels:
      app: store-products
  template:
    metadata:
      labels:
        app: store-products
    spec:
      containers:
      - name: store-products
        image: linuxacademycontent/store-products:1.0.0
        ports:
        - containerPort: 80
EOF

# Create a service for the store-products pods:
cat << EOF | kubectl apply -f -
kind: Service
apiVersion: v1
metadata:
  name: store-products
spec:
  selector:
    app: store-products
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
EOF

kubectl get svc store-products  #  service is up in the cluster
kubectl exec busybox -- curl -s store-products
```

##### Deploying a Microservice Application to Kubernetes
- http://$kube_master_public_ip:30080
- Code github [link](https://github.com/linuxacademy/robot-shop)
```bash
ssh cloud_user@PUBLIC_IP_ADDRESS
cd ~/
git clone https://github.com/linuxacademy/robot-shop.git
ll K8s/descriptors/
kubectl create namespace robot-shop
kubectl -n robot-shop create -f ~/robot-shop/K8s/descriptors/
kubectl get pods -n robot-shop
kubectl edit deployment mongodb -n robot-shop  # Open vim editor Edit, make mongodb 2 relicas Insert (press I) exit(ESC :wq), exit withoutedit (ESC:q)
# Under spec:, look for the line that says replicas: 1 and change it to replicas: 2
kubectl get pods -n robot-shop # YOu can see that 2 mongodb pods
```

##### Creating a Kubernetes Cluster
- Kubernetes cluster consisting of 1 master and 2 nodes
- Run below command all 3 terminials

```bash
sudo su

cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo # Docker Community Edition repository to yum
sudo yum install -y containerd.io
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl status containerd
sudo swapoff -a
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

- Complete the following section on the MASTER ONLY
```bash
kubeadm init --pod-network-cidr=10.244.0.0/16 # Initialize the cluster using the IP range for Flannel
mkdir -p $HOME/.kube   # Cross check folder created or not
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- Run this command to join Worker noders to control plane 
```bash
kubeadm join 10.0.1.100:6443 --token uhb4cx.vnvwlkcj2vcs9qrn \
        --discovery-token-ca-cert-hash sha256:126f4ac30baec439e5c09c91748d46dfd913ac0e1e0af1a5d6361cc49e15c1ad
```

- Complete the following section on the MASTER ONLY
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
kubectl get nodes # Include Control plane and worker nodes
```
- Create and Scale a Deployment Using kubectl
- These commands will only be run on the master node.
```bash
kubectl create deployment nginx --image=nginx
kubectl scale deployment nginx --replicas=4
kubectl get pods
```

## Working with Kubernetes Cluster

##### 