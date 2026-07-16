# Kubernetes – Day 08 - Resource Requests, Limits, and Probes

My Pods are running, but Kubernetes has no idea how much CPU or memory they need — and no way to tell if they are actually healthy. Today I set resource requests and limits for smart scheduling, then add probes so Kubernetes can detect and recover from failures automatically.

---

### ✅ Task 1: Resource Requests and Limits

1. Write a Pod manifest with `resources.requests` (cpu: 100m, memory: 128Mi) and `resources.limits` (cpu: 250m, memory: 256Mi)

```yml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx-app
    image: nginx:1.25
    resources:
      requests:
        memory: 128Mi
        cpu: 100m
      limits:
        memory: 256Mi
        cpu: 250m
```

2. Apply and inspect with `kubectl describe pod` — look for the Requests, Limits, and QoS Class sections

3. Since requests and limits differ, the QoS class is `Burstable`. If equal, it would be `Guaranteed`. If missing, `BestEffort`.

CPU is in millicores: `100m` = 0.1 CPU. Memory is in mebibytes: `128Mi`.

**Requests** = guaranteed minimum (scheduler uses this for placement). **Limits** = maximum allowed (kubelet enforces at runtime).

**Verify**: What QoS class does your Pod have ?

It shows Burstable.

---

### ✅ Task 2 : OOMKilled — Exceeding Memory Limits

1. Write a Pod manifest using the `polinux/stress` image with a memory limit of `100Mi`

```yml
apiVersion: v1
kind: Pod
metadata:
  name: stress
spec:
  containers:
  - name: stress
    image: polinux/stress

    resources:
      limits:
        memory: 100Mi
```

2. Set the stress command to allocate 200M of memory: `command: ["stress"] args: ["--vm", "1", "--vm-bytes", "200M", "--vm-hang", "1"]`

```yml
apiVersion: v1
kind: Pod
metadata:
  name: stress
spec:
  containers:
  - name: stress
    image: polinux/stress
    command: ["stress"] 
    args: ["--vm", "1", "--vm-bytes", "200M", "--vm-hang", "1"]
    
    resources:
      limits:
        memory: 100Mi
```

3. Apply and watch — the container gets killed immediately

CPU is throttled when over limit. Memory is killed — no mercy.

Check `kubectl describe pod` for `Reason: OOMKilled and Exit Code: 137` (128 + SIGKILL).

**Verify**: What exit code does an OOMKilled container have ?

It has exit code 137.

---

### ✅ Task 3 : Pending Pod — Requesting Too Much

1. Write a Pod manifest requesting `cpu: 100` and `memory: 128Gi`

```yml
apiVersion: v1
kind: Pod
metadata:
  name: pending-pod

spec:
  containers:
  - name: heavy-container
    image: nginx:alpine
    resources:
      requests:
        cpu: "100"
        memory: 128Gi
```

2. Apply and check — STATUS stays `Pending` forever

3. Run `kubectl describe pod` and read the Events — the scheduler says exactly why: insufficient resources

**Verify**: What event message does the scheduler produce ?

It produces `Warning  FailedScheduling  10s   default-scheduler  0/3 nodes are available: 3 Insufficient cpu, 3 Insufficient memory.`

---

### ✅ Task 4 : Liveness Probe

A liveness probe detects stuck containers. If it fails, Kubernetes restarts the container.

1. Write a Pod manifest with a busybox container that creates `/tmp/healthy` on startup, then deletes it after 30 seconds

```yml
apiVersion: v1
kind: Pod
metadata:
  name: probe-pod

spec:
  containers:
  - name: busybox
    image: busybox:1.36
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600
```

2. Add a liveness probe using `exec` that runs `cat /tmp/healthy`, with `periodSeconds: 5` and `failureThreshold: 3`

```yml
apiVersion: v1
kind: Pod
metadata:
  name: probe-pod

spec:
  containers:
  - name: busybox
    image: busybox:1.36
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600

    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      periodSeconds: 5
      failureThreshold: 3
```

3. After the file is deleted, 3 consecutive failures trigger a restart. Watch with `kubectl get pod -w`

Verify: How many times has the container restarted ?

The container should restart approximately 45 seconds after startup because:

* File exists for the first 30 seconds.
* Probe checks every 5 seconds.
* It takes 3 consecutive failures (`failureThreshold: 3`) before Kubernetes restarts the container.

The restart count (`RESTARTS`) should be at least 1 after the first cycle and will continue to increase as the container repeats the same behavior.

