# Kubernetes – Day 07 - Kubernetes StatefulSets

Deployments work great for stateless apps, but what about databases ? You need stable pod names, ordered startup, and persistent storage per replica. Today I will learn StatefulSets — the workload designed for stateful applications like MySQL, PostgreSQL, and Kafka.

---

### ✅ Task 1: See the Problem — Data Lost on Pod Deletion

**1. Write a Pod manifest that uses an `emptyDir` volume and writes a timestamped message to `/data/message.txt`**

```yml
kind: Pod
apiVersion: v1

metadata:
  name: emptydir

spec:
  containers:
    - name: writer
      image: busybox
      command: [ "/bin/sh", "-c" ]
      args:
      - |
        echo "Log created at: $(date)" > /data/message.txt
        # Keep the container running so we can inspect it
        sleep 3600

      volumeMounts:
      - name: data-volume
        mountPath: /data

  volumes:
  - name: data-volume
    emptyDir: {}
```

**2. Apply it, verify the data exists with `kubectl exec`**

```bash
kubectl apply -f empty-pod.yml
 kubectl exec emptydir -- cat /data/message.txt
```

**3. Delete the Pod, recreate it, check the file again — the old message is gone**

```bash
kubectl delete pod emptydir
kubectl apply -f empty-pod.yml
```

**Verify**: Is the timestamp the same or different after recreation?

Timestamp is different 

---

### ✅ Task 2 : Create a PersistentVolume (Static Provisioning)

**1. Write a PV manifest with `capacity: 1Gi`, `accessModes: ReadWriteOnce`, `persistentVolumeReclaimPolicy: Retain`, and `hostPath` pointing to `/tmp/k8s-pv-data`**

```yml
kind: PersistentVolume
apiVersion: v1

metadata:
  name: pv-volume

spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: "/tmp/k8s-pv-data"
```

**2. Apply it and check `kubectl get pv` — status should be Available**

```bash
kubectl apply -f pv.yaml
kubectl get pv
```

Access modes to know:

* `ReadWriteOnce (RWO)` — read-write by a single node
* `ReadOnlyMany (ROX)` — read-only by many nodes
* `ReadWriteMany (RWX)` — read-write by many nodes

`hostPath` is fine for learning, not for production.

Verify: What is the STATUS of the PV?

Status of PV is Available

---

### ✅ Task 3 : Create a PersistentVolumeClaim

**1. Write a PVC manifest requesting `500Mi` of storage with `ReadWriteOnce` access**

```yml
kind: PersistentVolumeClaim
apiVersion: v1

metadata:
 name: pv-claim

spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

To bind volumes use `storageClassName: manual` necessarily

**2. Apply it and check both `kubectl get pvc` and `kubectl get pv`**

```bash
kubectl get pvc
kubectl get pv
```

**3. Both should show `Bound` — Kubernetes matched them by capacity and access mode**


Verify: What does the VOLUME column in `kubectl get pvc` show ?

It shows name of Persistent volume from where it's bound.

---

### ✅ Task 4 : Use the PVC in a Pod — Data That Survives

**1. Write a Pod manifest that mounts the PVC at `/data` using `persistentVolumeClaim.claimName`**

```yml
apiVersion: v1
kind: Pod
metadata:
  name: pv-pod
spec:
  containers:
  - name: nginx
    image: nginx:latest
    volumeMounts:
    - name: secure-storage
      mountPath: /data
  volumes:
  - name: secure-storage
    persistentVolumeClaim:
      claimName: pv-claim
```

**2. Write data to `/data/message.txt`, then delete and recreate the Pod**

```yml
apiVersion: v1
kind: Pod
metadata:
  name: pv-pod
spec:
  containers:
  - name: nginx
    image: nginx:latest
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "Hello from inside at: $(date)" >> /data/message.txt
      sleep 3600
    volumeMounts:
    - name: secure-storage
      mountPath: /data
  volumes:
  - name: secure-storage
    persistentVolumeClaim:
      claimName: pv-claim
```

**3. Check the file — it should contain data from both Pods**

Verify: Does the file contain data from both the first and second Pod ?

Yes ! It contains data from both containers.

---

### ✅ Task 5 : StorageClasses and Dynamic Provisioning

**1. Run `kubectl get storageclass` and `kubectl describe storageclass`**

**2. Note the provisioner, reclaim policy, and volume binding mode**

**3. With dynamic provisioning, developers only create PVCs — the StorageClass handles PV creation automatically**

**Verify**: What is the default StorageClass in your cluster ?

Default StorageClass is Standard in my cluster.

---

### ✅ Task 6 : Dynamic Provisioning

**1. Write a PVC manifest that includes `storageClassName: standard` (or your cluster's default)**

```yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  storageClassName: standard
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

**2. Apply it — a PV should appear automatically in `kubectl get pv`**

After applying it  a PV will not appear automatically. Kubernetes does not create the PV immediately after the PVC is created.

Instead, it waits until a Pod actually uses the PVC.

**3. Use this PVC in a Pod, write data, verify it works**

```yml
apiVersion: v1
kind: Pod
metadata:
  name: dynamic-pod
spec:
  containers:
  - name: app
    image: busybox
    command:
      - sh
      - -c
      - |
        echo "Hello Dynamic PV" > /data/message.txt
        sleep 3600

    volumeMounts:
    - name: storage
      mountPath: /data

  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: dynamic-pvc
```

**Verify**: How many PVs exist now? Which was manual, which was dynamic ?

2 PVs exist now :

* pv-volume: Manual
* pvc-1b19d0ef-c72f-442d-9c2b-216f1bdb1b05 : Dynamic

---

### ✅ Task 7 : Clean Up

**1. Delete all pods first**

**2. Delete PVCs — check `kubectl get pv` to see what happened**

**3. The dynamic PV is gone (Delete reclaim policy). The manual PV shows `Released` (Retain policy).**

**4. Delete the remaining PV manually**

**Verify**: Which PV was auto-deleted and which was retained ? Why ?

Dynamic PV is gone , but Manual PV is retained.

**Note**:

* PVs are cluster-wide (not namespaced), PVCs are namespaced
* PV status: `Available` -> `Bound` -> `Released`
* If a PVC stays `Pending`, check for matching capacity and access modes
* `hostPath` data is lost if the Pod moves to a different node
* `storageClassName: ""` disables dynamic provisioning
* Reclaim policies: `Retain` (keep data) vs Delete (remove data)

---
