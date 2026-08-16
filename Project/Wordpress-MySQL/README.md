# 🚀 Kubernetes WordPress + MySQL Deployment (Kind Cluster)

A production-style Kubernetes project that deploys a **WordPress + MySQL** application on a **Kind (Kubernetes in Docker)** cluster using Kubernetes manifests.

This project demonstrates how to deploy a real-world stateful application while following Kubernetes best practices including ConfigMaps, Secrets, StatefulSets, Deployments, Persistent Volumes, Health Probes, Resource Limits, and Horizontal Pod Autoscaling.

---

# 📌 Project Architecture

```
                    Browser
                       │
                       │
              NodePort Service
                       │
        ┌──────────────┴──────────────┐
        │                             │
  WordPress Pod                  WordPress Pod
 (Deployment)                   (Deployment)
        │                             │
        └──────────────┬──────────────┘
                       │
          mysql-0.mysql Service
             (Headless Service)
                       │
                  MySQL Pod
                (StatefulSet)
                       │
               Persistent Volume
```

---

# 📂 Project Structure

```
.
├── namespace.yaml
├── secret.yaml
├── configmap.yaml
├── mysql-headless-service.yaml
├── mysql-statefulset.yaml
├── wordpress-deployment.yaml
├── wordpress-service.yaml
├── hpa.yaml
└── README.md
```

---

# ✨ Features

- ✅ MySQL deployed using StatefulSet
- ✅ Headless Service for stable MySQL networking
- ✅ Secrets for database credentials
- ✅ ConfigMap for application configuration
- ✅ Liveness Probe
- ✅ Readiness Probe
- ✅ CPU & Memory Requests/Limits
- ✅ Horizontal Pod Autoscaler (HPA)
- ✅ Self-Healing Pods
- ✅ Data Persistence after Pod restart

---

# 🛠 Technologies Used

- Kubernetes
- Kind
- Docker
- kubectl
- WordPress
- MySQL 8.0
- YAML

---


# 🚀 Getting Started

## Prerequisites

- Docker
- Kind
- kubectl

Verify installation

```bash
docker --version
kind --version
kubectl version --client
```

---

# Step 1: Create Kind Cluster

```bash
kind create cluster --name wordpress-cluster
```

Verify

```bash
kubectl cluster-info
kubectl get nodes
```

---

# Step 2: Create Namespace

```bash
kubectl apply -f namespace.yaml
```

or

```bash
kubectl create namespace capstone
kubectl config set-context --current --namespace=capstone
```

---

# Step 3: Deploy MySQL

Apply the following manifests:

```bash
kubectl apply -f secret.yaml

kubectl apply -f mysql-headless-service.yaml

kubectl apply -f mysql-statefulset.yaml
```

Verify

```bash
kubectl get pods

kubectl get pvc

kubectl get svc
```

---

# Step 4: Verify MySQL

Connect to MySQL

```bash
kubectl exec -it mysql-0 -- bash
```

Inside container

```bash
mysql -u wordpress -p
```

Show databases

```sql
SHOW DATABASES;
```

You should see:

```
wordpress
```

---

# Step 5: Deploy WordPress

```bash
kubectl apply -f configmap.yaml

kubectl apply -f wordpress-deployment.yaml

kubectl apply -f wordpress-service.yaml
```

Verify

```bash
kubectl get pods

kubectl get deploy

kubectl get svc
```

Wait until both WordPress Pods are

```
Running
```

and

```
READY 1/1
```

---

# Step 6: Access WordPress

Since this project uses **Kind**, expose the service using port-forward:

```bash
kubectl port-forward svc/wordpress 8080:80
```

Open your browser

```
http://localhost:8080
```

Complete the WordPress installation wizard.

---

# Step 7: Configure Horizontal Pod Autoscaler

```bash
kubectl apply -f hpa.yaml
```

Verify

```bash
kubectl get hpa
```

---

# 📊 Verify Deployment

```bash
kubectl get all -n capstone
```

---

## Installing Argo CD

- Create a namespace for Argo CD:
  ```bash
  kubectl create namespace argocd
  ```

- Apply the Argo CD manifest:
  ```bash
  kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
  ```

- Check services in Argo CD namespace:
  ```bash
  kubectl get svc -n argocd
  ```

- Expose Argo CD server using NodePort:
  ```bash
  kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
  ```

- Forward ports to access Argo CD server:
  ```bash
  kubectl port-forward -n argocd service/argocd-server 8443:443 &
  ```

## Argo CD Initial Admin Password

- Retrieve Argo CD admin password:
  ```bash
  kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
  ```

## Install Kube Prometheus Stack

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add stable https://charts.helm.sh/stable
helm repo update
kubectl create namespace monitoring
helm install kind-prometheus prometheus-community/kube-prometheus-stack --namespace monitoring --set prometheus.service.nodePort=30000 --set prometheus.service.type=NodePort --set grafana.service.nodePort=31000 --set grafana.service.type=NodePort --set alertmanager.service.nodePort=32000 --set alertmanager.service.type=NodePort --set prometheus-node-exporter.service.nodePort=32001 --set prometheus-node-exporter.service.type=NodePort
kubectl get svc -n monitoring
kubectl get namespace
```

---

```bash
kubectl port-forward svc/kind-prometheus-kube-prome-prometheus -n monitoring 9090:9090 --address=0.0.0.0 &
kubectl port-forward svc/kind-prometheus-grafana -n monitoring 31000:80 --address=0.0.0.0 &
```


---

## Prometheus Queries

```bash
sum (rate (container_cpu_usage_seconds_total{namespace="default"}[1m])) / sum (machine_cpu_cores) * 100

sum (container_memory_usage_bytes{namespace="default"}) by (pod)


sum(rate(container_network_receive_bytes_total{namespace="default"}[5m])) by (pod)
sum(rate(container_network_transmit_bytes_total{namespace="default"}[5m])) by (pod)

```
  
---

# 📷 Sample Commands

View resources

```bash
kubectl get all -n capstone
```

Describe Pod

```bash
kubectl describe pod <pod-name>
```

View Logs

```bash
kubectl logs <pod-name>
```

Delete Everything

```bash
kubectl delete namespace capstone
```

Delete Cluster

```bash
kind delete cluster --name wordpress-cluster
```

---

# 🎯 Learning Outcomes

Through this project I learned:

- Kubernetes object relationships
- Deploying stateless vs stateful applications
- Managing sensitive data using Secrets
- Application configuration using ConfigMaps
- Persistent storage using PVCs
- Service discovery with Headless Services
- Resource management
- Health checks using Liveness & Readiness Probes
- Horizontal Pod Autoscaling
- Kubernetes self-healing capabilities
- Deploying applications on a Kind cluster

---