---

### ✅ Task 5 : Readiness Probe

A readiness probe controls traffic. Failure removes the Pod from Service endpoints but does NOT restart it.

1. Write a Pod manifest with nginx and a `readinessProbe` using `httpGet` on path `/` port `80`

```yml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-pod
  labels:                     # Must add to expose it to a service
    app: readiness-test
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80

    readinessProbe:
      httpGet:
        path: /
        port: 80
      periodSeconds: 5
      failureThreshold: 3
```

2. Expose it as a Service: `kubectl expose pod <name> --port=80 --name=readiness-svc`

To Expose it as service you must add label.

```bash
kubectl expose pod readiness-pod --port=80 --name=readiness-svc
```

3. Check `kubectl get endpoints readiness-svc` — the Pod IP is listed

4. Break the probe: `kubectl exec <pod> -- rm /usr/share/nginx/html/index.html`

```bash
kubectl exec readiness-pod -- rm /usr/share/nginx/html/index.html
```

5. Wait 15 seconds — Pod shows `0/1` READY, endpoints are empty, but the container is NOT restarted

**Verify**: When readiness failed, was the container restarted ?

No, the container was not restarted.

Unlike a liveness probe (which restarts unhealthy containers), a readiness probe only controls traffic routing.

* The Pod’s status changes to 0/1 READY.
* The Pod’s IP is removed from the readiness-svc endpoints so clients no longer hit a broken container.
* The container itself continues running uninterrupted (restarts will remain at 0), giving you a chance to inspect or debug the running container without losing its state to a restart.

---

### ✅ Task 6 : Startup Probe

A startup probe gives slow-starting containers extra time. While it runs, liveness and readiness probes are disabled.

1. Write a Pod manifest where the container takes 20 seconds to start (e.g., sleep 20 && touch /tmp/started)

```yml
apiVersion: v1
kind: Pod
metadata:
  name: slow-pod

spec:
  containers:
  - name: busybox
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
    - |
      sleep 20
      touch /tmp/started
```

Add a startupProbe checking for /tmp/started with periodSeconds: 5 and failureThreshold: 12 (60 second budget)

```yml
apiVersion: v1
kind: Pod
metadata:
  name: slow-pod

spec:
  containers:
  - name: busybox
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
    - |
      sleep 20
      touch /tmp/started

    startupProbe:
      exec:
        command:
        - cat
        - /tmp/started
      periodSeconds: 5
      failureThreshold: 12
```

Add a livenessProbe that checks the same file — it only kicks in after startup succeeds

```yml
apiVersion: v1
kind: Pod
metadata:
  name: slow-pod

spec:
  containers:
  - name: busybox
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
    - |
      sleep 20
      touch /tmp/started

    startupProbe:
      exec:
        command:
        - cat
        - /tmp/started
      periodSeconds: 5
      failureThreshold: 12

    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/started
      periodSeconds: 5
      failureThreshold: 3
```

Verify: What would happen if `failureThreshold` were 2 instead of 12 ?

If the failureThreshold on the startupProbe were set to 2 (with a periodSeconds of 5):

* The Startup Budget is Cut Short: The total startup budget would be reduced to only 10 seconds (2 * 5 seconds).
* Premature Failures & Restart Loop: Since the container is hardcoded to sleep for 20 seconds before creating the `/tmp/started` file, the startup probe will fail its first check (at ~5 seconds) and its second check (at ~10 seconds).
* The Outcome: Because the probe hits 2 consecutive failures before the file is created, Kubernetes will conclude that the container has failed to start. It will kill the container and restart it. This will trap the Pod in an endless, frustrating crash-restart loop because the container can never get the 20 seconds it needs to boot up successfully.

---

### ✅ Task 7 : Clean Up

Delete all pods and services you created.

**Note**:

* CPU is compressible (throttled); memory is incompressible (OOMKilled)
* CPU: `1` = 1 core = `1000m`. Memory: `Mi` (mebibytes), `Gi` (gibibytes)
* QoS: Guaranteed (requests == limits), Burstable (requests < limits), BestEffort (none set)
* Probe types: `httpGet`, `exec`, `tcpSocket`
* Liveness failure = restart. Readiness failure = remove from endpoints. Startup failure = kill.
* `initialDelaySeconds`, `periodSeconds`, `failureThreshold` control probe timing
* Exit code 137 = OOMKilled (128 + SIGKILL)

---
