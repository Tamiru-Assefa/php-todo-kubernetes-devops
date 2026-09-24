# ☸️ Kubernetes Deployment Guide

This document explains how to deploy the **PHP Todo web application** and its **MySQL database** onto the Kubernetes cluster prepared in the previous **Environment Setup Guide**.

This guide continues directly from that environment and uses the same server names, IP addresses, and architecture throughout the deployment.

---

# 1. 🏗️ Environment We Already Prepared

Our DevOps environment consists of **3 physical/virtual servers**.

| Server                   | Hostname                |     IP Address | Role                                    | Main Software                                          |
| ------------------------ | ----------------------- | -------------: | --------------------------------------- | ------------------------------------------------------ |
| Kubernetes Control Plane | `desktop-control-plane` | `192.168.1.10` | Kubernetes Control Plane + Jenkins Host | kubeadm, kubelet, kubectl, containerd, Docker, Jenkins |
| Kubernetes Worker 1      | `desktop-worker`        | `192.168.1.11` | Kubernetes Worker                       | kubeadm, kubelet, containerd                           |
| Kubernetes Worker 2      | `desktop-worker2`       | `192.168.1.12` | Kubernetes Worker                       | kubeadm, kubelet, containerd                           |

### Important Architecture Decision

Jenkins is **not running on a separate `jenkins-server`**.

Instead, Jenkins runs as a **Docker container on the Kubernetes control-plane server**:

```text
desktop-control-plane
192.168.1.10
    │
    ├── Kubernetes Control Plane
    │
    └── Jenkins Docker Container
```

This means the machine has two responsibilities:

```text
Kubernetes Control Plane
        +
CI/CD Server
```

Jenkins is still **not a Kubernetes worker**.

The Jenkins container does not become a Kubernetes node.

Kubernetes continues to consist of:

```text
desktop-control-plane
desktop-worker
desktop-worker2
```

---

# 2. 🧩 Complete Project Architecture

The complete environment now looks like this:

```text
                         Developer
                             │
                             │ git push
                             ▼
                    ┌──────────────────┐
                    │      GitHub      │
                    │ devops-php-todo  │
                    └────────┬─────────┘
                             │
                             │ Webhook
                             ▼
              ┌────────────────────────────────┐
              │      desktop-control-plane     │
              │          192.168.1.10          │
              │                                │
              │  ┌──────────────────────────┐  │
              │  │     Jenkins Container    │  │
              │  │        :8081             │  │
              │  │                          │  │
              │  │ Docker CLI               │  │
              │  │ kubectl                   │  │
              │  └────────────┬─────────────┘  │
              │               │                │
              │               │ CI/CD          │
              │               ▼                │
              │  ┌──────────────────────────┐  │
              │  │ Kubernetes Control Plane │  │
              │  └──────────────────────────┘  │
              └───────────────┬────────────────┘
                              │
                     Kubernetes Scheduler
                              │
                ┌─────────────┴─────────────┐
                │                           │
                ▼                           ▼
       ┌─────────────────┐        ┌─────────────────┐
       │ desktop-worker  │        │ desktop-worker2 │
       │ 192.168.1.11    │        │ 192.168.1.12   │
       │                 │        │                 │
       │ Kubernetes      │        │ Kubernetes      │
       │ Worker          │        │ Worker          │
       └────────┬────────┘        └────────┬────────┘
                │                          │
                └────────────┬─────────────┘
                             ▼
                     devops-todo namespace
                             │
                  ┌──────────┴──────────┐
                  │                     │
                  ▼                     ▼
             PHP Application       MySQL Database
                3 replicas            1 replica
                  │                     │
                  │                     ▼
                  │              Persistent Storage
                  │
                  ▼
             NodePort :30080
                  │
                  ▼
               Browser
```

---

# 3. 🔑 Understanding the Architecture

There are two different systems running on `desktop-control-plane`.

## Kubernetes

The server is part of the Kubernetes cluster as:

```text
Kubernetes Control Plane
```

It manages:

```text
API Server
Scheduler
Controller Manager
etcd
```

and communicates with:

```text
desktop-worker
desktop-worker2
```

---

## Jenkins

Jenkins runs separately as a Docker container on the same operating-system host:

```text
desktop-control-plane
    │
    └── Docker
         │
         └── Jenkins Container
```

Jenkins is responsible for:

