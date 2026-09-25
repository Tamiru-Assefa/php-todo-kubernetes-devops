# Kubernetes Deployment Guide

This document explains how to deploy the **PHP Todo web application** and its **MySQL database** onto the Kubernetes cluster prepared on this **Environment Setup Guide**.



---

# 1. Environment We Have to Prepare

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

# 2.  Complete Project Architecture

The complete environment now looks like this:
![alt text](Architecture/Overall%20Arch.png)


---

# 3.  Understanding the Architecture

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

# 4.  Where Do We Run Kubernetes Commands?

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

# 5.  Connect to the Kubernetes Control Plane

If you are currently on another machine, first connect to the Kubernetes control-plane server.

** Run from your administration/development machine:**

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

# 6.  Verify the Kubernetes Cluster

Before deploying the application, make sure all three Kubernetes nodes are available.

**📍 Run on: `desktop-control-plane`**

```bash
kubectl get nodes
```

Expected:
![alt text](ScreenShots/Screenshot%202026-09-23%20160841.png)



Jenkins does **not** appear as another node because Jenkins is running as a Docker container on the control-plane host.

---


