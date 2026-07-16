# Kubernetes – Day 10 - Helm — Kubernetes Package Manager

Over the past eight days I have written Deployments, Services, ConfigMaps, Secrets, PVCs, and more — all as individual YAML files. For a real application I have dozens of these. Helm is the package manager for Kubernetes, like apt for Ubuntu. Today I will install charts, customize them, and create your own.

---

### ✅ Task 1: Install Helm

1. Install Helm (brew, curl script, or chocolatey depending on your OS)

```bash
sudo snap install helm --classic
```

2. Verify with `helm version` and `helm env`

Three core concepts:

* **Chart** — a package of Kubernetes manifest templates
* **Release** — a specific installation of a chart in your cluster
* **Repository** — a collection of charts (like a package repo)

**Verify**: What version of Helm is installed ?

Version:"v4.2.3" is Insatlled.

---

### ✅ Task 2 : Add a Repository and Search

1. Add the Bitnami repository: `helm repo add bitnami https://charts.bitnami.com/bitnami`

2. Update: `helm repo update`

3. Search: `helm search repo nginx` and `helm search repo bitnami`

**Verify**: How many charts does Bitnami have ?

Bitnami offers over 140 Helm charts.

---

### ✅ Task 3 : Install a Chart

1. Deploy nginx: `helm install my-nginx bitnami/nginx`

2. Check what was created: `kubectl get all`

Pod, Service and Deployments are created automatically for nginx.

3. Inspect the release: `helm list`, `helm status my-nginx`, `helm get manifest my-nginx`

One command replaced writing a Deployment, Service, and ConfigMap by hand.

Verify: How many Pods are running? What Service type was created ?

1 Pod is created. Service type created is LoadBalancer.

---

### ✅ Task 4 : Customize with Values

1. View defaults: `helm show values bitnami/nginx`

It's used to inspect and print all the default configuration options (values) of the bitnami/nginx Helm chart directly to your terminal.

2. Install a custom release with `--set replicaCount=3 --set service.type=NodePort`

```bash
helm install my-nginx bitnami/nginx --set replicaCount=3 --set service.type=NodePort
```

3. Create a `custom-values.yaml` file with replicaCount, service type, and resource limits

```yml
replicaCount: 3

service:
  type: NodePort

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi
```

4. Install another release using `-f custom-values.yaml`

```bash
helm install my-nginx bitnami/nginx -f custom-values.yaml
```

5. Check overrides: `helm get values <release-name>`

```bash
helm get values my-nginx
```

```txt
USER-SUPPLIED VALUES:
replicaCount: 3
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi
service:
  type: NodePort
```

Verify: Does the values file release have the correct replicas and service type ?

Yes ! All values are correct.

---

### ✅ Task 5 : Upgrade and Rollback

1. Upgrade: `helm upgrade my-nginx bitnami/nginx --set replicaCount=5`

Number of pods will become 5.

2. Check history: `helm history my-nginx`

```txt
REVISION        UPDATED                         STATUS          CHART           APP VERSION     DESCRIPTION
1               Thu Jul 16 19:40:06 2026        superseded      nginx-25.0.14   1.31.3          Install complete
2               Thu Jul 16 19:44:00 2026        deployed        nginx-25.0.14   1.31.3          Upgrade complete
```

3. Rollback: `helm rollback my-nginx 1`

```txt
1               Thu Jul 16 19:40:06 2026        superseded      nginx-25.0.14   1.31.3          Install complete
2               Thu Jul 16 19:44:00 2026        superseded      nginx-25.0.14   1.31.3          Upgrade complete
3               Thu Jul 16 19:45:04 2026        deployed        nginx-25.0.14   1.31.3          Rollback to 1
```

4. Check history again — rollback creates a new revision (3), not overwriting revision 2

Same concept as Deployment rollouts from Day 3, but at the full stack level.

**Verify**: How many revisions after the rollback ?

3 revisions after the rollback.

---

### ✅ Task 6 : Create Your Own Chart

1. Scaffold: `helm create my-app`

```txt
Chart.yaml  charts  templates  values.yaml
```

2. Explore the directory: `Chart.yaml`, `values.yaml`, `templates/deployment.yaml`

3. Look at the Go template syntax in templates: `{{ .Values.replicaCount }}`, `{{ .Chart.Name }}`

4. Edit `values.yaml` — set replicaCount to 3 and image to nginx:1.25

5. Validate: `helm lint my-app`

6. Preview: `helm template my-release ./my-app`

7. Install: `helm install my-release ./my-app`

8. Upgrade: `helm upgrade my-release ./my-app --set replicaCount=5`

**Verify**: After installing, 3 replicas ? After upgrading, 5 ?

Yes !

---

### ✅ Task 7 : Clean Up

1. Uninstall all releases: `helm uninstall <name>` for each

```bash
helm uninstall my-nginx
```

2. Remove chart directory and values file

3. Use `--keep-history` if you want to retain release history for auditing

**Verify**: Does `helm list` show zero releases?

**Note**:

* To create a chart for a particular namespace:  `helm install my-nginx bitnami/nginx --namespace nginx --create-namespace`
* `helm show values <chart>` — see what you can customize
* `--set key=value` for single overrides, `-f values.yaml` for files
* Nested values use dots: `--set service.type=NodePort`
* `helm get values <release>` shows overrides, `--all` for everything
* `helm template` renders without installing — great for debugging
* `helm lint` validates chart structure before installing
* Templates: `{{ .Values.key }}`, `{{ .Chart.Name }}`, `{{ .Release.Name }}`


---