```text
GitHub
   ↓
Checkout
   ↓
Docker Build
   ↓
Docker Hub
   ↓
kubectl
   ↓
Kubernetes
```

Therefore:

```text
Jenkins
   ≠
Kubernetes Node
```

Jenkins simply uses the Kubernetes API to deploy the application.

---

# 4. 📍 Where Do We Run Kubernetes Commands?

This is important because we have three servers.

For Kubernetes administration, we primarily use:

```text
desktop-control-plane
192.168.1.10
```

Therefore, commands such as:

```bash
kubectl get nodes
kubectl apply -f ...
kubectl get pods
kubectl get services
```

are executed from:

```text
desktop-control-plane
```

You do **not** need to manually SSH into:

```text
desktop-worker
desktop-worker2
```

to deploy the application.

Kubernetes automatically decides which worker should run each Pod.

---

# 5. 🔐 Connect to the Kubernetes Control Plane

If you are currently on another machine, first connect to the Kubernetes control-plane server.

**📍 Run from your administration/development machine:**

```bash
ssh <your-username>@192.168.1.10
```

You are now connected to:

```text
desktop-control-plane
```

Confirm the hostname:

```bash
hostname
```

Expected:

```text
desktop-control-plane
```

From this point onward, unless a step specifically says otherwise:

> **Kubernetes commands in this guide are executed on `desktop-control-plane`.**

---

# 6. 🔍 Verify the Kubernetes Cluster

Before deploying the application, make sure all three Kubernetes nodes are available.

**📍 Run on: `desktop-control-plane`**

```bash
kubectl get nodes
```

Expected:

```text
NAME                    STATUS   ROLES           AGE   VERSION
desktop-control-plane   Ready    control-plane   ...   ...
desktop-worker          Ready    <none>          ...   ...
desktop-worker2         Ready    <none>          ...   ...
```

All three nodes should have:

```text
STATUS
Ready
```

The important point is:

```text
3 Kubernetes Nodes
```

not four.

Jenkins does **not** appear as another node because Jenkins is running as a Docker container on the control-plane host.

---

# 7. 🔎 Verify Jenkins on the Control Plane

Because Jenkins is running as a Docker container on `desktop-control-plane`, verify that the container is running.

**📍 Run on: `desktop-control-plane`**

```bash
docker ps
```

You should see a Jenkins container similar to:

```text
CONTAINER ID   IMAGE                 PORTS
xxxxxxxx       devops-jenkins:1.0   0.0.0.0:8081->8080/tcp
```

The exact container ID and image name may differ.

You can also check:

```bash
docker ps --filter "name=jenkins"
```

Jenkins should be running.

---

# 8. 🌐 Verify Jenkins

Open Jenkins from your browser:

```text
http://192.168.1.10:8081
```

The request flow is:

```text
Browser
    │
    ▼
192.168.1.10:8081
    │
    ▼
Docker
    │
    ▼
Jenkins Container
```

Jenkins is therefore hosted by the same machine that acts as the Kubernetes control plane.

---

# 9. 📥 Get the Application Source Code

The PHP Todo application is stored in GitHub.

Repository:

```text
https://github.com/Tamiru-Assefa/devops-php-todo
```

For the initial manual Kubernetes deployment, the repository can be cloned on `desktop-control-plane`.

**📍 Run on: `desktop-control-plane`**

```bash
git clone https://github.com/Tamiru-Assefa/devops-php-todo.git
```

Then:

```bash
cd devops-php-todo
```

Verify the Kubernetes directory:

```bash
ls Kubernetes
```

You should have files similar to:

```text
namespace.yaml
deployment.yaml
service.yaml
mysql-pvc.yaml
mysql-deployment.yaml
mysql-service.yaml
```

Later, Jenkins will clone the same repository automatically during the CI/CD pipeline.

---

# 10. 📦 Create the Kubernetes Namespace

Now we begin creating Kubernetes resources.

**📍 Run on: `desktop-control-plane`**

```bash
kubectl apply -f Kubernetes/namespace.yaml
```

Expected:

```text
namespace/devops-todo created
```

Verify:

```bash
kubectl get namespaces
```

You should see:

```text
devops-todo   Active
```

---

# 11. 🔐 Create the MySQL Secret

The database credentials should not be written directly into the application configuration.

Kubernetes Secrets allow us to store sensitive configuration separately.

Our Secret will be:

```text
mysql-secret
```

**📍 Run on: `desktop-control-plane`**

```bash
kubectl create secret generic mysql-secret \
  --from-literal=MYSQL_ROOT_PASSWORD=rootpassword \
  --from-literal=MYSQL_USER=root \
  --from-literal=MYSQL_PASSWORD=rootpassword \
  -n devops-todo
```

Expected:

```text
secret/mysql-secret created
```

Verify:

```bash
kubectl get secrets -n devops-todo
```

Expected:

```text
NAME           TYPE     DATA   AGE
mysql-secret   Opaque   3      ...
```

> **Note:** `rootpassword` is being used for this local learning environment. A production deployment should use properly managed credentials and stronger secret-management practices.

---

# 12. 🐳 PHP Docker Image

The PHP application runs inside a Docker container.

The Kubernetes Deployment references a Docker image such as:

```yaml
image: ybtamiru/devops-php-todo:latest
```

Before continuing, open:

```text
Kubernetes/deployment.yaml
```

and verify the image name.

The image referenced there must exist in Docker Hub and be accessible by the Kubernetes cluster.

Our Docker Hub repository is:

```text
ybtamiru/devops-php-todo
```

The CI/CD pipeline will eventually automate:

```text
GitHub
    ↓
Jenkins
    ↓
Docker Build
    ↓
Docker Hub
    ↓
Kubernetes Deployment
```

---

# 13. 🚀 Deploy the PHP Application

Now we create the PHP application Deployment.

**📍 Run on: `desktop-control-plane`**

```bash
kubectl apply -f Kubernetes/deployment.yaml
```

Expected:

```text
deployment.apps/todo-app created
```

The Deployment requests:

```text
3 replicas
```

Kubernetes will therefore attempt to maintain three PHP Pods.

The Pods can be scheduled across eligible Kubernetes nodes according to the scheduler and the Deployment configuration.

You do **not** manually create a PHP container on each worker.

Kubernetes does that automatically.

---

# 14. 🔎 Verify the PHP Pods

**📍 Run on: `desktop-control-plane`**

```bash
kubectl get pods -n devops-todo -o wide
```

The `-o wide` option also shows the node running each Pod.

For example:

```text
NAME                       READY   STATUS    RESTARTS   AGE   IP          NODE
todo-app-xxxxx-aaa         1/1     Running   0          ...   10.244.x.x  desktop-worker
todo-app-xxxxx-bbb         1/1     Running   0          ...   10.244.x.x  desktop-worker2
todo-app-xxxxx-ccc         1/1     Running   0          ...   10.244.x.x  desktop-worker
```

The exact Pod names and IP addresses will be different.

You requested:

```text
3 replicas
```

Kubernetes attempts to maintain three Pods according to its scheduling rules.

---

# 15. 🌐 Create the PHP Service

The PHP Pods have dynamic IP addresses, so users should not connect directly to individual Pods.

Instead, we create:

```text
todo-service
```

The Service provides a stable endpoint for the application.

**📍 Run on: `desktop-control-plane`**

```bash
kubectl apply -f Kubernetes/service.yaml
```

Expected:

```text
service/todo-service created
```

---

# 16. 🔍 Verify the PHP Service

**📍 Run on: `desktop-control-plane`**

```bash
kubectl get service -n devops-todo
```

You should see something similar to:

```text
NAME           TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)
todo-service   NodePort   10.x.x.x       <none>        80:30080/TCP
```

This means:

```text
Container Port = 80
Service Port   = 80
NodePort       = 30080
```

The request flow is:

```text
Browser
   │
   ▼
Node IP :30080
   │
   ▼
todo-service :80
   │
   ├── PHP Pod 1
   ├── PHP Pod 2
   └── PHP Pod 3
```

---

# 17. 🎯 Check the PHP Service Endpoints

**📍 Run on: `desktop-control-plane`**

```bash
kubectl get endpoints -n devops-todo
```

You should see the IP addresses of the PHP Pods.

You can also use EndpointSlices:

```bash
kubectl get endpointslices -n devops-todo
```

This confirms that the Service has discovered the application Pods.

---

# 18. 💾 Create MySQL Persistent Storage

Now we deploy the database.

MySQL needs persistent storage because database information must survive Pod recreation.

We therefore create a:

```text
PersistentVolumeClaim
```

called:

```text
mysql-pvc
```

