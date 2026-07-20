# Kubernetes – Day 11 - Capstone: Deploy WordPress + MySQL on Kubernetes

Ten days of Kubernetes — clusters, Pods, Deployments, Services, ConfigMaps, Secrets, storage, StatefulSets, resource management, autoscaling, and Helm. Today I put it all together. Deploy a real WordPress + MySQL application using every major concept I have learned.

---

### ✅ Task 1: Create the Namespace (Day 3)

1. Create a `capstone` namespace

2. Set it as your default: `kubectl config set-context --current --namespace=capstone`

---

### ✅ Task 2 : Deploy MySQL (Days 5-7)

1. Create a Secret with `MYSQL_ROOT_PASSWORD`, `MYSQL_DATABASE`, `MYSQL_USER`, and `MYSQL_PASSWORD` using `stringData`

```yml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: capstone
type: Opaque
stringData:
  MYSQL_ROOT_PASSWORD: "rootpassword123"
  MYSQL_DATABASE: "wordpress"
  MYSQL_USER: "wpuser"
  MYSQL_PASSWORD: "wppassword123"
```

2. Create a Headless Service (`clusterIP: None`) for MySQL on port 3306

```yml
kind: Service
apiVersion: v1

metadata:
  name: mysql
  namespace: capstone
  labels:
    app: capstone

spec:
  clusterIP: None
  selector:
    app: capstone

  ports:
    - port: 3306
      targetPort: 3306
```


3. Create a StatefulSet for MySQL with:

* Image: `mysql:8.0`
* `envFrom` referencing the Secret
* Resource requests (cpu: 250m, memory: 512Mi) and limits (cpu: 500m, memory: 1Gi)
* A `volumeClaimTemplates` section requesting 1Gi of storage, mounted at `/var/lib/mysql`

```yml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: capstone
  labels:
    app: capstone

spec:
  selector:
    matchLabels:
      app: capstone
  serviceName: mysql
  replicas: 1
  template:
    metadata:
      labels:
        app: capstone
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
          name: mysql                        
        resources:
          requests:
            memory: 512Mi
            cpu: 250m
          limits:
            memory: 1Gi
            cpu: 500m
        envFrom:
        - secretRef:
            name: mysql-secret

        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql

  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: standard
      resources:
        requests:
          storage: 1Gi
```

4. Verify MySQL works: `kubectl exec -it mysql-0 -- mysql -u <user> -p<password> -e "SHOW DATABASES;"`


```bash
kubectl exec -it mysql-0 -- mysql -u wpuser -pwppassword123 -e "SHOW DATABASES;"
```

Verify: Can you see the `wordpress` database?

Yes !

```txt
+--------------------+
| Database           |
+--------------------+
| information_schema |
| performance_schema |
| wordpress          |
+--------------------+
```

---

### ✅ Task 3 : Deploy WordPress (Days 3, 5, 8)

1. Create a ConfigMap with `WORDPRESS_DB_HOST` set to `mysql-0.mysql.capstone.svc.cluster.local:3306` and `WORDPRESS_DB_NAME`

```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: wordpress-config
  namespace: capstone
  labels:
    app: wordpress    
data:
  WORDPRESS_DB_HOST: mysql-0.mysql.capstone.svc.cluster.local:3306
  WORDPRESS_DB_NAME: wordpress
```

2. Create a Deployment with 2 replicas using `wordpress:latest` that:

* Uses `envFrom` for the ConfigMap
*Uses `secretKeyRef` for `WORDPRESS_DB_USER` and `WORDPRESS_DB_PASSWORD` from the MySQL Secret
* Has resource requests and limits
* Has a liveness probe and readiness probe on `/wp-login.php` port 80

```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: wordpress-config
  namespace: capstone
  labels:
    app: wordpress    
data:
  WORDPRESS_DB_HOST: mysql-0.mysql.capstone.svc.cluster.local:3306
  WORDPRESS_DB_NAME: wordpress


harsh@Lap:~/k8s-project$ cat deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: capstone
  labels:
    app: wordpress
spec:
  replicas: 2
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - name: wordpress
        image: wordpress:latest
        ports:
        - containerPort: 80

        envFrom:
        - configMapRef:
            name: wordpress-config

        env:
        - name: WORDPRESS_DB_USER
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_USER
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_PASSWORD

        resources:
          requests:
            memory: 512Mi
            cpu: 250m
          limits:
            memory: 1Gi
            cpu: 500m

        livenessProbe:
            httpGet:
              path: /wp-login.php
              port: 80
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
        readinessProbe:
            httpGet:
              path: /wp-login.php
              port: 80
            initialDelaySeconds: 15
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
```

3. Wait until both pods show `1/1 Running`

Verify: Are both WordPress pods running and ready ?

Yes ! Both WordPress pods are ready and running 

```txt
NAME                             READY   STATUS    RESTARTS   AGE
pod/wordpress-6bcc97f586-t2vj9   1/1     Running   0          50s
pod/wordpress-6bcc97f586-xm7xx   1/1     Running   0          50s
```

