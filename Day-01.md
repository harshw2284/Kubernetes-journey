# Kubernetes – Day 01 -  Kubernetes Architecture and Cluster Setup

I have been building and shipping containers with Docker. But what happens when I need to run hundreds of containers across multiple servers? You need an orchestrator. Today I started my Kubernetes journey — understand the architecture, set up a local cluster, and run your first kubectl commands.

This is where things get real.

---

### ✅ Task 1: Recall the Kubernetes Story

**Before touching a terminal, write down from memory:**

**1. Why was Kubernetes created? What problem does it solve that Docker alone cannot ?**

Kubernetes was created by Google to automate the deployment, scaling, and management of containerized applications across large fleets of servers. It is essentially a production-grade orchestrator that handles the operational heavy lifting of running thousands of interconnected microservices reliably.

Docker revolutionized the industry by allowing developers to package code and dependencies into a single, portable container. However, when running applications in production, Docker operates entirely on a single node (one machine) and leaves unresolved gaps in system reliability.

--

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



**After drawing, verify your understanding:**

**1. What happens when you run ``kubectl apply -f pod.yaml``? Trace the request through each component.**

When kubectl apply -f pod.yaml is executed, kubectl sends the Pod definition to the API Server, which validates and stores it in etcd. The Scheduler assigns a worker node, the Kubelet starts the container using the container runtime, Kube-Proxy configures networking, and the Pod enters the Running state.

**Flow-Diagram:**

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

### ✅ Task 3 : Understand the Anatomy

**`on`**: Defines the event that triggers the workflow, such as a push, pull request, or schedule.

**`jobs`**: Contains all the jobs that the workflow will execute.

**`runs-on`**: Specifies the operating system or runner (virtual machine) where the job will run (for example, Ubuntu).

**`steps`**: Lists the individual actions or commands inside a job.

**`uses`**: Runs a pre-built action created by GitHub or the community.

**`run`**: Executes a shell command or script directly in the workflow.

**`name`**: Gives a step a readable name so it is easier to identify in the workflow logs.

---

### ✅ Task 4 : Add More Steps

**Update `hello.yml` to also:**

* Print the current date and time
* Print the name of the branch that triggered the run (hint: GitHub provides this as a variable)
* List the files in the repo
* Print the runner's operating system

```yml

print:                                     #Job name
                                           #Runner
        runs-on: ubuntu-latest
        steps:
            - name: Print the current date and time
              run: date
            
            - name: Print the name of the branch that triggered the run
              run: echo "This run was triggered by a push to the ${{ github.ref_name }} branch."

            - name: List the files in the repo
              run: ls

            - name: Print the runner's operating system 
              run: echo "Running on ${{ runner.os }}"

```

---

### ✅ Task 5 : Break It On Purpose

I added a step that runs a command that will fail (misspelled command)

I Pushed it and observed what happened in Actions tab

Then I Fixed it and Push again

A failed pipeline usually shows a red X, failed status, or error message in the CI/CD tool. The pipeline stops at the stage or step where the problem occurred, such as testing, building, or deployment.

---
