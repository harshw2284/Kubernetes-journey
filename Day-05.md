# Kubernetes – Day 05 - Kubernetes ConfigMaps and Secrets

Oour application needs configuration — database URLs, feature flags, API keys. Hardcoding these into container images means rebuilding every time a value changes. Kubernetes solves this with ConfigMaps for non-sensitive config and Secrets for sensitive data.

---

### ✅ Task 1: Create a ConfigMap from Literals

**1. Use `kubectl create configmap` with `--from-literal` to create a ConfigMap called `app-config` with keys `APP_ENV=production`, `APP_DEBUG=false`, and `APP_PORT=8080**`**

```bash
kubectl create configmap app-config --from-literal=APP_ENV=production --from-literal=APP_DEBUG=false --from-literal=APP_PORT=8080
```

**2. Inspect it with `kubectl describe configmap app-config` and `kubectl get configmap app-config -o yaml`**

```yml
apiVersion: v1
data:
  APP_DEBUG: "false"
  APP_ENV: production
  APP_PORT: "8080"
kind: ConfigMap
metadata:
  creationTimestamp: "2026-07-13T17:46:17Z"
  name: app-config
  namespace: default
  resourceVersion: "57809"
  uid: 1e614740-7f69-40d4-ad24-3f0173801b4e
```

**3. Notice the data is stored as plain text — no encoding, no encryption**

**Verify**: Can you see all three key-value pairs ?

Yes !

---

### ✅ Task 2 : Create a ConfigMap from a File

ClusterIP is the default Service type. It gives your Pods a stable internal IP that is only reachable from within the cluster.

Create `clusterip-service.yaml`:

```yml
apiVersion: v1
kind: Service
metadata:
  name: web-app-clusterip
spec:
  type: ClusterIP
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
```
Key fields:

* `selector.app: web-app` — this Service routes traffic to all Pods with the label app: web-app
* `port: 80` — the port the Service listens on
* `targetPort: 80` — the port on the Pod to forward traffic to

```bash
kubectl apply -f clusterip-service.yaml
kubectl get services
```

You should see `web-app-clusterip` with a CLUSTER-IP address. This IP is stable — it will not change even if Pods restart.

Now test it from inside the cluster:

```bash
# Run a temporary pod to test connectivity
kubectl run test-client --image=busybox:latest --rm -it --restart=Never -- sh

# Inside the test pod, run:
wget -qO- http://web-app-clusterip
exit
```

You should see the Nginx welcome page. The Service load-balanced your request to one of the 3 Pods.

Verify: Does the Service respond ? Try running the wget command multiple times — the Service distributes traffic across all healthy Pods.

Yes ! Service responded.

---

### ✅ Task 3 : Create Your First Deployment

Kubernetes has a built-in DNS server. Every Service gets a DNS entry automatically:

Test this:

```bash
<service-name>.<namespace>.svc.cluster.local
Test this:

kubectl run dns-test --image=busybox:latest --rm -it --restart=Never -- sh

# Inside the pod:
# Short name (works within the same namespace)
wget -qO- http://web-app-clusterip

# Full DNS name
wget -qO- http://web-app-clusterip.default.svc.cluster.local

# Look up the DNS entry
nslookup web-app-clusterip
exit
```

Both the short name and the full DNS name resolve to the same ClusterIP. In practice, you use the short name when communicating within the same namespace and the full name when reaching across namespaces.

Verify: What IP does `nslookup` return? Does it match the CLUSTER-IP from `kubectl get services` ?

---

### ✅ Task 4 : Create a Secret

**1. Use `kubectl create secret generic db-credentials` with `--from-literal` to store `DB_USER=admin` and `DB_PASSWORD=s3cureP@ssw0rd`**

```bash
kubectl create secret generic db-credentials --from-literal=DB_USER=admin --from-literal=DB_PASSWORD=s3cureP@ssw0rd
```

**2. Inspect with `kubectl get secret db-credentials -o yaml` — the values are base64-encoded**

```yml
apiVersion: v1
items:
- apiVersion: v1
  data:
    DB_PASSWORD: czNjdXJlUEBzc3cwcmQ=
    DB_USER: YWRtaW4=
  kind: Secret
  metadata:
    creationTimestamp: "2026-07-14T11:45:08Z"
    name: db-credentials
    namespace: default
    resourceVersion: "63696"
    uid: 1fbd16f9-0bb9-49fd-ad35-8e59bf3fb65c
  type: Opaque
kind: List
metadata:
  resourceVersion: ""
```

**3. Decode a value: `echo '<base64-value>' | base64 --decode`**

base64 is encoding, not encryption. Anyone with cluster access can decode Secrets. The real advantages are RBAC separation, tmpfs storage on nodes, and optional encryption at rest.

Verify:Can you decode the password back to plaintext ?

Yes !

---

### ✅ Task 5 : Use Secrets in a Pod

**1. Write a Pod manifest that injects `DB_USER` as an environment variable using `secretKeyRef`**

```yml
kind: Pod
apiVersion: v1

metadata:
  name: secret-pod

spec:
  containers:
  - name: mypod
    image: nginx:latest

    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: DB_USER
```

**2. In the same Pod, mount the entire `db-credentials` Secret as a volume at `/etc/db-credentials` with `readOnly: true`**

```yml
kind: Pod
apiVersion: v1

metadata:
  name: secret-pod

spec:
  containers:
  - name: mypod
    image: nginx:latest

    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: DB_USER

    volumeMounts:
      - name: db-credentials
        mountPath: "/etc/db-credentials"
        readOnly: true

  volumes:
  - name: db-credentials
    secret: 
      secretName: db-credentials
```

**3. Verify: each Secret key becomes a file, and the content is the decoded plaintext value**

```bash
kubectl exec -it secret-demo -- printenv DB_USER   [ Output -> admin ]
kubectl exec -it secret-demo -- ls /etc/db-credentials  [ Output -> DB_PASSWORD  DB_USER ] 
```
**Verify**: Are the mounted file values plaintext or base64 ?

Mounted file values are plaintext.

---

### ✅ Task 6 : Rolling Update

Check all three services:

```bash
kubectl get services -o wide
```

Compare them:

| Type | Accessible From | Use Case |
|------|----------------|----------|
| ClusterIP | Inside the cluster only | Internal communication between services |
| NodePort | Outside via `<NodeIP>:<NodePort>` | Development, testing, direct node access |
| LoadBalancer | Outside via cloud load balancer | Production traffic in cloud environments |

Each type builds on the previous one:

* LoadBalancer creates a NodePort, which creates a ClusterIP
* So a LoadBalancer service also has a ClusterIP and a NodePort

Verify this:

```bash
kubectl describe service web-app-loadbalancer
```

You should see all three: a ClusterIP, a NodePort, and the LoadBalancer configuration.

Verify: Does the LoadBalancer service also have a ClusterIP and NodePort assigned?

Yes !

---

### ✅ Task 7 : Clean Up

Delete all pods, ConfigMaps, and Secrets you created.

**Note**:

* `kubectl get endpoints <service-name>` shows which Pod IPs a Service is currently routing to
* `port` is what the Service listens on; `targetPort` is what the Pod listens on — they do not have to be the same number
* To test ClusterIP services, you must test from inside the cluster (use a temporary pod)

---
