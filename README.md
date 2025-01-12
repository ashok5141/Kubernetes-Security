# Kubernetes 
Kubernetes (K8s) is an open-source container orchestration platform that automates the deployment, scaling, and management of containerized applications. It simplifies operations by abstracting infrastructure management and providing declarative configurations for containerized workloads.

![Kubernetes Architecture](https://raw.githubusercontent.com/ashok5141/Kubernetes-Security/refs/heads/main/K8s-Project/K8-Architecture.jpg)
## Kubernetes Terms
- The Kubernetes architecture is based on a master-worker design, consisting of the following core components
#### 1. Control Plane (Master Node)
> The control plane manages the Kubernetes cluster, ensuring the desired state of the system. It consists of:

**I. API Server** 

    - Acts as the frontend for the Kubernetes cluster.
    - Handles REST API requests and serves as the primary interface for cluster management.

**II. Controller Manager**
- Runs various controllers to regulate the cluster's state, such as:
    - Node Controller: Monitors node health.
    - Replication Controller: Ensures the desired number of pod replicas.
    - Endpoints Controller: Manages service endpoints.

**III. Scheduler**
    - Assigns pods to nodes based on resource requirements, policies, and other constraints.

**IV. etcd**
    - A distributed key-value store that stores all cluster data (e.g., configurations, secrets, and state).

#### 2. Worker Nodes
> Worker nodes host the application workloads. They include:

**I. kubelet**
    - Agent running on each worker node.
    - Ensures containers are running as per the pod specifications.

**II. kube-proxy**
    - Manages network communication between pods and services.
    - Handles routing, forwarding, and load balancing.

**III. Container Runtime**
    - Runs and manages containers on each node (e.g., Docker, containerd).
#### 3. Cluster Networking
> Kubernetes abstracts networking with the following features:

- **Pod-to-Pod Communication**: All pods can communicate with each other by default.
- **Services**: Expose workloads to other pods or external traffic.
- **Ingress**: Manages external HTTP/S access to services.

## Kubernetes Components
![Kubernetes Components](https://raw.githubusercontent.com/ashok5141/Kubernetes-Security/refs/heads/main/K8s-Project/k8s-componets.png)
- **Pod** : The smallest deployable unit in Kubernetes, representing one or more containers running together with shared resources (like storage and network).
- **Service** : Provides a stable endpoint for accessing a group of pods, enabling load balancing and reliable connectivity.
- **Ingress** : Manages external HTTP/S traffic to the cluster and routes it to specific services based on rules.
- **ConfigMap** : Stores configuration data in key-value pairs to decouple application settings from container images.
- **Secret** : Securely stores sensitive data such as passwords, API keys, or tokens, to avoid hardcoding them into the application.
- **Deployment** : Manages the desired state of pods, ensuring scaling, updates, and rollbacks are handled efficiently.
- **StatefulSet** : Manages stateful applications, ensuring pods are created and scaled in a specific order, with persistent identities and storage.
- **DaemonSet** : Ensures that a copy of a specific pod runs on every node in the cluster (or a subset of nodes).


#### Kubernetes vs Docker 
> Docker and Kubernetes are closely related in the container ecosystem, and they work together to manage and deploy containerized applications. Here's how Docker relates to Kubernetes and pods.

- **Container Runtime**: Docker is a container runtime, responsible for creating and managing containers. Kubernetes uses container runtimes like Docker (or alternatives such as containerd or CRI-O) to run the containers in its pods.

| Feature | Docker Alone | Docker with Kubernetes |
|:- |:- |:- |
|Scaling |Manual scaling (run more containers) |Automatic scaling based on load (Horizontal Pod Autoscaler). |
|High Availability |No built-in redundancy |Ensures redundancy using ReplicaSets and multiple nodes. |
|Self-Healing |Manual intervention for crashes |Automatically restarts failed pods. |
|Load Balancing |Basic or external tools |Built-in load balancing with Services. |
|Networking |Simple container networking (bridge mode) |Advanced networking with services, Ingress, and DNS. |

# Kubernetes Security  
