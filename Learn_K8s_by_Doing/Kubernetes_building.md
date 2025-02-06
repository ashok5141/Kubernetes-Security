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

- Run this command to join Worker noders to join control plane 
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

##### Deploying a Pod to a Node with a Lable in Kubernetes
- Get All Node Labels
- Create and Apply the Pod YAML
- Verify The pod is Running on the Correct Node
- Find the lable ```disk=ssd```
```bash
kubectl get nodes # 1 Master, 2 worker nodes
kubectl get nodes --show-labels  # 1 Master, 2 worker nodes detailed names
kubectl get pods --all-namespaces
```
- Find the IP address of the API server running on the master node
```bash
kubectl get pods --all-namespaces -o wide
kubectl get no --show-labels # We should see the label disk=ssd
```
- Create the pod YAML that will run on the node labeled ```disk=ssd```
- Name of the pod is  ```vi pod.yaml```
```bash
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx
  nodeSelector:
    disk: ssd
```
- Verify that the pod is running on the correct node
```bash
kubectl get po -o wide # Show you the Pods running on nginx
```

##### Installing and Testing the Components of a Kubernetes Cluster
- temporarily unavailable!

## Service Discovery, Scheduling and Lifecycle Management

##### Creating a Service and Discovering DNS Names in Kubernetes
- Create an nginx deployment, and verify it was successful.
- Create a service, and verify the service was successful.
- Create a pod that will allow you to query DNS, and verify it’s been created.
- Perform a DNS query to the service.
```bash
kubectl create deployment nginx --image=nginx
kubectl get deployments # Show the deplyments
kubectl get nodes # 1 clontrol and 2 worker nodes
```
- Create a service, and verify the service was successful
```bash
kubectl expose deployment nginx --port 80 --type NodePort # Service/nginx exposed
kubectl get services # Shows the Controler and Nginx with port
```
- Create a pod that will allow you to query DNS, and verify it’s been created
- Using an editor of your choice (e.g., Vim and the command ```vim busybox.yaml```), enter the following YAML to create the busybox pod spec: save (ESC :wq)
```bash
apiVersion: v1
kind: Pod
metadata:
  name: busybox
spec:
  containers:
  - image: busybox:1.28.4
    command:
      - sleep
      - "3600"
    name: busybox
  restartPolicy: Always
```
- create the busybox pod
```bash
kubectl create -f busybox.yaml
kubectl get pods
```
- Perform a DNS query to the service
```bash
kubectl exec busybox -- nslookup nginx # DNS name - nginx.default.svc.cluster.local
```

##### Scheduling Pods with Taints and TOlerations in Kubernetes
- Taint one of the worker nodes to repel work.
- Schedule a pod to the dev environment.
- Allow a pod to be scheduled to the prod environment.
- Verify each pod has been scheduled and verify the toleration.'

- Taint one of the worker nodes to repel work.
```bash
kubectl get nodes

# Output start
cloud_user@ip-10-0-1-101:~$ kubectl get nodes
NAME            STATUS   ROLES                  AGE   VERSION
ip-10-0-1-101   Ready    control-plane,master   17m   v1.23.0
ip-10-0-1-102   Ready    <none>                 17m   v1.23.0
ip-10-0-1-103   Ready    <none>                 17m   v1.23.0
# Output End

kubectl taint node <NODE_NAME> node-type=prod:NoSchedule
kubectl taint node ip-10-0-1-103 node-type=prod:NoSchedule
kubectl describe node ip-10-0-1-103 # Output as (Taints:node-type=prod:NoSchedule)
```

- Schedule a pod to the dev environment
- vim dev-pod.yaml
```bash
apiVersion: v1
kind: Pod
metadata:
  name: dev-pod
  labels:
    app: busybox
spec:
  containers:
  - name: dev
    image: busybox
    command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']
```
- create pod dev-pod.yaml
```bash
kubectl create -f dev-pod.yaml
```

- Schedule a pod to the prod environment
- vim prod-deployment.yaml
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prod
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prod
  template:
    metadata:
      labels:
        app: prod
    spec:
      containers:
      - args:
        - sleep
        - "3600"
        image: busybox
        name: main
      tolerations:
      - key: node-type
        operator: Equal
        value: prod
        effect: NoSchedule
