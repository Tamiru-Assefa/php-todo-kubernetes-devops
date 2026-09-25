#  DevOps PHP Todo — Kubernetes & CI/CD Project

This project demonstrates the deployment and automation of a **PHP Todo web application** using **Docker, Kubernetes, Jenkins, Docker Hub, and GitHub Webhooks**.

The project has been developed step by step, starting from the Kubernetes environment setup, followed by application deployment,Monitoring(Grafana, Prometheus, cAdvisor, kube-state-metrics) and finally automated CI/CD using Jenkins.

The complete project repository is available here:

```text
https://github.com/Tamiru-Assefa/devops-php-todo/
```

All the necessary files required for the Kubernetes deployment and CI/CD setup are included in the repository.

---

#  Project Structure

The main documentation and configuration files for this project are organized as follows:

```text
devops-php-todo/
│
├── Architecture/
│
├── Kubernetes/
│
├── ScreenShots/
│
├── Jenkins Automation.md
│
├── Jenkinsfile
│
├── Kubernetes-Deployment-and-CICD.md
│
├── PRE-ENVIRONMENT-SETUP.md
│
└── README.md
```

###  Architecture/

Contains the architecture diagrams used to explain the project infrastructure and deployment design.

###  Kubernetes/

Contains the Kubernetes configuration files required to deploy the PHP application and MySQL database to the Kubernetes cluster.

All the necessary Kubernetes resources for this project are provided in this directory.

###  ScreenShots/

Contains screenshots showing the results of the commands, configurations, deployments, Jenkins setup, and other steps described throughout the documentation.

###  PRE-ENVIRONMENT-SETUP.md

Contains the instructions for preparing the environment before deploying the application.

This includes the required infrastructure, Kubernetes cluster setup, nodes, tools, and other environment requirements.

###  Kubernetes-Deployment-and-CICD.md

Contains the instructions for deploying the PHP Todo application and MySQL database to Kubernetes.

It covers the Kubernetes resources, application deployment, services, persistent storage, Jenkins setup, Docker image creation, and the initial CI/CD configuration.

###  Jenkins Automation.md

Contains the final step of the project: automating Jenkins using a GitHub Webhook.

This allows a `git push` to automatically trigger Jenkins, build and push the Docker image, and update the application running on Kubernetes.

###  Jenkinsfile

Contains the Jenkins Pipeline configuration used to automate the CI/CD process.

---

#  Recommended Documentation Order

To complete the project correctly, follow the documentation in this order:

```text
1. PRE-ENVIRONMENT-SETUP.md
              ↓
2. Kubernetes-Deployment-and-CICD.md
              ↓
3. Jenkins Automation.md
```

### 1️⃣ PRE-ENVIRONMENT-SETUP.md

Start here.

Prepare the required environment and Kubernetes cluster before deploying the application.

### 2️⃣ Kubernetes-Deployment-and-CICD.md

After the environment is ready, continue with the Kubernetes deployment.

This step deploys the application and database and prepares Jenkins for the CI/CD process.

### 3️⃣ Jenkins Automation.md

Finally, configure the GitHub Webhook.

This completes the automation so that pushing new code to GitHub automatically triggers the Jenkins pipeline and updates the Kubernetes application.

---

#  Complete Project Flow

The complete project workflow is:

                         ┌───────────────┐
                         │   Developer   │
                         └───────┬───────┘
                                 │
                              git push
                                 │
                                 ▼
                         ┌───────────────┐
                         │    GitHub     │
                         └───────┬───────┘
                                 │
                              Webhook
                                 │
                                 ▼
                         ┌───────────────┐
                         │    Jenkins    │
                         │     CI/CD     │
                         └───────┬───────┘
                                 │
                         Build Docker Image
                                 │
                                 ▼
                         ┌───────────────┐
                         │  Docker Hub   │
                         └───────┬───────┘
                                 │
                           Pull Image
                                 │
                                 ▼
              ┌─────────────────────────────────┐
              │       Kubernetes Cluster        │
              │                                 │
              │  ┌───────────────────────────┐  │
              │  │       PHP Todo App        │  │
              │  └─────────────┬─────────────┘  │
              │                │                │
              │                ▼                │
              │        ┌──────────────┐         │
              │        │    MySQL     │         │
              │        └──────────────┘         │
              │                                 │
              │        Monitoring               │
              │                                 │
              │   cAdvisor ────────────┐        │
              │   kube-state-metrics ──┤        │
              │   Node Exporter ───────┤        │
              │                         ▼        │
              │                    Prometheus    │
              │                         │        │
              │                         ▼        │
              │                      Grafana     │
              │                                 │
              └─────────────────────────────────┘

All required instructions, commands, configurations, and screenshots are provided in the documentation files above.

**Start with `PRE-ENVIRONMENT-SETUP.md` and follow the documents in order.**
