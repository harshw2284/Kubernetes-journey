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

**1. Write a custom Nginx config file that adds a `/health` endpoint returning "healthy"**

```conf
server {
    listen       80;
    server_name  localhost;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    # Custom health check endpoint
    location /health {
        access_log off;
        default_type text/plain;
        return 200 'healthy';
    }

    # Error pages redirection
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

**2. Create a ConfigMap from this file using `kubectl create configmap nginx-config --from-file=default.conf=<your-file>`**

**3. The key name (`default.conf`) becomes the filename when mounted into a Pod**

**Verify**: Does `kubectl get configmap nginx-config -o yaml` show the file contents ?

```yml
apiVersion: v1
data:
  default.conf: |
    server {
        listen       80;
        server_name  localhost;

        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }

        # Custom health check endpoint
        location /health {
            access_log off;
            default_type text/plain;
            return 200 'healthy';
        }

        # Error pages redirection
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   /usr/share/nginx/html;
        }
    }
kind: ConfigMap
metadata:
  creationTimestamp: "2026-07-14T18:51:50Z"
  name: nginx-config
  namespace: default
  resourceVersion: "88592"
  uid: 453ec1c7-b73e-4b6a-9948-b6362bab912a
```

Yes ! , The file contents are shown.

---

### ✅ Task 3 : Use ConfigMaps in a Pod

**1. Write a Pod manifest that uses `envFrom` with `configMapRef` to inject all keys from `app-config` as environment variables. Use a busybox container that prints the values.**

```yml
apiVersion: v1
kind: Pod
metadata:
  name: env-configmap-pod
spec:
  containers:
    - name: busybox
      command: [ "/bin/sh", "-c", "env && sleep 3600" ]
      image: busybox:1.36
      envFrom:
        - configMapRef:
            name: nginx-config
```

**2. Write a second Pod manifest that mounts `nginx-config` as a volume at `/etc/nginx/conf.d`. Use the nginx image.**

```yml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-volume-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    volumeMounts:
    - name: config-volume
      mountPath: /etc/nginx/conf.d
  volumes:
  - name: config-volume
    configMap:
      name: nginx-config
```

**3. Test that the mounted config works: `kubectl exec <pod> -- curl -s http://localhost/health`**

Use environment variables for simple key-value settings. Use volume mounts for full config files.

**Verify**: Does the `/health` endpoint respond ?

Yes ! It responded with `healthy` .

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