```

- creste pod prod-deployment.yaml
```bash
kubectl create -f prod-deployment.yaml
```

- Verify each pod has been scheduled to the correct environment
```bash
kubectl get pods -o wide
kubectl scale deployment/prod --replicas=3    # Scale up the deployment 3 deployment includeing first one
kubectl get pods -o wide
```

##### Performing a Rolling Updates of an Application in Kubernetes
- Create and Roll Out Version 1 of the Application, and Verify a Successful Deployment
- Scale Up the Application to Create High Availability
- Create a Service So Users Can Access the Application
- Perform a Rolling Update to Version 2 of the Application, and Verify Its Success

```bash
kubectl get nodes # 1 master(Controller) and 2 worker nodes
kubectl get pods # no pods or namespaces
```


- Create and Roll Out Version 1 of the Application, and Verify a Successful Deployment
- vim kubeserve-deployment.yaml
- Commands paste ```:set paste``` save exit ```:wq```
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubeserve
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kubeserve
  template:
    metadata:
      name: kubeserve
      labels:
        app: kubeserve
    spec:
      containers:
      - image: linuxacademycontent/kubeserve:v1
        name: app
```

- Create deployment
```bash
kubectl apply -f kubeserve-deployment.yaml
```
- Verify the deployment was successful:
```bash
kubectl rollout status deployments kubeserve # deployment "kubeserve" successfully rolled out
```
- Verify the app is at the correct version:
```bash
kubectl describe deployment kubeserve
```

- Scale Up the Application to Create High Availability
```bash
kubectl get pods # 3 pods
kubectl scale deployment kubeserve --replicas=5 # 5 replicas including above 3
kubectl get pods # 5 pods
```

- Create a Service So Users Can Access the Application
```bash
kubectl expose deployment kubeserve --port 80 --target-port 80 --type NodePort
kubectl get services # Verify the 
# Output start
cloud_user@ip-10-0-1-101:~$ kubectl get service
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        52m
kubeserve    NodePort    10.102.52.146   <none>        80:31584/TCP   11s
# Output end
```

- Perform a Rolling Update to Version 2 of the Application, and Verify Its Success
```bash
# Open terminal with Control(Master) node with new ssh session
while true; do curl http://<ip-address-of-the-service>; done
while true; do curl http://10.102.52.146; done    # Copy the IP address from above command output - kubectl get services 
# It will send the continous requests
kubectl set image deployments/kubeserve app=linuxacademycontent/kubeserve:v2 --v 6 # second terminal
kubectl set image deployments/kubeserve app=linuxacademycontent/kubeserve:v2  # --v is not working
while true; do curl http://10.102.52.146; done # In this terminal version is updated receving v2 continously, stop CTRL+C
```
- In the first terminal, view the additional ReplicaSet created during the update:
```bash
kubectl get replicasets # 2 replica sets
kubectl get pods  # You should still see 5 pod replicas.
kubectl rollout history deployment kubeserve  # View the rollout history:
```

## Storage and Security

##### Creating Persistent Storage for Pods in Kubernetes
- Create a PersistentVolume.
- Create a PersistentVolumeClaim.
- Create the redispod image, with a mounted volume to mount path `/data`
- Connect to the container and write some data.
- Delete `redispod` and create a new pod named `redispod2`.
- Verify the volume has persistent data.

- Create a PersistentVolume
- create file ```vim redis-pv.yaml```
```bash
apiVersion: v1
kind: PersistentVolume
metadata:
  name: redis-pv
spec:
  storageClassName: ""
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
```

- Then, create the PersistentVolume
```bash
kubectl apply -f redis-pv.yaml
kubectl get pv # Check the see the persistant volume
```
- Create a PersistentVolumeClaim name of the file ```vim redis-pvc.yaml```
```bash
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redisdb-pvc
spec:
  storageClassName: ""
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

- Then, create the PersistentVolumeClaim:
```bash
kubectl apply -f redis-pvc.yaml
```

- Create a pod from the redispod image, with a mounted volume to mount path `/data`, name of the file ```vim redispod.yaml```
```bash
apiVersion: v1
kind: Pod
metadata:
  name: redispod
spec:
  containers:
  - image: redis
    name: redisdb
    volumeMounts:
    - name: redis-data
      mountPath: /data
    ports:
    - containerPort: 6379
      protocol: TCP
  volumes:
  - name: redis-data
    persistentVolumeClaim:
      claimName: redisdb-pvc