The PVC requests:

```text
1 GiB
```

of persistent storage.

**📍 Run on: `desktop-control-plane`**

```bash
kubectl apply -f Kubernetes/mysql-pvc.yaml
```

Expected:

```text
persistentvolumeclaim/mysql-pvc created
```

---

# 19. 🔍 Verify the MySQL PVC

**📍 Run on: `desktop-control-plane`**

```bash
kubectl get pvc -n devops-todo
```

Expected:

```text
NAME        STATUS   VOLUME   CAPACITY   ACCESS MODES
mysql-pvc   Bound    ...      1Gi        ...
```

The important status is:

```text
Bound
```

If it says:

```text
Pending
```

check:

```bash
kubectl get storageclass
```

and:

```bash
kubectl get pv
```

Do not continue until the storage issue is understood.

---

# 20. 🗄️ Deploy MySQL

Now deploy the MySQL database.

**📍 Run on: `desktop-control-plane`**

```bash
kubectl apply -f Kubernetes/mysql-deployment.yaml
```

Expected:

```text
deployment.apps/mysql created
```

Kubernetes will schedule the MySQL Pod on an eligible Kubernetes node.

You do not manually install MySQL on:

```text
desktop-worker
desktop-worker2
```

The MySQL container contains the MySQL software.

---

# 21. 🔌 Create the MySQL Service

The PHP application needs to communicate with MySQL.

We therefore create an internal Kubernetes Service named:

```text
db
```

**📍 Run on: `desktop-control-plane`**

```bash
kubectl apply -f Kubernetes/mysql-service.yaml
```

Expected:

```text
service/db created
```

The MySQL Service should use:

```text
ClusterIP
```

rather than:

```text
NodePort
```

This means MySQL is available internally inside the Kubernetes cluster but is not intentionally exposed through a node port.

The PHP application can connect using:

```text
db:3306
```

---

# 22. 🔎 Verify MySQL

**📍 Run on: `desktop-control-plane`**

```bash
kubectl get pods -n devops-todo -o wide
```

You should now see approximately:

```text
3 PHP Pods
1 MySQL Pod
```

For example:

```text
NAME                       READY   STATUS    NODE
mysql-xxxxx                1/1     Running   desktop-worker2
todo-app-xxxxx-aaa         1/1     Running   desktop-worker
todo-app-xxxxx-bbb         1/1     Running   desktop-worker2
todo-app-xxxxx-ccc         1/1     Running   desktop-worker
```

The exact Pod names and node placement may differ.

---

# 23. 🌐 Access the Application

Because the PHP Service uses:

```text
NodePort: 30080
```

the application can be accessed through a Kubernetes node.

Our nodes are:

```text
desktop-control-plane   192.168.1.10
desktop-worker          192.168.1.11
desktop-worker2         192.168.1.12
```

Try:

```text
http://192.168.1.10:30080
```

or:

```text
http://192.168.1.11:30080
```

or:

```text
http://192.168.1.12:30080
```

The request is handled by the Kubernetes Service and forwarded to one of the PHP Pods.

---

# 24. 🧪 Test Database Persistence

Create a Todo item through the web application.

Then identify the MySQL Pod.

**📍 Run on: `desktop-control-plane`**

```bash
kubectl get pods -n devops-todo
```

Delete the MySQL Pod:

```bash
kubectl delete pod <mysql-pod-name> -n devops-todo
```

For example:

```bash
kubectl delete pod mysql-xxxxx -n devops-todo
```

The MySQL Deployment should automatically create a replacement Pod.

Check:

```bash
kubectl get pods -n devops-todo
```

Once the new MySQL Pod is:

```text
Running
```

open the Todo application again.

The previously created data should remain because MySQL is using:

```text
mysql-pvc
```

and its associated persistent storage.

---

# 25. 🔍 Final Kubernetes Verification

Run the following commands from:

```text
desktop-control-plane
```

## Nodes

```bash
kubectl get nodes
```

Expected:

```text
desktop-control-plane   Ready
desktop-worker          Ready
desktop-worker2         Ready
```

---

## Pods

```bash
kubectl get pods -n devops-todo -o wide
```

Expected approximately:

```text
3 PHP Pods
1 MySQL Pod
```

---

## Deployments

```bash
kubectl get deployments -n devops-todo
```

Expected approximately:

```text
NAME       READY
todo-app   3/3
mysql      1/1
```

