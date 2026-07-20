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

- ✅ Dedicated Kubernetes Namespace
- ✅ MySQL deployed using StatefulSet
- ✅ Persistent Storage using PVC
- ✅ Headless Service for stable MySQL networking
- ✅ WordPress Deployment with 2 replicas
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

# 📖 Kubernetes Concepts Covered

This project demonstrates the following Kubernetes resources:

| Resource | Purpose |
|----------|----------|
| Namespace | Isolate project resources |
| Secret | Store MySQL credentials securely |
| ConfigMap | Store WordPress configuration |
| Deployment | Manage WordPress replicas |
| StatefulSet | Deploy stateful MySQL |
| Headless Service | Stable DNS for StatefulSet |
| NodePort Service | Expose WordPress |
| PersistentVolumeClaim | Persistent MySQL storage |
| Liveness Probe | Restart unhealthy containers |
| Readiness Probe | Route traffic only to healthy Pods |
| Resource Requests/Limits | Efficient resource management |
| Horizontal Pod Autoscaler | Automatic scaling |

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

Expected resources:

- Namespace
- Deployment
- StatefulSet
- Services
- Pods
- PVC
- ConfigMap
- Secret
- HPA

---

# 🔁 Self-Healing Test

Delete a WordPress Pod

```bash
kubectl delete pod <wordpress-pod-name>
```

A new Pod should be created automatically.

Delete the MySQL Pod

```bash
kubectl delete pod mysql-0
```

The StatefulSet recreates the Pod while preserving the database using the Persistent Volume.

---

# 💾 Persistence Test

1. Create a blog post.
2. Delete the MySQL Pod.

```bash
kubectl delete pod mysql-0
```

3. Wait until MySQL becomes Running.
4. Refresh WordPress.

The blog post should still exist, demonstrating persistent storage.

---

# 📈 Horizontal Pod Autoscaler

Configured with:

- Minimum Replicas: **2**
- Maximum Replicas: **10**
- CPU Utilization Target: **50%**

Verify

```bash
kubectl get hpa
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

# 📸 Screenshots

You can add screenshots here:

- WordPress Home Page
- WordPress Admin Dashboard
- `kubectl get all`
- HPA Output
- PVC
- StatefulSet
- Kind Cluster

---

# 👨‍💻 Author

**Harsh Bhagat**

If you found this project helpful, feel free to ⭐ the repository.