```

- Then, create the pod:
```bash
kubectl apply -f redispod.yaml
kubectl get pods # Verify the pods
```
- Connect to the container and write some data.
```bash
kubectl exec -it redispod -- redis-cli  # Connect to the container and run the redis-cli
SET server:name "redis server"  # Set the key space server:name and value "redis server":
GET server:name # Run the GET command to verify the value was set:
QUIT # Exit the redis-cli:
```

- Delete redispod and create a new pod named redispod2
```bash
kubectl delete pod redispod # Delete the existing redispod
kubectl get pods # no pods in that
name: redispod2 # Open the file redispod.yaml and change line 4 from name: redispod to:
kubectl apply -f redispod.yaml # Create a new pod named redispod2
```

- Verify the volume has persistent data
```bash
kubectl exec -it redispod2 -- redis-cli # Connect to the container and run redis-cli
kubectl get pods # New pod redis2 is created
kubectl exec -it redispod2 -- redis-cli  # it will open the trminal session
GET server:name # Run the GET command to retrieve the data written previously inputed
QUIT # Exit the redis-cli:
```

##### Creating a ClusterRole to Access a PV in Kubernetes
- View the Persistent Volume
- Create a ClusterRole and ClusterRoleBinding
- Create a pod to access the PV
- Request access to the PV from the pod
-
```bash
kubectl get pv # View the Persistent Volume "database-pv"
kubectl create clusterrole pv-reader --verb=get,list --resource=persistentvolumes # Create a ClusterRole
kubectl get clusterole # it will all roles pv-reader
kubectl create clusterrolebinding pv-test --clusterrole=pv-reader --serviceaccount=web:default # Create a ClusterRoleBinding
kubectl get clusterrolebinding pv-test # pv-test 
```
- Create a Pod to Access the PV ```vim curlpod.yaml```
```bash
apiVersion: v1
kind: Pod
metadata:
  name: curlpod
  namespace: web
spec:
  containers:
  - image: curlimages/curl
    command: ["sleep", "9999999"]
    name: main
  - image: linuxacademycontent/kubectl-proxy
    name: proxy
  restartPolicy: Always 
```

- Create the pod:
```bash
kubectl apply -f curlpod.yaml
kubectl get pod -n web # we can see the pod
```

- Request Access to the PV from the Pod
```bash
kubectl exec -it curlpod -n web -- sh # Open a new shell to the pod:
curl localhost:8001/api/v1/persistentvolumes # Curl the PV resource:, we can access the persistant data of the pod "database-pv" seen start of the lab
```

## Testing Your Cluster

##### Smoke Testing a Kubernetes CLuster
> your team has just finished setting up a new kubernetes cluster. However, before moving your company's online store to the cluster, they want to make sure that cluster is set up correctly. The team has asked you to run a series of smoke tests against the cluster in order to make sure that everything is working
- You will test functionality of the following:
    - Data encryption
    - Deployment
    - Port forwarding
    - Logs
    - Exec
    - Services
- Verify the cluster's ability to perform data encryption.
- Verify that deployments work.
- Verify that remote access works via port forwarding.
- Verify that you can access container logs with kubectl logs.
- Verify that you can execute commands inside a container with `kubectl exec`.
- Verify that services work.

- Verify the cluster's ability to perform data encryption.
```bash
kubectl create secret generic kubernetes-the-hard-way --from-literal="mykey=mydata"

sudo ETCDCTL_API=3 etcdctl get \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem\
  /registry/secrets/default/kubernetes-the-hard-way | hexdump -C     # Something like 00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
```
- Verify that Deployments Work
```bash
kubectl run nginx --image=nginx # Create a new deployment:
kubectl get deployments # list the deployments
kubectl get pods -l run=nginx # Verify the deployment created a pod and that the pod is running:
```

- Verify Remote Access Works via Port Forwarding
```bash

