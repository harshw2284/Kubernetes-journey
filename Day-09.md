# Kubernetes – Day 09 - Metrics Server and Horizontal Pod Autoscaler (HPA)

Yesterday I set resource requests and limits. Today I put that to work. Install the Metrics Server so Kubernetes can see actual resource usage, then set up a Horizontal Pod Autoscaler that scales our app up under load and back down when things calm down.

---

### ✅ Task 1: Install the Metrics Server

1. Check if it is already running: `kubectl get pods -n kube-system | grep metrics-server`

2. If not, install it:
* Minikube: `minikube addons enable metrics-server`
* Kind/kubeadm: apply the official manifest from the metrics-server GitHub releases

3. On local clusters, you may need the `--kubelet-insecure-tls` flag (never in production)

4. Wait 60 seconds, then verify: `kubectl top nodes` and `kubectl top pods -A`

**Verify**: What is the current CPU and memory usage of your node ?

```txt
NAME                        CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)
hwb-cluster-control-plane   230m         1%       640Mi           8%
hwb-cluster-worker          39m          0%       328Mi           4%
hwb-cluster-worker2         42m          0%       356Mi           4%
hwb-cluster-worker3         52m          0%       382Mi           4%
```

---

### ✅ Task 2 : Explore kubectl top

1. Run `kubectl top nodes`, `kubectl top pods -A`, `kubectl top pods -A --sort-by=cpu`

2. `kubectl top` shows real-time usage, not requests or limits — these are different things

3. Data comes from the Metrics Server, which polls kubelets every 15 seconds

Verify: Which pod is using the most CPU right now ?

`kube-apiserver-hwb-cluster-control-plane` in `kube-system` namespace is using the most CPU.

---

### ✅ Task 3 : Pending Pod — Requesting Too Much

1. Write a Deployment manifest using the `registry.k8s.io/hpa-example` image (a CPU-intensive PHP-Apache server)

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
  labels:
    app: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
```

It's Important to add Labels.

2. Set `resources.requests.cpu: 200m` — HPA needs this to calculate utilization percentages

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
  labels:
    app: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m
```

3. Expose it as a Service: `kubectl expose deployment php-apache --port=80`

Without CPU requests, HPA cannot work — this is the most common HPA setup mistake.

**Verify**: What is the current CPU usage of the Pod ?

Current CPU usage of php-apache pod is 1m.

---

### ✅ Task 4 : Create an HPA (Imperative)

1. Run: `kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10`

2. Check: `kubectl get hpa` and `kubectl describe hpa php-apache`

```txt
NAME         REFERENCE               TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   cpu: 0%/50%   1         10        1          20s
```

3. TARGETS may show `<unknown>` initially — wait 30 seconds for metrics to arrive

This scales up when average CPU exceeds 50% of requests, and down when it drops below.

Verify: What does the TARGETS column show ?

 TARGETS column show `cpu: 0%/50%`.

---

### ✅ Task 5 : Generate Load and Watch Autoscaling

1. Start a load generator: `kubectl run load-generator --image=busybox:1.36 --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://php-apache; done"`

2. Watch HPA: `kubectl get hpa php-apache --watch`

3. Over 1-3 minutes, CPU climbs above 50%, replicas increase, CPU stabilizes

```txt
php-apache   Deployment/php-apache   cpu: 0%/50%     1         10        1          9m11s
php-apache   Deployment/php-apache   cpu: 257%/50%   1         10        1          9m37s
php-apache   Deployment/php-apache   cpu: 490%/50%   1         10        4          9m52s
php-apache   Deployment/php-apache   cpu: 319%/50%   1         10        8          10m
php-apache   Deployment/php-apache   cpu: 164%/50%   1         10        10         10m      <-- Max Replicas Form
```

4. Stop the load: `kubectl delete pod load-generator`

5. Scale-down is slow (5-minute stabilization window) — you do not need to wait

Verify: How many replicas did HPA scale to under load ?

It scale Max upto 10 replicas to handle the load.

---

### ✅ Task 6 : Create an HPA from YAML (Declarative)

1. Delete the imperative HPA: `kubectl delete hpa php-apache`

2. Write an HPA manifest using `autoscaling/v2` API with CPU target at 50% utilization

```yml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

3. Add a `behavior` section to control scale-up speed (no stabilization) and scale-down speed (300 second window)

```yml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
```

4. Apply and verify with `kubectl describe hpa`

`autoscaling/v2` supports multiple metrics and fine-grained scaling behavior that the imperative command cannot configure.

**Verify**: What does the behavior section control ?

It allows to configure:

* `Scale-Up` Speed: You can control how quickly the cluster adds replicas when load spikes (e.g., opting for no stabilization window to scale up immediately to handle sudden traffic surges).

* `Scale-Down` Speed: You can establish a cooldown window (such as a 300-second stabilization window) to hold onto peak replica counts temporarily. This prevents "thrashing"—the rapid, inefficient looping of scaling up and down when traffic fluctuates slightly.

---

### ✅ Task 7 : Clean Up

Delete the HPA, Service, Deployment, and load-generator pod. Leave the Metrics Server installed.

**Note**:

* HPA requires `resources.requests` — without them TARGETS shows `<unknown>`
* `kubectl top` = actual usage. `kubectl describe pod` = configured requests/limits
* HPA checks every 15 seconds. Scale-up is fast, scale-down has a 5-minute stabilization window
* `autoscaling/v1` = CPU only. autoscaling/v2 = CPU + memory + custom metrics
* Formula: `desiredReplicas = ceil(currentReplicas * (currentUsage / targetUsage))`
* HPA works with Deployments, StatefulSets, and ReplicaSets

---
