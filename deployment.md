Kubernetes Deployment Guide
This document explains how to deploy the PHP Todo web application and its MySQL database onto the Kubernetes cluster we prepared in the previous document.

It is written for someone who is new to DevOps. It explains what each command does, how the application is structured, and how to verify that everything is working correctly.

1. Introduction
At this point, you should have a working Kubernetes cluster with 3 nodes (desktop-control-plane, desktop-worker, desktop-worker2).

We will now deploy our application. Our application consists of two main parts:

PHP Todo App (Frontend/Backend): The web application that users interact with.

MySQL Database: Where the Todo tasks are permanently stored.

To make this robust, we will deploy 3 replicas of the PHP app (so if one crashes, the app stays online) and 1 replica of MySQL (backed by persistent storage).

Where to run these commands?
You will run all of these commands from your local terminal (PowerShell on Windows) where you have kubectl configured to talk to your cluster. You do not need to SSH into the master node.

2. Clone the Project
First, we need to get the application code onto our machine. We do this by cloning the GitHub repository.

Open your terminal and run:

bash
git clone https://github.com/Tamiru-Assefa/devops-php-todo.git
cd devops-php-todo
3. Create the Namespace
A namespace in Kubernetes is like a virtual cluster inside your physical cluster. It allows us to isolate our project from other projects so we don't accidentally delete someone else's work.

We have a namespace.yaml file in our Kubernetes directory. Let's apply it:

bash
kubectl apply -f Kubernetes/namespace.yaml
Expected Output: namespace/devops-todo created

Verify it was created:

bash
kubectl get namespaces
You should see devops-todo in the list with a status of Active.

4. Log in to Docker Hub
Our PHP application needs to be packaged as a Docker image. Even though we built it locally, Jenkins will eventually push it to Docker Hub. For now, let's log in to your Docker Hub account from the terminal so Kubernetes has permission to pull the image.

bash
docker login
Enter your Docker Hub username and password when prompted.

5. Create the Database Secret
We never want to hardcode database passwords in our application code or YAML files. Instead, we store them in a Kubernetes Secret.

Run this command to create the secret for our MySQL database (this matches the exact command in your screenshots):

bash
kubectl create secret generic mysql-secret `
  --from-literal=MYSQL_ROOT_PASSWORD=rootpassword `
  --from-literal=MYSQL_USER=root `
  --from-literal=MYSQL_PASSWORD=rootpassword `
  -n devops-todo
