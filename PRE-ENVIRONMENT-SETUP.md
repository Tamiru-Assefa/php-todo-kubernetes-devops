# Kubernetes Environment Setup Guide

This guide prepares the complete DevOps environment and then deploys the **PHP Todo web application** and its **MySQL database** onto the Kubernetes cluster.

The guide is designed for the actual architecture used in this project:

* 1 Kubernetes Control Plane
* 2 Kubernetes Workers
* Jenkins running as a Docker container on the Control Plane
* Docker Hub for container image storage
* Kubernetes for application orchestration
* MySQL with persistent storage
* NodePort for application access

------------------------------------------------------------------------

## 1. Server Environment
Our DevOps environment consists of **3 servers**.

| Server                   | Hostname                | IP Address     | Role                                    | Main Software                                                   |
| ------------------------ | ----------------------- | -------------- | --------------------------------------- | --------------------------------------------------------------- |
| Kubernetes Control Plane | `desktop-control-plane` | `192.168.1.10` | Kubernetes Control Plane + Jenkins Host | kubeadm, kubelet, kubectl, containerd, Docker, Jenkins, Java 21 |
| Kubernetes Worker 1      | `desktop-worker`        | `192.168.1.11` | Kubernetes Worker                       | kubeadm, kubelet, containerd                                    |
| Kubernetes Worker 2      | `desktop-worker2`       | `192.168.1.12` | Kubernetes Worker                       | kubeadm, kubelet, containerd                                    |
---------------------------------------------------

### Architecture

``` text
                         Jenkins
                    Docker Container
                           |
                           |
              +------------+------------+
              |                         |
              v                         v
     desktop-control-plane       Kubernetes Cluster
         192.168.1.10                  |
         Control Plane          +-------+-------+
                                |               |
                                v               v
                         desktop-worker   desktop-worker2
                          192.168.1.11     192.168.1.12
```

Jenkins runs as a Docker container on `desktop-control-plane`. It is not
a separate Kubernetes node.

![alt text](Architecture/Overall%20Arch.png)

------------------------------------------------------------------------

# 2. Prepare All Three Servers

The following steps must be performed on **all three servers**.

## 2.1 Set the Hostname

### Run on: `desktop-control-plane`

``` bash
sudo hostnamectl set-hostname desktop-control-plane
```

### Run on: `desktop-worker`

``` bash
sudo hostnamectl set-hostname desktop-worker
```

### Run on: `desktop-worker2`

``` bash
sudo hostnamectl set-hostname desktop-worker2
```

Verify:

``` bash
hostname
```

------------------------------------------------------------------------

## 2.2 Configure `/etc/hosts`

Add the following entries on **all three servers**.

### Run on: All Servers

``` bash
sudo nano /etc/hosts
```

Add:

``` text
192.168.1.10 desktop-control-plane
192.168.1.11 desktop-worker
192.168.1.12 desktop-worker2
```

Test connectivity:

### Run on: `desktop-control-plane`

``` bash
ping -c 2 desktop-worker
ping -c 2 desktop-worker2
```

### Run on: `desktop-worker`

``` bash
ping -c 2 desktop-control-plane
ping -c 2 desktop-worker2
```

### Run on: `desktop-worker2`

``` bash
ping -c 2 desktop-control-plane
ping -c 2 desktop-worker
```

------------------------------------------------------------------------

# 3. Disable Swap

Kubernetes requires swap to be disabled.

### Run on: All Servers

``` bash
sudo swapoff -a
```

Disable it permanently:

``` bash
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

Verify:

``` bash
free -h
```

The `Swap` value should show `0B`.

------------------------------------------------------------------------

# 4. Configure Kubernetes Kernel Modules

### Run on: All Servers

``` bash
sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF
```

Load the modules:

``` bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Configure networking:

``` bash
sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```

Apply the settings:

``` bash
sudo sysctl --system
```

------------------------------------------------------------------------

# 5. Install Container Runtime

Kubernetes needs a container runtime.

In this environment:

-   `desktop-control-plane` uses Docker Engine and its containerd
    runtime.
-   `desktop-worker` uses containerd.
-   `desktop-worker2` uses containerd.

## 5.1 Install Docker on the Control Plane

Docker is installed on the control plane because Jenkins will run as a
Docker container there.

### Run on: `desktop-control-plane`

``` bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
```

``` bash
sudo install -m 0755 -d /etc/apt/keyrings
```

``` bash
sudo curl -fsSL https://download.docker.com/linux/debian/gpg \
  -o /etc/apt/keyrings/docker.asc
```

``` bash
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

``` bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install Docker:

``` bash
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Start Docker:

``` bash
sudo systemctl enable --now docker
```

Verify:

``` bash
docker --version
```

``` bash
sudo docker run hello-world
```

Docker's `containerd` service will be used as the Kubernetes container
runtime on this server.

------------------------------------------------------------------------

## 5.2 Install Containerd on the Workers

### Run on: `desktop-worker`

``` bash
sudo apt-get update
sudo apt-get install -y containerd
```

### Run on: `desktop-worker2`

``` bash
sudo apt-get update
sudo apt-get install -y containerd
```

Configure containerd on both workers.

### Run on: `desktop-worker`

``` bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
```

``` bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

``` bash
sudo systemctl enable --now containerd
sudo systemctl restart containerd
```

### Run on: `desktop-worker2`

``` bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
```

``` bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

``` bash
sudo systemctl enable --now containerd
sudo systemctl restart containerd
```

Verify on both workers:

``` bash
sudo systemctl status containerd
```

------------------------------------------------------------------------

# 6. Install Kubernetes Components

Install:

-   `kubeadm` --- creates and manages the Kubernetes cluster
-   `kubelet` --- runs Kubernetes workloads on each server
-   `kubectl` --- command-line tool for managing Kubernetes

## 6.1 Add Kubernetes Repository

### Run on: All Servers

``` bash
sudo mkdir -p -m 755 /etc/apt/keyrings
```

``` bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

``` bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Update packages:

``` bash
sudo apt-get update
```

------------------------------------------------------------------------

## 6.2 Install Kubernetes

### Run on: All Servers

``` bash
sudo apt-get install -y kubelet kubeadm kubectl
```

Prevent automatic package changes:

``` bash
sudo apt-mark hold kubelet kubeadm kubectl
```

Verify:

``` bash
kubeadm version
kubectl version --client
```

Enable kubelet:

``` bash
sudo systemctl enable kubelet
```

------------------------------------------------------------------------

# 7. Create the Kubernetes Control Plane

The control plane is initialized only on `desktop-control-plane`.

### Run on: `desktop-control-plane`

``` bash
sudo kubeadm init \
  --apiserver-advertise-address=192.168.1.10 \
  --pod-network-cidr=10.244.0.0/16
```

When the command finishes, Kubernetes will display a `kubeadm join`
command.

Save that command. It will be used on both worker servers.

------------------------------------------------------------------------

# 8. Configure kubectl on the Control Plane

### Run on: `desktop-control-plane`

``` bash
mkdir -p $HOME/.kube
```

``` bash
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```

``` bash
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Test:

``` bash
kubectl get nodes
```

At this point, the control plane should be visible.

------------------------------------------------------------------------

# 9. Install the Kubernetes Network Plugin

The cluster needs a network plugin so that Pods can communicate with
each other.

### Run on: `desktop-control-plane`

``` bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Check the system Pods:

``` bash
kubectl get pods -n kube-flannel
```

Wait until the Flannel Pods are running.

------------------------------------------------------------------------

# 10. Join the Worker Nodes

Use the `kubeadm join` command generated during `kubeadm init`.

It will look similar to:

``` bash
sudo kubeadm join 192.168.1.10:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

Do not copy the example token above. Use the actual command generated by
your control plane.

## Join Worker 1

### Run on: `desktop-worker`

``` bash
sudo kubeadm join 192.168.1.10:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

## Join Worker 2

### Run on: `desktop-worker2`

``` bash
sudo kubeadm join 192.168.1.10:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

------------------------------------------------------------------------

# 11. Verify the Kubernetes Cluster

All cluster management commands are now run from the control plane.

### Run on: `desktop-control-plane`

``` bash
kubectl get nodes
```

Expected result:

``` text
NAME                    STATUS   ROLES           AGE   VERSION
desktop-control-plane   Ready    control-plane   ...   v1.36.x
desktop-worker          Ready    <none>          ...   v1.36.x
desktop-worker2         Ready    <none>          ...   v1.36.x
```

Check all system Pods:

``` bash
kubectl get pods -A
```

The important result is that the Kubernetes nodes are `Ready` and the
system Pods are running.

------------------------------------------------------------------------

# 12. SSH Access

Once the three servers are prepared, SSH to the Kubernetes control
plane:

``` bash
ssh <your-username>@192.168.1.10
```

Verify the hostname:

``` bash
hostname
```

Expected:

``` text
desktop-control-plane
```

From this server, Kubernetes can be managed with:

``` bash
kubectl get nodes
```
![alt text](ScreenShots/Screenshot%202026-09-23%20160841.png)
The Jenkins container runs on this same control-plane server which we will got on the next part. 