---

### ✅ Task 4 : Expose WordPress (Day 4)

1. Create a NodePort Service on port 30080 targeting the WordPress pods

```yml
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  namespace: capstone
  labels:
    app: wordpress
spec:
  type: NodePort
  selector:
    app: wordpress
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
      protocol: TCP
```

2. Access WordPress in your browser:
* Minikube: `minikube service wordpress -n capstone`
* Kind: `kubectl port-forward svc/wordpress 8080:80 -n capstone`

3. Complete the setup wizard and create a blog post

Verify: Can you see the WordPress setup page ?

<img width="495" height="641" alt="1-wp-setup-start" src="https://github.com/user-attachments/assets/cae3ef3b-d6bd-4cea-86af-b554099be432" />

---

### ✅ Task 5 : Test Self-Healing and Persistence

1. Delete a WordPress pod — watch the Deployment recreate it within seconds. Refresh the site.

```bash
kubectl delete pod/wordpress-6bcc97f586-cs2rp
```

2. Delete the MySQL pod: `kubectl delete pod mysql-0 -n capstone` — watch the StatefulSet recreate it

3. After MySQL recovers, refresh WordPress — your blog post should still be there

Verify: After deleting both pods, is your blog post still there ?

Yes ! They are still there.

---

### ✅ Task 6 : Set Up HPA (Day 9)

1. Write an HPA manifest targeting the WordPress Deployment with CPU at 50%, min 2, max 10 replicas

```yml
kind: HorizontalPodAutoscaler
apiVersion: autoscaling/v2

metadata:
  name: wordpress-hpa
  namespace: capstone

spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: wordpress
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

2. Apply and check: `kubectl get hpa -n capstone`

```bash
kubectl apply -f hpa.yml
```

3. Run `kubectl get all -n capstone` for the complete picture

```txt
NAME                             READY   STATUS    RESTARTS   AGE
pod/mysql-0                      1/1     Running   0          73m
pod/wordpress-6bcc97f586-t2vj9   1/1     Running   0          92m
pod/wordpress-6bcc97f586-xm7xx   1/1     Running   0          74m

NAME                TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
service/mysql       ClusterIP   None          <none>        3306/TCP       7h4m
service/wordpress   NodePort    10.96.49.29   <none>        80:30080/TCP   85m

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/wordpress   2/2     2            2           92m

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/wordpress-6bcc97f586   2         2         2       92m

NAME                     READY   AGE
statefulset.apps/mysql   1/1     7h4m

NAME                                                REFERENCE              TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/wordpress-hpa   Deployment/wordpress   cpu: 2%/50%   2         10        2          64m
```

Verify: Does the HPA show correct min/max and target ?

Yes !

```txt
NAME                                                REFERENCE              TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/wordpress-hpa   Deployment/wordpress   cpu: 2%/50%   2         10        2          2m
```

---

### ✅ Task 7 : (Bonus) Compare with Helm (Day 10)

1. Install WordPress using `helm install wp-helm bitnami/wordpress` in a separate namespace

2. Compare: how many resources did each approach create? Which gives more control?

3. Clean up the Helm deployment

### ✅ Task 8 : Clean Up and Reflect

1. Take a final look: `kubectl get all -n capstone`

```txt
NAME                             READY   STATUS    RESTARTS   AGE
pod/mysql-0                      1/1     Running   0          73m
pod/wordpress-6bcc97f586-t2vj9   1/1     Running   0          92m
pod/wordpress-6bcc97f586-xm7xx   1/1     Running   0          74m

NAME                TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
service/mysql       ClusterIP   None          <none>        3306/TCP       7h4m
service/wordpress   NodePort    10.96.49.29   <none>        80:30080/TCP   85m

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/wordpress   2/2     2            2           92m

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/wordpress-6bcc97f586   2         2         2       92m

NAME                     READY   AGE
statefulset.apps/mysql   1/1     7h4m

NAME                                                REFERENCE              TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/wordpress-hpa   Deployment/wordpress   cpu: 2%/50%   2         10        2          64m
```

2. Count the concepts you used: Namespace, Secret, ConfigMap, PVC, StatefulSet, Headless Service, Deployment, NodePort Service, Resource Limits, Probes, HPA, Helm — twelve concepts in one deployment

3. Delete the namespace: `kubectl delete namespace capstone`

4. Reset default: `kubectl config set-context --current --namespace=default`

Verify: Did deleting the namespace remove everything ?

Yes, deleting the capstone namespace removes everything created within it.

**Note**:

- If MySQL takes long to start, check: `kubectl logs mysql-0 -n capstone`
- `WORDPRESS_DB_HOST` must match the StatefulSet DNS pattern: `<pod>.<service>.<namespace>.svc.cluster.local`
- WordPress probes may fail initially — `initialDelaySeconds` gives it time to boot
- If PVC stays Pending, check `kubectl get storageclass`
- `nodePort` must be in range 30000-32767
- The Bitnami chart uses MariaDB instead of MySQL — compatible but not identical

---