POD_NAME=$(kubectl get pods -l run=nginx -o jsonpath="{.items[0].metadata.name}")  # First, get the pod name of the nginx pod and store it as an environment variable:
echo $POD_NAME # creating object for pod name, same this with this command (kubectl get pods -l run=nginx)
kubectl port-forward $POD_NAME 8081:80 # Forward port 8081 to the nginx pod:
curl --head http://127.0.0.1:8081 # 200 OK , Open up a new terminal, log in to the controller server, and verify the port forward works:
# For me port-forwarding is not working but able to access the on port 8080, error-error: error upgrading connection: unable to upgrade connection: Forbidden (user=kubernetes, verb=create, resource=nodes, subresource=proxy)
```

- Verify You Can Access Container Logs with kubectl Logs
```bash
POD_NAME=$(kubectl get pods -l run=nginx -o jsonpath="{.items[0].metadata.name}") # Go back to the first terminal to run this test. If you are in a new terminal, you may need to set the <POD_NAME> environment variable again
kubectl logs $POD_NAME # Get the logs from the nginx pod:,  127.0.0.1 - - [10/Sep/2018:19:29:01 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.47.0" "-"
# Not able run the logs commands 
```
- Verify You Can Execute Commands Inside a Container with `kubectl exec` 
```bash
kubectl exec -ti $POD_NAME -- nginx -v # Execute a simple nginx -v command inside the nginx pod:, nginx version: nginx/1.15.3
# error: unable to upgrade connection: Forbidden (user=kubernetes, verb=create, resource=nodes, subresource=proxy)
```

- Verify Services Work
```bash
kubectl expose deployment nginx --port 80 --type NodePort # Create a service to expose the nginx deployment:
NODE_PORT=$(kubectl get svc nginx --output=jsonpath='{range .spec.ports[0]}{.nodePort}') # Get the node port assigned to the newly created service and assign it to an environment variable:
curl -I 10.0.1.102:$NODE_PORT # 200 OK, Access the service on one of the worker nodes from the controller like this (10.0.1.102 is the private IP of one of the workers):
```

##### Upgrading the Kubernetes Cluster Using 
>While doing this lab my kubectl is  v1.23.0, if i for lab veriosn it's downgrade, not performing this lab
- Install Version 1.18.5 of kubeadm on Master Node
- Upgrade Control Plane Components using kubeadm
- Install Version 1.18.5 of kubelet on Master Node
- Install Version 1.18.5 of kubectl on Master Node
- Install Version 1.18.5 of kubelet on The Worker Nodes

- Install Version 1.18.5 of kubeadm, Master node
- Master node Commands to run
```bash
kubectl get nodes # On the master node, check the current version of kubeadm, then check for the Client Version and Server Version:
kubectl version --short
sudo apt-mark unhold kubeadm kubelet # Take the hold off of kubeadm and kubelet:
sudo apt install -y kubeadm=1.18.5-00 # Install them using the package manager:
kubeadm version # Check the version again (which should show v1.18.5):
sudo kubeadm upgrade plan # Plan the upgrade to check for errors:
sudo kubeadm upgrade apply v1.18.5 # Apply the upgrade of the kube-scheduler and kube-controller-manager:
kubectl get nodes # Look at what version our nodes are at:
```

- Install the Latest Version of kubelet on the Master Node 
- Master node Commands to run
```bash
sudo apt-mark unhold kubelet # Make sure the kubelet package isn't on hold:
sudo apt install -y kubelet=1.18.5-00 # Install the latest version of kubelet:
kubectl get nodes # Verify that the installation was successful:
kubectl version --short # we'll see that the Server Version is v1.18.5, and the Client Version is at v1.17.8.
```
- Install the Latest Version of kubectl on the Master Node
- Master node Commands to run
```bash
sudo apt-mark unhold kubectl # Make sure the kubectl package isn't on hold:
sudo apt install -y kubectl=1.18.5-00  # Install the latest version of kubectl:
kubectl version --short #Verify that the installation was successful:
```

- Install the Newer Version of kubelet on the Worker Nodes 
- Worker nodes
```bash
# Worker 1
sudo apt-mark unhold kubelet # Make sure the kubelet package isn't on hold:
sudo apt install -y kubelet=1.18.5-00 # Install the latest version of kubelet:
# Worker 0
sudo apt-mark unhold kubelet # Make sure the kubelet package isn't on hold:
sudo apt install -y kubelet=1.18.5-00 # Install the latest version of kubelet:
```

## Logging and Monitering

##### Moniter and Output Logs to a File in Kubernetes
> You can troubleshoot broken pods quickly using the kubectl logs command. You can manipulate the output and save it to a file in order to capture important data. In this hands-on lab, you will be presented with a broken pod, and you must collect the logs and save them to a file in order to better understand the issue.
- Identify the problematic pod in your cluster.
- Collect the logs from the pod.
- Output the logs to a file.
```bash
# After log ssh, ssh cloud_user@34.203.221.52
kubectl get pods --all-namespaces  # pod3 is status is CrashLoopBackOff, Identify the problematic pod in your cluster.
# kubectl logs <pod_name> -n <namespace_name>
kubectl logs pod4 -n web # Collect the logs from the pod.
kubectl logs pod4 -n web > broken-pod.log
#Look like this -  2025/02/05 17:32:14 [error] 7#7: *1 open() "/etc/nginx/html/ealthz" failed (2: No such file or directory), client: 10.244.1.1, server: , request: "GET /ealthz HTTP/1.1", host: "10.244.1.3:8081"
#10.244.1.1 - - [05/Feb/2025:17:32:14 +0000] "GET /ealthz HTTP/1.1" 404 153 "-" "kube-probe/1.23"
```

##### Configuring Prometheus to Use Service Discovery
> Recently, your team has deployed Prometheus to the companies Kubernetes cluster. Now it is time to use service discovery to find targets for cAdvisor and the Kubernetes API. You have been tasked with modifying the Prometheus Config Map that is used to create the prometheus.yml file. Create the scrape config and add the jobs for kubernetes-apiservers and kubernetes-cadvisor. Then, propagate the changes to the Prometheus pod.
- Configure the Service Discovery Targets
- Apply the Changes to the Prometheus Configuration Map
- Delete the Prometheus Pod

- Logging In and Setting up the Environment
```bash
sudo su -
cd /root/prometheus
./bootstrap.sh
kubectl get pods -n monitoring
```

- Configure the Service Discovery Targets
- File name ```vi prometheus-config-map.yml```
```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-server-conf
  labels:
    name: prometheus-server-conf
  namespace: monitoring