(Note: If you are on Linux/Mac, replace the backticks with backslashes` for line continuation).

Verify the secret exists:

bash
kubectl get secret -n devops-todo
You should see mysql-secret with a type of Opaque.

6. Deploy the PHP Application
Now we deploy the PHP app. Look at the deployment.yaml file. It contains instructions for Kubernetes. Let's apply it:

bash
kubectl apply -f Kubernetes/deployment.yaml
Expected Output: deployment.apps/todo-app created

⚙️ Understanding the Deployment Settings
In this file, we defined several important settings:

Replicas: 3: Kubernetes will create 3 identical copies (Pods) of your PHP container. If one fails, the other two keep the site running.

Resources:

requests: The minimum resources guaranteed to the container. (100m CPU = 10% of a core, 128Mi RAM).

limits: The maximum resources the container can use before it gets throttled or killed. (500m CPU = 50% of a core, 256Mi RAM).

ReadinessProbe: Kubernetes checks if the app is ready to receive traffic before sending users to it. It waits 5 seconds, then checks every 10 seconds.

LivenessProbe: Kubernetes checks if the app is still alive. If it freezes, Kubernetes will restart it. It waits 15 seconds, then checks every 20 seconds.

Verify the Pods are running:

bash
kubectl get pods -n devops-todo
You should see 3 todo-app-xxxxx pods with 1/1 in the READY column and Running in the STATUS column.

7. Expose the PHP Application (Service)
Right now, your PHP pods are running, but they are only accessible inside the cluster. To access them from your web browser, we need a Service.

We use a NodePort service, which opens a specific port on every node in the cluster (e.g., 30080) and forwards traffic to port 80 inside our containers.

Apply the service:

bash
kubectl apply -f Kubernetes/service.yaml
Expected Output: service/todo-service created

Verify the Service:

bash
kubectl get service -n devops-todo
You should see todo-service. Notice the PORT(S) column says 80:30080/TCP. This means port 80 inside the cluster is mapped to port 30080 on your local machine.

Check Endpoints:
A service routes traffic to specific Pod IPs. To verify it found our 3 PHP pods:

bash
kubectl get endpoints -n devops-todo
Note: You might see a warning that Endpoints are deprecated in v1.33+. This is normal. Kubernetes is transitioning to EndpointSlices, but the functionality is exactly the same.
You should see 3 IP addresses listed under ENDPOINTS. These are your 3 PHP pods!

8. Deploy MySQL Database and pvc
Now we need our database. We will deploy MySQL and a Service for it.
Note: We do not expose MySQL with a NodePort. It should only be accessible internally by our PHP app for security reasons.

First, we apply the PVC file. This asks Kubernetes to allocate 1 Gigabyte of storage for our database.

bash
kubectl apply -f Kubernetes/mysql-pvc.yaml
Expected Output: persistentvolumeclaim/mysql-pvc created

Verify the PVC is bound:

bash
kubectl get pvc -n devops-todo
You should see mysql-pvc with a STATUS of Bound. This means Kubernetes successfully found and attached the storage.

Apply the MySQL deployment and service:

bash
kubectl apply -f Kubernetes/mysql-deployment.yaml
kubectl apply -f Kubernetes/mysql-service.yaml
Expected Output:
deployment.apps/mysql created
service/db created

Verify MySQL is running:

bash
kubectl get pods -n devops-todo
You should now see a mysql-xxxxx pod running alongside your 3 PHP pods.











# 📊 Monitoring Setup: Prometheus, Grafana, cAdvisor & Kubernetes Metrics

This document explains how to build a complete monitoring stack for the Kubernetes cluster hosting the PHP Todo application.

The monitoring architecture uses:

* **Prometheus** — collects and stores time-series metrics
* **Grafana** — visualizes metrics through dashboards
* **cAdvisor** — collects container CPU, memory, filesystem, and network metrics
* **kube-state-metrics** — exposes Kubernetes object/state metrics such as Deployment replicas, Pod status, and desired/current replica counts

The final monitoring architecture is:

```text
                         Kubernetes Cluster
                                │
             ┌──────────────────┼──────────────────┐
             │                  │                  │
             ▼                  ▼                  ▼
     desktop-control-plane  desktop-worker   desktop-worker2
        192.168.1.10         192.168.1.11      192.168.1.12
             │                  │                  │
             │                  │                  │
             │             cAdvisor            cAdvisor
             │                  │                  │
             └──────────────────┼──────────────────┘
                                │
                                ▼
                         Prometheus
                                │
                                │ PromQL
                                ▼
                            Grafana
                                │
                                ▼
                         Web Dashboard
```

---

# 1. Monitoring Architecture

The monitoring stack contains several different responsibilities.

## Prometheus

Prometheus periodically **scrapes** metrics from monitoring endpoints and stores them as time-series data.

For example:

```text
CPU usage
Memory usage
Pod count
Container statistics
Deployment replicas
Node metrics
```

Prometheus does not normally receive metrics automatically.

Instead, it asks monitoring endpoints:

```text
Prometheus
    │
    ├── scrape ──► cAdvisor
    │
    ├── scrape ──► kube-state-metrics
    │
    └── scrape ──► Kubernetes / node endpoints
```

---

# 2. cAdvisor vs kube-state-metrics

These two components solve different problems.

## cAdvisor

cAdvisor provides **container resource metrics**, such as:

```text
CPU usage
Memory usage
Network traffic
Filesystem usage
Container statistics
```

For example:

```text
container_cpu_usage_seconds_total
container_memory_working_set_bytes
```

These are useful for questions such as:

> How much CPU is my PHP container using?

---

## kube-state-metrics

kube-state-metrics exposes information about the **state of Kubernetes objects**.

For example:

```text
Deployment desired replicas
Deployment available replicas
Pod status
DaemonSet status
Node status
```

This provides metrics such as:

```text
kube_deployment_status_replicas_available
```

Therefore, our monitoring stack uses:

```text
cAdvisor
     ↓
Container-level metrics

kube-state-metrics
     ↓
Kubernetes object/state metrics
```

---

# 3. Where Commands Are Run

For this project, Kubernetes administration is performed from:

```text
Server:
desktop-control-plane

IP:
192.168.1.10
```

SSH into the control plane:

```bash
ssh <your-user>@192.168.1.10
```

Verify:

```bash
hostname
```

Expected:

```text
desktop-control-plane
```

All `kubectl` commands in this document should be executed on:

```text
desktop-control-plane
```

The worker nodes:

```text
desktop-worker
192.168.1.11

desktop-worker2
192.168.1.12
```

are managed by Kubernetes.

You normally **do not manually deploy monitoring components on the workers**.

Kubernetes schedules the required Pods there.

---

# 4. Verify the Kubernetes Cluster

## Run on: `desktop-control-plane`

```bash
kubectl get nodes -o wide
```

Expected:

```text
NAME                    STATUS   ROLES           INTERNAL-IP
desktop-control-plane   Ready    control-plane   192.168.1.10
desktop-worker          Ready    <none>          192.168.1.11
desktop-worker2         Ready    <none>          192.168.1.12
```

All nodes should show:

```text
Ready
```

---

# 5. Create the Monitoring Namespace

We isolate monitoring components from the application.

The PHP Todo application runs in:

```text
devops-todo
```

Monitoring runs in:

```text
monitoring
```

This separation makes the cluster easier to organize.

## Run on: `desktop-control-plane`

Create the namespace:

```bash
kubectl create namespace monitoring
```

If it already exists, Kubernetes may return:

```text
Error from server (AlreadyExists)
```

That is not a problem.

Verify:

```bash
kubectl get namespaces
```

You should see:

```text
default
devops-todo
monitoring
kube-system
...
```

---

# 6. Monitoring Kubernetes Files

The Kubernetes directory should contain monitoring manifests similar to:

```text
Kubernetes/
│
├── prometheus-rbac.yaml
├── prometheus-config.yaml
├── prometheus-deployment.yaml
├── prometheus-service.yaml
│
├── grafana-deployment.yaml
├── grafana-service.yaml
│
├── cadvisor.yaml
│
├── kube-state-metrics.yaml
│
└── ...
```

If persistent storage is configured separately, you may also have:

```text
├── prometheus-pvc.yaml
└── grafana-pvc.yaml
```

---

# 7. Deploy Prometheus RBAC

Prometheus needs permission to discover Kubernetes resources.

RBAC stands for:

```text
Role-Based Access Control
```

Prometheus uses Kubernetes API access for service discovery and monitoring.

The RBAC manifest normally creates:

```text
ServiceAccount
ClusterRole
ClusterRoleBinding
```

## Run on: `desktop-control-plane`

```bash
kubectl apply -f Kubernetes/prometheus-rbac.yaml
```

Verify:

```bash
kubectl get serviceaccount -n monitoring
```

Check ClusterRoles:

```bash
kubectl get clusterrole | grep prometheus
```

Check bindings:

```bash
kubectl get clusterrolebinding | grep prometheus
```

---

# 8. Configure Prometheus

The Prometheus configuration is normally stored inside a ConfigMap.

The configuration file is:

```text
prometheus.yml
```

It defines:

* Scrape interval
* Scrape targets
* Kubernetes service discovery
* cAdvisor targets
* kube-state-metrics targets
* Other monitoring endpoints

## Run on: `desktop-control-plane`

```bash
kubectl apply -f Kubernetes/prometheus-config.yaml
```

Verify:

```bash
kubectl get configmap -n monitoring
```

You should see:

```text
prometheus-config
```

You can inspect it with:

```bash
kubectl get configmap prometheus-config \
  -n monitoring \
  -o yaml
```

---

# 9. Deploy Prometheus

The Prometheus Deployment creates the Prometheus Pod.

## Run on: `desktop-control-plane`

```bash
kubectl apply -f Kubernetes/prometheus-deployment.yaml
```

Verify:

```bash
kubectl get deployments -n monitoring
```

Expected:

```text
NAME         READY   UP-TO-DATE   AVAILABLE
prometheus   1/1     1            1
```

Check the Pod:

```bash
kubectl get pods -n monitoring -o wide
```

You should see something similar to:

```text
prometheus-xxxxx   1/1   Running   ...   desktop-worker
```

The exact node can vary because Kubernetes chooses where to schedule the Pod.

---

# 10. Verify Prometheus Logs

If the Pod is running but Prometheus does not appear to work, check its logs.

## Run on: `desktop-control-plane`

First get the Pod name:

```bash
kubectl get pods -n monitoring
```

Then:

```bash
kubectl logs -n monitoring <prometheus-pod-name>
```

For example:

```bash
kubectl logs -n monitoring prometheus-xxxxx
```

Look for errors involving:

```text
configuration
RBAC
permission denied
scrape
connection refused
```

---

# 11. Expose Prometheus

Create the Prometheus Service:

## Run on: `desktop-control-plane`

```bash
kubectl apply -f Kubernetes/prometheus-service.yaml
```

Verify:

```bash
kubectl get service -n monitoring
```

Expected:

```text
NAME         TYPE       CLUSTER-IP    PORT(S)
prometheus   NodePort   10.x.x.x      9090:30090/TCP
```

The mapping:

```text
9090:30090/TCP
```

means:

```text
Container/Service Port: 9090
NodePort:              30090
```

---

# 12. Access Prometheus

From your computer's browser:

```text
http://192.168.1.10:30090
```

You can also try:

```text
http://192.168.1.11:30090
```

or:

```text
http://192.168.1.12:30090
```

depending on your network configuration.

The Prometheus interface should open.

---

# 13. Verify Prometheus Targets

Inside the Prometheus web interface, go to:

```text
Status
    ↓
Targets
```

You should see your configured monitoring targets.

The important thing is that targets should eventually show:

```text
UP
```

If a target shows:

```text
DOWN
```

do not continue directly to Grafana.

First investigate the failed target.

---

# 14. Deploy kube-state-metrics

This component is important for Kubernetes-level metrics.

Without kube-state-metrics, queries such as:

```promql
kube_deployment_status_replicas_available
```

may return no data.

## Run on: `desktop-control-plane`

```bash
kubectl apply -f Kubernetes/kube-state-metrics.yaml
```

Verify:

```bash
kubectl get pods -n monitoring
```

You should see a Pod similar to:

```text
kube-state-metrics-xxxxx   1/1   Running
```

Verify its Service:

```bash
kubectl get service -n monitoring
```

You should see:

```text
kube-state-metrics
```

---

# 15. Verify kube-state-metrics

Check the Service:

```bash
kubectl get svc kube-state-metrics -n monitoring
```

Typically it exposes port:

```text
8080
```

Prometheus should be configured to scrape:

```text
kube-state-metrics:8080
```

Because both services are in the `monitoring` namespace, the short DNS name works:

```text
kube-state-metrics:8080
```

---

# 16. Deploy cAdvisor

cAdvisor collects container-level resource metrics.

It should normally run as a:

```text
DaemonSet
```

A DaemonSet tells Kubernetes:

> Run one Pod on every eligible node.

This is important because container metrics are generated locally on each node.

---

# 17. Important cAdvisor Node Consideration

Your cluster contains:

```text
desktop-control-plane
desktop-worker
desktop-worker2
```

However, kubeadm control-plane nodes normally have a taint that prevents ordinary workloads from being scheduled there.

Therefore, depending on the `cadvisor.yaml` configuration, cAdvisor may run on:

```text
desktop-worker
desktop-worker2
```

but not:

```text
desktop-control-plane
```

If you want cAdvisor to monitor **all three nodes**, including the control plane, the DaemonSet needs an appropriate toleration for the control-plane taint.

Do not simply assume that:

```text
DaemonSet = three Pods
```

Always verify the actual result.

---

# 18. Deploy cAdvisor

## Run on: `desktop-control-plane`

```bash
kubectl apply -f Kubernetes/cadvisor.yaml
```

Check the DaemonSet:

```bash
kubectl get daemonset -n monitoring
```

Expected example:

```text
NAME       DESIRED   CURRENT   READY
cadvisor   2         2         2
```

or, if the control plane is also tolerated:

```text
NAME       DESIRED   CURRENT   READY
cadvisor   3         3         3
```

---

# 19. Check Which Nodes Run cAdvisor

This is important.

## Run on: `desktop-control-plane`

```bash
kubectl get pods -n monitoring -o wide
```

Example:

```text
NAME              READY   STATUS    NODE
cadvisor-xxxxx    1/1     Running   desktop-worker
cadvisor-yyyyy    1/1     Running   desktop-worker2
```

If configured to run on the control plane:

```text
NAME              READY   STATUS    NODE
cadvisor-xxxxx    1/1     Running   desktop-control-plane
cadvisor-yyyyy    1/1     Running   desktop-worker
cadvisor-zzzzz    1/1     Running   desktop-worker2
```

---

# 20. Why cAdvisor Runs on Multiple Nodes

Suppose:

```text
desktop-worker
```

is running:

```text
PHP Pod A
PHP Pod B
```

and:

```text
desktop-worker2
```

is running:

```text
PHP Pod C
```

Each node has its own containers.

Therefore:

```text
desktop-worker
      ↓
cAdvisor
      ↓
container metrics

desktop-worker2
      ↓
cAdvisor
      ↓
container metrics
```

Prometheus collects the metrics from the cAdvisor instances.

---

# 21. Verify cAdvisor Metrics

Check the cAdvisor Service:

```bash
kubectl get service -n monitoring
```

You should see:

```text
cadvisor
```

You can also inspect the endpoints:

```bash
kubectl get endpoints -n monitoring cadvisor
```

The endpoints should correspond to the available cAdvisor Pods.

---

# 22. Deploy Grafana

Grafana is the visualization layer.

It does not normally collect metrics itself.

Instead:

```text
cAdvisor
      ↓
Prometheus
      ↓
Grafana
```

Grafana sends PromQL queries to Prometheus.

## Run on: `desktop-control-plane`

```bash
kubectl apply -f Kubernetes/grafana-deployment.yaml
```

Verify:

```bash
kubectl get deployment -n monitoring
```

Expected:

```text
grafana   1/1   1   1
```

Check the Pod:

```bash
kubectl get pods -n monitoring -o wide
```

---

# 23. Check Grafana Logs

If Grafana does not start:

```bash
kubectl logs -n monitoring deployment/grafana
```

You can also inspect the Deployment:

```bash
kubectl describe deployment grafana -n monitoring
```

---

# 24. Expose Grafana

Apply the Service:

## Run on: `desktop-control-plane`

```bash
kubectl apply -f Kubernetes/grafana-service.yaml
```

Verify:

```bash
kubectl get service -n monitoring
```

Expected:

```text
NAME      TYPE       CLUSTER-IP    PORT(S)
grafana   NodePort   10.x.x.x      3000:30300/TCP
```

This means:

```text
Grafana Service Port: 3000
NodePort:             30300
```

---

# 25. Access Grafana

Open:

```text
http://192.168.1.10:30300
```

You can also try:

```text
http://192.168.1.11:30300
```

or:

```text
http://192.168.1.12:30300
```

depending on your network.

---

# 26. Grafana Login

If the Deployment explicitly configures:

```text
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=admin
```

then the initial credentials are:

```text
Username:
admin

Password:
admin
```

If your manifest does not define these environment variables, check the actual credentials configured in your Grafana deployment.

After logging in, change the default password.

> Do not use `admin/admin` for a publicly exposed production Grafana instance.

---

# 27. Persistent Grafana Storage

Grafana stores important information such as:

```text
Dashboards
Data source configuration
Users
Plugins
Grafana settings
```

Therefore, production-style deployments should use persistent storage instead of relying only on the container filesystem.

Check whether the Grafana deployment already uses a PVC:

```bash
kubectl get pvc -n monitoring
```

If a Grafana PVC exists, verify:

```text
STATUS = Bound
```

For example:

```text
NAME            STATUS   VOLUME
grafana-pvc     Bound    pvc-xxxxx
```

---

# 28. Persistent Prometheus Storage

Prometheus also benefits from persistent storage.

Without persistence, deleting/recreating the Prometheus Pod can remove its locally stored historical metrics.

Check:

```bash
kubectl get pvc -n monitoring
```

If Prometheus has a PVC:

```text
prometheus-pvc   Bound
```

then historical metrics can survive Pod recreation, subject to the storage configuration and retention policy.

---

# 29. Check the StorageClass

If a PVC remains:

```text
Pending
```

check the available StorageClasses.

## Run on: `desktop-control-plane`

```bash
kubectl get storageclass
```

Check PersistentVolumes:

```bash
kubectl get pv
```

Check PersistentVolumeClaims:

```bash
kubectl get pvc -n monitoring
```

A PVC should eventually become:

```text
Bound
```

---

# 30. Connect Grafana to Prometheus

Log into Grafana.

Navigate to:

```text
Connections
    ↓
Data sources
```

Click:

```text
Add data source
```

Select:

```text
Prometheus
```

For the URL, use the Kubernetes Service name.

If Grafana and Prometheus are both in:

```text
monitoring
```

use:

```text
http://prometheus:9090
```

This works because Kubernetes provides internal DNS.

The flow is:

```text
Grafana
   │
   │ http://prometheus:9090
   ▼
Prometheus Service
   │
   ▼
Prometheus Pod
```

Click:

```text
Save & Test
```

You should receive a successful connection message.

---

# 31. Why We Do Not Use the NodePort for Grafana → Prometheus

The browser uses:

```text
192.168.1.10:30300
```

to access Grafana.

But Grafana itself should use the internal Kubernetes Service:

```text
http://prometheus:9090
```

rather than:

```text
http://192.168.1.10:30090
```

Internal communication should use Kubernetes Services whenever possible.

Therefore:

```text
Browser
   ↓
192.168.1.10:30300
   ↓
Grafana
   ↓
http://prometheus:9090
   ↓
Prometheus
```

---

# 32. Verify the Prometheus Data Source

In Grafana:

```text
Connections
    ↓
Data sources
    ↓
Prometheus
```

Click:

```text
Save & Test
```

The connection should succeed.

If it fails, verify:

```bash
kubectl get svc prometheus -n monitoring
```

and:

```bash
kubectl get pods -n monitoring
```

---

# 33. Test Prometheus Before Building Dashboards

Before creating Grafana panels, verify that Prometheus actually has data.

Open Prometheus:

```text
http://192.168.1.10:30090
```

Go to:

```text
Graph
```

Try:

```promql
up
```

Click:

```text
Execute
```

You should receive time-series results.

This confirms Prometheus is collecting metrics.

---

# 34. Test Container Metrics

Run:

```promql
container_cpu_usage_seconds_total
```

If cAdvisor is being successfully scraped, this query should return container metrics.

You can also test:

```promql
container_memory_working_set_bytes
```

---

# 35. Test Kubernetes State Metrics

Run:

```promql
kube_deployment_status_replicas_available
```

If kube-state-metrics is correctly deployed and scraped, Kubernetes Deployment metrics should appear.

Test specifically:

```promql
kube_deployment_status_replicas_available{
  namespace="devops-todo",
  deployment="todo-app"
}
```

You should get the currently available PHP replicas.

---

# 36. Create the Grafana Dashboard

In Grafana:

```text
Dashboards
    ↓
New
    ↓
New Dashboard
```

Click:

```text
Add visualization
```

Select:

```text
Prometheus
```

We will create four main panels.

---

# 37. Panel 1 — PHP Todo CPU Usage

This panel shows CPU consumption of the PHP Todo Pods.

Use:

```promql
sum by (pod) (
  rate(container_cpu_usage_seconds_total{
    namespace="devops-todo",
    container!="",
    container!="POD"
  }[5m])
)
```

Select:

```text
Visualization:
Time series
```

Title:

```text
PHP Todo CPU Usage
```

Recommended unit:

```text
CPU
```

or:

```text
Percent (0.0-1.0)
```

depending on how you want the value displayed.

---

# 38. Understanding the CPU Query

The important metric is:

```text
container_cpu_usage_seconds_total
```

This is a cumulative counter.

Therefore we use:

```promql
rate(...[5m])
```

to calculate the CPU usage rate over the previous five minutes.

Then:

```promql
sum by (pod)
```

groups the result by PHP Pod.

Therefore, if you have:

```text
todo-app-aaa
todo-app-bbb
todo-app-ccc
```

Grafana can show CPU usage separately for each Pod.

---

# 39. Panel 2 — PHP Todo Memory Usage

Create another visualization.

Use:

```promql
sum by (pod) (
  container_memory_working_set_bytes{
    namespace="devops-todo",
    container!="",
    container!="POD"
  }
)
```

Visualization:

```text
Time series
```

Title:

```text
PHP Todo Memory Usage
```

Set the unit to:

```text
Data (IEC)
```

and:

```text
bytes
```

This allows Grafana to display values such as:

```text
50 MiB
100 MiB
250 MiB
```

instead of large raw byte numbers.

---

# 40. Understanding the Memory Query

The metric:

```text
container_memory_working_set_bytes
```

represents the current working memory set of the container.

We group it:

```promql
sum by (pod)
```

so each PHP Pod gets its own series.

This lets you identify whether one replica is consuming significantly more memory than the others.

---

# 41. Panel 3 — Current CPU Usage by Pod

Create another visualization.

Use:

```promql
sum by (pod) (
  rate(container_cpu_usage_seconds_total{
    namespace="devops-todo",
    container!="",
    container!="POD"
  }[5m])
)
```

Visualization:

```text
Bar chart
```

Title:

```text
Current CPU Usage by Pod
```

This gives you an easier comparison between the PHP replicas.

Example:

```text
todo-app-aaa   █████
todo-app-bbb   ███
todo-app-ccc   ███████
```

---

# 42. Panel 4 — Available PHP Replicas

This panel uses kube-state-metrics.

Use:

```promql
kube_deployment_status_replicas_available{
  namespace="devops-todo",
  deployment="todo-app"
}
```

Visualization:

```text
Time series
```

Title:

```text
Available PHP Replicas
```

The application is configured for:

```text
3 replicas
```

Therefore, under normal conditions, the metric should be:

```text
3
```

If it drops to:

```text
2
```

one replica is unavailable.

If it drops to:

```text
1
```

two replicas are unavailable.

If it reaches:

```text
0
```

the application has no available PHP replicas.

---

# 43. Add Desired Replica Count

A useful additional panel is to compare the desired and available replicas.

Desired replicas:

```promql
kube_deployment_spec_replicas{
  namespace="devops-todo",
  deployment="todo-app"
}
```

Available replicas:

```promql
kube_deployment_status_replicas_available{
  namespace="devops-todo",
  deployment="todo-app"
}
```

Use a:

```text
Time series
```

visualization.

Title:

```text
PHP Todo Desired vs Available Replicas
```

This makes Kubernetes availability easier to understand.

---

# 44. Add Pod Restart Monitoring

Another useful panel is Pod restarts.

Use:

```promql
sum by (pod) (
  kube_pod_container_status_restarts_total{
    namespace="devops-todo"
  }
)
```

Visualization:

```text
Time series
```

Title:

```text
PHP Todo Pod Restarts
```

A continuously increasing restart count can indicate application crashes, configuration problems, resource pressure, or other failures.

---

# 45. Add Node CPU Monitoring

To monitor the Kubernetes nodes themselves, you can create a node-level panel.

For example, depending on the available node metrics:

```promql
100 * (
  1 - avg by (instance) (
    rate(node_cpu_seconds_total{mode="idle"}[5m])
  )
)
```

Visualization:

```text
Time series
```

Title:

```text
Kubernetes Node CPU Usage
```

> This query requires node-exporter-style metrics. If your current Prometheus configuration does not scrape node-exporter, this panel will not return data. In that case, deploy node-exporter or use the node metrics already available in your configuration.

---

# 46. Add Node Memory Monitoring

Similarly, node-level memory monitoring can be provided using node-exporter metrics.

Example:

```promql
100 * (
  1 -
  (
    node_memory_MemAvailable_bytes /
    node_memory_MemTotal_bytes
  )
)
```

Visualization:

```text
Time series
```

Title:

```text
Kubernetes Node Memory Usage
```

Again, this requires the relevant node-exporter metrics to be available.

---

# 47. Recommended Dashboard Layout

A useful dashboard structure is:

```text
┌───────────────────────────────┬───────────────────────────────┐
│ PHP Todo CPU Usage            │ PHP Todo Memory Usage         │
│                               │                               │
│          Time Series          │          Time Series          │
├───────────────────────────────┼───────────────────────────────┤
│ Current CPU by Pod            │ Available PHP Replicas        │
│                               │                               │
│          Bar Chart            │          Time Series          │
├───────────────────────────────┼───────────────────────────────┤
│ Pod Restarts                  │ Desired vs Available          │
│                               │ Replicas                       │
│          Time Series          │          Time Series          │
└───────────────────────────────┴───────────────────────────────┘
```

This provides both:

```text
Resource monitoring
```

and:

```text
Application/Kubernetes health monitoring
```

---

# 48. Save the Dashboard

Once the panels are complete:

```text
Save dashboard
```

Use:

```text
PHP Todo Application Monitoring
```

as the dashboard name.

The dashboard should contain at least:

1. PHP Todo CPU Usage
2. PHP Todo Memory Usage
3. Current CPU Usage by Pod
4. Available PHP Replicas
5. Pod Restarts
6. Desired vs Available Replicas

---

# 49. Verify Monitoring From Kubernetes

## Run on: `desktop-control-plane`

Check all monitoring Pods:

```bash
kubectl get pods -n monitoring -o wide
```

Check Deployments:

```bash
kubectl get deployments -n monitoring
```

Check DaemonSets:

```bash
kubectl get daemonsets -n monitoring
```

Check Services:

```bash
kubectl get services -n monitoring
```

Check PVCs:

```bash
kubectl get pvc -n monitoring
```

A useful complete command is:

```bash
kubectl get all -n monitoring
```

---

# 50. Expected Monitoring Components

At the end, the namespace should contain components similar to:

```text
monitoring
│
├── prometheus
│   ├── Deployment
│   ├── Pod
│   └── Service
│
├── grafana
│   ├── Deployment
│   ├── Pod
│   └── Service
│
├── cadvisor
│   ├── DaemonSet
│   ├── Pods
│   └── Service
│
└── kube-state-metrics
    ├── Deployment
    ├── Pod
    └── Service
```

---

# 51. Useful Monitoring Commands

## Check everything

```bash
kubectl get all -n monitoring
```

## Check Pods

```bash
kubectl get pods -n monitoring -o wide
```

## Check Services

```bash
kubectl get svc -n monitoring
```

## Check Deployments

```bash
kubectl get deployments -n monitoring
```

## Check DaemonSets

```bash
kubectl get daemonsets -n monitoring
```

## Check ConfigMaps

```bash
kubectl get configmaps -n monitoring
```

## Check PVCs

```bash
kubectl get pvc -n monitoring
```

---

# 52. Troubleshooting — Prometheus Pod Not Running

Check:

```bash
kubectl get pods -n monitoring
```

If the Pod is:

```text
Pending
```

describe it:

```bash
kubectl describe pod <prometheus-pod> -n monitoring
```

Look at:

```text
Events
```

Possible causes include:

```text
Insufficient resources
PVC Pending
Scheduling problems
Configuration errors
```

---

# 53. Troubleshooting — Prometheus Has No Data

First check:

```bash
kubectl get pods -n monitoring
```

Then check Prometheus targets in:

```text
Status → Targets
```

If a target is:

```text
DOWN
```

check the target's error message.

Also check Prometheus logs:

```bash
kubectl logs -n monitoring deployment/prometheus
```

---

# 54. Troubleshooting — Grafana Cannot Connect to Prometheus

Check the Prometheus Service:

```bash
kubectl get svc prometheus -n monitoring
```

Check Prometheus:

```bash
kubectl get pods -n monitoring -l app=prometheus
```

From inside the Grafana Pod, test DNS/network connectivity.

First find the Grafana Pod:

```bash
kubectl get pods -n monitoring -l app=grafana
```

Then:

```bash
kubectl exec -it <grafana-pod> -n monitoring -- \
  curl http://prometheus:9090/-/healthy
```

A healthy Prometheus instance should return a successful response.

---

# 55. Troubleshooting — cAdvisor Pods Are Missing

Check:

```bash
kubectl get daemonset -n monitoring
```

Then:

```bash
kubectl get pods -n monitoring -o wide
```

If cAdvisor is running only on the workers:

```text
desktop-worker
desktop-worker2
```

but not:

```text
desktop-control-plane
```

check the control-plane taints:

```bash
kubectl describe node desktop-control-plane | grep Taints
```

A kubeadm control plane commonly has a taint similar to:

```text
node-role.kubernetes.io/control-plane:NoSchedule
```

If monitoring the control plane is required, the cAdvisor DaemonSet needs an appropriate toleration.

---

# 56. Troubleshooting — Grafana Page Does Not Open

Check the Service:

```bash
kubectl get svc grafana -n monitoring
```

Verify the NodePort:

```text
3000:30300/TCP
```

Check the Pod:

```bash
kubectl get pods -n monitoring -l app=grafana
```

Check logs:

```bash
kubectl logs -n monitoring deployment/grafana
```

Then try:

```text
http://192.168.1.10:30300
```

---

# 57. Troubleshooting — Prometheus Query Returns No Results

First determine which component should provide the metric.

For container metrics:

```text
cAdvisor
```

For Kubernetes object metrics:

```text
kube-state-metrics
```

For node-exporter metrics:

```text
node-exporter
```

For example:

```promql
container_memory_working_set_bytes
```

requires container metrics.

Whereas:

```promql
kube_deployment_status_replicas_available
```

requires kube-state-metrics.

---

# 58. Monitoring Validation Test

After the monitoring stack is running, perform a complete test.

## Step 1 — Verify PHP Pods

```bash
kubectl get pods -n devops-todo -o wide
```

You should see your PHP replicas.

---

## Step 2 — Open Grafana

```text
http://192.168.1.10:30300
```

---

## Step 3 — Open the Monitoring Dashboard

Open:

```text
PHP Todo Application Monitoring
```

---

## Step 4 — Verify CPU

The PHP Pods should appear in:

```text
PHP Todo CPU Usage
```

---

## Step 5 — Verify Memory

The PHP Pods should appear in:

```text
PHP Todo Memory Usage
```

---

## Step 6 — Verify Replicas

The dashboard should show:

```text
Available PHP Replicas = 3
```

when all three application replicas are healthy.

---

## Step 7 — Verify Restarts

Check:

```text
PHP Todo Pod Restarts
```

This should allow you to see whether Pods have restarted.

---

# 59. Test Kubernetes Monitoring During a Deployment

This is especially useful because the project already has Jenkins CI/CD.

Run a Jenkins deployment.

The flow is:

```text
GitHub
   ↓
Jenkins
   ↓
Docker Build
   ↓
Docker Hub
   ↓
kubectl set image
   ↓
Kubernetes Rolling Update
   ↓
New PHP Pods
   ↓
Prometheus
   ↓
Grafana
```

During the deployment, Grafana can show changes in:

```text
CPU
Memory
Pod availability
Pod restarts
Replica availability
```

This demonstrates that the monitoring system is actually observing the application rather than simply displaying static dashboards.

---

# 60. Final Monitoring Architecture

The completed DevOps environment now looks like:

```text
                         GitHub
                            │
                            ▼
                     ┌────────────┐
                     │   Jenkins  │
                     └─────┬──────┘
                           │
                    Docker Build
                           │
                           ▼
                      Docker Hub
                           │
                           ▼
                ┌─────────────────────┐
                │ Kubernetes Cluster  │
                │                     │
                │ desktop-control     │
                │ desktop-worker      │
                │ desktop-worker2     │
                └──────────┬──────────┘
                           │
                    Application Pods
                           │
                           ▼
                       cAdvisor
                           │
                           │ metrics
                           ▼
                      Prometheus
                           ▲
                           │
                  kube-state-metrics
                           │
                           │
                           ▼
                        Grafana
                           │
                           ▼
                    Monitoring Dashboard
```

---

# 61. Final Monitoring Checklist

Before considering monitoring complete, verify:

### Kubernetes

```bash
kubectl get nodes
```

All three nodes:

```text
Ready
```

### Monitoring Namespace

```bash
kubectl get namespace monitoring
```

### Prometheus

```bash
kubectl get deployment prometheus -n monitoring
```

### Grafana

```bash
kubectl get deployment grafana -n monitoring
```

### cAdvisor

```bash
kubectl get daemonset cadvisor -n monitoring
```

### kube-state-metrics

```bash
kubectl get deployment kube-state-metrics -n monitoring
```

### Monitoring Services

```bash
kubectl get svc -n monitoring
```

### Monitoring Pods

```bash
kubectl get pods -n monitoring -o wide
```

### Prometheus

Open:

```text
http://192.168.1.10:30090
```

### Grafana

Open:

```text
http://192.168.1.10:30300
```

### Prometheus Query Test

```promql
up
```

### Container Metrics Test

```promql
container_memory_working_set_bytes
```

### Kubernetes State Metrics Test

```promql
kube_deployment_status_replicas_available{
  namespace="devops-todo",
  deployment="todo-app"
}
```

---

# 62. What This Project Now Demonstrates

With Jenkins, Docker, Kubernetes, Prometheus, Grafana, cAdvisor, and kube-state-metrics, the project demonstrates a complete DevOps workflow:

```text
                    DEVELOPMENT
                         │
                         ▼
                       GitHub
                         │
                         ▼
                      Jenkins
                         │
              ┌──────────┴──────────┐
              │                     │
              ▼                     ▼
         Docker Build          CI/CD Pipeline
              │                     │
              ▼                     ▼
          Docker Hub           Kubernetes
                                    │
                         ┌──────────┴──────────┐
                         │                     │
                         ▼                     ▼
                      Worker 1              Worker 2
                         │                     │
                         └──────────┬──────────┘
                                    │
                              PHP Todo Pods
                                    │
                     ┌──────────────┴──────────────┐
                     │                             │
                     ▼                             ▼
                  cAdvisor               kube-state-metrics
                     │                             │
                     └──────────────┬──────────────┘
                                    ▼
                               Prometheus
                                    │
                                    ▼
                                 Grafana
                                    │
                                    ▼
                         Monitoring Dashboard
```

The project therefore demonstrates four major DevOps areas:

```text
1. Containerization
   Docker

2. Orchestration
   Kubernetes

3. Continuous Integration / Deployment
   Jenkins + GitHub + Docker Hub

4. Monitoring / Observability
   Prometheus + Grafana + cAdvisor + kube-state-metrics
```

This gives the PHP Todo project a much more complete **end-to-end DevOps architecture** suitable for demonstrating your Kubernetes, Jenkins, containerization, CI/CD, and monitoring skills.
