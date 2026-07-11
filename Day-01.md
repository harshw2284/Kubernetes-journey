# Kubernetes – Day 01 -  Kubernetes Architecture and Cluster Setup

I have been building and shipping containers with Docker. But what happens when I need to run hundreds of containers across multiple servers? You need an orchestrator. Today I started my Kubernetes journey — understand the architecture, set up a local cluster, and run your first kubectl commands.

This is where things get real.

---

### ✅ Task 1: Recall the Kubernetes Story

**Before touching a terminal, write down from memory:**

**1. Why was Kubernetes created? What problem does it solve that Docker alone cannot ?**

Kubernetes was created by Google to automate the deployment, scaling, and management of containerized applications across large fleets of servers. It is essentially a production-grade orchestrator that handles the operational heavy lifting of running thousands of interconnected microservices reliably.

Docker revolutionized the industry by allowing developers to package code and dependencies into a single, portable container. However, when running applications in production, Docker operates entirely on a single node (one machine) and leaves unresolved gaps in system reliability.

**2. Who created Kubernetes and what was it inspired by ?**

Kubernetes was created by Google employees in 2014. The project’s design and development were heavily inspired by Google's internal cluster management systems, **Borg** and **Omega**, which Google used for over a decade to manage massive-scale container workloads.

To honor its predecessor, the developers codenamed Kubernetes "Project 7" (after the Star Trek ex-Borg character, Seven of Nine), which is why its logo features a seven-spoke ship's wheel.

**3. What does the name "Kubernetes" mean ?**

The name Kubernetes comes from the Greek word "κυβερνήτης" (kubernetes), which means "helmsman," "pilot," or "ship captain." It represents someone who steers and manages a ship.

Similarly, Kubernetes is a container orchestration platform that automatically manages, deploys, scales, and monitors containerized applications, acting like a captain that controls and coordinates containers across multiple machines.

---

### ✅ Task 2 : Draw the Kubernetes Architecture

**Draw or Describe the Kubernetes architecture. Your diagram should include:**

**Control Plane (Master Node):**

* API Server — the front door to the cluster, every command goes through it
* etcd — the database that stores all cluster state
* Scheduler — decides which node a new pod should run on
* Controller Manager — watches the cluster and makes sure the desired state matches reality

**Worker Node:**

* kubelet — the agent on each node that talks to the API server and manages pods
* kube-proxy — handles networking rules so pods can communicate
* Container Runtime — the engine that actually runs containers (containerd, CRI-O)

**Diagram:**

<img width="1070" height="510" alt="Screenshot 2026-07-11 173004" src="https://github.com/user-attachments/assets/12c24637-a515-4a6c-93c5-e9a8d4b344df" />

<img width="1536" height="1024" alt="ChatGPT Image Jul 11, 2026, 04_45_16 PM" src="https://github.com/user-attachments/assets/a9863821-e855-4c0e-b840-dc2c070142d3" />

**After drawing, verify your understanding:**

**1. What happens when you run ``kubectl apply -f pod.yaml``? Trace the request through each component.**

When kubectl apply -f pod.yaml is executed, kubectl sends the Pod definition to the API Server, which validates and stores it in etcd. The Scheduler assigns a worker node, the Kubelet starts the container using the container runtime, Kube-Proxy configures networking, and the Pod enters the Running state.

**Flow-Diagram:**


```text
kubectl
    │
    ▼
API Server
    │
    ▼
etcd (stores desired state)
    │
    ▼
Scheduler (selects Worker Node)
    │
    ▼
API Server
    │
    ▼
Kubelet (on Worker Node)
    │
    ▼
Container Runtime (pulls image & starts container)
    │
    ▼
Kube-Proxy (configures networking)
    │
    ▼
Pod Running
```

**2. What happens if the API server goes down ?**

The API Server is the central component of the Kubernetes control plane. It receives and processes all requests from users, kubectl, controllers, and other cluster components.

If the Kubernetes API server goes down:

* your running applications will continue to function normally, but you will completely lose the ability to manage or change the cluster state.
* No new operations can be performed, such as creating, updating, or deleting Pods, Services, or Deployments.
* Controllers and the Scheduler cannot manage or monitor the cluster, as they depend on the API Server.
* Applications remain available as long as the worker nodes and Pods are healthy.
* Existing Pods continue running on worker nodes if they are already running.

**3. What happens if a worker node goes down ?**

A Worker Node is responsible for running Pods and containerized applications. 

If a worker node goes down:

* All Pods running on that node become unavailable.
* The Node Controller detects that the node is not responding and marks it as NotReady.
* Kubernetes reschedules the affected Pods to other healthy worker nodes, provided they are managed by a Deployment, ReplicaSet, or StatefulSet.
* Users may experience temporary downtime until the Pods are recreated on another node.
* If there are no other available worker nodes, the Pods remain in a Pending state until a healthy node becomes available.

---

### ✅ Task 3 : Install kubectl

**`kubectl` is the CLI tool you will use to talk to your Kubernetes cluster.**

Install it:

```bash
# macOS
brew install kubectl

# Linux (amd64)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Windows (with chocolatey)
choco install kubernetes-cli
```

Verify:

```bash
kubectl version --client
```

---

### ✅ Task 4 : Set Up Your Local Cluster

**Choose one of the following. Both give you a fully functional Kubernetes cluster on your machine.**

Option A: kind (Kubernetes in Docker)

```bash

# Install kind
# macOS
brew install kind

# Linux
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Create a cluster
kind create cluster --name devops-cluster

# Verify
kubectl cluster-info
kubectl get nodes
```

Option B: minikube

```bash
# Install minikube
# macOS
brew install minikube

# Linux
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start a cluster
minikube start

# Verify
kubectl cluster-info
kubectl get nodes
```

**Which one did you choose and why ?**

I choosed option A: Kind because it run clusters by creating Docker containers which can act as nodes.

---

### ✅ Task 5 : Explore Your Cluster

Now that your cluster is running, explore it:

```bash
# See cluster info
kubectl cluster-info

# List all nodes
kubectl get nodes

# Get detailed info about your node
kubectl describe node <node-name>

# List all namespaces
kubectl get namespaces

# See ALL pods running in the cluster (across all namespaces)
kubectl get pods -A
```
Look at the pods running in the `kube-system` namespace:

```bash
kubectl get pods -n kube-system
```

You should see pods like `etcd`, `kube-apiserver`, `kube-scheduler`, `kube-controller-manager`, `coredns`, and `kube-proxy`. These are the architecture components you drew in Task 2 — running as pods inside the cluster.

---

### ✅ Task 6 : Practice Cluster Lifecycle

Build muscle memory with cluster operations:

```bash
# Delete your cluster
kind delete cluster --name devops-cluster
# (or: minikube delete)

# Recreate it
kind create cluster --name devops-cluster
# (or: minikube start)

# Verify it is back
kubectl get nodes
```
Try these useful commands:

```bash
# Check which cluster kubectl is connected to
kubectl config current-context

# List all available contexts (clusters)
kubectl config get-contexts

# See the full kubeconfig
kubectl config view
```

**What is a kubeconfig? Where is it stored on your machine?**

A kubeconfig is a YAML file used by the Kubernetes command-line tool (`kubectl`) and other components to access and authenticate with a Kubernetes cluster. It holds the cluster endpoint URLs, security certificates, tokens, and user credentials.

By default, it is stored in the `.kube` directory in your user's home folder:

* Linux/macOS: `~/.kube/config`
* Windows: `%USERPROFILE%\.kube\config`

---
