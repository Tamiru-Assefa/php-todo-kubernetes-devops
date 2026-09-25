# 🚀 Automate Jenkins with GitHub Webhooks

Until now, the CI/CD pipeline was started manually by clicking **Build Now** in Jenkins.

In this final step, we will remove that manual action.

After configuring a **GitHub Webhook**, every time code is pushed to the `devops-php-todo` repository, GitHub will notify Jenkins and Jenkins will automatically:

1. Checkout the latest code.
2. Build the Docker image.
3. Push the image to Docker Hub.
4. Deploy the new image to Kubernetes.
5. Wait for the Kubernetes rollout to complete.

The final workflow becomes:

```text
Developer
    │
    │ git push
    ▼
GitHub Repository
Tamiru-Assefa/devops-php-todo
    │
    │ HTTPS Webhook
    ▼
Jenkins
desktop-control-plane
    │
    ├── Checkout
    ├── Docker Build
    ├── Docker Push
    │
    ▼
Docker Hub
ybtamiru/devops-php-todo
    │
    │ kubectl
    ▼
Kubernetes Cluster
    │
    ├── desktop-control-plane
    ├── desktop-worker
    └── desktop-worker2
         │
         ▼
    devops-todo namespace
         │
         └── todo-app
```

---

# 1. Make Jenkins Accessible from GitHub

Our Jenkins is running on the `desktop-control-plane` machine and is available locally through:

```text
http://192.168.1.10:8081
```

This works inside our local network.

However, GitHub is outside our local network, so it cannot directly send a webhook to:

```text
http://192.168.1.10:8081/github-webhook/
```

Therefore, we need to expose Jenkins through a publicly accessible URL.

For our local project, we will use **ngrok**.

Run this on the machine where Jenkins is running:

```bash
ngrok http 8081
```

ngrok will provide a public HTTPS URL similar to:

```text
https://abcd-1234.ngrok-free.app
```

Your URL will be different.

Our Jenkins webhook URL will therefore be:

```text
https://abcd-1234.ngrok-free.app/github-webhook/
```

> **Note:** Keep the ngrok tunnel running while testing the webhook.

---

# 2. Verify Jenkins Through the Public URL

Open the generated HTTPS URL in your browser.

For example:

```text
https://abcd-1234.ngrok-free.app
```

It should open the same Jenkins instance that is available locally through:

```text
http://192.168.1.10:8081
```

The public URL simply forwards requests to our local Jenkins container.

---

# 3. Configure the Jenkins Job

Open Jenkins:

```text
http://192.168.1.10:8081
```

Open the existing job:

```text
devops-php-todo
```

Click:

```text
Configure
```

Find:

```text
Build Triggers
```

Enable:

```text
☑ GitHub hook trigger for GITScm polling
```

Then click:

```text
Save
```

Jenkins is now ready to receive GitHub webhook notifications.

---

# 4. Add the GitHub Webhook

Open our GitHub repository:

```text
https://github.com/Tamiru-Assefa/devops-php-todo
```

Go to:

```text
Settings
    ↓
Webhooks
    ↓
Add webhook
```

For **Payload URL**, enter our public Jenkins URL followed by:

```text
/github-webhook/
```

For example:

```text
https://abcd-1234.ngrok-free.app/github-webhook/
```

Set:

```text
Content type:
application/json
```

Select:

```text
Just the push event
```

Make sure:

```text
☑ Active
```

is enabled.

Then click:

```text
Add webhook
```

---

# 5. Test the GitHub Webhook

Now we will test the complete automation.

Make a small change to the project.

Then run:

```bash
git add .
```

```bash
git commit -m "Test GitHub webhook"
```

```bash
git push origin main
```

We should **not** click **Build Now** in Jenkins.

The process should now be:

```text
git push
    │
    ▼
GitHub
    │
    │ Webhook
    ▼
Jenkins
    │
    ▼
Pipeline starts automatically
```

---

# 6. Verify the Webhook

Go back to the GitHub repository:

```text
Settings
    ↓
Webhooks
```

Open the webhook we created.

Under:

```text
Recent Deliveries
```

we should see the latest push event.

A successful delivery means GitHub successfully sent the webhook request to Jenkins.

---

# 7. Verify Jenkins

Go back to Jenkins and open:

```text
devops-php-todo
```

A new build should now appear automatically.

For example:

```text
Build #2
Build #3
Build #4
```

The exact build number depends on the previous builds.

Open the new build and select:

```text
Console Output
```

The Jenkins pipeline should perform:

```text
Checkout
    ↓
Docker Build
    ↓
Docker Hub Login
    ↓
Docker Push
    ↓
Kubernetes Deployment Update
    ↓
Kubernetes Rollout
    ↓
Docker Hub Logout
```

If everything succeeds, Jenkins should show:

```text
Finished: SUCCESS
```

---

# 8. Verify Docker Hub

After Jenkins finishes the build, open the Docker Hub repository:

```text
ybtamiru/devops-php-todo
```

A new image should have been pushed using the Jenkins build number.

For example:

```text
ybtamiru/devops-php-todo:2
```

The exact number depends on the Jenkins build number.

---

# 9. Verify Kubernetes Deployment

After Jenkins completes the deployment, check the Kubernetes deployment from the `desktop-control-plane`:

```bash
kubectl get deployment todo-app -n devops-todo
```

Then check the Pods:

```bash
kubectl get pods -n devops-todo
```

We should still have our 3 PHP application replicas running.

To verify the image currently used by the deployment:

```bash
kubectl get deployment todo-app -n devops-todo \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

It should show the new image version pushed by Jenkins.

For example:

```text
ybtamiru/devops-php-todo:2
```

Finally, verify that Kubernetes successfully completed the rollout:

```bash
kubectl rollout status deployment/todo-app -n devops-todo
```

Expected output:

```text
deployment "todo-app" successfully rolled out
```

---

# 10. Final CI/CD Workflow

Our `devops-php-todo` project is now fully automated.

Previously:

```text
Developer
    ↓
git push
    ↓
Jenkins
    ↓
Build Now
    ↓
Docker Hub
    ↓
Kubernetes
```

Now:

```text
Developer
    ↓
git push
    ↓
GitHub
    ↓
Webhook
    ↓
Jenkins
    ↓
Docker Build
    ↓
Docker Hub
    ↓
Kubernetes
    ↓
Application Updated
```

From now on, when we push changes to the `main` branch:

```bash
git push origin main
```

GitHub will notify Jenkins automatically, and Jenkins will build and deploy the updated application.

This completes the automated **CI/CD pipeline** for the `devops-php-todo` project.