---

## Services

```bash
kubectl get services -n devops-todo
```

Expected:

```text
todo-service   NodePort
db             ClusterIP
```

---

## Persistent Storage

```bash
kubectl get pvc -n devops-todo
```

Expected:

```text
mysql-pvc   Bound
```

---

## Secrets

```bash
kubectl get secrets -n devops-todo
```

Expected:

```text
mysql-secret
```

---

# 26. 🛠️ Troubleshooting

If a Pod is not running:

**📍 Run on: `desktop-control-plane`**

```bash
kubectl get pods -n devops-todo
```

Then:

```bash
kubectl describe pod <pod-name> -n devops-todo
```

Check the events at the bottom.

For application logs:

```bash
kubectl logs <pod-name> -n devops-todo
```

For MySQL logs:

```bash
kubectl logs deployment/mysql -n devops-todo
```

For PHP application logs:

```bash
kubectl logs deployment/todo-app -n devops-todo
```

---

# 27. 🐳 Troubleshooting Jenkins

Because Jenkins is running as a Docker container on the control-plane server, Jenkins troubleshooting is performed on:

```text
desktop-control-plane
```

Check the container:

```bash
docker ps --filter "name=jenkins"
```

Check Jenkins logs:

```bash
docker logs jenkins
```

If the container is not running:

```bash
docker start jenkins
```

Check the Jenkins port:

```bash
docker ps
```

You should see a port mapping similar to:

```text
0.0.0.0:8081->8080/tcp
```

Jenkins should then be available at:

```text
http://192.168.1.10:8081
```

---

# 28. 🔗 How Jenkins Connects to Kubernetes

Jenkins is located on:

```text
desktop-control-plane
192.168.1.10
```

The Jenkins container needs access to the Kubernetes API so that the pipeline can run commands such as:

```bash
kubectl set image ...
kubectl rollout status ...
```

The important concept is:

```text
Jenkins Container
       │
       │ kubectl
       ▼
Kubernetes API Server
       │
       ▼
Kubernetes Scheduler
       │
       ├── desktop-worker
       └── desktop-worker2
```

Jenkins does **not** manually deploy containers to the workers.

Instead, Jenkins tells Kubernetes:

```text
"Deploy this application image."
```

Kubernetes then handles scheduling and maintaining the desired state.

---

# 29. 🧠 Important: What Each Server Does

At this stage, it is important to understand the responsibility of each machine.

## `desktop-control-plane`

```text
192.168.1.10
```

This machine has **two responsibilities**.

### Kubernetes

It acts as the:

```text
Kubernetes Control Plane
```

It manages the Kubernetes cluster.

We use it to run commands such as:

```bash
kubectl apply
kubectl get
kubectl describe
kubectl logs
```

### Jenkins

The same host also runs:

```text
Jenkins Docker Container
```

Jenkins is responsible for CI/CD:

```text
GitHub
   ↓
Checkout
   ↓
Docker Build
   ↓
Docker Hub
   ↓
kubectl
   ↓
Kubernetes
```

Therefore:

```text
desktop-control-plane
│
├── Kubernetes Control Plane
│
└── Jenkins Container
```

---

## `desktop-worker`

```text
192.168.1.11
```

This is a:

```text
Kubernetes Worker
```

Kubernetes may schedule application Pods here.

We normally do **not** manually deploy applications on this server.

---

## `desktop-worker2`

```text
192.168.1.12
```

This is also a:

```text
Kubernetes Worker
```

Kubernetes may schedule application Pods here.

Again, we normally do **not** manually deploy applications on this server.

---

# 30. ⚠️ Important Jenkins Architecture Note

Running Jenkins on the Kubernetes control-plane host is convenient for this learning project, but it means the control-plane machine has both:

```text
Kubernetes Control Plane
+
CI/CD Infrastructure
```

The Jenkins container may also require access to:

```text
Docker
Kubernetes API
```

Those permissions should therefore be treated carefully.

For a production environment, Jenkins is commonly separated onto its own server or managed infrastructure.

For this project, however, the architecture is intentionally:

```text
                    desktop-control-plane
                         192.168.1.10
                              │
                 ┌────────────┴────────────┐
                 │                         │
                 ▼                         ▼
          Kubernetes CP             Jenkins Container
                 │                         │
                 │                         │
                 └────────────┬────────────┘
                              │
                              ▼
                       Kubernetes API
                              │
                  ┌───────────┴───────────┐
                  ▼                       ▼
          desktop-worker          desktop-worker2
```