data:
  prometheus.yml: |-
    global:
      scrape_interval: 5s
      evaluation_interval: 5s

    scrape_configs:
      - job_name: 'kubernetes-apiservers'

        kubernetes_sd_configs:
        - role: endpoints
        scheme: https

        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: default;kubernetes;https

      - job_name: 'kubernetes-cadvisor'

        scheme: https

        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        kubernetes_sd_configs:
        - role: node

        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
```

- Apply the Changes to the Prometheus Configuration Map
```bash
kubectl apply -f prometheus-config-map.yml
```
- Delete the Prometheus Pod
```bash
kubectl get pods -n monitoring # List the pods to find the name of the Prometheus pod:
kubectl delete pods <POD_NAME> -n monitoring # Delete the Prometheus pod:
kubectl delete pods prometheus-deployment-6684c87bbc-ssmkq -n monitoring
http://3.86.197.158:30000/graph   # Open up a new web browser tab, and navigate to the Expression browser. This will be at the public IP of the lab server, on port 30000:
```

##### Creating Alerting Rules
> After deploying a Prometheus environment to our Kubernetes cluster, the team has decided to test its monitoring capabilities by configuring alerting of our Redis deployment. We have been tasked with writing two alerting rules. The first rule will fire an alert if any of the Redis pods are down for 10 minutes. The second alert will fire if there are no pods available for 1 minute.
- Create a ConfigMap That Will Be Used to Manage the Alerting Rules.
- Apply the Changes Made to `prometheus-rules-config-map.yml`
- Delete the Prometheus Pod

- Logging In and Setting up the Environment 
```bash
 sudo su -
cd /root/prometheus
./bootstrap.sh
kubectl get pods -n monitoring
```

- Create a ConfigMap That Will Be Used to Manage the Alerting Rules
- Edit `prometheus-rules-config-map.yml` and add the Redis alerting rules. It should look like this when we're done:
```bash
apiVersion: v1
kind: ConfigMap
metadata:
  creationTimestamp: null
  name: prometheus-rules-conf
  namespace: monitoring
data:
  redis_rules.yml: |
    groups:
    - name: redis_rules
      rules:
      - record: redis:command_call_duration_seconds_count:rate2m
        expr: sum(irate(redis_command_call_duration_seconds_count[2m])) by (cmd, environment)
      - record: redis:total_requests:rate2m
        expr: rate(redis_commands_processed_total[2m])
  redis_alerts.yml: |
    groups:
    - name: redis_alerts
      rules:
      - alert: RedisServerDown
        expr: redis_up{app="media-redis"} == 0
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: Redis Server {{ $labels.instance }} is down!
      - alert: RedisServerGone
        expr:  absent(redis_up{app="media-redis"})
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: No Redis servers are reporting!
```

- Apply the Changes Made to
```bash
kubectl apply -f prometheus-rules-config-map.yml
```
- Delete the Prometheus Pod
```bash
kubectl get pods -n monitoring 
kubectl delete pods <POD_NAME> -n monitoring
kubectl delete pods prometheus-deployment-fd5569cf9-lml2d -n monitoring
http://<IP>:30080
http://54.145.118.56:30080 # Tried changin gthe ports prometheus-service.yml file port 8080, 9090, 30080 nothing works moved on
```

## Troubleshooting and Repairing Your Cluster
##### Repairing Failed Pods in Kubernetes
> As a Kubernetes Administrator, you will come across broken pods. Being able to identify the issue and quickly fix the pods is essential to maintaining uptime for your applications running in Kubernetes. In this hands-on lab, you will be presented with a number of broken pods. You must identify the problem and take the quickest route to resolve the problem in order to get your cluster back up and running.
- Identify the broken pods.
- Find out why the pods are broken.
- Repair the broken pods.
- Ensure pod health by accessing the pod directly.

- Identify the broken pods.
```bash
kubectl get all --all-namespaces # command to see what’s in the cluster
kubectl get svc,po,deploy -n web # To make this a little easier to read, you could run the following command to view services, pods, and deployments:
```

- Find out why the pods are broken.
```bash
kubectl describe pod POD_NAME -n web # Use the following command to inspect one of the broken pods and view the events, substituting the name of the pod for POD_NAME
kubectl describe pod nginx-856876659f-sc96q  -n web # Error: ErrImagePull pulling image "nginx:191"
```

- Repair the broken pods.
- Where it says image: nginx:191, change it to image: nginx. Save and exit.
```bash
kubectl edit deploy nginx -n web
kubectl get po -n web # get pod name 
kubectl edit nginx-856876659f-dgrfb  -n web # Image to nginx:191 to nginx, Directly repair the deploy
kubectl get rs -n web # replicaset checing after changing the ngnix server, spanup new pod and terminated old pods
```
- Ensure pod health by accessing the pod directly.
```bash
kubectl get po -n web -o wide # Including IP Address, got IP from here 10.244.1.13
kubectl run busybox --image=busybox --rm -it --restart=Never -- sh # Busybox pox
wget -qO- POD_IP_ADDRESS:80  # Use the following command to access the pod directly via its container port, replacing POD_IP_ADDRESS with an appropriate pod IP
wget -qO- 10.244.1.13:80
```
## Doing Things "The Hard Way"

##### Creating a Certificate Authority and TLS Certificates for Kubernetes
> The various components of Kubernetes require certificates in order to authenticate with one another. Provisioning a certificate authority and using it to generate those certificates is a necessary step in bootstrapping a Kubernetes cluster from scratch. This activity will guide you through the process of provisioning a certificate authority and generating the certificates Kubernetes needs.
- Here is the cluster architecture for which you will need to generate certificates. Note that these are not real servers, just values that we will use for the purposes of this activity.
    - Controllers:
        - Hostname: controller0.mylabserver.com, IP: 172.34.0.0
        - Hostname: controller1.mylabserver.com, IP: 172.34.0.1
    - Workers:
        - Hostname: worker0.mylabserver.com, IP: 172.34.1.0
        - Hostname: worker1.mylabserver.com, IP: 172.34.1.1
    - Kubernetes API Load Balancer:
        - Hostname: kubernetes.mylabserver.com, IP: 172.34.2.0
- Process
    - Provision the certificate authority (CA).
    - Generate the necessary Kubernetes client certs, as well as kubelet client certs for two worker nodes.
    - Generate the Kubernetes API server certificate.
    - Generate a Kubernetes service account key pair.

- Provision the Certificate Authority (CA)
```bash
{

cat > ca-config.json << EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ca-csr.json << EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

}
```
- Check the files created or by using this ```ls``` command.

- Generate the Kubernetes Client Certificates and Kubelet Client Certificates for Two Worker Nodes
```bash
{

cat > admin-csr.json << EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:masters",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin

}
```
- Check the files created or by using this ```ls``` command.

- Generate the Kubelet Client Certificates
```bash
{
cat > worker0.mylabserver.com-csr.json << EOF
{
  "CN": "system:node:worker0.mylabserver.com",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=172.34.1.0,worker0.mylabserver.com \
  -profile=kubernetes \
  worker0.mylabserver.com-csr.json | cfssljson -bare worker0.mylabserver.com

cat > worker1.mylabserver.com-csr.json << EOF
{
  "CN": "system:node:worker1.mylabserver.com",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=172.34.1.1,worker1.mylabserver.com \
  -profile=kubernetes \
  worker1.mylabserver.com-csr.json | cfssljson -bare worker1.mylabserver.com

}
```
- - Check the files created or by using this ```ls``` command.

- Generate the Kube-Controller-Manager Client Certificate
```bash
{

cat > kube-controller-manager-csr.json << EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-controller-manager",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

}
```
- Check the files created or by using this ```ls``` command.

- Generate the Kube-Proxy Client Certificate
```bash
{

cat > kube-proxy-csr.json << EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:node-proxier",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy

}
```
- Check the files created or by using this ```ls``` command.

- Generate the Kube-Scheduler Client Certificate
```bash
{

cat > kube-scheduler-csr.json << EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-scheduler",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler

}
```
- Check the files created or by using this ```ls``` command.

- Generate the Kubernetes API Server Certificate
```bash
{

cat > kubernetes-csr.json << EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,172.34.0.0,controller0.mylabserver.com,172.34.0.1,controller1.mylabserver.com,172.34.2.0,kubernetes.mylabserver.com,127.0.0.1,localhost,kubernetes.default \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes

}
```
- Check the files created or by using this ```ls``` command.

- Generate a Kubernetes Service Account Key Pair
```bash
{

cat > service-account-csr.json << EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account

}
```
- Check the files created or by using this ```ls``` command.

##### Generating Kubeconfigs for a New Kubernetes Cluster
> To set up a new Kubernetes cluster from scratch, we need to provide various components of the cluster with kubeconfig files so that they can locate and authenticate with the Kubernetes API. In this learning activity, you will generate a set of kubeconfigs that can be used to build a Kubernetes cluster.

- We are going to create kubeconfig files for the following:
    - Kubelet (one kubeconfig for each worker node)
    - Kube-proxy
    - Kube-controller-manager
    - Kube-scheduler
    - Admin
- We will be using the following cluster architecture:
    - Controllers:
        - Hostname: controller0.mylabserver.com, IP: 172.34.0.0
        - Hostname: controller1.mylabserver.com, IP: 172.34.0.1
    - Workers:
        - Hostname: worker0.mylabserver.com, IP: 172.34.1.0
        - Hostname: worker1.mylabserver.com, IP: 172.34.1.1
    - Kubernetes API Load Balancer:
        - Hostname: kubernetes.mylabserver.com, IP: 172.34.2.0
- Process
    - Generate kubelet kubeconfigs for each worker node.
    - Generate a kube-proxy kubeconfig.
    - Generate a kube-controller-manager kubeconfig.
    - Generate a kube-scheduler kubeconfig.
    - Generate an admin kubeconfig.

- Generate Kubelet Kubeconfigs for Each Worker Node
```bash
ls
KUBERNETES_PUBLIC_ADDRESS=172.34.2.0 # Set an environment variable called KUBERNETES_PUBLIC_ADDRESS and set it equal to the IP address of the load balancer:

for instance in worker0.mylabserver.com worker1.mylabserver.com; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
```
- Check the files created or by using this ```ls``` command.

- Generate a Kube-Proxy Kubeconfig
```bash
KUBERNETES_PUBLIC_ADDRESS=172.34.2.0

{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.pem \
    --client-key=kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
}
```
- Check the files created or by using this ```ls``` command.

- Generate a Kube-Controller-Manager Kubeconfig
```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.pem \
    --client-key=kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
}
```
- Check the files created or by using this ```ls``` command.

- Generate a Kube-Scheduler Kubeconfig
```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.pem \
    --client-key=kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
}
```
- Check the files created or by using this ```ls``` command.

- Generate an Admin Kubeconfig
```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default --kubeconfig=admin.kubeconfig
}
```
- Check the files created or by using this ```ls``` command.

-
```bash

```

-
```bash

```
-
```bash

```

-
```bash

```

-
```bash

```

-
```bash

```
-
```bash

```

-
```bash

```

-
```bash

```

-
```bash

```
-
```bash

```

-
```bash

```

-
```bash

```

-
```bash

```

-
```bash

```

-
```bash

```

-
```bash

```

-
```bash

```
-
```bash

```

-
```bash

```

-
```bash

```

-
```bash

```