This keeps the infrastructure simple while still demonstrating a complete CI/CD workflow.

---

# 31. 🚀 What Happens Next?

At this point, the PHP Todo application has been deployed manually.

The current process is:

```text
Developer
    │
    ▼
GitHub
    │
    │
    ▼
desktop-control-plane
    │
    │ kubectl
    ▼
Kubernetes
    │
    ▼
PHP Todo Application
```

However, Jenkins is already running on the same control-plane server.

Therefore, the next stage is to automate this process.

Our CI/CD architecture will become:

```text
Developer
     │
     │ git push
     ▼
GitHub
     │
     │ webhook
     ▼
Jenkins Container
desktop-control-plane
192.168.1.10
     │
     ├── Checkout
     │
     ├── Build
     │
     ├── Docker Build
     │
     ├── Docker Push
     │
     └── kubectl
             │
             ▼
        Kubernetes API
             │
             ▼
     Kubernetes Cluster
             │
       ┌─────┴─────┐
       ▼           ▼
desktop-worker  desktop-worker2
       │           │
       └─────┬─────┘
             ▼
      PHP Todo Application
```

---

# 32. 🎯 Final Project Architecture

After completing the CI/CD and monitoring stages, the complete project architecture will be:

```text
                         Developer
                             │
                             │ git push
                             ▼
                    ┌──────────────────┐
                    │      GitHub      │
                    │ devops-php-todo  │
                    └────────┬─────────┘
                             │
                             │ GitHub Webhook
                             ▼
        ┌─────────────────────────────────────────┐
        │          desktop-control-plane          │
        │              192.168.1.10               │
        │                                         │
        │  ┌───────────────────────────────────┐  │
        │  │       Jenkins Docker Container    │  │
        │  │              :8081                │  │
        │  │                                   │  │
        │  │  Docker CLI                       │  │
        │  │  kubectl                          │  │
        │  └───────────────┬───────────────────┘  │
        │                  │                      │
        │                  │ CI/CD                │
        │                  ▼                      │
        │       Kubernetes Control Plane         │
        └──────────────────┬──────────────────────┘
                           │
                           │ Kubernetes API
                           ▼
                ┌───────────────────────┐
                │  Kubernetes Scheduler │
                └───────────┬───────────┘
                            │
               ┌────────────┴────────────┐
               │                         │
               ▼                         ▼
      ┌─────────────────┐       ┌─────────────────┐
      │ desktop-worker  │       │ desktop-worker2 │
      │ 192.168.1.11    │       │ 192.168.1.12   │
      └────────┬────────┘       └────────┬────────┘
               │                         │
               └────────────┬────────────┘
                            ▼
                    devops-todo namespace
                            │
                  ┌─────────┴─────────┐
                  │                   │
                  ▼                   ▼
             todo-app              MySQL
            3 replicas             1 replica
                  │                   │
                  │                   ▼
                  │              mysql-pvc
                  │
                  ▼
             todo-service
              NodePort
                :30080
                  │
                  ▼
               Browser
```

---

# 33. 🎉 Final Result

The Kubernetes portion of the project now demonstrates:

* Kubernetes Control Plane
* Two Kubernetes Workers
* Docker containerized PHP application
* MySQL container
* Kubernetes Deployments
* Kubernetes Services
* NodePort
* ClusterIP
* Kubernetes Secrets
* PersistentVolumeClaim
* Multi-replica application deployment
* Kubernetes scheduling
* Database persistence
* Jenkins running in Docker
* Jenkins-to-Kubernetes integration

The infrastructure is now:

```text
                    3 SERVERS
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
   Control Plane    Worker 1       Worker 2
   + Jenkins
        │
        │
        ▼
   Kubernetes
        │
        ▼
   PHP Todo + MySQL
```

The next stage connects this Kubernetes deployment to the Jenkins CI/CD pipeline:

```text
git push
    ↓
GitHub
    ↓
Webhook
    ↓
Jenkins Container
    ↓
Docker Build
    ↓
Docker Hub
    ↓
kubectl
    ↓
Kubernetes
    ↓
Rolling Deployment
    ↓
Updated PHP Todo Application
```

This completes the Kubernetes deployment foundation for the project.
